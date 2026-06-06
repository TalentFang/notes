
## `EventDispatcher` 类详解

### 一、定位与设计哲学

```
┌─────────────────────────────────────────────────────────────────┐
│                    brpc IO 架构中的 EventDispatcher              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   传统 IO 线程模型                    brpc EventDispatcher 模型   │
│   ──────────────────                  ────────────────────────    │
│                                                                  │
│   ┌─────────────┐                    ┌─────────────┐              │
│   │  IO线程1     │ ←── 读+处理          │  EventDispatcher │ ←── 只分发事件    │
│   │  - epoll_wait│                    │  - epoll_wait   │              │
│   │  - 读取数据  │                    │  - 启动 bthread  │ ←── 处理交给 bthread │
│   │  - 业务处理  │ ←── 阻塞点           │                 │              │
│   └─────────────┘                    └─────────────┘              │
│   ┌─────────────┐                          ↓                     │
│   │  IO线程2     │                    ┌─────────────┐              │
│   │  ...        │                    │  bthread 池   │ ←── 动态处理       │
│   └─────────────┘                    │  - 读取数据    │              │
│                                      │  - 业务处理    │              │
│   问题：一个线程忙会阻塞其他fd          │  - 可窃取执行  │ ←── 负载均衡        │
│                                      └─────────────┘              │
│                                                                  │
│   核心设计：分离"事件分发"和"IO处理"                               │
│   • EventDispatcher 只做轻量级事件分发（O(1)）                    │
│   • 实际 IO 和处理由 bthread 执行（可扩展）                        │
│   • 避免一个 fd 慢影响其他 fd（长尾延迟）                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### 二、核心类结构

```
┌─────────────────────────────────────────────────────────────────┐
│                    EventDispatcher 类层次                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   IOEventData (VersionedRefWithId<IOEventData>)                   │
│   ├── 封装 fd 和回调函数                                          │
│   ├── 支持版本控制（SetFailed/Revive）                            │
│   └── 存储在 epoll/kqueue 的 data 字段中                          │
│                                                                  │
│   EventDispatcher                                                 │
│   ├── 管理 epoll/kqueue 实例                                      │
│   ├── 分发事件到 IOEventData                                      │
│   └── 每个线程标签 + fd 哈希 一个实例                              │
│                                                                  │
│   IOEvent<T> (模板包装)                                            │
│   ├── 简化 IOEventData 的使用                                     │
│   ├── 模板参数 T 提供静态回调函数                                 │
│   └── RAII 管理生命周期                                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### 三、完整源代码解析

#### 1. `IOEventData`：事件数据封装

```cpp
// 继承 VersionedRefWithId，支持版本控制和延迟回收
class IOEventData : public VersionedRefWithId<IOEventData> {
public:
    explicit IOEventData(Forbidden f)
        : VersionedRefWithId<IOEventData>(f)
        , _options{ NULL, NULL, NULL } {}

    // 调用用户注册的回调
    int CallInputEventCallback(uint32_t events, const bthread_attr_t& thread_attr) {
        return _options.input_cb(_options.user_data, events, thread_attr);
    }

    int CallOutputEventCallback(uint32_t events, const bthread_attr_t& thread_attr) {
        return _options.output_cb(_options.user_data, events, thread_attr);
    }

private:
    friend class VersionedRefWithId<IOEventData>;

    // 必需钩子：创建时初始化
    int OnCreated(const IOEventDataOptions& options) {
        if (!options.input_cb || !options.output_cb) {
            return -1;
        }
        _options = options;
        return 0;
    }

    // 必需钩子：回收前清理
    void BeforeRecycled() {
        _options = { NULL, NULL, NULL };
    }

    IOEventDataOptions _options;  // 回调配置
};
```

**关键设计**：
- 使用 `VersionedRefWithId` 管理生命周期，支持 `Create`/`SetFailed`/`Address`
- `IOEventDataId` 存储在 epoll/kqueue 事件数据中，避免直接存指针（安全）
- 回调函数由用户注册，支持各种 IO 类型（socket、pipe、timerfd 等）

---

#### 2. `EventDispatcher`：事件分发器

```cpp
class EventDispatcher {
public:
    EventDispatcher();
    virtual ~EventDispatcher();

    // 在 bthread 中启动分发器
    int Start(const bthread_attr_t* thread_attr);

    // 添加 fd 监听（边缘触发）
    int AddConsumer(IOEventDataId event_data_id, int fd);
    
    // 注册/注销可写事件
    int RegisterEvent(IOEventDataId event_data_id, int fd, bool pollin);
    int UnregisterEvent(IOEventDataId event_data_id, int fd, bool pollin);

    bool Running() const;
    void Stop();
    void Join();

private:
    // 平台相关的 epoll/kqueue fd
    int _event_dispatcher_fd;
    
    // 停止标志
    volatile bool _stop;
    
    // 运行 EventDispatcher 的 bthread ID
    bthread_t _tid;
    
    // 用户回调的 bthread 属性
    bthread_attr_t _thread_attr;
    
    // 用于唤醒 epoll_wait 的管道（退出机制）
    int _wakeup_fds[2];
};
```

