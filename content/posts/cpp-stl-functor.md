---
title: "C++ STL 函数对象：从 operator() 到标准库函数对象"
date: 2021-08-16T10:00:00+08:00
tags: ["C++", "STL", "函数对象", "Functor", "operator()"]
categories: ["技术"]
summary: "深入剖析 C++ 函数对象的本质——重载 operator() 的类。从最简单的函子到标准库提供的算术、比较、逻辑函数对象，再到 C++11 std::function 的类型擦除与 Lambda 的关系，讲清楚函子在 STL 算法中的核心角色。"
ShowToc: true
---

## 什么是函数对象（Functor）

函数对象，也叫**函子**（functor），本质就是一个重载了 `operator()` 的类实例。

因为重载了函数调用运算符，这个对象可以像函数一样被"调用"，但它同时是一个对象，拥有构造函数、成员变量、析构函数，具备普通函数所没有的**状态保持能力**。

```cpp
// A functor is just a class with operator()
struct AddValue {
    int val;

    explicit AddValue(int v) : val(v) {}

    int operator()(int x) const {
        return x + val;
    }
};

int main() {
    AddValue adder(10);     // construct a functor object
    int result = adder(5);  // "call" the object — actually calls adder.operator()(5)
    // result == 15
}
```

`adder(5)` 看起来像函数调用，实际上是编译器帮你调用了 `adder.operator()(5)`。

> **核心定义**：任何重载了 `operator()` 的类型，其实例都称为**函数对象**。

## 最简单的 Functor

先看一个没有任何成员变量的例子，它和普通函数做的事情一模一样：

```cpp
struct PrintInt {
    void operator()(int x) const {
        std::cout << x << ' ';
    }
};

int main() {
    PrintInt printer;
    printer(42);   // output: 42
    printer(100);  // output: 100
}
```

你可能会问：这不就是一个打印函数吗？为什么要搞一个类出来？

答案在于 **STL 算法的设计**。标准库的算法（`std::sort`、`std::for_each`、`std::transform` 等）都接受**可调用对象**（callable）作为参数。函数指针是一种可调用对象，functor 是另一种，lambda 是第三种。functor 比函数指针多了一个关键优势：**它可以持有状态**。

| 特性 | 函数指针 | Functor | Lambda |
|------|----------|---------|--------|
| 可调用 | ✓ | ✓ | ✓ |
| 持有状态 | ✗ | ✓ | ✓（捕获） |
| 可内联 | 通常不行 | ✓ | ✓ |
| 类型安全 | 弱 | 强 | 强 |
| 编译期多态 | ✗ | ✓ | ✓ |

## 带状态的 Functor：相比函数指针的关键优势

函数指针最大的短板是**没有状态**。一个函数指针只能指向一个函数，但你没法让它在不同调用之间"记住"什么东西。

Functor 可以。因为它本质是个对象，成员变量就是它的状态。

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

// A functor that counts how many times it's been called
struct CountingPrinter {
    int call_count = 0;

    void operator()(int x) {
        ++call_count;
        std::cout << "#" << call_count << ": " << x << '\n';
    }
};

int main() {
    std::vector<int> data = {10, 20, 30, 40, 50};

    CountingPrinter printer;
    std::for_each(data.begin(), data.end(), std::ref(printer));

    std::cout << "Total calls: " << printer.call_count << '\n';
    // #1: 10
    // #2: 20
    // #3: 30
    // #4: 40
    // #5: 50
    // Total calls: 5
}
```

注意 `std::for_each` 传参时会拷贝 functor，所以如果想在外部拿到状态，需要用 `std::ref` 包一层引用包装。或者利用 `std::for_each` 的返回值：

```cpp
// for_each returns the moved-in functor, so we can get state back
auto final_printer = std::for_each(data.begin(), data.end(), CountingPrinter{});
std::cout << "Total calls: " << final_printer.call_count << '\n';
```

再看一个更实用的例子：一个阈值过滤器。

```cpp
struct AboveThreshold {
    int threshold;

