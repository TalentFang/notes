// 文字 -> Excalidraw 桥（方案A）
// 由 pi 生成 / 更新。在 Obsidian 中执行命令 "(Script) AI-绘图桥" 触发。
// 插件用 new AsyncFunction("ea","utils", 去 frontmatter 全文) 编译本文件 —— 必须是纯 JS，不能有 md 代码围栏。
// 运行日志写在 Excalidraw/_bridge-log.md（pi 通过 REST 回读）。

const diagrams = [
  // ==================== Kafka ====================
  {
    folder: "Excalidraw/Kafka",
    filename: "Kafka 架构总览.excalidraw.md",
    chart: `flowchart TB
  subgraph CL["客户端层"]
    Producer["Producer<br/>分区器 + 批量 + 压缩"]
    Consumer["Consumer Group<br/>fetch 拉取 + 位移提交"]
    Eco["Connect / Streams / Admin"]
  end
  subgraph BR["Broker 进程（每节点一个）"]
    Net["SocketServer 网络层<br/>num.network.threads"]
    Req["RequestChannel 请求队列"]
    Io["KafkaRequestHandler 线程池<br/>num.io.threads"]
    Rm["ReplicaManager<br/>副本读写与 ISR 维护"]
    Lm["LogManager<br/>分区日志生命周期"]
    Gc["GroupCoordinator<br/>消费组与位移"]
    Tc["TransactionCoordinator"]
  end
  subgraph MT["集群元数据"]
    Ctl["Controller<br/>分区状态机 + Leader 选举"]
    Kft["KRaft Controller Quorum<br/>Raft 复制元数据日志"]
  end
  subgraph ST["存储层"]
    Log[("分区日志<br/>顺序追加 + 稀疏索引")]
  end
  Producer --> Net
  Consumer --> Net
  Eco --> Net
  Net --> Req --> Io
  Io --> Rm
  Io --> Gc
  Io --> Tc
  Rm --> Lm --> Log
  Kft --> Ctl --> Rm
  Gc --> Log`,
  },
  {
    folder: "Excalidraw/Kafka",
    filename: "Kafka 存储与索引原理.excalidraw.md",
    chart: `flowchart LR
  subgraph DIR["分区目录 topic-partition"]
    Logf[".log<br/>消息批次顺序追加"]
    Idxf[".index<br/>偏移索引，每 4KB 一条"]
    Timef[".timeindex<br/>时间戳索引"]
    Chk[".snapshot 与 leader-epoch-checkpoint"]
  end
  Msg["消息批次 v2<br/>varint 可变长 + CRC32C<br/>批量头 + 压缩块"] --> Logf
  Logf --> Seg["LogSegment<br/>按 segment.bytes 滚动<br/>文件名即 baseOffset"]
  Seg --> Sparse["稀疏索引<br/>二分定位最近的索引项"]
  Sparse --> Scan["从索引位置顺序扫描<br/>log.index.interval.bytes 控制粒度"]
  Seg --> Pc["OS Page Cache<br/>读写都命中页缓存"]
  Pc --> Zero["sendfile 零拷贝<br/>数据不经用户态直达网卡"]
  Seg --> Clean["日志清理策略<br/>delete 按时间或大小<br/>compact 按 key 只留最新"]
  Comp["compaction 后段内 key 去重<br/>产生墓碑标记"] --> Clean
  Idxf --> Sparse
  Timef --> Sparse`,
  },
  {
    folder: "Excalidraw/Kafka",
    filename: "Kafka 副本与一致性原理.excalidraw.md",
    chart: `flowchart TB
  subgraph LD["Leader 副本"]
    Leo["LEO 下一条待写偏移"]
    Hw["HW 高水位 = ISR 中最小的 LEO"]
    Epoch["Leader Epoch<br/>防止落后副本错位截断"]
  end
  subgraph FL["Follower 副本"]
    Fq["fetch 请求携带自身 LEO"]
    Fw["写入本地日志并回报进度"]
  end
  P["Producer<br/>acks=0 不等待 / 1 Leader 落盘 / all ISR 最小副本数"] --> Leo
  Leo --> Fq --> Fw
  Fw -->|"落后量"| Isr["ISR 集合<br/>replica.lag.time.max.ms 内保持同步"]
  Isr --> Hw
  Leo --> Hw
  Hw -->|"HW 之前的消息才对消费者可见"| Cons["Consumer"]
  Minisr["min.insync.replicas<br/>ISR 小于该值直接拒绝写入"] --> Isr
  Ule["unclean.leader.election.enable<br/>允许非 ISR 副本当 Leader<br/>可用性提升但可能丢数据"] --> Epoch
  Epoch --> Hw
  Ctrl["Controller 监听 ISR 收缩<br/>从存活 ISR 中选新 Leader"] --> Leo`,
  },
  {
    folder: "Excalidraw/Kafka",
    filename: "Kafka 消费组与再平衡.excalidraw.md",
    chart: `flowchart TB
  subgraph CG["消费组 group.id"]
    Ca["Consumer A<br/>分区 0,1"]
    Cb["Consumer B<br/>分区 2,3"]
  end
  Gc["GroupCoordinator<br/>管理者组成员与位移"] --> CG
  Hb["心跳 heartbeat.interval.ms<br/>session.timeout.ms 判活"] --> Gc
  subgraph RB["再平衡流程"]
    R1["触发：成员加入/离开、订阅变化、分区数变化"]
    R2["JoinGroup 选出 Leader 消费者"]
    R3["SyncGroup 下发分配方案"]
    R4["各成员恢复拉取<br/>期间 stop-the-world 不可消费"]
  end
  Ca --> R1
  Cb --> R1
  R1 --> R2 --> R3 --> R4
  R4 --> Ca
  R4 --> Cb
  subgraph Strategy["分区分配策略"]
    S1["Range 按区间连续分配<br/>多主题时易倾斜"]
    S2["RoundRobin 轮询最均衡"]
    S3["Sticky / CooperativeSticky<br/>尽量少搬分区，支持增量再平衡"]
  end
  Strategy --> R3
  subgraph Off["位移管理"]
    O1["__consumer_offsets 内部主题<br/>50 分区，key 为 group+topic+partition"]
    O2["自动提交可能重复消费"]
    O3["commitSync / commitAsync 手动提交"]
  end
  Gc --> Off
  Lag["消费延迟 = HW 减 已提交位移"] --> Off`,
  },
  {
    folder: "Excalidraw/Kafka",
    filename: "Kafka 3 节点集群拓扑.excalidraw.md",
    chart: `flowchart TB
  subgraph Q["KRaft 控制器仲裁组（3 节点 Raft）"]
    Kv["controller.quorum.voters<br/>多数派 2 台可选举"]
    Ks["__cluster_metadata 元数据日志"]
    Kv --> Ks
  end
  subgraph N1["节点 1 broker.id=1"]
    B1["Broker 进程"]
    D1[("log.dirs 数据盘")]
    B1 --> D1
  end
  subgraph N2["节点 2 broker.id=2"]
    B2["Broker 进程"]
    D2[("log.dirs 数据盘")]
    B2 --> D2
  end
  subgraph N3["节点 3 broker.id=3"]
    B3["Broker 进程"]
    D3[("log.dirs 数据盘")]
    B3 --> D3
  end
  Ks -->|"元数据复制"| B1
  Ks -->|"元数据复制"| B2
  Ks -->|"元数据复制"| B3
  subgraph PT["分区副本布局 replication.factor=3"]
    T0["主题分区 0<br/>Leader=1 Follower=2,3"]
    T1["主题分区 1<br/>Leader=2 Follower=3,1"]
    T2["主题分区 2<br/>Leader=3 Follower=1,2"]
  end
  B1 --- T0
  B2 --- T1
  B3 --- T2
  B2 -->|"节点 2 宕机"| Fix["Controller 从 ISR 选新 Leader<br/>分区 1 提升副本 3"]
  Fix --> Guard["min.insync.replicas=2<br/>副本剩 2 台仍可写<br/>再挂 1 台即拒绝写入"]
  Guard --> Down["数据面可容忍 1 台故障<br/>控制面同样容忍 1 台"]`,
  },

  // ==================== Elasticsearch ====================
  {
    folder: "Excalidraw/Elasticsearch",
    filename: "ES 架构总览.excalidraw.md",
    chart: `flowchart TB
  subgraph Cli["客户端"]
    Rest["REST / bulk / search 请求"]
  end
  subgraph Node["节点角色（可组合）"]
    M["Master 节点<br/>集群状态与分片分配"]
    D["Data 节点<br/>持有分片执行读写"]
    Co["Coordinating 节点<br/>分发请求并归并结果"]
    In["Ingest 节点<br/>pipeline 数据预处理"]
  end
  subgraph Cluster["集群协调"]
    Disc["discovery 模块<br/>节点发现与 Master 选举"]
    Cs["ClusterState<br/>索引、映射、路由表"]
  end
  subgraph Shard["分片与 Lucene"]
    Prim["Primary Shard 主分片"]
    Repl["Replica Shard 副本分片"]
    Luc["Lucene 索引<br/>不可变段 + 倒排 + DocValues"]
  end
  Rest --> Co
  Co --> In --> D
  Co --> D
  D --> Prim --> Luc
  Prim -->|"复制请求"| Repl
  Disc --> M --> Cs --> Co
  Cs --> D`,
  },
  {
    folder: "Excalidraw/Elasticsearch",
    filename: "ES 存储与段合并原理.excalidraw.md",
    chart: `flowchart LR
  Doc["写入文档"] --> Buf["内存 Index Buffer"]
  Doc --> Tl["translog 预写日志<br/>durability=request 时同步刷盘"]
  Buf -->|"refresh 默认 1s<br/>或 buffer 满"| NewSeg["生成新 Lucene 段<br/>文档变为可搜索"]
  Tl -->|"恢复未刷盘数据"| Rec["节点重启回放 translog"]
  NewSeg --> Comm["commit point 提交点<br/>记录全部可用段"]
  Comm -->|"flush 触发：translog 阈值、定时、显式调用"| Fs["fsync 段文件并写提交点"]
  NewSeg --> Merge["段合并 merge<br/>tiered 策略按大小分层"]
  Merge --> Phys["物理删除 .del 标记的文档<br/>收回磁盘与内存"]
  Merge --> Big["合并出更大段<br/>减少段数量与查询开销"]
  Del["删除与更新"] --> Mark["只写 .del 标记<br/>真正的物理删除延后到合并"]
  Mark --> Merge
  Fm["force merge 手工合并<br/>只读索引可压到 1 段"] --> Merge`,
  },
  {
    folder: "Excalidraw/Elasticsearch",
    filename: "ES 索引与查询原理.excalidraw.md",
    chart: `flowchart TB
  subgraph IX["索引结构"]
    Fst["词典 FST<br/>前缀共享自动机常驻堆外"]
    Post["postings 倒排表<br/>docId 有序 + skip list 跳表"]
    Dv["DocValues 列存<br/>排序、聚合、脚本取值"]
    Src["_source 原始文档<br/>取回阶段还原结果"]
  end
  Term["term 查询"] --> Fst --> Post
  Sort["sort / aggs"] --> Dv
  subgraph Q["search 两阶段"]
    Qp["query 阶段：每个分片本地评分<br/>返回 TopN docId + score"]
    Fp["fetch 阶段：按 _id 取回 _source<br/>组装命中结果"]
  end
  Post --> Qp
  Qp --> Co["协调节点归并全局 TopN"]
  Co --> Fp --> Src
  subgraph Sem["检索语义"]
    Bm["BM25 打分：TF、IDF、字段长度归一"]
    Flt["filter context 不打分<br/>可缓存 bitset"]
    Dfs["DFS_query_then_fetch<br/>先收集全局词频再打分"]
  end
  Sem --> Qp
  subgraph Agg["聚合"]
    AggQ["按分片局部聚合<br/>coordinator 做最终归并"]
  end
  AggQ --> Dv`,
  },
  {
    folder: "Excalidraw/Elasticsearch",
    filename: "ES 写入链路与写一致性.excalidraw.md",
    chart: `flowchart LR
  Bulk["bulk 批量写入"] --> Coord["协调节点<br/>按 _id 路由到主分片"]
  Coord --> P["主分片 Primary"]
  P --> V["版本控制 _version<br/>乐观并发，冲突返回 409"]
  V --> W["写入内存 buffer + translog"]
  W --> AckQ["写一致性 wait_for_active_shards<br/>默认 1 或 quorum"]
  AckQ --> Rep["并行转发副本分片"]
  Rep --> Ack["所有活跃副本确认后返回客户端"]
  W --> Ref["refresh 变成可搜索"]
  Ref --> Vis["NRT 近实时：默认 1s 可见"]
  subgraph Fail["失败与重试"]
    F1["主分片故障：副本提升为主"]
    F2["副本失败：主分片继续写并重建副本"]
    F3["写入重试需业务幂等<br/>可用 _version 或外部版本号去重"]
  end
  P --> Fail
  subgraph Route["分片路由"]
    R1["shard = hash(routing) mod 主分片数"]
    R2["主分片数创建后不可改<br/>改分片数需 reindex"]
  end
  Route --> Coord`,
  },
  {
    folder: "Excalidraw/Elasticsearch",
    filename: "ES 3 节点集群拓扑与分片分配.excalidraw.md",
    chart: `flowchart TB
  subgraph Elec["选举与脑裂防护"]
    Quorum["7.x 起自动维护 voting 配置<br/>多数派在线才能选出 Master"]
    Split["防止脑裂：少数派分区不选主"]
    Quorum --> Split
  end
  subgraph N1["节点 1 · master+data"]
    M1["Master 候选与投票"]
    P0["主分片 P0"]
    R1["副本 R1"]
    T1[("数据盘")]
    M1 --> T1
    P0 --> T1
    R1 --> T1
  end
  subgraph N2["节点 2 · master+data"]
    M2["Master 候选与投票"]
    P1["主分片 P1"]
    R2["副本 R2"]
    T2[("数据盘")]
    M2 --> T2
    P1 --> T2
    R2 --> T2
  end
  subgraph N3["节点 3 · master+data"]
    M3["Master 候选与投票"]
    P2["主分片 P2"]
    R0["副本 R0"]
    T3[("数据盘")]
    M3 --> T3
    P2 --> T3
    R0 --> T3
  end
  Quorum --> M1
  Quorum --> M2
  Quorum --> M3
  subgraph Layout["3 节点典型布局"]
    L1["主分片 3 个"]
    L2["副本 1 份，共 6 个分片"]
    L3["每台既是主分片宿主又是别的分片副本"]
    L4["任一台数据节点故障，数据不丢"]
    L1 --> L2 --> L3 --> L4
  end
  P0 --> Layout
  subgraph Alloc["分配与恢复"]
    A1["分片分配器均衡节点与磁盘"]
    A2["awareness 机架或可用区感知"]
    A3["恢复限流 recovery.max_bytes_per_sec"]
  end
  Layout --> Alloc`,
  },
  {
    folder: "Excalidraw/Elasticsearch",
    filename: "ES 集群健康与故障恢复.excalidraw.md",
    chart: `flowchart TB
  subgraph Health["集群健康"]
    G["green 主副本全部就绪"]
    Y["yellow 主分片就绪，存在未分配副本<br/>单节点常见"]
    R["red 存在未分配主分片，部分数据不可读"]
  end
  subgraph Detect["故障检测"]
    Pg["节点间 ping 保活"]
    Mm["Master 失联则重新选举"]
    Nm["Data 节点失联则标记分片不可用"]
  end
  Detect --> Alloc2["分配器尝试重新分配分片"]
  Alloc2 --> Promote["副本提升为新的主分片<br/>未分配副本重新构建"]
  subgraph Rec["恢复类型"]
    Gt["gateway 从本地磁盘恢复未分配分片"]
    Pe["peer 从其他节点复制分片"]
    Sn["snapshot 从快照恢复"]
    Lo["local shard 已有数据直接加载"]
  end
  Promote --> Rec
  Rec --> Sync["恢复完成后追平 translog 增量"]
  Sync --> Health
  subgraph Safe["安全操作"]
    Ro["滚动重启：先关闭分片分配再改配置"]
    Wa["操作前确认副本数大于等于 1"]
    Snapshot["定期 snapshot 是唯一可靠备份"]
  end
  Health --> Safe`,
  },

  // ==================== ClickHouse ====================
  {
    folder: "Excalidraw/ClickHouse",
    filename: "ClickHouse 架构总览.excalidraw.md",
    chart: `flowchart TB
  subgraph Iface["接口层"]
    Tcp["TCP 9000 与 HTTP 8123"]
    Clients["clickhouse-client / JDBC / 各种集成"]
  end
  subgraph Parse["SQL 处理"]
    Parser["解析器生成 AST"]
    Analyser["语义分析与类型推导"]
    Opt["查询优化器<br/>谓词下推、聚合下推、join 重排"]
  end
  subgraph Exec["执行层"]
    Pipeline["处理器 pipeline<br/>拉取式流式执行"]
    Vector["向量化：一次处理一个列块"]
    Threads["多线程并行 + SIMD"]
  end
  subgraph Storage["存储引擎"]
    Mt["MergeTree 家族<br/>ReplacingSumAggregating 等"]
    Other["Log / Memory / Distributed / Kafka 等"]
    Disk["Disk 与 Volume 抽象<br/>本地盘、对象存储、S3"]
  end
  subgraph Meta["元数据与集群"]
    Keeper["ZooKeeper 或 ClickHouse Keeper<br/>副本元数据与分布式 DDL"]
    Sys["system 系统表<br/>慢查询、part、副本队列"]
  end
  Iface --> Parse --> Opt --> Pipeline
  Pipeline --> Vector --> Threads
  Pipeline --> Mt
  Mt --> Disk
  Other --> Disk
  Keeper --> Mt
  Sys --> Mt`,
  },
  {
    folder: "Excalidraw/ClickHouse",
    filename: "ClickHouse MergeTree 存储原理.excalidraw.md",
    chart: `flowchart LR
  Ins["INSERT 数据"] --> Part["数据目录 part<br/>一次性写入，之后不可变"]
  subgraph Pf["part 内部结构"]
    Col["column.bin 列文件<br/>每列独立连续存储"]
    Mrk[".mrk 标记文件<br/>记录每个 granule 的偏移"]
    Pk["primary.idx 主键索引<br/>稀疏索引"]
    Skip["skp_idx 跳数索引<br/>minmax set bloom_filter ngrambf"]
  end
  Part --> Pf
  Ord["ORDER BY 排序键<br/>决定数据在 part 内有序"] --> Pk
  Pk --> Gran["granule 粒度<br/>index_granularity 默认 8192 行"]
  Mrk --> Gran
  Gran --> Prune["查询时按主键与跳数索引裁剪 granule"]
  Skip --> Prune
  Part --> Merge["后台 merge<br/>同分区 part 归并成更大 part"]
  Merge --> Stable["减少 part 数量<br/>提升扫描效率"]
  subgraph Extra["分区与生命周期"]
    Partition["PARTITION BY 分区键<br/>按月或按天切分，便于批量删除"]
    Ttl["TTL 过期自动删除或降冷"]
    Proj["projection 预聚合投影"]
  end
  Stable --> Partition
  Partition --> Ttl`,
  },
  {
    folder: "Excalidraw/ClickHouse",
    filename: "ClickHouse 查询执行原理.excalidraw.md",
    chart: `flowchart TB
  Sql["SQL 文本"] --> Ast["AST 解析"]
  Ast --> Ana["语义分析：表、列、类型"]
  Ana --> Opt["优化"]
  subgraph Opts["关键优化"]
    O1["谓词下推到存储层"]
    O2["分区裁剪：只读命中分区"]
    O3["主键与跳数索引裁剪 granule"]
    O4["聚合下推、表达式下推、join 重排"]
  end
  Opt --> Opts
  Opts --> Pipe["执行 pipeline<br/>Source 到 Sink 的处理器链"]
  subgraph Ex["执行特征"]
    E1["向量化：按列块批量计算"]
    E2["多线程：max_threads 并行"]
    E3["流式：结果边算边返回，低内存"]
    E4["算子级并行与流水线重叠 IO 与计算"]
  end
  Pipe --> Ex
  Ex --> Con["结果返回客户端"]
  subgraph Acc["常见加速手段"]
    A1["预聚合物化视图"]
    A2["projection 与 skip index"]
    A3["列裁剪：只读需要的列"]
    A4["近似函数 uniqCombined、quantileTDigest"]
  end
  Acc --> Opts`,
  },
  {
    folder: "Excalidraw/ClickHouse",
    filename: "ClickHouse 写入与合并链路.excalidraw.md",
    chart: `flowchart LR
  Bat["批量 INSERT<br/>建议单批至少 1000 行"] --> Sort["按排序键在内存中排序并分组"]
  Sort --> Write["一次写入一个新 part 目录"]
  Write --> Fsync["磁盘落盘，part 立即可查询"]
  Fsync --> Mq["后台 merge 队列"]
  Mq --> Merge["同分区 part 归并<br/>减小 part 数量"]
  Merge --> Drop["旧 part 标记删除"]
  subgraph Async["异步写入"]
    A1["async_insert=1 服务端攒批"]
    A2["适合大量小写入场景"]
  end
  Async --> Sort
  subgraph Mv["物化视图"]
    M1["源表插入时触发"]
    M2["只处理新插入数据块<br/>做增量聚合写目标表"]
  end
  Write --> Mv
  subgraph Anti["常见反模式"]
    N1["小批量高频写入产生大量小 part"]
    N2["过多分区导致 part 爆炸"]
    N3["宽表随机更新需要 ReplacingMergeTree 合并"]
  end
  Write --> Anti`,
  },
  {
    folder: "Excalidraw/ClickHouse",
    filename: "ClickHouse 3 节点集群拓扑与副本.excalidraw.md",
    chart: `flowchart TB
  subgraph Cfg["集群配置"]
    Rs["remote_servers 定义分片与副本"]
    Mac["macros 定义 shard 与 replica 编号"]
    Keep["ClickHouse Keeper 3 节点<br/>多数派提供元数据一致性"]
  end
  subgraph S1["分片 1"]
    A1["副本 1-1<br/>ReplicatedMergeTree"]
    A2["副本 1-2<br/>ReplicatedMergeTree"]
    A1 -->|"复制日志同步"| A2
  end
  subgraph S2["分片 2"]
    B1["副本 2-1<br/>ReplicatedMergeTree"]
    B2["副本 2-2<br/>ReplicatedMergeTree"]
    B1 -->|"复制日志同步"| B2
  end
  subgraph S3["分片 3"]
    C1["副本 3-1<br/>ReplicatedMergeTree"]
    C2["副本 3-2<br/>ReplicatedMergeTree"]
    C1 -->|"复制日志同步"| C2
  end
  Keep --> A1
  Keep --> B1
  Keep --> C1
  subgraph Dist["分布式访问层"]
    Dt["Distributed 表<br/>本地表 + 分片键路由"]
    Dt --> S1
    Dt --> S2
    Dt --> S3
  end
  subgraph Typ["3 节点常见形态"]
    T1["1 分片 3 副本：高可用与读扩展"]
    T2["3 分片 1 副本：容量与写扩展"]
    T3["3 分片 2 副本：生产最常用均衡形态"]
    T4["3 节点 Keeper 仲裁避免脑裂"]
  end
  Dist --> Typ`,
  },
  {
    folder: "Excalidraw/ClickHouse",
    filename: "ClickHouse 分布式查询与副本路由.excalidraw.md",
    chart: `flowchart TB
  Q["客户端查询 Distributed 表"] --> Init["发起节点解析分片拓扑"]
  Init --> Fwd["把查询下推到各分片本地表"]
  Fwd --> L1["分片 1 本地执行"]
  Fwd --> L2["分片 2 本地执行"]
  Fwd --> L3["分片 3 本地执行"]
  subgraph Local["分片内执行"]
    Ll["按 granule 裁剪并向量化扫描"]
    Rr["副本选择策略 load_balancing<br/>随机、最近主机、按错误率"]
    Ll --> Rr
  end
  L1 --> Local
  L2 --> Local
  L3 --> Local
  Local --> Partial["各分片返回局部聚合结果"]
  Partial --> Final["发起节点做最终归并"]
  subgraph Join["跨分片关联"]
    J1["GLOBAL JOIN 先把右表广播到各分片"]
    J2["本地 join 避免重复拉取右表"]
  end
  Join --> Final
  subgraph Prec["精度与代价"]
    P1["uniqExact 精确但耗内存"]
    P2["uniqCombined 近似去重更省资源"]
    P3["分布式查询放大：分片越多越要控制返回行数"]
  end
  Prec --> Final`,
  },
];

