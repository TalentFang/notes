
## `ResourcePool` 详解

### 一、定位与设计哲学

```
┌─────────────────────────────────────────────────────────────────┐
│                    brpc 内存架构中的 ResourcePool                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   传统内存分配                      ResourcePool 设计              │
│   ────────────                      ────────────────               │
│                                                                  │
│   ┌─────────────┐                    ┌─────────────────────┐       │
│   │  malloc/new  │ ←── 慢，碎片化      │  预分配 Block 数组    │       │
│   │  - 系统调用   │                    │  - O(1) 分配         │       │
│   │  - 锁竞争    │                    │  - 无锁（TLS）        │       │
│   │  - 缓存不友好 │                    │  - 批量分配/归还      │       │
│   └─────────────┘                    └─────────────────────┘       │
│                                                                  │
│   核心设计：分离"频繁分配/归还"与"实际内存管理"                      │
│   • 线程本地缓存（LocalPool）：90%+ 操作无锁                        │
│   • 全局 Block 数组：统一内存管理，支持通过 ID 安全访问               │
│   • 延迟归还：批量处理，减少全局竞争                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### 二、核心数据结构

```
┌─────────────────────────────────────────────────────────────────┐
│                    ResourcePool 层级结构                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ResourceId<T> (64位 ID)                                          │
│  ├── 全局唯一标识一个对象                                          │
│  └── value = block_index * BLOCK_NITEM + offset                   │
│                                                                  │
│  Block (缓存行对齐)                                                │
│  ├── items[sizeof(T) * BLOCK_NITEM]  // 实际对象存储               │
│  └── nitem                           // 已使用数量                 │
│                                                                  │
│  BlockGroup (管理最多 65536 个 Block)                               │
│  ├── blocks[RP_GROUP_NBLOCK]         // Block 指针数组             │
│  └── nblock                          // 已分配 Block 数            │
│                                                                  │
│  LocalPool (线程本地，无锁访问)                                     │
│  ├── _cur_block                      // 当前分配的 Block            │
│  ├── _cur_free                       // 本地空闲 ID 缓存            │
│  └── _pool                           // 指向全局 ResourcePool      │
│                                                                  │
│  ResourcePool (全局单例)                                            │
│  ├── _block_groups[65536]            // BlockGroup 数组            │
│  ├── _free_chunks                    // 全局空闲列表（加锁访问）      │
│  └── _local_pool (TLS)               // 线程本地 LocalPool         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### 三、完整源代码解析

#### 1. `ResourceId`：全局唯一标识符

```cpp
template <typename T>
struct ResourceId {
    uint64_t value;  // 64位全局唯一 ID

    // 隐式转换为 uint64_t，便于使用
    operator uint64_t() const { return value; }

    // 类型转换：不同 ResourcePool 类型间转换 ID
    template <typename T2>
    ResourceId<T2> cast() const {
        ResourceId<T2> id = { value };
        return id;
    }
};
```

**ID 编码规则**：
```
value = block_index * BLOCK_NITEM + offset

例如：BLOCK_NITEM = 256
- ID 0 ~ 255   → Block 0, offset 0~255
- ID 256 ~ 511 → Block 1, offset 0~255
- ...
```

---

#### 2. `Block` 和 `BlockGroup`：内存管理单元

```cpp
// 缓存行对齐，避免伪共享
struct BAIDU_CACHELINE_ALIGNMENT Block {
    char items[sizeof(T) * BLOCK_NITEM];  // 预分配内存，未构造对象
    size_t nitem;                          // 已构造对象数量

    Block() : nitem(0) {}
};

// 管理 65536 个 Block
struct BlockGroup {
    butil::atomic<size_t> nblock;  // 已分配 Block 数（原子）
    butil::atomic<Block*> blocks[RP_GROUP_NBLOCK];  // Block 指针数组

    BlockGroup() : nblock(0) {
        // 初始化为 NULL，address_resource 看到 NULL 知道未分配
        memset(blocks, 0, sizeof(butil::atomic<Block*>) * RP_GROUP_NBLOCK);
    }
};
```

---

#### 3. `LocalPool`：线程本地分配器（核心优化）