    explicit AboveThreshold(int t) : threshold(t) {}

    bool operator()(int x) const {
        return x > threshold;
    }
};

int main() {
    std::vector<int> data = {1, 5, 10, 15, 20, 25};

    // find the first element greater than 12
    auto it = std::find_if(data.begin(), data.end(), AboveThreshold(12));
    if (it != data.end()) {
        std::cout << *it << '\n';  // output: 15
    }
}
```

`AboveThreshold(12)` 创建了一个 functor，它的内部状态 `threshold = 12`，每次被调用就拿这个阈值去比较。用函数指针做不到同样的事情（除非用全局变量，但那既丑陋又不安全）。

> **规则**：如果你的可调用逻辑需要参数化行为或保持跨调用状态，写一个 functor。

## Functor 在 STL 算法中的应用

STL 算法大量使用 functor。标准库中几乎所有接受"操作"参数的算法都可以用 functor。下面逐个演示最常见的场景。

### std::sort — 自定义排序

```cpp
#include <algorithm>
#include <vector>
#include <string>

struct Person {
    std::string name;
    int age;
};

struct CompareByAge {
    bool operator()(const Person& a, const Person& b) const {
        return a.age < b.age;
    }
};

struct CompareByName {
    bool operator()(const Person& a, const Person& b) const {
        return a.name < b.name;
    }
};

int main() {
    std::vector<Person> people = {
        {"Alice", 30},
        {"Bob", 25},
        {"Charlie", 35},
    };

    // sort by age ascending
    std::sort(people.begin(), people.end(), CompareByAge{});

    // sort by name alphabetical
    std::sort(people.begin(), people.end(), CompareByName{});
}
```

### std::find_if — 条件查找

```cpp
#include <algorithm>
#include <vector>

struct IsMultipleOf {
    int divisor;

    explicit IsMultipleOf(int d) : divisor(d) {}

    bool operator()(int x) const {
        return x % divisor == 0;
    }
};

int main() {
    std::vector<int> nums = {7, 14, 21, 28, 35};

    // find first multiple of 21
    auto it = std::find_if(nums.begin(), nums.end(), IsMultipleOf(21));
    // *it == 21
}
```

### std::transform — 元素变换

```cpp
#include <algorithm>
#include <vector>
#include <cmath>

struct ScaleBy {
    double factor;

    explicit ScaleBy(double f) : factor(f) {}

    double operator()(double x) const {
        return x * factor;
    }
};

int main() {
    std::vector<double> input = {1.0, 2.0, 3.0, 4.0};
    std::vector<double> output(input.size());

    std::transform(input.begin(), input.end(), output.begin(), ScaleBy(2.5));
    // output: {2.5, 5.0, 7.5, 10.0}
}
```

### std::accumulate — 自定义累加

```cpp
#include <numeric>
#include <vector>
#include <string>

struct ConcatWithSeparator {
    std::string sep;

    explicit ConcatWithSeparator(const std::string& s) : sep(s) {}

    std::string operator()(const std::string& acc, const std::string& next) const {
        return acc.empty() ? next : acc + sep + next;
    }
};

int main() {
    std::vector<std::string> words = {"Hello", "World", "C++"};

    std::string result = std::accumulate(
        words.begin(), words.end(),
        std::string{},
        ConcatWithSeparator(", ")
    );
    // result: "Hello, World, C++"
}
```

### std::remove_if — 条件删除

```cpp
#include <algorithm>
#include <vector>

struct IsNegative {
    bool operator()(int x) const {
        return x < 0;
    }
};

int main() {
    std::vector<int> data = {3, -1, 4, -5, 9, -2, 6};

    auto new_end = std::remove_if(data.begin(), data.end(), IsNegative{});
    data.erase(new_end, data.end());
    // data: {3, 4, 9, 6}
}
```

### std::for_each — 逐元素操作

```cpp
#include <algorithm>
#include <vector>
#include <iostream>

struct SquarePrinter {
    void operator()(int x) const {
        std::cout << x << "^2 = " << (x * x) << '\n';
    }
};

