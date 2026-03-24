

## CRTP 静态多态详解

### 一、什么是 CRTP（Curiously Recurring Template Pattern，即奇异递归模板模式）？

CRTP 是一种 C++ 模板编程技巧，让**派生类继承自以自身为模板参数的基类**：

```cpp
// 基类模板
template <typename Derived>
class Base {
    // 可以使用 Derived 的类型信息
};

// 派生类
class Derived : public Base<Derived> {  // 把自己传给基类
    // ...
};
```

**关键特征**：编译期确定类型，零运行时开销。

---

### 二、静态多态 vs 动态多态

| 特性 | 动态多态（虚函数） | 静态多态（CRTP） |
|:---|:---|:---|
| 实现机制 | 虚函数表（vtable） | 模板实例化 + 静态转换 |
| 调用方式 | 运行时通过 vptr 间接调用 | 编译期直接内联调用 |
| 性能 | 有间接跳转开销（约 3-10 ns） | **零开销**，直接函数调用 |
| 内存占用 | 每个对象 +8~16 字节（vptr） | **无额外开销** |
| 代码大小 | vtable 放在 .rodata 段 | 每个实例化生成独立代码（可能膨胀）|
| 灵活性 | 运行时动态绑定 | 编译期固定，无法运行时更换 |
| 调试 | 容易（虚函数表可见） | 模板展开后较复杂 |

---

### 三、`VersionedRefWithId` 中的 CRTP 实现

#### 1. 基础 CRTP 结构

```cpp
// 基类模板：T 是派生类
template <typename T>
class VersionedRefWithId {
public:
    // 静态转换为派生类
    T* derived() {
        return static_cast<T*>(this);
    }
    
    const T* derived() const {
        return static_cast<const T*>(this);
    }
    
    // 调用派生类的方法（编译期确定）
    void someOperation() {
        // 不是虚函数调用，直接内联
        derived()->onOperation();  
    }
};
```

#### 2. 派生类实现

```cpp
class Socket : public VersionedRefWithId<Socket> {  // CRTP 关键
public:
    explicit Socket(Forbidden f) 
        : VersionedRefWithId<Socket>(f) {}  // 传递自身类型
    
private:
    // 基类通过 static_cast 调用这些方法
    friend class VersionedRefWithId<Socket>;  // 允许基类访问私有方法
    
    int OnCreated(int fd, const SocketOptions& opt) {
        _fd = fd;
        // 初始化...
        return 0;
    }
    
    void BeforeRecycled() {
        close(_fd);  // 清理资源
    }
    
    void OnFailed(int error_code, const std::string& error_text) {
        LOG(ERROR) << "Socket failed: " << error_text;
    }
    
    // 可选钩子
    std::string OnDescription() const {
        return butil::string_printf("fd=%d", _fd);
    }
    
    int _fd;
};
```

---

### 四、CRTP 如何实现"多态"

#### 动态多态的做法（对比）

```cpp
// 动态多态：虚函数
class Base {
public:
    virtual int onCreated() = 0;  // 纯虚函数
    virtual ~Base() = default;
};

class Derived : public Base {
public:
    int onCreated() override { return 0; }  // 重写
};

// 调用
Base* obj = new Derived();
obj->onCreated();  // 运行时通过 vtable 查找 Derived::onCreated
```

#### CRTP 静态多态的做法

```cpp
// CRTP：模板 + 静态转换
template <typename T>
class Base {
public:
    int callOnCreated() {
        // 编译期就知道是 T，直接调用 T::onCreated
        return static_cast<T*>(this)->onCreated();
    }
};

class Derived : public Base<Derived> {
public:
    int onCreated() { return 0; }  // 不需要 virtual
};

// 调用
Derived d;
d.callOnCreated();  // 编译期内联为 Derived::onCreated，无运行时开销
```

---

### 五、`VersionedRefWithId` 中的高级技巧

#### 1. `WRAPPER_OF` 宏：可选钩子的编译期检查

```cpp
// 检查派生类是否实现了可选方法
#define WRAPPER_OF(class_type, func_name, return_type) \
    struct func_name ## Wrapper { \
        /* SFINAE 检查是否存在 func_name 方法 */ \
        template <typename... Args> \
        static auto Test(int) -> decltype( \
            std::declval<class_type>().func_name(std::declval<Args>()...), \
            std::true_type()); \
        \
        template <typename... Args> \
        static auto Test(...) -> std::false_type; \
        \
        /* 如果存在，调用实际方法 */ \
        template <typename... Args> \
        typename std::enable_if< \
            decltype(Test<Args...>(0))::value, return_type>::type \
        Call(class_type* obj, Args&&... args) { \
            /* 编译期检查返回类型匹配 */ \
            BAIDU_CASSERT((butil::is_result_same< \
                return_type, decltype(&T::func_name), T, Args...>::value), \
                "Params or return type mismatch"); \
            return obj->func_name(std::forward<Args>(args)...); \
        } \
        \
        /* 如果不存在，返回默认值 */ \
        template <typename... Args> \
        typename std::enable_if< \
            !decltype(Test<Args...>(0))::value, return_type>::type \
        Call(class_type* obj, Args&&...) { \
            return ReturnEmpty<return_type>();  // 空操作 \
        } \
    }
```

**作用**：
- **编译期检查**：派生类是否实现了 `OnFailed`、`AfterRevived` 等可选钩子
- **默认行为**：如果没实现，提供空实现（`ReturnEmpty`）
- **类型安全**：检查参数和返回类型是否匹配

#### 2. 使用示例

