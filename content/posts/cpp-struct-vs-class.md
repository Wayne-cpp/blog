---
title: "C++ struct 与 class：从历史渊源到工程抉择"
date: 2021-07-13T10:00:00+08:00
tags: ["C++", "struct", "class", "面向对象"]
categories: ["技术"]
summary: "C++ 同时拥有 struct 和 class 两个关键字，它们在语言层面几乎等价，却在工程实践中承载着不同的语义约定。本文从 C 语言的历史遗留出发，讲清楚两者的技术差异、现代 C++ 的聚合体初始化演变，以及 Google C++ Style Guide 与 C++ Core Guidelines 的使用建议。"
ShowToc: true
---

每一个 C++ 程序员都见过 `struct` 和 `class`，也都听过"它们几乎一样"这种说法。问题在于——"几乎"二字里藏着很多东西：默认访问权限的差异、与 C 语言兼容性的考量、聚合体初始化的微妙演变，以及社区里几十年沉淀下来的语义约定。这篇文章把它们一次性讲透。

## 先回答一个经典面试题：区别是什么

### 技术层面——两个差异，仅此而已

在整个 C++ 语言规范中，`struct` 和 `class` 的技术差异**只有两个**：

| 差异点 | `struct` | `class` |
|---|---|---|
| 默认成员访问权限 | `public` | `private` |
| 默认继承访问权限 | `public` | `private` |

仅此而已。下面的代码完全合法：

```cpp
struct S {
    int x;  // 默认 public
};

class C {
    int x;  // 默认 private
};
```

但如果你显式写出访问修饰符，两者产生的代码**完全一致**：

```cpp
struct S {
private:
    int x;
public:
    int GetX() const { return x; }
};

class C {
private:   // 写不写都一样，class 默认就是 private
    int x;
public:
    int GetX() const { return x; }
};

// S 和 C 在内存布局、行为、编译产物上没有任何区别
static_assert(sizeof(S) == sizeof(C));
```

同样，在继承时：

```cpp
struct DerivedS : Base {};    // 等价于 struct DerivedS : public Base {};
class DerivedC : Base {};     // 等价于 class DerivedC : private Base {};
```

除了这两处默认值不同，`struct` 和 `class` 在 C++ 中**完全等价**——它们定义的都是"类类型"（class type），都可以有构造函数、析构函数、虚函数、继承、模板等一切面向对象特性。

> **面试速答**：`struct` 默认 `public`，`class` 默认 `private`。就这一个区别。

---

## 历史渊源：为什么 C++ 需要两个关键字

要理解这个设计，得回到 C++ 的诞生背景。

### C 语言的 struct：没有行为的数据包

C 语言中的 `struct` 是纯粹的**数据聚合体**——只能存放数据成员，不能有成员函数，不能有访问控制：

```c
/* C 语言 */
struct Point {
    double x;
    double y;
};

/* 操作必须通过外部函数 */
double Distance(struct Point a, struct Point b) {
    return sqrt((a.x - b.x) * (a.x - b.x) + (a.y - b.y) * (a.y - b.y));
}
```

数据和对数据的操作是分离的，没有"封装"的概念。

### C++ 的设计抉择：扩展而非替换

Bjarne Stroustrup 在设计 C++ 时的核心原则之一是**尽可能兼容 C**。他面临的问题是：

1. C 程序员已经在大量使用 `struct`
2. 要引入面向对象的封装、继承、多态
3. 不能破坏已有的 C 代码

他的方案是**扩展 `struct`**，让它支持成员函数、访问控制、继承等特性。这样所有 C 代码中的 `struct` 定义在 C++ 中仍然合法——零成本兼容。

但 Stroustrup 也观察到，C 程序员对 `struct` 的心理预期是"一组公开的数据"，而面向对象编程中更常见的模式是"私有数据 + 公开接口"。如果让 `struct` 的默认访问权限变成 `private`，就会破坏兼容性，也会违背 C 程序员的直觉。

于是他引入了 `class` 关键字：

- **`struct`** 保持 `public` 默认——尊重 C 的传统，"数据包"的自然延续
- **`class`** 使用 `private` 默认——面向对象编程中"封装"的默认选择

这不是技术上的需要，而是**语义设计上的考量**——用两个关键字区分两种不同的设计意图。

> Stroustrup 本人在 *The Design and Evolution of C++* 中解释过：引入 `class` 关键字的主要原因是语法上需要一种方式来引入"默认私有"的用户定义类型，同时保持与 C `struct` 的完全兼容。

