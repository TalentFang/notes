
## 一、按模块
阶段 1：基础层（1-2周）
├── IOBuf (src/butil/iobuf.cpp)
│   └── 理解零拷贝、块引用计数
├── resource_pool.cpp - 资源池
├── 基础工具 (src/butil/)
│   ├── atomicops.h - 原子操作封装
│   ├── flat_map.h - 高性能哈希表
│   └── endpoint.cpp - 网络地址封装
└── 日志系统 (src/butil/logging.cpp)
    └── 理解日志系统、日志级别、日志格式



阶段 2：协程层（2-3周）
├── task_meta.h - 协程元数据
├── task_group.cpp - 调度核心
├── stack.cpp - 栈管理
└── butex.cpp - 协程同步原语

阶段 3：网络层（2-3周）
├── socket.cpp - 连接对象
├── socket_map.cpp - 连接池
├── event_dispatcher.cpp - epoll 封装
└── input_message.cpp - 消息框架

阶段 4：协议层（1-2周）
├── policy/baidu_rpc_protocol.cpp - 主协议
├── policy/http*.cpp - HTTP 实现
└── protocol.cpp - 协议注册机制

阶段 5：应用层（1-2周）
├── server.cpp - 服务端
├── channel.cpp - 客户端
├── controller.cpp - 请求上下文
└── load_balancer.cpp - 负载均衡


## 当前进度
- [ ] 基础层：IOBuf、资源池、基础工具、日志系统
* resource_pool资源池
CAS无锁编程，解决ABA问题
- [ ] 协程层：task_meta、task_group、stack、butex
-