```cpp
// 在基类中调用派生类的可选钩子
template <typename... Args>
int SetFailedImpl(Args&&... args) {
    // ...
    
    // 编译期决定：如果 T 实现了 OnFailed，则调用；否则调用空函数
    WRAPPER_CALL(OnFailed, static_cast<T*>(this), 
                 std::forward<Args>(args)...);
    
    // ...
}
```

展开后相当于：
```cpp
// 如果 Socket 实现了 OnFailed：
OnFailedWrapper().Call(this, error_code, error_text);
// → 编译为 this->OnFailed(error_code, error_text);

// 如果 Socket 没实现 OnFailed：
OnFailedWrapper().Call(this, error_code, error_text);
// → 编译为 ReturnEmpty<void>();  // 什么都不做
```

---

### 六、CRTP 的优势在 `VersionedRefWithId` 中的体现

#### 1. **零开销抽象**

```cpp
// 虚函数版本（假设）
class VersionedRefWithIdBase {
public:
    virtual void BeforeRecycled() = 0;  // 虚函数
    virtual ~VersionedRefWithIdBase() = default;
};

// 每个 Socket 对象需要 8 字节 vptr
// 每次调用 BeforeRecycled 需要两次间接跳转（vtable → 函数）

// CRTP 版本（实际）
template <typename T>
class VersionedRefWithId {
    void Recycle() {
        static_cast<T*>(this)->BeforeRecycled();  // 直接调用
    }
};

// 无 vptr，无额外内存
// BeforeRecycled 直接内联，单条指令
```

#### 2. **编译期类型检查**

```cpp
// 错误：派生类忘记实现必需的 OnCreated
class BadSocket : public VersionedRefWithId<BadSocket> {
    // 没有 OnCreated！
};

// 编译错误：BAIDU_CASSERT 触发
// "T::OnCreated must accept Args params and return int"
```

#### 3. **灵活的可选接口**

```cpp
// 简单派生类：只实现必需方法
class SimpleSocket : public VersionedRefWithId<SimpleSocket> {
    int OnCreated() { return 0; }
    void BeforeRecycled() {}
};

// 复杂派生类：实现所有可选钩子
class AdvancedSocket : public VersionedRefWithId<AdvancedSocket> {
    int OnCreated() { return 0; }
    void BeforeRecycled() {}
    void OnFailed(int code, const std::string& text) { /* 日志 */ }
    void AfterRevived() { /* 重置状态 */ }
    std::string OnDescription() const { return "advanced"; }
};
```

**基类自动适配**：无需修改基类代码，编译期自动检测并调用。

---

### 七、CRTP 的局限性与 brpc 的应对

| 局限性 | brpc 的解决方案 |
|:---|:---|
| 代码膨胀（每个 T 生成一份代码） | 只用于核心基础类（Socket、Controller 等），实例化次数有限 |
| 无法运行时更换实现 | 设计为静态类型，配合 `VersionedRefWithId` 的 ID 机制实现动态替换 |
| 复杂的编译错误信息 | 使用 `BAIDU_CASSERT` 提供清晰的静态断言错误 |
| 头文件必须暴露实现 | 模板实现放在 `.h` 文件，符合 brpc 作为基础库的设计 |

---

### 八、总结

```
┌─────────────────────────────────────────────────────────────┐
│                    CRTP 在 brpc 中的应用                      │
├─────────────────────────────────────────────────────────────┤
│  基类: VersionedRefWithId<T>                                │
│     ├── 管理通用逻辑：引用计数、版本控制、生命周期             │
│     └── 通过 static_cast<T*> 调用派生类钩子                  │
│                      ↓                                       │
│  派生类: Socket / Controller / ...                          │
│     ├── 继承: public VersionedRefWithId<Socket>             │
│     ├── 实现: OnCreated(), BeforeRecycled() [必需]           │
│     └── 可选: OnFailed(), AfterRevived(), OnDescription()    │
│                      ↓                                       │
│  编译期: WRAPPER_OF 宏检查可选方法，生成最优调用代码           │
│  运行时: 零开销，直接函数调用，无 vtable 间接跳转             │
└─────────────────────────────────────────────────────────────┘
```

CRTP 让 `VersionedRefWithId` 在保持**高性能（零运行时开销）**的同时，实现了**灵活的多态扩展能力**，是 brpc 能够支撑百万级并发的重要设计基础。


 
### 九、`base<T>`中是转为`T*`进行函数调用进而实现静态多态，为何不能直接`T*` （this）直接调用，实现多态？
#### 原因

1. **类型安全**：
   - `this` 的类型是 `Base<T>*`，不是 `T*`。
   - 直接调用 `this->method()` 会触发编译错误，因为 `Base<T>` 没有 `method()` 方法。

2. **编译期绑定**：
   - C++ 的多态性是在运行时实现的（通过 vtable 指针）。
   - 在编译时，`this` 的类型是 `Base<T>*`，编译器无法确定 `this` 实际指向的是 `T` 类型的对象。

#### 解决方案

- **使用 `static_cast<T*>(this)` 转换**：
  - 这是实现 CRTP 静态多态的标准方法。
  - 它告诉编译器：“我知道 `this` 指向的是 `T` 类型的对象，你可以安全地调用 `T` 类的方法。”

### 示例

```cpp
template <typename T>
class Base {
    void func() {
        // ❌ this 是 Base<T>*，不是 T*！
        this->method();  // 调用的是 Base<T>::method，不是 T::method
        
        // ✅ 必须转换，让编译器知道"实际对象是 T 类型"
        static_cast<T*>(this)->method();  // 调用 T::method
    }
};
```