## 类图
```mermaid
classDiagram
    direction TB
    
    %% ==================== 核心数据块 ====================
    class IOBufBlock {
        +atomic~int~ nshared
        +uint16_t flags
        +uint32_t size
        +uint32_t cap
        +char* data
        +void inc_ref()
        +void dec_ref()
        +bool full() const
        +size_t left_space() const
    }
    
    %% ==================== 块引用 ====================
    class IOBufBlockRef {
        +uint32_t offset
        +uint32_t length
        +IOBufBlock* block
        +bool operator==(const IOBufBlockRef&)
    }
    
    %% ==================== 视图基类 ====================
    class IOBufSmallView {
        +IOBufBlockRef refs[2]
        +size_t nref() const
    }
    
    class IOBufBigView {
        +IOBufBlockRef* refs
        +size_t nref
        +size_t cap
        +void push_back(IOBufBlockRef)
        +void pop_back()
        +void resize(size_t)
    }
    
    %% ==================== 核心IOBuf类 ====================
    class IOBuf {
        <<class>>
        -IOBufBigView _bv
        -IOBufSmallView _sv
        -bool _is_small() const
        +IOBuf()
        +~IOBuf()
        +IOBuf(const IOBuf&)
        +IOBuf(IOBuf&&)
        +IOBuf& operator=(const IOBuf&)
        +void append(const IOBuf&)
        +void append(const void*, size_t)
        +void pop_front(size_t)
        +void cut(IOBuf*, size_t)
        +void cutn(void*, size_t)
        +ssize_t read_from_file_descriptor(int, int)
        +ssize_t write_to_file_descriptor(int) const
        +bool append_user_data(void*, size_t, Deleter)
        +size_t length() const
        +bool empty() const
        +void clear()
        +void swap(IOBuf&)
        +const_iterator begin() const
        +const_iterator end() const
    }
    
    %% ==================== IOPortal（网络专用） ====================
    class IOPortal {
        <<class>>
        -ssize_t pappend_file(int, void*, size_t)
        +ssize_t read_from_file_descriptor(int, int)
        +ssize_t read_from_SSL_channel(SSLChannel*, int)
        +ssize_t read_from_IChannel(IChannel*, int)
        +void return_cached_blocks()
    }
    
    %% ==================== 零拷贝流 ====================
    class IOBufAsZeroCopyInputStream {
        -const IOBuf* _buf
        -size_t _block_count
        -size_t _current_block
        -const void* _data
        -int _size
        -bool _backup
        +IOBufAsZeroCopyInputStream(const IOBuf*)
        +~IOBufAsZeroCopyInputStream()
        +bool Next(const void**, int*)
        +void BackUp(int)
        +bool Skip(int)
        +int64_t ByteCount() const
    }
    
    class IOBufAsZeroCopyOutputStream {
        -IOBuf* _buf
        -IOBuf::Block* _block
        -size_t _initial_length
        +IOBufAsZeroCopyOutputStream(IOBuf*)
        +~IOBufAsZeroCopyOutputStream()
        +bool Next(void**, int*)
        +void BackUp(int)
        +int64_t ByteCount() const
    }
    
    %% ==================== 切割工具 ====================
    class IOBufCutter {
        -IOBuf* _buf
        -IOBuf::BlockRef* _cur_ref
        -size_t _cur_buf_offset
        +explicit IOBufCutter(IOBuf*)
        +bool cutn(IOBuf*, size_t)
        +bool cutn(void*, size_t)
        +bool cut1(void*)
        +bool cut2(void*)
        +bool cut4(void*)
        +bool cut8(void*)
        +bool peek(void*, size_t)
        +bool peek1(void*)
        +bool skip(size_t)
        +size_t remaining() const
    }
    
    %% ==================== 构建工具 ====================
    class IOBufBuilder {
        -IOBuf _buf
        +IOBufBuilder()
        +~IOBufBuilder()
        +IOBufBuilder& operator<<(bool)
        +IOBufBuilder& operator<<(short)
        +IOBufBuilder& operator<<(unsigned short)
        +IOBufBuilder& operator<<(int)
        +IOBufBuilder& operator<<(unsigned int)
        +IOBufBuilder& operator<<(long)
        +IOBufBuilder& operator<<(unsigned long)
        +IOBufBuilder& operator<<(long long)
        +IOBufBuilder& operator<<(unsigned long long)
        +IOBufBuilder& operator<<(float)
        +IOBufBuilder& operator<<(double)
        +IOBufBuilder& operator<<(const void*)
        +IOBufBuilder& operator<<(const char*)
        +IOBufBuilder& operator<<(const std::string&)
        +IOBufBuilder& operator<<(const IOBuf&)
        +IOBufBuilder& operator<<(const StringPiece&)
        +size_t size() const
        +IOBuf& buf()
        +IOBuf movable()
    }
    
    %% ==================== Socket相关 ====================
    class Socket {
        <<class>>
        -int _fd
        -IOBuf _read_buf
        -IOBuf _write_buf
        -EventDispatcher* _dispatcher
        +int Read(IOPortal*)
        +int Write(const IOBuf&)
        +int Write(const void*, size_t)
        +int fd() const
        +void set_priority(Priority)
    }
    
    class EventDispatcher {
        <<class>>
        -int _epfd
        -bthread_t _tid
        +int Start()
        +int Stop()
        +int AddConsumer(Socket*, int)
        +int RemoveConsumer(Socket*)
        +void Run()
    }
    
    %% ==================== 内存分配器 ====================
    class IOBufBlockMemPool {
        -pthread_mutex_t _mutex
        -std::deque~IOBufBlock*~ _free_blocks
        -size_t _capacity
        +IOBufBlock* get()
        +void put(IOBufBlock*)
        +size_t size() const
    }
    
    %% ==================== 关系定义 ====================
    IOBufBlockRef --> IOBufBlock : references
    IOBufSmallView --> IOBufBlockRef : contains[2]
    IOBufBigView --> IOBufBlockRef : contains[N]
    
    IOBuf --> IOBufSmallView : uses (union)
    IOBuf --> IOBufBigView : uses (union)
    IOBuf ..> IOBufBlockRef : manages
    IOBuf ..> IOBufBlock : allocates
    
    IOPortal --|> IOBuf : extends
    IOPortal ..> IOBufBlock : appends
    
    IOBufAsZeroCopyInputStream ..> IOBuf : reads
    IOBufAsZeroCopyOutputStream ..> IOBuf : writes
    
    IOBufCutter ..> IOBuf : modifies
    IOBufCutter ..> IOBufBlockRef : iterates
    
    IOBufBuilder ..> IOBuf : builds
    
    Socket ..> IOPortal : uses for read
    Socket ..> IOBuf : uses for write
    Socket --> EventDispatcher : registered to
    
    IOBufBlockMemPool ..> IOBufBlock : manages pool
    ```

    以上Mermaid类图包含：
核心数据层：IOBuf、Block、BlockRef、SmallView/BigView的包含关系
网络IO层：IOPortal继承IOBuf，Socket使用IOPortal
工具类：IOBufCutter、IOBufBuilder对IOBuf的操作
零拷贝流：IOBufAsZeroCopyInputStream/OutputStream实现Protobuf接口
时序图：展示append→cut→writev的零拷贝流程
面试时可以重点讲解SmallView/BigView的联合优化和Block的引用计数机制，体现你对内存布局和性能优化的深入理解。