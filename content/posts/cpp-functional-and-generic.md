---
title: "C++ 函数式与泛型编程全景：从函数指针到 Lambda、模板与异步"
date: 2021-07-28T10:00:00+08:00
tags: ["C++", "Lambda", "模板", "类型推导", "完美转发", "std::function", "异步"]
categories: ["技术"]
summary: "从 C 语言函数指针出发，沿 C++ 演进脉络梳理 std::function、std::bind、Lambda 表达式、std::future、类型推导、完美转发与模板编程。不是一个个孤立特性的罗列，而是一条连贯的抽象之路——为什么 C++ 需要这些东西，它们解决的是什么问题，彼此如何替代与互补。"
ShowToc: true
---

C++11 到 C++20 引入的新特性数量之多，让不少程序员感到困惑：Lambda、`std::function`、`std::bind`、`auto`、`decltype`、完美转发、变参模板、`std::future`……这些看起来各自独立的东西，为什么要学这么多？它们之间是什么关系？

这篇文章用一条线索把它们串起来：**C++ 如何逐步让"行为"和"类型"成为可组合、可传递的一等公民。** 不是特性的堆砌，而是一条连贯的抽象之路。

## 一条贯穿的主线

先看全局。C++ 程序要解决的核心问题，归结起来有两件：

1. **传递行为**——把"做什么事"作为参数传来传去（回调、策略、事件处理）
2. **抽象类型**——写一份代码适用于多种类型（容器、算法、通用逻辑）

```
传递行为的演进:
  函数指针 → 函子(functor) → std::function / Lambda → 协程

抽象类型的演进:
  宏 / void* → 模板 → 类型推导(auto/decltype) → 概念(concepts)

连接两者的桥梁:
  完美转发 + 万能引用
```

每一个新特性都不是凭空发明的，而是为了解决前一个方案留下的痛点。下面沿着这两条线逐步展开。

---

## 第一部分：传递行为

### 起点：函数指针（C 语言时代）

C 语言传递"行为"的唯一手段是函数指针：

```cpp
// 回调函数
typedef void (*Callback)(int event);

void RegisterHandler(Callback cb) {
    cb(42);
}

void MyHandler(int event) {
    std::cout << "Event: " << event << std::endl;
}

int main() {
    RegisterHandler(MyHandler);  // 传递函数指针
    return 0;
}
```

**函数指针的致命局限**：

1. **不能携带状态**——函数指针只指向代码，没有地方存储上下文数据
2. **类型严格**——签名必须精确匹配，不能用成员函数
3. **不能内联**——编译器很难通过函数指针做优化

```cpp
// 问题：我需要在回调中访问外部变量
int threshold = 10;

// 错误: 函数指针无法捕获 threshold
// void Handler(int event) { if (event > threshold) ... }  // threshold 从哪来？
```

### 进化一：函子（Functor / Function Object）

C++98 的解决方案是重载 `operator()` 的类——函子可以携带状态：

```cpp
class ThresholdHandler {
public:
    explicit ThresholdHandler(int threshold) : threshold_(threshold) {}
    void operator()(int event) const {
        if (event > threshold_) {
            std::cout << "Alert: " << event << " > " << threshold_ << std::endl;
        }
    }
private:
    int threshold_;
};

int main() {
    ThresholdHandler handler(10);
    handler(42);  // 像函数一样调用
    return 0;
}
```

函子解决了状态问题，但代价是写一个完整的类——对于简单的逻辑来说太啰嗦了。

### 进化二：`std::function`——行为的多态容器

C++11 引入的 `std::function` 是一个**类型擦除**的包装器，可以持有任何签名匹配的可调用对象：

```cpp
#include <functional>
#include <iostream>

int Add(int a, int b) { return a + b; }

struct Multiply {
    int operator()(int a, int b) const { return a * b; }
};

int main() {
    // std::function 可以持有：函数指针、函子、Lambda、bind 表达式
    std::function<int(int, int)> op;

    op = Add;                  // 普通函数
    std::cout << op(3, 4) << std::endl;   // 7

    op = Multiply();           // 函子
    std::cout << op(3, 4) << std::endl;   // 12

    op = [](int a, int b) { return a - b; };  // Lambda
    std::cout << op(3, 4) << std::endl;   // -1

    return 0;
}
```

`std::function` 的价值在于**统一接口**——不管底层是函数指针、函子还是 Lambda，只要签名匹配就能存入同一个 `std::function`：