---

## 现代 C++ 中的细微差异

### 聚合体（Aggregate）与初始化

这里有一个容易被忽视的细节：`struct` 和 `class` 在**聚合体初始化**行为上可能不同——但原因仍然是默认访问权限，而非关键字本身。

**C++11/14 的规则**：聚合体不能有 `private` 或 `protected` 成员。由于 `class` 默认 `private`，如果你不写 `public:`，它就不是聚合体，不能用花括号初始化：

```cpp
struct Point {
    double x, y;  // 默认 public → 是聚合体
};

class Color {
    int r, g, b;  // 默认 private → 不是聚合体
};

Point p = {1.0, 2.0};    // OK: 聚合体初始化
// Color c = {255, 0, 0}; // 错误: Color 不是聚合体

// 但如果你显式写出 public，两者都可以：
class Color2 {
public:
    int r, g, b;  // 显式 public → 是聚合体
};

Color2 c2 = {255, 0, 0};  // OK
```

**C++17 的变化**：聚合体的定义被放宽，允许更多情况，但"不能有 `private`/`protected` 成员"这一条仍然保留。

**C++20 的变化**：引入了**指定初始化器**（designated initializers），进一步增强了聚合体初始化的表达力：

```cpp
struct Config {
    int max_connections = 100;
    int timeout_ms = 5000;
    bool verbose = false;
};

Config cfg = {
    .max_connections = 200,
    .verbose = true
};
```

注意：这些行为差异的本质是**默认访问权限不同**导致的是否满足"聚合体"条件，而不是 `struct` 和 `class` 关键字本身有什么特殊处理。

### POD 与平凡类型

C++11 之前，`struct` 和 `class` 在 POD（Plain Old Data）概念上有间接差异——因为 POD 类型要求所有非静态成员都是 `public`。同样，这是因为默认访问权限。

C++11 起，POD 的概念被拆分为**平凡类型**（trivial type）和**标准布局类型**（standard-layout type），判断标准不再直接和 `struct`/`class` 关键字绑定，但"所有非静态数据成员具有相同访问权限"仍然是标准布局类型的条件之一：

```cpp
struct MixedAccess {
public:
    int a;
private:
    int b;  // 不同访问权限 → 不是标准布局类型
};
```

---

## 工程实践：用哪个

### 语义约定——社区共识

虽然 `struct` 和 `class` 在技术上等价，C++ 社区形成了清晰的**语义约定**：

| 约定 | `struct` | `class` |
|---|---|---|
| 语义 | 被动数据容器 | 具有行为的对象 |
| 成员 | 直接暴露数据成员 | 数据私有，通过接口访问 |
| 不变量 | 无不变量或非常简单 | 维护复杂的不变量 |
| 虚函数 | 通常没有 | 可以有 |
| 例子 | `Point {x, y}`、`Color {r, g, b}` | `ThreadPool`、`FileReader` |

核心原则很简单：**如果它主要是在"携带数据"，用 `struct`；如果它需要"封装行为和维护不变量"，用 `class`。**

### Google C++ Style Guide 的建议

Google C++ Style Guide 明确规定了 struct 和 class 的使用场景：

> 使用 `struct` 仅限于被动对象——那些只携带数据的对象。`struct` 应该不建立不变量（invariants），所有字段应该直接访问，并且不应该有超出初始化/销毁以外的成员函数。

具体来说：

```cpp
// Good: 用 struct 表示纯数据
struct HttpRequest {
    std::string method;
    std::string url;
    std::map<std::string, std::string> headers;
    std::string body;
};

// Good: 用 class 封装行为和不变量
class HttpClient {
public:
    HttpResponse Send(const HttpRequest& request);
    void SetTimeout(int milliseconds);
private:
    int timeout_ms_ = 5000;
    std::unique_ptr<ConnectionPool> pool_;
};
```

Google 风格还规定：
- `struct` **不应该**有 `private`/`protected` 成员
- `struct` **不应该**有复杂的成员函数（简单构造函数除外）
- 如果需要添加行为，应该把 `struct` 改成 `class`

### C++ Core Guidelines 的建议

Bjarne Stroustrup 本人参与编写的 C++ Core Guidelines 也给出了类似建议（Rule C.2）：

> **C.2**: Use `class` if the class has an invariant; use `struct` if the data members can vary independently.

> **C.8**: Use `class` rather than `struct` if any member is non-public.

这条规则和 Google 的建议高度一致：**有不变量用 `class`，纯数据用 `struct`。**

