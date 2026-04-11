 我来详细解析 `InputMessenger` 类，这是 brpc 网络传输层的核心组件，负责处理所有网络读事件和消息解析。

## InputMessenger 核心定位

```
┌─────────────────────────────────────────────────────────┐
│                    InputMessenger                        │
│  "网络数据的入口处理器，连接 Socket 和 Protocol 的桥梁"    │
├─────────────────────────────────────────────────────────┤
│  职责：                                                  │
│  1. 处理 Socket 读事件 (epoll/kqueue 触发)               │
│  2. 从内核读取数据到 IOBuf                               │
│  3. 调用协议解析器切分消息边界                            │
│  4. 将完整消息分发给上层处理                              │
│  5. 管理解析状态和错误处理                                │
└─────────────────────────────────────────────────────────┘
```
我来绘制 `InputMessage` 的完整处理流程图，包括静态结构和动态调用。

## 一、相关类图
 我来基于最新 brpc 代码绘制 `InputMessage` 相关的完整类图。

### 完整类图（Mermaid 格式）

```mermaid
classDiagram
    %% 基础接口层
    class Destroyable {
        <<abstract>>
        +virtual ~Destroyable()
        +virtual void Destroy() = 0
    }

    %% 核心消息基类
    class InputMessageBase {
        <<abstract>>
        -int64_t _received_us
        -int64_t _base_real_us
        -SocketUniquePtr _socket
        -void (*_process)(InputMessageBase*)
        -const void* _arg
        +void Destroy()
        +Socket* ReleaseSocket()
        +Socket* socket() const
        +const void* arg() const
        +int64_t received_us() const
        +int64_t base_real_us() const
        #virtual void DestroyImpl() = 0
        #virtual ~InputMessageBase()
    }

    %% 具体消息实现
    class MostCommonMessage {
        <<struct>>
        +butil::IOBuf meta
        +butil::IOBuf payload
        +PipelinedInfo pi
        +static MostCommonMessage* Get()
        +void DestroyImpl()
    }

    %% HTTP 专用消息
    class HttpMessage {
        <<struct>>
        +HttpMessage* Get()
        +void DestroyImpl()
        +void Clear()
        +void Swap(HttpMessage*)
        +http_message::Header header
        +http_message::URL url
        +std::string status_code
        +std::string content_type
        +butil::IOBuf body
        +HttpMethod method
        +HttpVersion version
        +bool read_progressively
        +bool write_progressively
    }

    %% 其他协议消息
    class H2Context {
        <<struct>>
        +H2Context* Get()
        +void DestroyImpl()
        +H2StreamContext* FindStream(int)
        +H2StreamContext* RemoveStream(int)
        +void AddStream(H2StreamContext*)
        +std::map streams
        +H2Settings remote_settings
        +H2Settings local_settings
        +H2ConnectionState state
        +butil::IOBuf pending_data
    }

    class H2StreamContext {
        <<struct>>
        +H2StreamContext* Get()
        +void DestroyImpl()
        +int stream_id
        +H2StreamState state
        +butil::IOBuf data
        +H2HeaderTable* header_table
    }

    class MongoContext {
        <<struct>>
        +MongoContext* Get()
        +void DestroyImpl()
        +MongoReply* reply
        +int counter
    }

    class MongoReply {
        <<struct>>
        +MongoReply* Get()
        +void DestroyImpl()
        +butil::IOBuf meta
        +butil::IOBuf payload
    }

    class RtmpContext {
        <<struct>>
        +RtmpContext* Get()
        +void DestroyImpl()
        +RtmpStreamBase* stream
        +int chunk_size
        +uint32_t window_ack_size
        +uint32_t peer_bandwidth
        +butil::IOBuf pending_buf
    }

    %% 辅助类
    class DestroyingPtr~T~ {
        <<template>>
        +DestroyingPtr()
        +DestroyingPtr(T*)
        +T* get()
        +T* release()
        +void reset(T*)
    }

    class SocketUniquePtr {
        <<typedef>>
        std::unique_ptr~Socket, SocketDeleter~
    }

    class Socket {
        <<friend>>
        -SocketId _id
        -butil::IOBuf _read_buf
        +SocketId id() const
        +butil::EndPoint remote_side()
        +ssize_t DoRead(butil::IOBuf*)
        +int Write(butil::IOBuf*)
    }

    class InputMessenger {
        <<friend>>
        -InputMessageHandler* _handlers
        -butil::atomic _max_index
        +int AddHandler(const InputMessageHandler&)
        +int Create(const butil::EndPoint&, time_t, SocketId*)
        +static void OnNewMessages(Socket*)
        -ParseResult CutInputMessage(Socket*, size_t*, bool)
        -int ProcessNewMessage(Socket*, ssize_t, bool, uint64_t, uint64_t, InputMessageClosure&)
    }

    class InputMessageHandler {
        <<struct>>
        +Parse parse
        +Process process
        +Verify verify
        +const void* arg
        +const char* name
    }

    class InputMessageClosure {
        +InputMessageClosure()
        +~InputMessageClosure()
        +InputMessageBase* release()
        +void reset(InputMessageBase*)
        -InputMessageBase* _msg
    }

    class ParseResult {
        <<struct>>
        -ParseError _error
        -InputMessageBase* _msg
        +ParseError error() const
        +InputMessageBase* message() const
        +static ParseResult OK(InputMessageBase*)
        +static ParseResult Fail(ParseError)
    }

    %% 继承关系
    Destroyable <|-- InputMessageBase
    InputMessageBase <|-- MostCommonMessage
    InputMessageBase <|-- HttpMessage
    InputMessageBase <|-- H2Context
    InputMessageBase <|-- H2StreamContext
    InputMessageBase <|-- MongoContext
    InputMessageBase <|-- MongoReply
    InputMessageBase <|-- RtmpContext

    %% 关联关系
    InputMessageBase "1" *-- "1" SocketUniquePtr : owns
    SocketUniquePtr "1" --> "1" Socket : manages
    InputMessenger "1" --> "*" InputMessageHandler : uses
    InputMessenger "1" --> "1" InputMessageBase : processes
    InputMessageHandler "1" --> "1" InputMessageBase : creates
    MostCommonMessage "1" --> "1" PipelinedInfo : contains
    H2Context "1" --> "*" H2StreamContext : manages
    DestroyingPtr "1" --> "1" InputMessageBase : manages
    InputMessageClosure "1" --> "1" InputMessageBase : manages
```