```cpp
// 策略模式：无需继承，无需虚函数
class Sorter {
public:
    using Comparator = std::function<bool(int, int)>;
    void SetComparator(Comparator comp) { comp_ = std::move(comp); }
    void Sort(std::vector<int>& data) {
        std::sort(data.begin(), data.end(), comp_);
    }
private:
    Comparator comp_;
};
```

**`std::function` 的代价**：

| 方面 | 开销 |
|---|---|
| 内存 | 堆分配（如果可调用对象超过小缓冲区大小，通常 16-24 字节） |
| 调用 | 间接调用（虚函数 / 函数指针），不可内联 |
| 拷贝 | 深拷贝可调用对象 |

> **规则**：当你需要存储、传递、延迟调用"任意可调用对象"时用 `std::function`。如果类型在编译时确定，直接用模板参数更高效。

### 进化三：`std::bind`——历史遗留

`std::bind` 可以绑定参数到可调用对象，创建新的可调用对象：

```cpp
#include <functional>

int Multiply(int a, int b, int c) { return a * b * c; }

int main() {
    // 将第二个参数绑定为 10
    auto Double = std::bind(Multiply, std::placeholders::_1, 10, std::placeholders::_2);
    std::cout << Double(3, 4) << std::endl;  // 3 * 10 * 4 = 120
    return 0;
}
```

`std::bind` 有几个严重问题：

1. **可读性差**——`_1`、`_2` 占位符不够直观
2. **编译错误难以理解**——模板错误信息极度冗长
3. **绑定引用需要 `std::ref`**——容易忘记
4. **不能移动捕获**——Lambda 可以

```cpp
// bind 的引用陷阱
int value = 42;
// auto f = std::bind(SomeFunc, value);    // 拷贝 value
auto f = std::bind(SomeFunc, std::ref(value));  // 必须显式 std::ref
```

> **结论**：在现代 C++ 中，**Lambda 几乎全面优于 `std::bind`**。C++14 的初始化捕获和泛型 Lambda 消除了 `bind` 仅存的优势。Scott Meyers 在 *Effective Modern C++* 中明确建议：优先使用 Lambda 而非 `std::bind`。

### 进化四：Lambda 表达式——终极方案

C++11 引入 Lambda，C++14/17/20 逐步增强。Lambda 是现代 C++ 中传递行为的首选方式。

#### Lambda 的语法拆解

```cpp
auto lambda = [capture](parameters) mutable noexcept -> return_type { body };
//             ↑ 捕获    ↑ 参数列表   ↑ 可选    ↑ 可选     ↑ 可选的尾置返回类型
```

#### 捕获列表详解

```cpp
int a = 1, b = 2, c = 3;

[a, b]     (int x) { return a + b + x; };   // 值捕获 a, b
[&a, &b]   (int x) { return a + b + x; };   // 引用捕获 a, b
[=]        (int x) { return a + b + x; };   // 值捕获所有外部变量
[&]        (int x) { a += x; };             // 引用捕获所有外部变量
[=, &c]    (int x) { c += x; return a + b; }; // 默认值捕获，c 引用捕获
[&, c]     (int x) { return c + a + b + x; }; // 默认引用捕获，c 值捕获

// C++14: 初始化捕获（广义捕获）
auto p = std::make_unique<Widget>(42);
auto f = [ptr = std::move(p)]() { return ptr->GetValue(); };  // 移动捕获

// C++14: 泛型 Lambda
auto gl = [](auto x, auto y) { return x + y; };
gl(1, 2);       // int
gl(1.0, 2.0);   // double
gl(std::string("a"), std::string("b"));  // string
```

#### Lambda 的本质

Lambda 不是魔法——编译器为每个 Lambda 生成一个**唯一的函子类**：

```cpp
// 你写的 Lambda
int threshold = 10;
auto handler = [threshold](int event) {
    return event > threshold;
};

// 编译器生成的（等价代码）
class __UniqueLambdaName {
public:
    __UniqueLambdaName(int threshold) : threshold_(threshold) {}
    bool operator()(int event) const {
        return event > threshold_;
    }
private:
    int threshold_;
};

auto handler = __UniqueLambdaName(threshold);
```

这就是 Lambda 比 `std::function` 更高效的原因——编译器知道确切的类型，可以内联调用。

#### `mutable` Lambda

默认情况下 Lambda 的 `operator()` 是 `const` 的——不能修改值捕获的变量。`mutable` 取消这个限制：