int main() {
    std::vector<int> nums = {1, 2, 3, 4, 5};
    std::for_each(nums.begin(), nums.end(), SquarePrinter{});
    // 1^2 = 1
    // 2^2 = 4
    // 3^2 = 9
    // 4^2 = 16
    // 5^2 = 25
}
```

---

## 标准库函数对象

C++ 标准库在 `<functional>` 头文件中提供了一组现成的 functor，覆盖了常见的算术、比较和逻辑操作。不用自己手写，直接拿来用。

### 算术函数对象

| Functor | 等价操作 | 示例 |
|---------|----------|------|
| `std::plus<T>` | `a + b` | `std::plus<int>()(3, 4)` → `7` |
| `std::minus<T>` | `a - b` | `std::minus<int>()(10, 3)` → `7` |
| `std::multiplies<T>` | `a * b` | `std::multiplies<int>()(3, 4)` → `12` |
| `std::divides<T>` | `a / b` | `std::divides<int>()(10, 3)` → `3` |
| `std::modulus<T>` | `a % b` | `std::modulus<int>()(10, 3)` → `1` |
| `std::negate<T>` | `-a` | `std::negate<int>()(5)` → `-5` |

实际使用场景：

```cpp
#include <functional>
#include <numeric>
#include <vector>

int main() {
    std::vector<int> nums = {1, 2, 3, 4, 5};

    // default accumulate uses plus, but we can be explicit
    int sum = std::accumulate(nums.begin(), nums.end(), 0, std::plus<int>());
    // sum == 15

    // compute product
    int product = std::accumulate(nums.begin(), nums.end(), 1, std::multiplies<int>());
    // product == 120
}
```

### 比较函数对象

| Functor | 等价操作 |
|---------|----------|
| `std::equal_to<T>` | `a == b` |
| `std::not_equal_to<T>` | `a != b` |
| `std::greater<T>` | `a > b` |
| `std::less<T>` | `a < b` |
| `std::greater_equal<T>` | `a >= b` |
| `std::less_equal<T>` | `a <= b` |

其中 `std::greater` 和 `std::less` 使用频率最高：

```cpp
#include <algorithm>
#include <functional>
#include <vector>

int main() {
    std::vector<int> nums = {5, 2, 8, 1, 9, 3};

    // ascending sort (default, shown for completeness)
    std::sort(nums.begin(), nums.end(), std::less<int>());

    // descending sort
    std::sort(nums.begin(), nums.end(), std::greater<int>());
    // nums: {9, 8, 5, 3, 2, 1}
}
```

### 逻辑函数对象

| Functor | 等价操作 |
|---------|----------|
| `std::logical_and<T>` | `a && b` |
| `std::logical_or<T>` | `a \|\| b` |
| `std::logical_not<T>` | `!a` |

```cpp
#include <functional>
#include <algorithm>
#include <vector>

int main() {
    std::vector<bool> flags = {true, true, false, true};

    // check if all are true (C++ way before std::all_of)
    bool all_true = std::accumulate(
        flags.begin(), flags.end(),
        true,
        std::logical_and<bool>()
    );
    // all_true == false
}
```

---

## std::bind 和 std::placeholders

`std::bind` 可以把一个可调用对象和部分参数绑定在一起，生成一个新的可调用对象。`std::placeholders::_1`、`_2` ... 表示新函数的第几个参数。

```cpp
#include <functional>
#include <iostream>

void PrintSum(int a, int b) {
    std::cout << a + b << '\n';
}

int main() {
    // bind the second argument to 10
    auto PrintPlus10 = std::bind(PrintSum, std::placeholders::_1, 10);

    PrintPlus10(5);   // output: 15
    PrintPlus10(20);  // output: 30
}
```

也可以绑定成员函数：

```cpp
#include <functional>
#include <iostream>
#include <vector>
#include <algorithm>

struct Formatter {
    void Print(int x) const {
        std::cout << "[" << x << "] ";
    }
};