### 关键关系说明

| 关系类型 | 说明 |
|:---|:---|
| **继承** | `InputMessageBase` 继承 `Destroyable`，所有具体消息继承 `InputMessageBase` |
| **组合** | `InputMessageBase` 包含 `SocketUniquePtr`（消息来源） |
| **依赖** | `InputMessenger` 依赖 `InputMessageHandler` 回调创建和处理消息 |
| **关联** | `H2Context` 关联多个 `H2StreamContext`（流管理） |
| **模板** | `DestroyingPtr<T>` 模板类管理任意消息类型生命周期 |





## 二、动态处理流程图（核心）

### 主流程：从网络到业务

```mermaid
flowchart LR
    A[网络数据] --> B[epoll_wait]
    B --> C[OnNewMessages]
    
    subgraph 读取解析
        C --> D[DoRead]
        D --> E{解析成功?}
        E -->|否| F[等待/错误]
        E -->|是| G[MostCommonMessage::Get]
        G --> H[切分meta/payload]
    end
    
    subgraph 提交处理
        H --> I[填充_socket/time/arg/process]
        I --> J[bthread_start_background]
    end
    
    subgraph 业务执行
        J --> K[ProcessInputMessage]
        K --> L[msg->_process]
        L --> M[ProcessRpcRequest]
        M --> N[CallMethod]
        N --> O[SendRpcResponse]
    end
    
    subgraph 销毁回收
        O --> P[msg->Destroy]
        P --> Q[DestroyImpl]
        Q --> R[return_object]
    end
    
    F --> S[结束]
    R --> S
    
    style A fill:#e3f2fd
    style C fill:#bbdefb
    style G fill:#fff3e0
    style J fill:#c8e6c9
    style N fill:#a5d6a7
    style P fill:#ffebee
```
关键节点说明
| 节点        | 函数/类                       | 作用                   |
| :-------- | :------------------------- | :------------------- |
| **网络到达**  | `epoll_wait`               | 监听 Socket 可读事件       |
| **读取数据**  | `Socket::DoRead`           | 从内核读取到 `IOBuf`（零拷贝）  |
| **解析消息**  | `CutInputMessage`          | 调用协议 `parse` 回调切分消息  |
| **获取对象**  | `MostCommonMessage::Get`   | 从对象池分配消息（无锁）         |
| **填充元数据** | `ProcessNewMessage`        | 设置时间戳、Socket、处理函数    |
| **提交异步**  | `bthread_start_background` | 创建 bthread 执行，不阻塞 IO |
| **业务处理**  | `ProcessRpcRequest`        | 反序列化、查找方法、执行业务       |
| **销毁回收**  | `DestroyImpl`              | 清理 IOBuf，归还对象池       |