```cpp
int counter = 0;
auto increment = [counter]() mutable {
    return ++counter;  // 修改捕获的副本（不影响外部 counter）
};
```

#### Lambda vs `std::function` 对比

| 特性 | Lambda | `std::function` |
|---|---|---|
| 类型 | 编译器生成的唯一类 | 类型擦除的包装器 |
| 调用开销 | 零（可内联） | 间接调用（不可内联） |
| 存储异构类型 | 不行（每个 Lambda 类型不同） | 可以（统一接口） |
| 捕获状态 | 支持（包括移动捕获 C++14） | 通过初始化间接支持 |
| 适用场景 | 回调、算法谓词、局部行为 | 存储、传递、延迟调用多种可调用对象 |

```cpp
// Lambda 的类型不同，不能放进同一个容器
auto f1 = [](int x) { return x * 2; };
auto f2 = [](int x) { return x * 3; };
// std::vector<decltype(f1)> vec; // 只能存 f1 的类型

// std::function 统一类型
std::vector<std::function<int(int)>> ops;
ops.push_back(f1);
ops.push_back(f2);
ops.push_back([](int x) { return x + 1; });  // 也可以直接放 Lambda
```

> **选择规则**：用 Lambda 写行为，用 `std::function` 存行为。Lambda 是生产者，`std::function` 是容器。

---

## 第二部分：异步计算——`std::future`

### 异步计算的动机

有些计算耗时且可以并行。C++11 提供了 `std::future` / `std::promise` / `std::async` 来表达"一个将来可用的结果"：

```cpp
#include <future>
#include <iostream>

int HeavyComputation(int x) {
    // 模拟耗时计算
    std::this_thread::sleep_for(std::chrono::seconds(1));
    return x * x;
}

int main() {
    // 启动异步任务
    std::future<int> result = std::async(std::launch::async, HeavyComputation, 42);

    // 在等待结果的同时做其他事
    std::cout << "Doing other work..." << std::endl;

    // 获取结果（如果任务还没完成，会阻塞等待）
    std::cout << "Result: " << result.get() << std::endl;  // Result: 1764
    return 0;
}
```

### `std::future` 与 Lambda

`std::async` 的参数就是可调用对象——完美匹配 Lambda：

```cpp
auto future = std::async(std::launch::async, [](int a, int b) {
    return a + b;
}, 10, 20);

std::cout << future.get() << std::endl;  // 30
```

### `std::promise`：手动设置结果

```cpp
#include <future>
#include <thread>

void Producer(std::promise<int> prom) {
    std::this_thread::sleep_for(std::chrono::seconds(1));
    prom.set_value(42);  // 设置结果
}

int main() {
    std::promise<int> prom;
    std::future<int> fut = prom.get_future();

    std::thread t(Producer, std::move(prom));
    t.detach();

    std::cout << fut.get() << std::endl;  // 42
    return 0;
}
```

### `std::future` 的局限

| 局限 | 说明 |
|---|---|
| 不可共享 | `future` 只能 `get()` 一次，之后失效 |
| 不可组合 | 不能等"多个 future 中的任意一个完成" |
| 没有延续 | 不能说"完成后执行这个回调" |

C++20 引入了 `std::jthread` 和部分改进，但完整的异步组合能力（如 `when_all`、`when_any`、`then`）仍需第三方库（如 `folly::Future`、`cpprestsdk` 的 `pplx::task`）或等待 C++23 的 `std::expected` 和 executor。

> **实践建议**：`std::async` + `std::future` 适合简单的异步场景。复杂的异步流程用线程池 + 任务队列或第三方异步框架。

---

## 第三部分：抽象类型

### 起点：宏和 `void*`（C 语言时代）

C 语言没有泛型。模拟泛型有两种方式：

```c
// 方式一：宏——文本替换，无类型安全
#define MAX(a, b) ((a) > (b) ? (a) : (b))

// 方式二：void*——丢失类型信息
void QSort(void* base, size_t nmemb, size_t size,
           int (*compar)(const void*, const void*));

int CompareInt(const void* a, const void* b) {
    return *(int*)a - *(int*)b;  // 手动强转，没有类型检查
}
```

两者都有严重问题：宏有副作用（`MAX(i++, j)`），`void*` 丢失类型安全。

### 模板：编译时多态

C++ 模板在编译时生成类型安全的代码：