int main() {
    std::vector<int> data = {1, 2, 3};
    Formatter fmt;

    // bind member function with object instance
    auto printer = std::bind(&Formatter::Print, &fmt, std::placeholders::_1);
    std::for_each(data.begin(), data.end(), printer);
    // [1] [2] [3]
}
```

> **实际建议**：在现代 C++ 中，Lambda 几乎总是比 `std::bind` 更清晰。`std::bind` 存在参数求值顺序未定义、嵌套可读性差等问题。新代码优先用 Lambda。

```cpp
// Prefer this over std::bind
auto print_plus_10 = [](int a) { PrintSum(a, 10); };
```

---

## C++14 透明函数对象（Transparent Functors）

C++14 引入了一个重要改进：**透明函数对象**（也叫**钻石函数对象**，diamond functors）。

你可以写 `std::less<>` 或 `std::greater<void>`，模板参数留空。它的 `operator()` 是一个模板：

```cpp
// C++11: must specify the type
std::sort(v.begin(), v.end(), std::less<int>());

// C++14: transparent — type deduced from arguments
std::sort(v.begin(), v.end(), std::less<>());
```

为什么这很重要？看一个实际场景：

```cpp
#include <set>
#include <functional>
#include <string>

struct Person {
    std::string name;
    int age;
};

// C++11: must write full comparator type, can't mix types
struct PersonCompare {
    bool operator()(const Person& a, const Person& b) const {
        return a.name < b.name;
    }
};

// The real power: heterogeneous lookup
// With std::less<>, you can look up in a std::set<std::string>
// using a std::string_view key, without converting to std::string
int main() {
    std::set<std::string, std::less<>> names = {"Alice", "Bob", "Charlie"};

    // heterogeneous lookup — no std::string temporary created
    std::string_view key = "Bob";
    auto it = names.find(key);  // works because std::less<> accepts different types
}
```

透明函数对象的 `operator()` 接受任意类型参数并自动推导。这对关联容器的**异构查找**（heterogeneous lookup）意义重大，避免了不必要的类型转换开销。

```cpp
// How std::less<> is defined (simplified)
template <>
struct less<void> {
    // Template operator() — accepts any types
    template <typename T, typename U>
    constexpr auto operator()(T&& t, U&& u) const
        -> decltype(std::forward<T>(t) < std::forward<U>(u))
    {
        return std::forward<T>(t) < std::forward<U>(u);
    }
};
```

> **规则**：在 C++14 及以后，写 `std::less<>` 而不是 `std::less<SomeSpecificType>`，除非你有明确的理由需要锁定类型。

---

## Functor vs Lambda vs std::function

三者都是"可调用对象"，但机制和适用场景不同。

### Lambda

Lambda 是 C++11 引入的匿名函数对象。编译器会为每个 lambda 生成一个唯一的**匿名 functor 类**。所以 lambda 本质上就是个 functor，只是语法糖。

```cpp
// This lambda...
auto add_ten = [](int x) { return x + 10; };

// ...is roughly equivalent to:
struct __unnamed_lambda {
    int operator()(int x) const { return x + 10; }
};
auto add_ten = __unnamed_lambda{};
```

Lambda 带捕获时，生成的匿名类会包含对应的成员变量：

```cpp
int offset = 10;
auto adder = [offset](int x) { return x + offset; };

// roughly equivalent to:
struct __unnamed_lambda {
    int offset;  // captured by value
    __unnamed_lambda(int o) : offset(o) {}
    int operator()(int x) const { return x + offset; }
};
auto adder = __unnamed_lambda{10};
```

### std::function

`std::function` 是一个**类型擦除**（type erasure）包装器。它可以持有任何签名匹配的可调用对象：函数指针、functor、lambda、bind 表达式、成员函数指针。

```cpp
#include <functional>

int Add(int a, int b) { return a + b; }

struct Multiply {
    int operator()(int a, int b) const { return a * b; }
};