```cpp
class BAIDU_THREAD_LOCAL LocalPool {
public:
    explicit LocalPool(ResourcePool* pool)
        : _pool(pool)
        , _cur_block(NULL)
        , _cur_block_index(0) {
        _cur_free.nfree = 0;  // 本地空闲列表为空
    }

    ~LocalPool() {
        // 析构时，将本地空闲列表归还全局
        if (_cur_free.nfree) {
            _pool->push_free_chunk(_cur_free);
        }
        _pool->clear_from_destructor_of_local_pool();
    }

    // 获取资源（核心函数）
    T* get(ResourceId<T>* id) {
        // 1. 优先从本地空闲列表获取（无锁，最快）
        if (_cur_free.nfree) {
            const ResourceId<T> free_id = _cur_free.ids[--_cur_free.nfree];
            *id = free_id;
            return unsafe_address_resource(free_id);  // O(1) 地址计算
        }

        // 2. 本地无空闲，从全局获取一批（加锁，但批量）
        if (_pool->pop_free_chunk(_cur_free)) {
            --_cur_free.nfree;
            const ResourceId<T> free_id = _cur_free.ids[_cur_free.nfree];
            *id = free_id;
            return unsafe_address_resource(free_id);
        }

        // 3. 全局也无空闲，从本地 Block 新建对象
        if (_cur_block && _cur_block->nitem < BLOCK_NITEM) {
            id->value = _cur_block_index * BLOCK_NITEM + _cur_block->nitem;
            //  placement new：在预分配内存上构造对象
            T* p = new ((T*)_cur_block->items + _cur_block->nitem) T;
            ++_cur_block->nitem;
            return p;
        }

        // 4. 本地 Block 满，申请新 Block
        _cur_block = add_block(&_cur_block_index);
        if (_cur_block != NULL) {
            // ... 同上，在新 Block 上构造
        }
        return NULL;
    }

    // 归还资源（核心函数）
    int return_resource(ResourceId<T> id) {
        // 优先归还本地（无锁）
        if (_cur_free.nfree < ResourcePool::free_chunk_nitem()) {
            _cur_free.ids[_cur_free.nfree++] = id;
            return 0;
        }

        // 本地满，批量归还全局（加锁，但减少频率）
        if (_pool->push_free_chunk(_cur_free)) {
            _cur_free.nfree = 1;
            _cur_free.ids[0] = id;
            return 0;
        }
        return -1;
    }

private:
    ResourcePool* _pool;
    Block* _cur_block;
    size_t _cur_block_index;
    FreeChunk _cur_free;  // 本地空闲列表（小数组）
};
```

**关键设计**：
- **90%+ 操作无锁**：`get`/`return` 优先操作本地 `_cur_free`
- **批量处理**：本地满时才与全局交互，摊平锁开销
- **Thread Local Storage**：每个线程独立 `LocalPool`，无竞争

---

#### 4. `ResourcePool`：全局协调器

