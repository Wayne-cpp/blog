---
title: "C++ 中 static、const、extern 关键字的作用与区别"
date: 2021-07-10T14:00:00+08:00
tags: ["C++", "关键字", "编译链接"]
categories: ["技术"]
summary: "深入解析 C++ 中 static、const、extern 三个关键字的语义：它们在不同语境（变量、函数、类成员、翻译单元）下的行为差异、常见陷阱，以及如何组合使用。"
ShowToc: true
---

C++ 有几十个关键字，但日常工程中打交道最多的，莫过于 `static`、`const` 和 `extern`。它们都和"可见性"与"生命周期"有关，却在不同语境下表现出截然不同的语义。这篇文章把它们放在一起，系统地梳理一遍。

## 三个关键字，一个核心问题

在正式展开之前，先明确一个前提：这三个关键字本质上都在回答同一个问题——**一个名字在什么范围内可见、它的存储周期是什么、它是否可以被修改。**

| 关键字 | 核心语义 | 一句话概括 |
|---|---|---|
| `static` | 控制存储周期与链接性 | 让东西"活得更久"或"藏得更深" |
| `const` | 控制可修改性 | 告诉编译器"不许改" |
| `extern` | 控制链接性 | 告诉链接器"它在别处定义" |

接下来逐个拆解。

---

## static：一关键字，三种语义

`static` 是 C++ 中**重载最严重**的关键字之一。根据使用场景不同，它有三种截然不同的含义。

### 1. 局部静态变量——延长生命周期

在函数内部声明的 `static` 变量，其生命周期从首次执行到声明处开始，到程序结束才销毁。但它的作用域仍然是局部的。

```cpp
void Counter() {
    static int count = 0;  // 只初始化一次
    ++count;
    std::cout << count << std::endl;
}

int main() {
    Counter();  // 1
    Counter();  // 2
    Counter();  // 3
}
```

关键点：