int main() {
    std::function<int(int, int)> op;

    op = Add;                      // function pointer
    std::cout << op(3, 4) << '\n'; // 7

    op = Multiply{};               // functor
    std::cout << op(3, 4) << '\n'; // 12

    op = [](int a, int b) { return a - b; };  // lambda
    std::cout << op(10, 3) << '\n';            // 7
}
```

### 三者对比

| 维度 | Functor（手写） | Lambda | std::function |
|------|----------------|--------|---------------|
| 语法 | 需要定义结构体/类 | 简洁内联 | 包装器声明 |
| 状态 | 成员变量 | 捕获列表 | 持有任何 callable |
| 内联 | 编译器容易内联 | 编译器容易内联 | **通常无法内联** |
| 开销 | 零开销 | 零开销 | 堆分配可能性 + 虚调用 |
| 类型 | 每个类唯一类型 | 每个 lambda 唯一类型 | 统一类型 `std::function<Sig>` |
| 可存储 | 需要具体类型或模板 | 同左 | 可存入容器、成员变量 |
| 复用 | 可在多处实例化 | 通常一次使用 | 运行时多态场景 |

> **关键区别**：`std::function` 引入了运行时开销（可能堆分配 + 虚函数调用），而手写 functor 和 lambda 是编译期的零开销抽象。

---

## 什么时候手写 Functor，什么时候用 Lambda

### 用 Lambda 的场景

大多数情况下 Lambda 就够了：

```cpp
// Simple one-off operation
std::sort(v.begin(), v.end(), [](const auto& a, const auto& b) {
    return a.score > b.score;
});

// Capture local variables
int threshold = GetThreshold();
auto it = std::find_if(v.begin(), v.end(), [threshold](int x) {
    return x > threshold;
});
```

### 手写 Functor 的场景

**1. 需要在多处复用**

```cpp
// Reusable comparator — use in sort, set, priority_queue, etc.
struct CompareByPriority {
    bool operator()(const Task& a, const Task& b) const {
        return a.priority < b.priority;
    }
};

std::vector<Task> tasks;
std::sort(tasks.begin(), tasks.end(), CompareByPriority{});

std::priority_queue<Task, std::vector<Task>, CompareByPriority> queue;
```

**2. 需要复杂的状态或初始化逻辑**

```cpp
struct LevenshteinDistance {
    // Expensive-to-construct state
    std::vector<std::vector<int>> dp_matrix;

    explicit LevenshteinDistance(size_t max_len)
        : dp_matrix(max_len + 1, std::vector<int>(max_len + 1)) {}

    int operator()(const std::string& a, const std::string& b) const {
        // ... DP algorithm using dp_matrix ...
        return 0;  // simplified
    }
};
```

**3. 作为模板参数需要默认构造**

标准容器（如 `std::set`、`std::map`、`std::priority_queue`）的比较器需要是**可默认构造**的。Lambda 没有默认构造函数，不能直接用作容器的模板参数。手写 functor 可以。

**4. 需要 ADL（参数依赖查找）或重载**

```cpp
struct HashCombine {
    // Overloaded for different types
    size_t operator()(int x) const {
        return std::hash<int>{}(x);
    }
    size_t operator()(const std::string& s) const {
        return std::hash<std::string>{}(s);
    }
};
```

> **简单规则**：如果逻辑用一次，用 Lambda。如果多处复用、需要状态初始化、或作为容器模板参数，写 Functor。

---

## 性能：Functor 的编译期多态优势

Functor（包括 Lambda）相比 `std::function` 最大的性能优势在于**内联**。

### 编译器内联

```cpp
struct AddFunctor {
    int operator()(int a, int b) const { return a + b; }
};

int ViaFunctor(int x, int y) {
    return AddFunctor{}(x, y);
    // Compiler sees the exact type, inlines the call
    // Optimized to just: return x + y;
}