```cpp
template <typename T>
T Max(T a, T b) {
    return a > b ? a : b;
}

Max(3, 4);          // 实例化为 int Max(int, int)
Max(3.14, 2.72);    // 实例化为 double Max(double, double)
// Max(3, 3.14);    // 编译错误: T 不能同时是 int 和 double
```

#### 函数模板与类模板

```cpp
// 函数模板
template <typename T>
T Clamp(T value, T lo, T hi) {
    return value < lo ? lo : (value > hi ? hi : value);
}

// 类模板
template <typename T, size_t N>
class Stack {
public:
    void Push(const T& value) {
        data_[top_++] = value;
    }
    T Pop() {
        return data_[--top_];
    }
    bool Empty() const { return top_ == 0; }
private:
    T data_[N];
    size_t top_ = 0;
};

Stack<int, 100> int_stack;
Stack<std::string, 10> str_stack;
```

#### 模板特化与偏特化

```cpp
// 通用模板
template <typename T>
struct IsPointer { static constexpr bool value = false; };

// 偏特化：匹配所有指针类型
template <typename T>
struct IsPointer<T*> { static constexpr bool value = true; };

static_assert(!IsPointer<int>::value);
static_assert(IsPointer<int*>::value);
```

#### 变参模板（Variadic Templates）

C++11 允许模板接受任意数量的参数：

```cpp
// 递归终止
void Print() {}

// 递归展开
template <typename T, typename... Args>
void Print(T first, Args... rest) {
    std::cout << first << " ";
    Print(rest...);
}

Print(1, "hello", 3.14, 'c');  // 1 hello 3.14 c
```

C++17 的**折叠表达式**让变参操作更简洁：

```cpp
template <typename... Args>
auto Sum(Args... args) {
    return (args + ...);  // 右折叠: arg1 + (arg2 + (arg3 + ...))
}

Sum(1, 2, 3, 4, 5);  // 15
```

### 类型推导：让编译器推断类型

C++11 引入了 `auto` 和 `decltype`，C++14/17/20 进一步扩展：

#### `auto`：最常用的推导

```cpp
auto x = 42;              // int
auto pi = 3.14;           // double
auto name = std::string("hello");  // std::string

// 迭代器——再也不用写 std::map<std::string, std::vector<int>>::const_iterator
std::map<std::string, std::vector<int>> data;
for (auto it = data.begin(); it != data.end(); ++it) {
    // ...
}

// 范围 for（C++11）
for (const auto& [key, value] : data) {  // C++17 结构化绑定
    // ...
}
```

#### `auto` 的推导规则

`auto` 的推导和模板参数推导规则基本一致，但有一个例外：

```cpp
auto x = 42;       // int
auto& rx = x;      // int&
auto&& ux = x;     // int&（万能引用，x 是左值，折叠为 int&）
auto&& uy = 42;    // int&&（万能引用，42 是右值）

// 大坑: auto 默认推导会丢失引用和 const
int& ref = x;
auto copy = ref;    // int（不是 int&！auto 推导为值类型）
auto& ref2 = ref;   // int&（显式加 & 保留引用）

const int cx = 42;
auto y = cx;        // int（const 被丢弃）
const auto z = cx;  // const int（显式加 const）
```

#### `decltype`：精确的类型推导

`decltype` 保留表达式的**精确类型**，包括引用和 `const`：

```cpp
int x = 42;
int& ref = x;
const int& cref = x;

decltype(x) a = x;      // int
decltype(ref) b = x;    // int&
decltype(cref) c = x;   // const int&
decltype(x + 1) d = 0;  // int（表达式结果是右值）

// decltype 和 auto 的关键区别
auto v1 = x;      // int（丢弃引用）
decltype(auto) v2 = ref;  // int&（保留精确类型）
```

#### `decltype(auto)`：完美保留返回类型

C++14 允许函数返回类型用 `auto` 推导，但 `auto` 会丢弃引用。`decltype(auto)` 解决这个问题：

```cpp
template <typename Container>
decltype(auto) GetFirst(Container& c) {
    return c[0];  // 返回类型的引用性被保留
}

std::vector<int> v = {1, 2, 3};
GetFirst(v) = 10;  // OK: 返回 int&
```

#### `auto` 在 Lambda 中的使用（C++14）

```cpp
// 泛型 Lambda：参数用 auto
auto less_than = [](auto a, auto b) { return a < b; };

less_than(1, 2);                    // int
less_than(3.14, 2.72);             // double
less_than(std::string("a"), std::string("b"));  // string
```