```cpp
template <typename T>
class BAIDU_CACHELINE_ALIGNMENT ResourcePool {
public:
    // 单例模式（双重检查锁定）
    static inline ResourcePool* singleton() {
        ResourcePool* p = _singleton.load(butil::memory_order_consume);
        if (p) return p;
        
        pthread_mutex_lock(&_singleton_mutex);
        p = _singleton.load(butil::memory_order_consume);
        if (!p) {
            p = new ResourcePool();
            _singleton.store(p, butil::memory_order_release);
        }
        pthread_mutex_unlock(&_singleton_mutex);
        return p;
    }

    // 对外接口：获取资源
    inline T* get_resource(ResourceId<T>* id) {
        LocalPool* lp = get_or_new_local_pool();  // 获取/创建线程本地池
        if (__builtin_expect(lp != NULL, 1)) {
            return lp->get(id);
        }
        return NULL;
    }

    // 对外接口：归还资源
    inline int return_resource(ResourceId<T> id) {
        LocalPool* lp = get_or_new_local_pool();
        if (__builtin_expect(lp != NULL, 1)) {
            return lp->return_resource(id);
        }
        return -1;
    }

    // 通过 ID 获取地址（O(1)）
    static inline T* address_resource(ResourceId<T> id) {
        const size_t block_index = id.value / BLOCK_NITEM;
        const size_t group_index = (block_index >> RP_GROUP_NBLOCK_NBIT);
        
        // 分层查找：Group → Block → Item
        if (__builtin_expect(group_index < RP_MAX_BLOCK_NGROUP, 1)) {
            BlockGroup* bg = _block_groups[group_index].load(butil::memory_order_consume);
            if (__builtin_expect(bg != NULL, 1)) {
                Block* b = bg->blocks[block_index & (RP_GROUP_NBLOCK - 1)]
                           .load(butil::memory_order_consume);
                if (__builtin_expect(b != NULL, 1)) {
                    const size_t offset = id.value - block_index * BLOCK_NITEM;
                    if (__builtin_expect(offset < b->nitem, 1)) {
                        return (T*)b->items + offset;
                    }
                }
            }
        }
        return NULL;  // ID 无效或已回收
    }

private:
    // 获取或创建线程本地池
    inline LocalPool* get_or_new_local_pool() {
        LocalPool* lp = _local_pool;  // TLS 变量
        if (lp != NULL) return lp;
        
        lp = new(std::nothrow) LocalPool(this);
        if (NULL == lp) return NULL;
        
        BAIDU_SCOPED_LOCK(_change_thread_mutex);
        _local_pool = lp;
        butil::thread_atexit(LocalPool::delete_local_pool, lp);  // 线程退出时清理
        _nlocal.fetch_add(1, butil::memory_order_relaxed);
        return lp;
    }

    // 全局空闲列表操作（加锁）
    bool pop_free_chunk(FreeChunk& c) {
        if (_free_chunks.empty()) return false;
        pthread_mutex_lock(&_free_chunks_mutex);
        // ... 从 _free_chunks 尾部取一个
        pthread_mutex_unlock(&_free_chunks_mutex);
        return true;
    }

    bool push_free_chunk(const FreeChunk& c) {
        // ... 拷贝到 DynamicFreeChunk，加入 _free_chunks
        pthread_mutex_lock(&_free_chunks_mutex);
        _free_chunks.push_back(p);
        pthread_mutex_unlock(&_free_chunks_mutex);
        return true;
    }

    // 申请新 Block（全局操作，加锁）
    static Block* add_block(size_t* index) {
        Block* new_block = new(std::nothrow) Block;
        // ... CAS 方式加入 BlockGroup，失败时重试
    }

    // 成员变量
    static BAIDU_THREAD_LOCAL LocalPool* _local_pool;  // TLS
    static butil::static_atomic<ResourcePool*> _singleton;  // 单例
    static pthread_mutex_t _singleton_mutex;
    static butil::static_atomic<BlockGroup*> _block_groups[RP_MAX_BLOCK_NGROUP];
    
    std::vector<DynamicFreeChunk*> _free_chunks;  // 全局空闲列表
    pthread_mutex_t _free_chunks_mutex;
};
```

---

### 四、核心操作流程

#### 1. 获取资源 `get_resource`

```
┌─────────────────────────────────────────────────────────────────┐
│                    get_resource 流程                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 获取 LocalPool（TLS，无锁）                                    │
│     LocalPool* lp = _local_pool;                                 │
│                                                                  │
│  2. 本地空闲列表有？（90%+ 概率，无锁）                              │
│     if (_cur_free.nfree > 0)                                     │
│         return _cur_free.ids[--nfree]  // O(1)                   │
│                                                                  │
│  3. 本地无，全局空闲列表有？（批量获取，加锁）                        │
│     if (pop_free_chunk(_cur_free))                               │
│         return _cur_free.ids[--nfree]                            │
│                                                                  │
│  4. 全局也无，本地 Block 有空间？（placement new）                   │
│     if (_cur_block->nitem < BLOCK_NITEM)                       │
│         new (items + nitem) T;  // 原地构造                       │
│         return &items[nitem++]                                   │
│                                                                  │
│  5. 本地 Block 满，申请新 Block（全局操作，CAS）                     │
│     _cur_block = add_block(&index);                              │
│     goto 4                                                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### 2. 归还资源 `return_resource`

```
┌─────────────────────────────────────────────────────────────────┐
│                    return_resource 流程                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 本地空闲列表未满？（90%+ 概率，无锁）                            │
│     if (_cur_free.nfree < FREE_CHUNK_NITEM)                    │
│         _cur_free.ids[nfree++] = id;  // O(1)                    │
│         return 0;                                                │
│                                                                  │
│  2. 本地满，批量归还全局（加锁，但减少频率）                          │
│     if (push_free_chunk(_cur_free))  // 拷贝到全局                │
│         _cur_free.nfree = 1;                                     │
│         _cur_free.ids[0] = id;                                   │
│         return 0;                                                │
│                                                                  │
│  3. 极端情况：归还失败（内存不足）                                   │
│     return -1;                                                   │
│                                                                  │
│  注意：实际对象内存不释放，只是标记为空闲，复用                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### 五、关键设计亮点