// ---- 回读通道：把运行结果写进 vault 文件，pi 用 REST 读回 ----
const LOG_PATH = "Excalidraw/_bridge-log.md";
const vault =
  (utils && utils.scriptFile && utils.scriptFile.vault) ||
  (typeof app !== "undefined" && app && app.vault) ||
  (ea && ea.app && ea.app.vault) ||
  null;
const lines = ["# AI-绘图桥 运行日志", "", "start: " + new Date().toISOString(), "vault: " + (vault ? "ok" : "MISSING")];
const flush = async () => {
  if (!vault) return;
  const content = lines.join("\n") + "\n";
  const f = vault.getAbstractFileByPath(LOG_PATH);
  try {
    if (f) await vault.modify(f, content);
    else await vault.create(LOG_PATH, content);
  } catch (err) {
    console.error("[AI-绘图桥] log write failed", err);
  }
};

await flush();

try {
  lines.push("ea methods: addMermaid=" + typeof ea.addMermaid + " create=" + typeof ea.create + " clear=" + typeof ea.clear);
  await flush();

  for (const d of diagrams) {
    lines.push("");
    lines.push("## " + d.folder + "/" + d.filename);
    try {
      ea.clear();
      const r = await ea.addMermaid(d.chart);
      if (typeof r === "string") {
        lines.push("addMermaid -> ERROR " + r);
        await flush();
        continue;
      }
      const ids = Array.isArray(r) ? r : [];
      lines.push("addMermaid -> " + (Array.isArray(r) ? "ok ids=" + ids.length : typeof r));
      lines.push("elementsDict: " + Object.keys(ea.elementsDict || {}).length);
      await flush();
      const p = await ea.create({
        filename: d.filename,
        foldername: d.folder,
        silent: true,
        frontmatterKeys: { "excalidraw-plugin": "parsed" },
      });
      lines.push("create -> " + p);
    } catch (err) {
      lines.push("EXCEPTION: " + (err && err.stack ? err.stack : String(err)));
    }
    await flush();
  }
} catch (err) {
  lines.push("FATAL: " + (err && err.stack ? err.stack : String(err)));
}

lines.push("", "end: " + new Date().toISOString());
await flush();
return lines;