int ViaStdFunction(int x, int y) {
    std::function<int(int, int)> f = AddFunctor{};
    return f(x, y);
    // Compiler cannot inline — f could hold anything at runtime
    // Actual virtual call through type-erased wrapper
}
```

当 `std::sort` 接受一个 functor 时，`sort` 的模板实例化能"看到" functor 的 `operator()` 实体，编译器可以直接内联比较操作。这对排序这类 O(n log n) 的算法影响巨大。

```cpp
#include <algorithm>
#include <functional>
#include <chrono>
#include <vector>

struct MyCompare {
    bool operator()(int a, int b) const { return a < b; }
};

void BenchmarkSort(std::vector<int>& data) {
    // Fast — functor inlined into sort
    std::sort(data.begin(), data.end(), MyCompare{});

    // Slower — std::function prevents inlining
    std::function<bool(int, int)> cmp = MyCompare{};
    // Note: std::sort won't accept std::function as comparator directly
    // because sort requires the comparator to be a template argument
    // This is illustrative of the overhead concept
}
```

### 编译期多态 vs 运行时多态

传统面向对象的运行时多态用虚函数：

```cpp
struct Comparator {
    virtual ~Comparator() = default;
    virtual bool Compare(int a, int b) const = 0;
};

struct LessComparator : Comparator {
    bool Compare(int a, int b) const override { return a < b; }
};

// Virtual call — cannot inline
void SortWithVirtual(int* begin, int* end, const Comparator& cmp) {
    // ... uses cmp.Compare(*it, *jt) — virtual dispatch every comparison
}
```

Functor 提供的是**编译期多态**（通过模板），零虚函数调用开销：

```cpp
template <typename Cmp>
void SortWithFunctor(int* begin, int* end, Cmp cmp) {
    // cmp(a, b) is a direct call, fully inlineable
}

// Instantiated with MyCompare, all calls are resolved at compile time
```

这就是 STL 算法全部基于模板设计的原因：模板 + functor = 零开销抽象。

---

## 实战示例

### 自定义 Hash Functor 用于 unordered 容器

`std::unordered_map` 和 `std::unordered_set` 的第三个模板参数是哈希函数。默认的 `std::hash<T>` 只支持基本类型和 `std::string`。自定义类型需要提供自己的哈希 functor。

```cpp
#include <unordered_set>
#include <functional>
#include <string>

struct Student {
    int id;
    std::string name;
    double gpa;

    bool operator==(const Student& other) const {
        return id == other.id && name == other.name;
    }
};

// Custom hash functor
struct StudentHash {
    std::size_t operator()(const Student& s) const {
        // Combine hash of id and name
        std::size_t h1 = std::hash<int>{}(s.id);
        std::size_t h2 = std::hash<std::string>{}(s.name);

        // Boost-style hash combine
        return h1 ^ (h2 + 0x9e3779b9 + (h1 << 6) + (h1 >> 2));
    }
};

int main() {
    std::unordered_set<Student, StudentHash> students;

    students.insert({1001, "Alice", 3.8});
    students.insert({1002, "Bob", 3.5});
    students.insert({1003, "Charlie", 3.9});

    // Lookup — uses StudentHash for bucket placement
    Student key{1002, "Bob", 0.0};  // gpa doesn't matter for equality
    auto it = students.find(key);
    // found Bob
}
```

### 自定义比较器用于 priority_queue

`std::priority_queue` 默认是最大堆（`std::less<T>`），但很多场景需要最小堆或自定义优先级。

```cpp
#include <queue>
#include <vector>
#include <functional>
#include <iostream>

struct Task {
    std::string name;
    int priority;   // lower number = higher priority
    int deadline;   // Unix timestamp
};

// Compare by priority first, then by deadline
struct TaskComparator {
    bool operator()(const Task& a, const Task& b) const {
        if (a.priority != b.priority) {
            return a.priority > b.priority;  // lower priority number comes first
        }
        return a.deadline > b.deadline;  // earlier deadline comes first
    }
};