---

### 四、核心机制详解

#### 1. 全局分发器管理

```cpp
// 配置参数
DEFINE_int32(event_dispatcher_num, 1, "Number of event dispatcher");
DEFINE_int32(task_group_ntags, 1, "Number of task groups");

// 全局分发器数组（二维：tag × num）
static EventDispatcher* g_edisp = NULL;

// 延迟初始化（线程安全）
void InitializeGlobalDispatchers() {
    g_edisp = new EventDispatcher[FLAGS_task_group_ntags * FLAGS_event_dispatcher_num];
    
    for (int i = 0; i < FLAGS_task_group_ntags; ++i) {
        for (int j = 0; j < FLAGS_event_dispatcher_num; ++j) {
            bthread_attr_t attr = FLAGS_usercode_in_pthread ? 
                BTHREAD_ATTR_PTHREAD : BTHREAD_ATTR_NORMAL;
            attr.tag = (BTHREAD_TAG_DEFAULT + i) % FLAGS_task_group_ntags;
            
            g_edisp[i * FLAGS_event_dispatcher_num + j].Start(&attr);
        }
    }
    
    atexit(StopAndJoinGlobalDispatchers);  // 注册退出清理
}

// 获取分发器（基于 fd 哈希和 bthread 标签）
EventDispatcher& GetGlobalEventDispatcher(int fd, bthread_tag_t tag) {
    pthread_once(&g_edisp_once, InitializeGlobalDispatchers);  // 只执行一次
    
    if (FLAGS_task_group_ntags == 1 && FLAGS_event_dispatcher_num == 1) {
        return g_edisp[0];  // 快速路径
    }
    
    // 哈希分配：同一 fd 总是映射到同一分发器
    int index = butil::fmix32(fd) % FLAGS_event_dispatcher_num;
    return g_edisp[tag * FLAGS_event_dispatcher_num + index];
}
```

**设计要点**：
- 支持多 `task_group_ntags`（bthread 任务组），实现 NUMA 亲和性
- 同一 `fd` 通过 `fmix32` 哈希总是映射到同一 `EventDispatcher`，避免竞态
- 默认单线程，可通过 `FLAGS_event_dispatcher_num` 扩展

---

#### 2. 事件循环（平台抽象）

```cpp
// event_dispatcher_epoll.cpp (Linux)
void EventDispatcher::Run() {
    while (!_stop) {
        epoll_event e[32];
        
        // 等待事件（超时 -1 表示阻塞，0 表示非阻塞轮询）
        int n = epoll_wait(_event_dispatcher_fd, e, ARRAY_SIZE(e), -1);
        
        if (_stop) break;  // 检查停止标志
        
        // 第一遍：处理读事件（启动 bthread）
        for (int i = 0; i < n; ++i) {
            if (e[i].events & (EPOLLIN | EPOLLERR | EPOLLHUP | EPOLLRDHUP)) {
                // 关键：通过 IOEventDataId 获取回调，而非直接调用
                Socket::StartInputEvent(e[i].data.u64, e[i].events, _thread_attr);
            }
        }
        
        // 第二遍：处理写事件
        for (int i = 0; i < n; ++i) {
            if (e[i].events & (EPOLLOUT | EPOLLERR | EPOLLHUP)) {
                Socket::HandleEpollOut(e[i].data.u64);
            }
        }
    }
}

// event_dispatcher_kqueue.cpp (macOS/FreeBSD)
void EventDispatcher::Run() {
    while (!_stop) {
        struct kevent e[32];
        int n = kevent(_event_dispatcher_fd, NULL, 0, e, ARRAY_SIZE(e), NULL);
        
        // 类似处理...
        for (int i = 0; i < n; ++i) {
            if (e[i].filter == EVFILT_READ || (e[i].flags & EV_ERROR)) {
                Socket::StartInputEvent((SocketId)e[i].udata, ...);
            }
        }
        // ...
    }
}
```

**关键设计**：
- 边缘触发（Edge-Triggered）：只在状态变化时通知，减少 epoll 调用
- 分离读写处理：读事件优先，避免写饥饿
- 不直接处理 IO，而是启动 bthread 异步处理

---

#### 3. 边缘触发（EPOLLET）详解

```cpp
int EventDispatcher::AddConsumer(IOEventDataId event_data_id, int fd) {
#if defined(OS_LINUX)
    epoll_event evt;
    evt.data.u64 = event_data_id;  // 存储 ID 而非指针（安全）
    evt.events = EPOLLIN | EPOLLET;  // 边缘触发 + 可读
    
#ifdef BRPC_SOCKET_HAS_EOF
    evt.events |= EPOLLRDHUP;  // 对端关闭检测
#endif

    return epoll_ctl(_epfd, EPOLL_CTL_ADD, fd, &evt);
    
#elif defined(OS_MACOSX)
    struct kevent evt;
    EV_SET(&evt, fd, EVFILT_READ, EV_ADD | EV_ENABLE | EV_CLEAR,  // EV_CLEAR = 边缘触发
           0, 0, (void*)event_data_id);
    return kevent(_epfd, &evt, 1, NULL, 0, NULL);
#endif
}
```

