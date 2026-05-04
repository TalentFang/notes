以下是 Go 语言的核心语法速览，突出其**简洁、显式、并发原生**的设计哲学：

---

## 1. 程序结构

```go
package main  // 包声明，main 为可执行程序入口

import (
    "fmt"
    "os"
)

func main() {
    fmt.Println("Hello, Go!")
}
```

---

## 2. 变量与类型

### 声明方式
```go
// 完整声明
var name string = "Go"

// 类型推断
var age = 25

// 短变量声明（函数内最常用）
count := 10

// 常量
const Pi = 3.14

// 多变量
a, b, c := 1, 2, 3

// 匿名变量（忽略返回值）
val, _ := someFunction()
```

### 基础类型
```go
bool                    // 布尔
string                  // 字符串（不可变，UTF-8）
int, int8, int16, int32, int64
uint, uint8, uint16, uint32, uint64, uintptr
float32, float64
complex64, complex128   // 复数
byte // uint8 别名    rune // int32 别名，代表 Unicode 码点
```

---

## 3. 复合类型

### 数组与切片（Slice）
```go
// 数组（定长，值类型）
arr := [5]int{1, 2, 3, 4, 5}

// 切片（动态数组，引用类型，更常用）
slice := []int{1, 2, 3}
slice = append(slice, 4)           // 追加元素
sub := slice[1:3]                  // 切片操作 [2,3]

// make 创建
s := make([]int, 5, 10)            // 长度5，容量10
```

### Map
```go
m := make(map[string]int)
m["age"] = 25

// 遍历
for k, v := range m {
    fmt.Printf("%s: %d\n", k, v)
}

// 检查键存在
val, ok := m["name"]  // ok 为 false 表示不存在
```

### 结构体（无类，只有结构体）
```go
type Person struct {
    Name string
    Age  int
}

// 创建
p := Person{Name: "Alice", Age: 30}
p.Age = 31

// 指针（自动解引用，无需 ->）
ptr := &p
fmt.Println(ptr.Name)  // 等价于 (*ptr).Name
```

---

## 4. 控制流

### if（可带简短语句）
```go
if x := 10; x > 5 {  // x 只在 if 块内有效
    fmt.Println(x)
}
```

### for（Go 唯一循环关键字）
```go
// 标准 for
for i := 0; i < 10; i++ {}

// 条件循环（类似 while）
for i < 100 {}

// 无限循环
for {}

// 遍历（range）
for i, v := range slice {}      // 索引, 值
for k := range map {}           // 只取键
```

### switch（默认 break，无需写）
```go
switch os := runtime.GOOS; os {
case "darwin":
    fmt.Println("macOS")
case "linux":
    fmt.Println("Linux")
default:
    fmt.Println("Other")
}

// 无条件 switch（替代长 if-else）
switch {
case x < 0:
    fmt.Println("negative")
case x > 0:
    fmt.Println("positive")
}
```

### defer（延迟执行，栈顺序）
```go
func readFile() {
    f, err := os.Open("file.txt")
    if err != nil {
        return
    }
    defer f.Close()  // 函数返回前执行，先进后出
    
    // 处理文件...
}
```

---

## 5. 函数（一等公民）

### 多返回值（错误处理核心）
```go
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

// 调用
result, err := divide(4, 2)
if err != nil {
    log.Fatal(err)
}
```

### 变长参数
```go
func sum(nums ...int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}

sum(1, 2, 3)
sum(slice...)  // 展开切片
```

### 闭包
```go
func counter() func() int {
    count := 0
    return func() int {
        count++
        return count
    }
}

next := counter()
fmt.Println(next()) // 1
fmt.Println(next()) // 2
```

---

## 6. 方法（绑定到类型的函数）

```go
type Rectangle struct {
    Width, Height float64
}

// 值接收者（拷贝）
func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

// 指针接收者（修改原值，避免拷贝大对象）
func (r *Rectangle) Scale(factor float64) {
    r.Width *= factor
    r.Height *= factor
}
```

---

## 7. 接口（隐式实现）

**无需显式声明 `implements`，只要实现方法集就自动满足接口**：

```go
type Shape interface {
    Area() float64
}

// Rectangle 自动成为 Shape，无需声明
func printArea(s Shape) {
    fmt.Println(s.Area())
}

// 空接口（可存任何类型）
var anything interface{} = "hello"
```

---

## 8. 并发核心（Goroutine + Channel）

### Goroutine（轻量级线程）
```go
func say(s string) {
    for i := 0; i < 5; i++ {
        time.Sleep(100 * time.Millisecond)
        fmt.Println(s)
    }
}

go say("world")  // 新建协程执行
say("hello")     // 主协程执行
```

### Channel（协程通信）
```go
ch := make(chan int, 2)  // 缓冲通道（容量2）

// 发送
ch <- 100

// 接收
v := <-ch

// 遍历（需关闭）
close(ch)
for v := range ch {}

// select 多路复用
select {
case v1 := <-ch1:
    fmt.Println(v1)
case v2 := <-ch2:
    fmt.Println(v2)
case <-time.After(1 * time.Second):
    fmt.Println("timeout")
}
```

---

## 9. 错误处理（显式哲学）

**Go 没有 try-catch，错误作为返回值处理**：

```go
if err != nil {
    // 处理错误
    return err  // 或 log.Fatal(err), panic(err)
}

// 错误链
if err := doSomething(); err != nil {
    return fmt.Errorf("context: %w", err)  // Go 1.13+ 错误包装
}
```

---

## 10. 泛型（Go 1.18+）

```go
// 类型参数
func Min[T constraints.Ordered](a, b T) T {
    if a < b {
        return a
    }
    return b
}

// 使用
m1 := Min[int](1, 2)
m2 := Min("a", "b")  // 类型推断
```

---

## Go 语法设计哲学

| 特点 | 说明 |
|------|------|
| **极简** | 25 个关键字，无类、无继承、无泛型（早期）、无异常 |
| **显式优于隐式** | 错误必须处理，导入包必须使用 |
| **组合优于继承** | 通过嵌入结构体组合功能 |
| **并发原生** | `go` 关键字和 `chan` 是语法级支持 |
| **正交性** | 特性之间不重叠，如接口与实现完全解耦 |

掌握这些核心语法，即可写出地道的 Go 代码。