| 设计 | 说明 | 收益 |
|:---|:---|:---|
| **Thread Local Storage** | 每个线程独立 `LocalPool` | 90%+ 操作无锁，无竞争 |
| **批量处理** | `FreeChunk` 批量转移 | 摊平全局锁开销 |
| **placement new** | 预分配内存上构造 | 避免频繁 malloc/free |
| **ID 编码地址** | `value = block_index * NITEM + offset` | O(1) 地址计算，无哈希 |
| **延迟回收** | 归还到空闲列表，不立即释放内存 | 快速复用，减少系统调用 |
| **分层管理** | Block → BlockGroup → ResourcePool | 支持海量对象（65536×65536）|
| **缓存行对齐** | `BAIDU_CACHELINE_ALIGNMENT` | 避免伪共享，提升并发性能 |

---

### 六、与 `VersionedRefWithId` 的协作

```
┌─────────────────────────────────────────────────────────────────┐
│              ResourcePool + VersionedRefWithId 协作              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ResourcePool<T>                    VersionedRefWithId<T>        │
│  ────────────────                   ────────────────────         │
│                                                                  │
│  职责：内存分配/回收                  职责：对象生命周期管理          │
│  • get_resource() → 原始内存         • Create() → 初始化 + 版本控制  │
│  • return_resource() → 延迟回收        • SetFailed() → 标记失效      │
│  • address_resource() → O(1) 查找      • Address() → 安全访问检查      │
│                                                                  │
│  协作流程（Socket 创建）：                                          │
│                                                                  │
│  1. Socket::Create()                                              │
│     ├── butil::get_resource(&slot)  // ResourcePool 分配内存         │
│     │   └── 返回 Socket*（原始内存，未初始化）                      │
│     ├── new (ptr) Socket(Forbidden)  // placement new 构造          │
│     ├── _versioned_ref.fetch_add(1)  // 引用计数初始化              │
│     └── MakeVRefId(version, slot)    // 创建带版本的 ID            │
│                                                                  │
│  2. Socket::Address(id)                                           │
│     ├── SlotOfVRefId(id) → slot      // 解码 ID                    │
│     ├── address_resource(slot)       // ResourcePool O(1) 查找       │
│     ├── _versioned_ref.fetch_add(1)  // 增加引用                    │
│     └── version 检查 → 成功则返回                                     │
│                                                                  │
│  3. Socket::Dereference()                                         │
│     ├── _versioned_ref.fetch_sub(1)  // 减少引用                    │
│     └── nref==0 && version 检查 → 触发 return_resource(slot)        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### 七、性能对比

| 操作 | `malloc/free` | `ResourcePool` | 提升 |
|:---|:---|:---|:---|
| 分配 | ~100-500 ns | ~5-10 ns | **10-50x** |
| 归还 | ~100-500 ns | ~5-10 ns | **10-50x** |
| 查找（地址）| N/A（指针直接访问）| ~2-3 ns | 可接受 |
| 并发扩展性 | 差（全局锁）| 优（TLS 无锁）| **百倍提升** |

---

### 八、一句话总结

> **`ResourcePool` 是 brpc 的"内存引擎"**：它通过 **Thread Local Storage** 实现 90%+ 无锁分配，通过 **placement new + 延迟回收** 避免频繁系统调用，通过 **ID 编码地址** 实现 O(1) 安全访问，为 `VersionedRefWithId` 和整个 brpc 提供**高性能、高并发、线程安全**的对象生命周期管理基础。