**边缘触发 vs 水平触发**：

| 特性 | 水平触发（Level-Triggered） | 边缘触发（Edge-Triggered）|
|:---|:---|:---|
| 通知时机 | 只要 fd 可读就一直通知 | 只有从不可读变为可读时通知一次 |
| 代码复杂度 | 简单（可以只读部分数据） | 复杂（必须读到 EAGAIN）|
| epoll 开销 | 高（重复通知） | 低（单次通知）|
| 多线程安全 | 需要额外同步 | 天然避免惊群（thundering herd）|
| brpc 选择 | ❌ 不使用 | ✅ 使用（性能更优）|

---

#### 4. 事件分发到 bthread

```cpp
// EventDispatcher 模板方法：调用用户回调
template <bool IsInputEvent>
static int OnEvent(IOEventDataId event_data_id, uint32_t events,
                   const bthread_attr_t& thread_attr) {
    // 通过 ID 安全获取 IOEventData（可能已失效，Address 会检查版本）
    EventDataUniquePtr data;
    if (IOEventData::Address(event_data_id, &data) != 0) {
        return -1;  // 已失效或已回收
    }
    
    // 调用用户回调（在 bthread 中执行）
    return IsInputEvent ?
        data->CallInputEventCallback(events, thread_attr) :
        data->CallOutputEventCallback(events, thread_attr);
}

// 实际调用路径（以 Socket 为例）：
// 1. epoll_wait 返回
// 2. EventDispatcher::Run() 调用 Socket::StartInputEvent(id, ...)
// 3. StartInputEvent 内部启动 bthread 执行 OnInputEvent
// 4. OnInputEvent 调用实际的 Socket 读取和处理逻辑
```

**关键优势**：
- `EventDispatcher` 不阻塞，立即返回继续 `epoll_wait`
- 实际 IO 在 bthread 中执行，可并行处理多个 fd
- bthread 可窃取（work stealing），自动负载均衡

---

### 五、`IOEvent` 模板：简化使用

```cpp
template <typename T>
class IOEvent {
public:
    int Init(void* user_data) {
        IOEventDataOptions options{
            OnInputEvent,   // 静态回调
            OnOutputEvent,  // 静态回调
            user_data
        };
        return IOEventData::Create(&_event_data_id, options);
    }

    int AddConsumer(int fd) {
        return GetGlobalEventDispatcher(fd, _bthread_tag)
            .AddConsumer(_event_data_id, fd);
    }

    // ... 其他方法

private:
    // 模板派发到 T 的静态方法
    static int OnInputEvent(void* user_data, uint32_t events,
                            const bthread_attr_t& thread_attr) {
        return T::OnInputEvent(user_data, events, thread_attr);
    }
    
    static int OnOutputEvent(void* user_data, uint32_t events,
                             const bthread_attr_t& thread_attr) {
        return T::OnOutputEvent(user_data, events, thread_attr);
    }

    bool _init;
    IOEventDataId _event_data_id;
    bthread_tag_t _bthread_tag;
};
```

**使用示例**：
```cpp
class MyHandler {
public:
    static int OnInputEvent(void* user_data, uint32_t events,
                            const bthread_attr_t&) {
        // 读取数据...
        return 0;
    }
    
    static int OnOutputEvent(void* user_data, uint32_t events,
                             const bthread_attr_t&) {
        // 写入数据...
        return 0;
    }
};

// 使用
IOEvent<MyHandler> io_event;
io_event.Init(this);
io_event.AddConsumer(fd);  // 开始监听
```

---

### 六、设计亮点总结

| 设计 | 说明 | 收益 |
|:---|:---|:---|
| **分离分发与处理** | `EventDispatcher` 只分发，不处理 IO | 避免一个 fd 慢阻塞其他 fd |
| **边缘触发** | `EPOLLET` / `EV_CLEAR` | 减少 epoll 调用次数，避免重复通知 |
| **ID 而非指针** | epoll data 存 `IOEventDataId` | 安全，支持版本控制和延迟回收 |
| **bthread 执行** | 事件启动 bthread 处理 | 可扩展，自动负载均衡（work stealing）|
| **多维度扩展** | `task_group_ntags` × `event_dispatcher_num` | 支持 NUMA 亲和性和并行扩展 |
| **平台抽象** | epoll (Linux) / kqueue (macOS) | 跨平台，统一接口 |

---

### 七、一句话总结

> **`EventDispatcher` 是 brpc 的"事件路由器"**：它使用**边缘触发**的 epoll/kqueue 高效捕获 IO 事件，但**不直接处理数据**，而是通过 `IOEventDataId` 安全地启动 **bthread** 执行实际 IO 和处理，实现**高并发、低延迟、无长尾**的网络 IO 架构。