---

## 三、详细时序图（动态调用）

### 1. 服务端请求处理完整时序

```mermaid
sequenceDiagram
    autonumber
    participant Client
    participant Socket as Socket (fd)
    participant Epoll as EventDispatcher<br/>(epoll)
    participant IM as InputMessenger
    participant MCM as MostCommonMessage
    participant Bthread as bthread 池
    participant Service as 用户 Service
    
    Note over Client,Service: 阶段1: 网络接收
    Client->>Socket: 发送 RPC 请求 (PRPC 协议)
    Epoll->>Epoll: epoll_wait 检测到可读
    Epoll->>IM: 触发 OnNewMessages(Socket*)
    
    Note over IM: 阶段2: 消息解析
    IM->>Socket: DoRead(&_read_buf)
    Socket-->>IM: 返回读取字节数
    
    loop 直到数据不足或解析错误
        IM->>IM: CutInputMessage()
        IM->>IM: 调用 ParseRpcMessage()
        
        alt 数据不足
            IM-->>IM: 返回 PARSE_ERROR_NOT_ENOUGH_DATA<br/>退出循环等待下次可读
        else 协议不匹配
            IM-->>IM: 返回 PARSE_ERROR_TRY_OTHERS<br/>尝试其他协议
        else 解析成功
            IM->>MCM: MostCommonMessage::Get()
            MCM-->>IM: 返回消息对象 (对象池)
            IM->>MCM: IOBuf::cutn(&meta)<br/>IOBuf::cutn(&payload)
            IM->>IM: ProcessNewMessage()
            IM->>MCM: 设置 _socket, _received_us<br/>_arg, _process
        end
    end
    
    Note over IM,Bthread: 阶段3: 异步提交
    IM->>Bthread: bthread_start_background<br/>(ProcessInputMessage, msg)
    IM-->>Epoll: 返回，继续监听
    
    Note over Bthread,Service: 阶段4: 业务处理
    Bthread->>Bthread: ProcessInputMessage()
    Bthread->>MCM: msg->_process(msg)<br/>即 ProcessRpcRequest()
    
    ProcessRpcRequest->>MCM: static_cast<MostCommonMessage*>
    ProcessRpcRequest->>MCM: ParsePbFromIOBuf(&meta)
    ProcessRpcRequest->>Service: svc->CallMethod(method, cntl, req, res, done)
    
    Note over Service: 执行业务逻辑
    
    Service-->>ProcessRpcRequest: 业务完成，回调 done<br/>即 SendRpcResponse()
    
    Note over ProcessRpcRequest: 阶段5: 发送响应
    ProcessRpcRequest->>Socket: Socket::Write(&res_buf)
    Socket-->>Client: 发送 RPC 响应
    
    Note over MCM: 阶段6: 销毁回收
    ProcessRpcRequest->>MCM: msg->Destroy()
    MCM->>MCM: DestroyImpl()
    MCM->>MCM: meta.clear()<br/>payload.clear()
    MCM->>MCM: butil::return_object(this)<br/>归还对象池
```

---

## 四、关键函数调用链

### 1. 服务端完整调用链