### 模板 + 类型推导的综合运用

```cpp
// 通用工厂函数：推导并完美转发所有参数
template <typename T, typename... Args>
auto Make(Args&&... args) {
    return T(std::forward<Args>(args)...);
}

auto w = Make<Widget>(42, "hello");
// Args 推导为实际参数类型，forward 保持值类别
```

---

## 第四部分：完美转发——连接行为与类型的桥梁

### 为什么完美转发是桥梁

当你把**行为传递**和**类型抽象**结合起来时，会遇到一个核心问题：**模板中的参数如何保持原始的值类别？**

```cpp
// 问题：make_unique 需要把参数完美地传递给构造函数
template <typename T, typename... Args>
std::unique_ptr<T> MakeUnique(Args&&... args) {
    return std::unique_ptr<T>(new T(std::forward<Args>(args)...));
    //                                ^^^^^^^^^^^^^^^^^^^^
    // 如果没有 forward：不管传入左值还是右值，args 都是左值
    // 有了 forward：左值传左值，右值传右值，类型信息不丢失
}
```

### 完美转发的原理回顾

```cpp
template <typename T>
void Wrapper(T&& arg) {           // 万能引用：绑定左值或右值
    Target(std::forward<T>(arg)); // 条件性转发
}

// 当传入左值: T = int&,  forward 返回 int& && → int&（左值引用）
// 当传入右值: T = int,   forward 返回 int&&（右值引用）
```

### 现实中的完美转发

**标准库中的 `std::make_unique` / `std::make_shared`：**

```cpp
template <typename T, typename... Args>
unique_ptr<T> make_unique(Args&&... args) {
    return unique_ptr<T>(new T(std::forward<Args>(args)...));
}
```

**容器的 `emplace_back`：**

```cpp
template <typename... Args>
void vector<T>::emplace_back(Args&&... args) {
    // 在原地构造元素，避免临时对象
    // 完美转发确保移动语义不被丢失
}
```

**`std::function` 的构造：**

```cpp
template <typename F>
function(F f);  // F 可以是 Lambda、函子、函数指针
// 内部用完美转发将 f 移动到存储中
```

---

## 第五部分：组合运用

### 组合一：事件系统（Lambda + `std::function` + 完美转发）

```cpp
#include <functional>
#include <vector>
#include <string>
#include <iostream>

class EventBus {
public:
    using Handler = std::function<void(const std::string&)>;

    void Subscribe(Handler handler) {
        handlers_.push_back(std::move(handler));
    }

    template <typename... Args>
    void Publish(Args&&... args) {
        std::string event;
        ((event += std::forward<Args>(args)), ...);  // C++17 折叠表达式
        for (const auto& h : handlers_) {
            h(event);
        }
    }
private:
    std::vector<Handler> handlers_;
};

int main() {
    EventBus bus;
    int count = 0;

    bus.Subscribe([&count](const std::string& event) {
        std::cout << "Handler 1: " << event << std::endl;
        ++count;
    });

    bus.Subscribe([](const std::string& event) {
        std::cout << "Handler 2: " << event.size() << " chars" << std::endl;
    });

    bus.Publish("Hello", " ", "World");
    // Handler 1: Hello World
    // Handler 2: 11 chars
    std::cout << "Total events: " << count << std::endl;
    return 0;
}
```

### 组合二：通用缓存（模板 + Lambda + `std::future`）

```cpp
#include <future>
#include <unordered_map>
#include <mutex>
#include <memory>
#include <functional>

template <typename Key, typename Value>
class AsyncCache {
public:
    using Loader = std::function<Value(const Key&)>;

    explicit AsyncCache(Loader loader) : loader_(std::move(loader)) {}

    std::shared_future<Value> Get(const Key& key) {
        std::lock_guard<std::mutex> lock(mutex_);
        auto it = cache_.find(key);
        if (it != cache_.end()) {
            return it->second;  // 返回已有的 future
        }
        // 异步加载
        std::shared_future<Value> fut = std::async(std::launch::async, [this, key]() {
            return loader_(key);
        }).share();
        cache_[key] = fut;
        return fut;
    }
private:
    Loader loader_;
    std::unordered_map<Key, std::shared_future<Value>> cache_;
    std::mutex mutex_;
};

// 使用
int main() {
    AsyncCache<std::string, std::string> cache([](const std::string& url) {
        // 模拟网络请求
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        return "Response from " + url;
    });

    auto f1 = cache.Get("/api/users");
    auto f2 = cache.Get("/api/users");  // 命中缓存，返回同一个 future
    std::cout << f1.get() << std::endl;
    return 0;
}
```