int main() {
    // Min-heap based on custom priority logic
    std::priority_queue<
        Task,
        std::vector<Task>,
        TaskComparator
    > task_queue;

    task_queue.push({"Deploy server", 1, 1629000000});
    task_queue.push({"Fix bug #42", 1, 1628900000});
    task_queue.push({"Write docs", 3, 1629200000});
    task_queue.push({"Code review", 2, 1629100000});

    while (!task_queue.empty()) {
        auto t = task_queue.top();
        task_queue.pop();
        std::cout << "[" << t.priority << "] " << t.name
                  << " (deadline: " << t.deadline << ")\n";
    }
    // [1] Fix bug #42 (deadline: 1628900000)
    // [1] Deploy server (deadline: 1629000000)
    // [2] Code review (deadline: 1629100000)
    // [3] Write docs (deadline: 1629200000)
}
```

这里 `TaskComparator` 必须是可默认构造的，因为它是 `priority_queue` 的模板参数。Lambda 无法做到这一点，必须手写 functor。

### 组合多个 functor

实际项目中经常需要组合条件。用 functor 的组合模式可以让代码更清晰：

```cpp
#include <algorithm>
#include <vector>
#include <functional>

struct IsInRange {
    int low, high;
    IsInRange(int l, int h) : low(l), high(h) {}
    bool operator()(int x) const { return x >= low && x <= high; }
};

struct IsNotZero {
    bool operator()(int x) const { return x != 0; }
};

// A composite functor: combines two predicates with AND logic
template <typename Pred1, typename Pred2>
struct AndPredicate {
    Pred1 pred1;
    Pred2 pred2;

    AndPredicate(Pred1 p1, Pred2 p2) : pred1(p1), pred2(p2) {}

    bool operator()(int x) const {
        return pred1(x) && pred2(x);
    }
};

int main() {
    std::vector<int> data = {-5, 0, 3, 7, 10, 15, 20, 0, 25};

    // Find: in range [5, 20] AND not zero
    auto pred = AndPredicate<IsInRange, IsNotZero>(
        IsInRange(5, 20),
        IsNotZero{}
    );

    auto it = std::find_if(data.begin(), data.end(), pred);
    // *it == 7
}
```

> 这种组合模式在 C++20 中有了更好的方案：Ranges 库的视图组合（`std::views::filter`）。但在 C++17 及之前，functor 组合是惯用手法。

---

## 总结

回顾全文的关键要点：

1. **Functor 本质**：重载了 `operator()` 的类实例，可调用，可持有状态。
2. **相比函数指针**：functor 可以有成员变量（状态），可以被编译器内联。
3. **STL 算法**：`sort`、`find_if`、`transform`、`accumulate`、`remove_if`、`for_each` 等大量算法接受 functor 作为参数。
4. **标准库函数对象**：`<functional>` 提供了算术（`plus`、`minus`、`multiplies`...）、比较（`less`、`greater`、`equal_to`...）、逻辑（`logical_and`、`logical_or`、`logical_not`）三大类现成 functor。
5. **透明函数对象**：C++14 的 `std::less<>` 支持异构查找，避免不必要的类型转换。
6. **Lambda 就是 functor**：编译器为每个 lambda 生成匿名 functor 类，本质相同。
7. **std::function 是类型擦除**：统一包装任何 callable，但引入运行时开销。
8. **手写 vs Lambda**：一次使用用 Lambda，多处复用或需要默认构造用手写 functor。
9. **性能**：functor 是编译期多态，零虚调用开销，可被完全内联。
10. **实战**：自定义哈希 functor 用于 `unordered_map`，自定义比较器用于 `priority_queue`。

```
// Decision cheat sheet:
//
// Need a callable?
//   ├─ One-off use?                         → Lambda
//   ├─ Reuse across multiple places?        → Hand-written Functor
//   ├─ Need default construction (container)? → Hand-written Functor
//   ├─ Runtime polymorphism / type erasure? → std::function
//   └─ Standard operation (less, plus...)?  → std::less<>, std::plus<>, etc.
```

Functor 是 C++ 泛型编程的基石之一。理解它，才能理解 STL 算法的设计哲学，也才能写出既高效又可复用的代码。