```
InputMessenger::OnNewMessages(Socket* m)
    │
    ├── 1. 数据读取
    │   └── m->DoRead(&m->_read_buf)
    │       ├── recvmsg() / splice() [系统调用]
    │       └── 数据追加到 IOBuf (零拷贝)
    │
    ├── 2. 消息切分循环
    │   └── CutInputMessage(m, &index, read_eof)
    │       └── _handlers[i].parse(&m->_read_buf, m, read_eof, _handlers[i].arg)
    │           └── ParseRpcMessage(source, socket, read_eof, arg) [baidu_std协议]
    │               ├── 检查魔数 "PRPC"
    │               ├── 读取 body_size, meta_size
    │               ├── MostCommonMessage* msg = MostCommonMessage::Get()
    │               ├── source->cutn(&msg->meta, meta_size)
    │               ├── source->cutn(&msg->payload, body_size - meta_size)
    │               └── return MakeMessage(msg) → ParseResult
    │
    ├── 3. 消息处理
    │   └── ProcessNewMessage(m, bytes, read_eof, received_us, base_realtime, last_msg)
    │       ├── msg->_received_us = butil::cpuwide_time_us()
    │       ├── msg->_base_real_us = base_realtime
    │       ├── msg->_socket.reset(m) [引用计数+1]
    │       ├── msg->_arg = _handlers[index].arg
    │       ├── msg->_process = _handlers[index].process
    │       └── bthread_start_background(&tid, NULL, ProcessInputMessage, msg)
    │
    └── 4. 异步执行 (新 bthread)
        └── ProcessInputMessage(void* arg)
            └── msg->_process(msg) [即 ProcessRpcRequest]
                ├── DestroyingPtr<MostCommonMessage> msg_guard(msg)
                ├── ParsePbFromIOBuf(&rpc_meta, msg->meta)
                ├── server->FindMethodProperty() [查找服务方法]
                ├── messages = factory->Get() [获取请求/响应对象]
                ├── DeserializeRpcMessage(msg->payload, ...) [反序列化]
                ├── svc->CallMethod(method, cntl, req, res, done) [业务调用]
                └── SendRpcResponse(...) [发送响应]
                    └── msg->Destroy() [触发销毁]
                        └── MostCommonMessage::DestroyImpl()
                            ├── meta.clear()
                            ├── payload.clear()
                            └── butil::return_object(this) [归还对象池]
```

### 2. 客户端响应调用链

```
InputMessenger::OnNewMessages(Socket* m)
    │
    ├── ... [同上：读取、解析]
    │
    └── ProcessInputMessage(void* arg)
        └── msg->_process(msg) [即 ProcessRpcResponse]
            ├── DestroyingPtr<MostCommonMessage> msg_guard(msg)
            ├── ParsePbFromIOBuf(&rpc_meta, msg->meta)
            ├── bthread_id_lock(correlation_id, &cntl) [匹配请求]
            ├── DeserializeRpcMessage(msg->payload, ..., cntl->response())
            ├── accessor.OnResponse(correlation_id, saved_error)
            │   └── 唤醒等待的客户端调用
            └── msg->Destroy()
```

---

## 五、关键设计点标注

```mermaid
flowchart LR
    subgraph ZeroCopy [零拷贝设计]
        A[Socket 内核缓冲区] -->|splice/recvmsg| B[IOBuf 块]
        B -->|cutn 指针移动| C[MostCommonMessage<br/>meta/payload]
        C -->|ParseFromZeroCopyStream| D[Protobuf 对象]
        style A fill:#e3f2fd
        style B fill:#e3f2fd
        style C fill:#e3f2fd
    end
    
    subgraph ObjectPool [对象池设计]
        E[MostCommonMessage::Get] -->|butil::get_object| F[线程本地缓存]
        F -->|无可用| G[全局对象池]
        G -->|return_object| F
        style E fill:#f3e5f5
        style F fill:#f3e5f5
        style G fill:#f3e5f5
    end
    
    subgraph Async [异步处理]
        H[InputMessenger<br/>IO 线程] -->|bthread_start| I[bthread 池]
        I -->|并行执行| J[多个 ProcessInputMessage]
        style H fill:#e8f5e9
        style I fill:#e8f5e9
        style J fill:#e8f5e9
    end
```

---

## 六、总结：核心流程口诀

```
一读：Socket::DoRead 从内核读数据
二切：CutInputMessage 协议解析切消息
三填：ProcessNewMessage 填充时间戳和回调
四提：bthread_start_background 提交异步处理
五调：msg->_process 调用业务逻辑
六发：SendRpcResponse 发送响应
七销：msg->Destroy 销毁归还对象池
```