- **初始化只发生一次**，后续调用跳过初始化语句。
- **线程安全的**（C++11 起）。如果多个线程同时首次调用，编译器保证初始化只发生一次（[magic statics](https://en.cppreference.com/w/cpp/language/storage_duration#Static_local_variables)）。
- 程序结束时按逆序销毁（与全局变量相同的销毁时机）。

### 2. 全局/文件静态——内部链接

在全局作用域（函数和类之外）声明的 `static` 变量或函数，具有**内部链接性**（internal linkage）。这意味着该名字只在当前翻译单元（`.cpp` 文件）可见。

```cpp
// utils.cpp
static int helper_data = 42;       // 只在 utils.cpp 中可见

static void HelperFunc() {          // 只在 utils.cpp 中可见
    std::cout << helper_data << std::endl;
}
```

另一个 `.cpp` 文件即使写了 `extern int helper_data;` 也无法链接到它。

> **现代 C++ 的建议**：在 C++17 及以后，优先使用**未命名命名空间**替代文件级 `static`，语义更清晰：
>
> ```cpp
> namespace {
>     int helper_data = 42;
>     void HelperFunc() { /* ... */ }
> }
> ```

### 3. 类静态成员——属于类，不属于对象

在类中使用 `static` 声明的成员，属于**类本身**而非类的某个实例。

```cpp
class Config {
public:
    static int kMaxConnections;  // 声明
    static void Reset() {        // 静态成员函数可以直接定义
        kMaxConnections = 0;
    }
};

// 必须在类外定义（C++17 可用 inline 变量替代）
int Config::kMaxConnections = 100;
```

关键点：

- **静态数据成员**需要在类外定义（除非是 `inline static` 或 `constexpr`）。
- **静态成员函数**没有 `this` 指针，不能访问非静态成员。
- 可以通过 `ClassName::member` 直接访问，无需创建对象。

---

## const：不只是"常量"

`const` 的核心语义是"只读"，但它在不同位置的含义有微妙差别。

### 1. const 变量

```cpp
const int kBufferSize = 1024;
kBufferSize = 2048;  // 编译错误：不能修改 const 变量
```

在 C++ 中，`const` 变量**默认具有内部链接性**（这一点和 C 不同！）。也就是说：

```cpp
// a.cpp
const int kValue = 42;  // 内部链接，等同于 static const int kValue = 42;

// b.cpp
extern const int kValue;  // 链接错误！找不到定义
```

如果你需要跨翻译单元共享一个 `const` 变量，必须用 `extern` 显式声明外部链接：

```cpp
// a.cpp
extern const int kValue = 42;  // 显式外部链接

// b.cpp
extern const int kValue;       // 声明，链接到 a.cpp 的定义
```

### 2. const 与指针

`const` 和指针结合时，位置决定语义——这是面试高频考点：

```cpp
int value = 42;

// 指向 const int 的指针：不能通过指针修改指向的值
const int* ptr1 = &value;
// *ptr1 = 10;  // 错误
ptr1 = nullptr;  // OK，指针本身可以改

// int 类型的 const 指针：指针本身不能改
int* const ptr2 = &value;
*ptr2 = 10;      // OK，可以改指向的值
// ptr2 = nullptr;  // 错误

// 指向 const int 的 const 指针：都不行
const int* const ptr3 = &value;
```

一个阅读技巧：**从右往左读**。

- `const int* p` → `p` is a pointer to `int` that is `const`
- `int* const p` → `p` is a `const` pointer to `int`

### 3. const 与引用

`const` 引用可以绑定到右值，这是移动语义和临时对象生命周期延长的基石：

```cpp
const std::string& ref = std::string("hello");  // OK，临时对象生命周期延长到 ref 的作用域结束
// std::string& non_const_ref = std::string("hello");  // 错误：非 const 引用不能绑定右值
```

### 4. const 成员函数

在类成员函数后面加 `const`，表示该函数不会修改对象的任何非静态成员：

```cpp
class StringView {
public:
    char At(int index) const {  // const 成员函数
        return data_[index];     // 只读访问，OK
    }

    void Set(int index, char ch) {  // 非 const 成员函数
        data_[index] = ch;          // 修改成员，OK
    }

private:
    char* data_;
    int size_;
};

const StringView csv;
csv.At(0);     // OK，const 对象可以调用 const 成员函数
// csv.Set(0, 'a');  // 错误，const 对象不能调用非 const 成员函数
```

> **实践建议**：能加 `const` 就加。const 正确性是 C++ 最重要的类型安全机制之一。如果你不把该加 `const` 的地方加上，下游使用你接口的人会非常痛苦。

### 5. constexpr vs const

C++11 引入的 `constexpr` 比 `const` 更严格：`constexpr` 保证值在**编译期**确定，而 `const` 只保证"运行期不可修改"。

```cpp
constexpr int kSize = 10;       // 编译期常量
const int kSize2 = ReadInput(); // 运行期初始化，之后不可修改
```

优先使用 `constexpr` 来定义真正的编译期常量。

---

## extern：跨翻译单元的桥梁

`extern` 的核心作用是**声明一个在其他翻译单元中定义的实体**，实现跨文件共享。

### 1. extern 变量

```cpp
// config.cpp
int g_max_threads = 16;  // 定义（分配存储）

// main.cpp
extern int g_max_threads;  // 声明（不分配存储，引用 config.cpp 中的定义）

int main() {
    std::cout << g_max_threads << std::endl;  // 16
}
```

**定义 vs 声明**是理解 `extern` 的关键：

- **定义**（definition）：分配存储空间，只能出现在一个翻译单元中。
- **声明**（declaration）：告诉编译器"这个名字存在，类型是什么"，可以出现在多个翻译单元中。
- `extern` 声明如果带初始化，就变成了定义：`extern int x = 42;` 等同于 `int x = 42;`。

### 2. extern "C"——与 C 代码互操作

`extern` 的另一个重要用途是与 C 代码交互。C++ 编译器会对函数名做 name mangling（名称修饰），导致 C++ 编译的符号和 C 编译的符号不兼容。`extern "C"` 告诉编译器按 C 的规则生成符号名：

```cpp
// C++ 代码调用 C 库
extern "C" {
    #include "legacy_c_library.h"  // C 头文件中的函数声明
}

// 或手动声明
extern "C" void C_ApiInit();    // 按 C 规则链接
extern "C" void C_ApiCleanup();
```

这在以下场景很常见：

- 封装 C 语言第三方库（如 SQLite、Lua）。
- 在 C++ 项目中复用遗留 C 代码。
- 编写同时供 C 和 C++ 使用的头文件。

### 3. extern 模板（C++11）

显式实例化声明，告诉编译器"别在这个翻译单元实例化这个模板，其他地方会实例化"：

```cpp
// header.h
template <typename T>
class MyContainer { /* ... */ };

extern template class MyContainer<int>;   // 声明：别在这实例化
```

```cpp
// source.cpp
template class MyContainer<int>;  // 显式实例化定义：只在这实例化一次
```

这可以显著减少大型项目中模板的编译时间。

---

## 三者的组合与对比

单独看每个关键字不难，真正的难点在于它们的**组合使用**。以下是几种常见的组合：

### static const / const static

文件作用域中，`static const` 定义一个**当前翻译单元私有的编译期常量**：

```cpp
// logger.cpp
static const char* kLogPrefix = "[INFO] ";  // 内部链接 + 不可修改
```

在 C++ 中，`const` 全局变量已经默认是内部链接的，所以这里的 `static` 是冗余的。但写上 `static` 可以**明确表达意图**——"这个常量是本文件私有的"。

### extern const

如前所述，`const` 默认内部链接，加 `extern` 可以使其外部链接：

```cpp
// shared.h
extern const int kSharedValue;  // 声明

// shared.cpp
extern const int kSharedValue = 42;  // 定义（extern + 初始化 = 定义）
```

### static 与单例模式

`static` 局部变量天然适合实现懒加载的单例：

```cpp
class Singleton {
public:
    static Singleton& Instance() {
        static Singleton instance;  // 线程安全的懒加载
        return instance;
    }

    Singleton(const Singleton&) = delete;
    Singleton& operator=(const Singleton&) = delete;

private:
    Singleton() = default;
};
```

这是 Scott Meyers 提出的单例模式实现，C++11 起保证线程安全。

---

## 常见陷阱

### 陷阱 1：const 全局变量的链接性

```cpp
// header.h
const int kValue = 42;  // 每个包含此头文件的 .cpp 都有自己的一份副本！
```

如果多个翻译单元包含了同一个定义了 `const` 变量的头文件，每个 `.cpp` 都会得到一个独立的副本。对于简单类型（`int`、`double`）这通常没问题，但如果取地址或用引用绑定，不同翻译单元可能得到不同的地址。

**解决方案**：使用 `inline` 变量（C++17）或 `constexpr`：

```cpp
// header.h (C++17)
inline const int kValue = 42;     // 所有翻译单元共享同一个实体

// 或者
constexpr int kValue = 42;        // 编译期常量，不需要链接
```

### 陷阱 2：static 初始化顺序问题

不同翻译单元中的全局/静态对象的初始化顺序是**未定义的**（static initialization order fiasco）：

```cpp
// a.cpp
static std::string g_name = "hello";  // g_name 可能还没初始化

// b.cpp
static int g_length = g_name.size();  // 未定义行为！
```

**解决方案**：使用函数内的 `static` 局部变量（Construct on First Use Idiom）：

```cpp
// b.cpp
static int GetLength() {
    static int length = GetName().size();  // 首次调用时才初始化
    return length;
}
```

### 陷阱 3：类静态成员的 ODR 违规

```cpp
// header.h
class Foo {
    static int count;  // 声明
};
// 忘记在 .cpp 中定义！
```

如果只在头文件中声明静态成员但不定义，链接时会报 `undefined reference` 错误。C++17 可以用 `inline` 解决：

```cpp
class Foo {
    inline static int count = 0;  // 声明 + 定义，C++17
};
```

---

## 速查表

| 场景 | `static` | `const` | `extern` |
|---|---|---|---|
| 局部变量 | 延长生命周期至程序结束 | 变量不可修改 | ❌ 不适用 |
| 全局变量 | 内部链接（仅当前 `.cpp` 可见） | 默认内部链接 | 外部链接（跨 `.cpp` 共享） |
| 函数 | 内部链接（仅当前 `.cpp` 可见） | 返回值不可修改（仅对指针/引用有意义） | 外部链接（默认行为） |
| 类成员变量 | 属于类而非实例 | 实例不可修改该成员 | ❌ 不适用 |
| 类成员函数 | 无 `this` 指针 | 承诺不修改对象状态 | ❌ 不适用 |
| 模板 | ❌ 不适用 | ❌ 不适用 | `extern template` 避免重复实例化 |

---

## 面试中如何回答

这三个关键字是 C++ 面试的"保留节目"。下面给出几种典型提问的回答思路。

### Q：说说 static 的作用

> **参考回答：**
>
> `static` 在 C++ 中有三种含义，取决于使用场景：
>
> 1. **局部静态变量**——在函数内部使用，变量生命周期延长到整个程序运行期，但作用域仍限于函数内部。初始化只执行一次，C++11 起保证线程安全。
> 2. **全局/文件级静态**——在全局作用域使用，变量或函数具有内部链接性，仅在当前翻译单元可见。现代 C++ 推荐用匿名命名空间替代。
> 3. **类静态成员**——属于类而非实例，静态成员函数没有 `this` 指针。
>
> 三种场景的核心思想是一致的：**控制可见范围，延长生命周期**。

### Q：const 和 constexpr 有什么区别

> **参考回答：**
>
> `const` 告诉编译器"这个值初始化后不可修改"，但初始化可以在运行期完成。`constexpr` 更严格，要求值必须在**编译期**就能确定。
>
> ```cpp
> const int a = std::rand();     // OK，运行期初始化
> constexpr int b = std::rand(); // 错误，编译期无法求值
> constexpr int c = 42;          // OK
> ```
>
> 另外注意：C++ 中 `const` 全局变量默认是**内部链接**的，这和 C 语言不同。

### Q：extern 有什么用

> **参考回答：**
>
> `extern` 主要有两种用途：
>
> 1. **跨翻译单元共享变量**——在一个 `.cpp` 中定义全局变量，在另一个 `.cpp` 中用 `extern` 声明来引用它。本质上是告诉编译器"这个符号在别处定义，链接时再解析"。
> 2. **`extern "C"`**——禁用 C++ 的 name mangling，让 C++ 代码能链接 C 编译的库。
>
> 另外 C++11 还有 `extern template`，可以阻止模板在当前翻译单元隐式实例化，减少编译时间。

### Q：const 全局变量能被其他文件访问吗

> **参考回答：**
>
> 默认不能。C++ 中 `const` 全局变量具有**内部链接性**，等同于加了 `static`。每个包含该头文件的翻译单元会得到独立副本。
>
> 如果需要跨文件共享，有两种方式：
>
> ```cpp
> // 方式一：extern 显式声明外部链接
> // def.cpp
> extern const int kValue = 42;
> // user.cpp
> extern const int kValue;
>
> // 方式二：C++17 inline 变量
> // header.h
> inline const int kValue = 42;
> ```

### Q：手写一个线程安全的单例

> **参考回答：**
>
> 利用 C++11 的 magic static 特性（函数内 `static` 局部变量的初始化是线程安全的），可以写出最简洁的单例：
>
> ```cpp
> class Singleton {
> public:
>     static Singleton& Instance() {
>         static Singleton inst;
>         return inst;
>     }
>     Singleton(const Singleton&) = delete;
>     Singleton& operator=(const Singleton&) = delete;
> private:
>     Singleton() = default;
> };
> ```
>
> 无需加锁、无需双重检查，编译器保证线程安全和延迟初始化。

### Q：const int*、int* const、const int* const 有什么区别

> **参考回答：**
>
> 从右往左读：
>
> | 声明 | 含义 |
> |---|---|
> | `const int* p` | 指针指向的值不可改，指针本身可改 |
> | `int* const p` | 指针本身不可改，指向的值可改 |
> | `const int* const p` | 都不可改 |
>
> 判断技巧：`const` 修饰它**左边**的东西。如果左边没有东西（最左边），就修饰右边。

### 回答策略

面试中回答这类问题，建议遵循 **"总-分-联系"** 结构：

1. **总**：一句话概括关键字的本质（如 `static` 控制生命周期和链接性）。
2. **分**：按场景逐一说明（局部、全局、类成员），每个场景配一个简洁的代码例子。
3. **联系**：点出与其他关键字的关系（如 `const` 默认内部链接、`extern` 显式外部链接），或者实际工程中的使用模式。

不要背定义，要展示你理解**为什么**这样设计——面试官考察的不是记忆力，而是对语言机制的理解深度。

---

## 总结

- **`static`** 的三种语义（延长生命周期、内部链接、类级别共享）看似不相关，但核心思想一致：**限制作用域，延长存在时间**。
- **`const`** 不仅仅是"不可修改"——它影响链接性、函数重载决议、模板推导，是类型系统的重要组成。
- **`extern`** 是跨翻译单元通信的基础，理解定义与声明的区别是正确使用它的前提。

三者经常组合使用（`extern const`、`static const`、`inline static const`），理解各自在链接性、存储周期和可修改性维度上的作用，才能在工程中做出正确的选择。

最后给出一条实践准则：**能加 `const` 就加 `const`，能用命名空间作用域就不要用全局 `static`，能用 `constexpr` 就不要用 `const`。** 这三条规则能帮你避开大部分坑。