### 实际项目中的用法示例

**用 struct 的典型场景**：

```cpp
// 1. 函数返回多个值
struct ParseResult {
    bool success;
    int value;
    std::string error_message;
};

ParseResult ParseInt(std::string_view input) {
    // ...
}

// 2. 配置/选项
struct RenderOptions {
    bool enable_shadows = true;
    int msaa_samples = 4;
    float fov_degrees = 60.0f;
};

// 3. POD 式数据传输对象
struct NetworkPacket {
    uint32_t sequence;
    uint8_t type;
    std::array<uint8_t, 1024> payload;
};
```

**用 class 的典型场景**：

```cpp
// 1. RAII 资源管理
class FileHandle {
public:
    explicit FileHandle(const std::string& path);
    ~FileHandle();

    FileHandle(const FileHandle&) = delete;
    FileHandle& operator=(const FileHandle&) = delete;

    size_t Read(void* buffer, size_t size);
    void Write(const void* data, size_t size);

private:
    int fd_ = -1;
};

// 2. 具有不变量的类型
class Temperature {
public:
    static Temperature FromCelsius(double c) {
        return Temperature(c);
    }
    static Temperature FromFahrenheit(double f) {
        return Temperature((f - 32.0) * 5.0 / 9.0);
    }

    double Celsius() const { return value_; }
    double Fahrenheit() const { return value_ * 9.0 / 5.0 + 32.0; }

private:
    explicit Temperature(double celsius) : value_(celsius) {}
    double value_;  // 内部始终以摄氏度存储，这是不变量
};
```

---

## 常见误区

### 误区一："struct 在栈上，class 在堆上"

**错误。** 两者在内存分配上没有任何区别。分配位置取决于如何使用，而非关键字：

```cpp
struct S { int x; };
class C { public: int x; };

S s;           // 栈上
S* sp = new S; // 堆上
C c;           // 栈上
C* cp = new C; // 堆上
```

### 误区二："struct 不能有构造函数/析构函数"

**错误。** 在 C++ 中，`struct` 拥有和 `class` 完全相同的能力：

```cpp
struct Vec3 {
    double x, y, z;

    Vec3() : x(0), y(0), z(0) {}
    Vec3(double x, double y, double z) : x(x), y(y), z(z) {}

    double Length() const {
        return std::sqrt(x * x + y * y + z * z);
    }

    ~Vec3() {
        // 完全合法，尽管通常不需要
    }
};
```

### 误区三："struct 不能继承"

**错误。** `struct` 可以继承，也可以被继承：

```cpp
struct Base {
    int id;
    virtual void Print() const { std::cout << id << std::endl; }
    virtual ~Base() = default;
};

struct Derived : Base {  // 默认 public 继承
    std::string name;
    void Print() const override {
        std::cout << id << ": " << name << std::endl;
    }
};
```

### 误区四："class 比 struct 性能更好/更差"

**错误。** 两者编译生成的机器码完全相同。关键字不影响任何运行时行为——虚函数表布局、内存对齐、内联决策等全部一致。

---

## 前向声明的细节

无论用 `struct` 还是 `class` 做前向声明，在 C++ 中它们是**完全可互换**的：

```cpp
struct Foo;   // 前向声明

class Foo {   // 定义，用 class 而非 struct —— 完全合法
    // ...
};

// 反过来也一样
class Bar;    // 前向声明

struct Bar {  // 定义，用 struct 而非 class —— 也完全合法
    // ...
};
```

这是因为在 C++ 的类型系统中，`struct` 和 `class` 声明的是同一种东西——类类型（class type）。前向声明中的关键字不需要和定义中的关键字匹配。

不过，**工程实践中建议保持一致**——声明和定义使用相同的关键字，避免让阅读者困惑。

---

## 总结

| 维度 | `struct` | `class` |
|---|---|---|
| 默认成员访问 | `public` | `private` |
| 默认继承访问 | `public` | `private` |
| 技术能力 | 完全相同 | 完全相同 |
| 语义约定 | 被动数据容器 | 具有行为的对象 |
| 适用场景 | 无不变量的数据聚合 | 需要封装和维护不变量的类型 |

记住一句话就够了：**技术上只有一个区别（默认访问权限），实践中有一条约定（数据用 struct，行为用 class）。**

C++ 同时保留这两个关键字，是语言设计中"兼容性"与"表达力"平衡的缩影——`struct` 连接了 C 的过去，`class` 面向了面向对象的未来，而两者的等价性则避免了不必要的复杂性。