### 组合三：类型安全的信号槽（模板 + 变参 + 完美转发）

```cpp
#include <functional>
#include <vector>
#include <utility>

template <typename... Args>
class Signal {
public:
    using SlotType = std::function<void(Args...)>;
    using SlotId = size_t;

    SlotId Connect(SlotType slot) {
        slots_.push_back(std::move(slot));
        return slots_.size() - 1;
    }

    void Emit(Args... args) const {
        for (const auto& slot : slots_) {
            slot(args...);
        }
    }

    void Disconnect(SlotId id) {
        if (id < slots_.size()) {
            slots_[id] = nullptr;
        }
    }
private:
    std::vector<SlotType> slots_;
};

int main() {
    Signal<std::string, int> onMessage;

    auto id = onMessage.Connect([](const std::string& msg, int code) {
        std::cout << "[" << code << "] " << msg << std::endl;
    });

    onMessage.Emit("Connected", 200);     // [200] Connected
    onMessage.Emit("Error", 500);         // [500] Error
    onMessage.Disconnect(id);
    onMessage.Emit("Nobody hears", 404);  // 无输出
    return 0;
}
```

---

## 全景对比：何时用什么

### 行为传递工具选择

| 工具 | 类型安全 | 携带状态 | 性能 | 适用场景 |
|---|---|---|---|---|
| 函数指针 | ✅（签名匹配） | ❌ | 最快 | C API 回调、简单函数 |
| 函子 | ✅ | ✅ | 快（可内联） | 需要状态的复杂逻辑 |
| `std::function` | ✅ | ✅ | 中（间接调用） | 存储多种可调用对象 |
| `std::bind` | ✅ | ✅ | 中 | **不推荐**，用 Lambda 替代 |
| Lambda | ✅ | ✅（捕获） | 最快（可内联） | **首选**：回调和局部行为 |

### 类型抽象工具选择

| 工具 | 编译时检查 | 灵活性 | 复杂度 | 适用场景 |
|---|---|---|---|---|
| 模板 | ✅ | 高 | 高 | 通用算法、容器 |
| `auto` | ✅ | 中 | 低 | 局部变量、迭代器、Lambda |
| `decltype` | ✅ | 高 | 中 | 精确保留类型、泛型编程 |
| `std::function` | ✅ | 中 | 低 | 运行时多态的"行为类型" |
| 概念（C++20） | ✅ | 高 | 中 | 约束模板参数 |

### 异步工具选择

| 工具 | 适用场景 |
|---|---|
| `std::async` | 简单的一次性异步任务 |
| `std::future` | 获取异步结果 |
| `std::promise` | 手动设置异步结果 |
| 线程池 + 任务队列 | 高并发、任务复用 |
| 第三方异步框架 | 复杂异步流程（依赖、组合、取消） |

---

## 总结：设计思想

这些特性不是为了增加复杂度而发明的。每一步都解决了一个真实的问题：

```
问题 1: 函数指针不能携带状态
  → 函子（operator()）解决状态问题
  → 但写函子太啰嗦
    → Lambda 提供简洁语法

问题 2: 多种可调用对象类型不统一
  → std::function 类型擦除，统一接口

问题 3: C 的泛型（宏 / void*）不安全
  → 模板提供编译时类型安全的多态

问题 4: 模板中类型太长，手写困难
  → auto / decltype 让编译器推导类型

问题 5: 模板中传递参数丢失值类别
  → 完美转发 + 万能引用保持左值/右值信息

问题 6: 阻塞式计算浪费资源
  → std::future / std::async 表达异步计算
```

**贯穿的思想**：让编译器做更多事，让程序员写更少代码，同时保持零开销抽象的承诺。

几条实践规则：

1. **传递行为用 Lambda**——简洁、高效、可内联。
2. **存储行为用 `std::function`**——类型擦除，统一接口。
3. **不要用 `std::bind`**——Lambda 做得更好。
4. **让编译器推导类型**——用 `auto` 和 `decltype(auto)` 减少冗余。
5. **模板参数转发用 `std::forward`**——保持值类别，别手动推演。
6. **简单异步用 `std::async`**——复杂场景上线程池或异步框架。
7. **写泛型代码时优先用模板 + 概念**——运行时多态用虚函数，编译时多态用模板。
