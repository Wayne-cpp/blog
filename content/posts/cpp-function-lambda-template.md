---
title: "C++ 函数、Lambda、Bind 与模板：从回调到泛型的思想演进"
date: 2021-07-28T10:00:00+08:00
tags: ["C++", "lambda", "模板", "泛型编程", "类型推导", "std::function"]
categories: ["技术"]
summary: "系统梳理 C++ 中可调用对象的演进脉络：从函数指针到 std::function、从 std::bind 到 lambda 表达式、从手动指定类型到 auto/decltype 类型推导、从具体类型到模板泛型编程。讲清楚这些机制各自解决什么问题、为什么要引入这么多层抽象，以及在实际工程中如何选择。"
ShowToc: true
---

C++ 提供了多种方式来表达"一段可执行的代码"：函数指针、`std::function`、`std::bind`、lambda 表达式、函数对象（functor）。与此同时，模板和类型推导（`auto`、`decltype`）让这些机制得以泛化——不绑定于某个具体类型，而是适用于一族类型。

初学者常常困惑：为什么需要这么多东西？一个函数指针不够用吗？答案是不够——因为现实中的"回调"需求远比"调用一个固定签名的函数"复杂得多。这篇文章从**问题驱动**的角度出发，展示每种机制解决的具体痛点，把函数、lambda、bind、模板、类型推导串成一条完整的演进链路。

## 起点：函数指针——C 时代的回调机制

### 函数指针能做什么

C 语言用函数指针实现回调——把"行为"作为参数传递：

```cpp
// 排序比较函数
int CompareInt(const void* a, const void* b) {
    int x = *static_cast<const int*>(a);
    int y = *static_cast<const int*>(b);
    return (x > y) - (x < y);
}

int main() {
    int arr[] = {3, 1, 4, 1, 5, 9};
    std::qsort(arr, 6, sizeof(int), CompareInt);  // 传递函数指针
    return 0;
}
```

C++ 的 `<algorithm>` 也大量使用函数指针风格的回调：

```cpp
bool IsEven(int n) { return n % 2 == 0; }

int main() {
    std::vector<int> v = {1, 2, 3, 4, 5, 6};
    auto it = std::find_if(v.begin(), v.end(), IsEven);  // 传入函数名，隐式退化為函数指针
    // it 指向 2
    return 0;
}
```

### 函数指针的局限

**局限一：无法携带状态**

```cpp
// 需求：找到第一个大于 threshold 的元素
// 但 threshold 是运行时变量，函数指针没有地方存它
bool GreaterThan(int n) {
    return n > ???;  // threshold 从哪来？只能用全局变量——丑陋且线程不安全
}
```

**局限二：签名必须严格匹配**

```cpp
void Process(int x) { /* ... */ }

int main() {
    void (*fp)(int) = Process;  // OK
    // void (*fp2)(int) = &SomeClass::Method;  // 错误: 成员函数指针是另一种类型
    return 0;
}
```

**局限三：无法指向内联代码**

函数指针指向的是一段具名函数的地址，无法直接表示"在这里内联写一小段逻辑"。

> 这三个局限驱动了后续所有机制的诞生：函数对象解决状态问题，`std::function` 解决类型统一问题，lambda 解决内联问题。

---

## 函数对象（Functor）：带状态的"函数"

### 运算符重载让对象像函数一样调用

C++ 允许重载 `operator()`，使对象可以像函数一样被调用——这就是**函数对象**（functor）：

```cpp
class GreaterThan {
public:
    explicit GreaterThan(int threshold) : threshold_(threshold) {}
    bool operator()(int n) const { return n > threshold_; }
private:
    int threshold_;
};

int main() {
    std::vector<int> v = {1, 2, 3, 4, 5, 6, 7, 8};
    int threshold = 5;

    // Functor 携带状态——解决了函数指针的第一个局限
    auto it = std::find_if(v.begin(), v.end(), GreaterThan(threshold));
    // it 指向 6
    return 0;
}
```

### 为什么函数对象比函数指针更高效

函数指针在编译时是**间接调用**——编译器不知道具体调用哪个函数，无法内联。函数对象的类型在编译时是确定的，编译器可以直接内联 `operator()`：

```cpp
struct Add {
    int operator()(int a, int b) const { return a + b; }
};

int main() {
    Add add;
    int result = add(1, 2);  // 编译器可以完全内联——和手写 1 + 2 一样快
    return 0;
}
```

这也是为什么标准库算法偏好函数对象——它给编译器最大的优化空间。

### STL 中的函数对象

C++ 标准库内置了大量函数对象：

```cpp
#include <functional>

int main() {
    std::vector<int> v = {3, 1, 4, 1, 5, 9};
    std::sort(v.begin(), v.end(), std::greater<int>());  // 降序排序
    // v = {9, 5, 4, 3, 1, 1}

    int sum = 0;
    std::for_each(v.begin(), v.end(), [&sum](int n) { sum += n; });  // lambda 计算总和
    return 0;
}
```

---

## Lambda 表达式：内联的函数对象

### Lambda 的本质

C++11 引入的 lambda 表达式，本质上是**编译器自动生成的函数对象**。它让"在调用点直接写一小段逻辑"成为可能：

```cpp
int main() {
    int threshold = 5;
    std::vector<int> v = {1, 2, 3, 4, 5, 6, 7, 8};

    // Lambda: 编译器自动生成一个 unnamed functor
    auto it = std::find_if(v.begin(), v.end(), [threshold](int n) {
        return n > threshold;
    });
    // 等价于手写 GreaterThan functor，但代码就在使用处
    return 0;
}
```

编译器为上面的 lambda 生成类似这样的代码：

```cpp
// 编译器内部生成的（简化）
class __lambda_13_38 {
public:
    __lambda_13_38(int threshold) : threshold_(threshold) {}
    bool operator()(int n) const { return n > threshold_; }
private:
    int threshold_;
};
```

### Lambda 的语法结构

```cpp
[capture](parameters) mutable noexcept -> return_type { body }
  │         │          │        │           │            │
  │         │          │        │           │            └─ 函数体
  │         │          │        │           └─ 尾置返回类型（可省略）
  │         │          │        └─ 异常说明（可选）
  │         │          └─ 是否允许修改捕获的变量（可选）
  │         └─ 参数列表
  └─ 捕获列表：如何访问外部变量
```

### 捕获列表详解

捕获列表决定了 lambda 如何访问其所在作用域中的变量：

| 捕获方式 | 含义 | 变量访问方式 |
|---|---|---|
| `[]` | 不捕获任何变量 | 只能使用参数和全局变量 |
| `[x]` | 按值捕获 `x` | 拷贝一份，lambda 内部是副本 |
| `[&x]` | 按引用捕获 `x` | 直接访问原变量 |
| `[=]` | 按值捕获所有使用的局部变量 | 全部拷贝 |
| `[&]` | 按引用捕获所有使用的局部变量 | 全部引用 |
| `[=, &x]` | 默认按值，`x` 按引用 | 混合捕获 |
| `[&, x]` | 默认按引用，`x` 按值 | 混合捕获 |
| `[this]` | 捕获当前对象的 `this` 指针 | 可访问成员变量和函数 |
| `[x = expr]` | 初始化捕获（C++14） | 移动捕获或计算捕获 |

```cpp
int main() {
    int a = 10;
    int b = 20;

    // 按值捕获：lambda 持有副本
    auto f1 = [a, b]() { return a + b; };
    a = 100;  // 不影响 lambda 中的 a
    std::cout << f1() << std::endl;  // 30

    // 按引用捕获：lambda 直接访问原变量
    auto f2 = [&a, &b]() { return a + b; };
    a = 100;
    std::cout << f2() << std::endl;  // 120

    // C++14 初始化捕获：移动一个 unique_ptr 进 lambda
    auto ptr = std::make_unique<int>(42);
    auto f3 = [p = std::move(ptr)]() { return *p; };
    // ptr 现在是 nullptr，p 持有资源
    std::cout << f3() << std::endl;  // 42

    return 0;
}
```

### 泛型 Lambda（C++14）

C++14 允许 lambda 参数使用 `auto`，使其成为泛型函数对象：

```cpp
auto add = [](auto a, auto b) { return a + b; };

int main() {
    std::cout << add(1, 2) << std::endl;       // 3 (int)
    std::cout << add(1.5, 2.5) << std::endl;   // 4.0 (double)
    std::cout << add(std::string("hi "), std::string("there")) << std::endl;  // "hi there"
    return 0;
}
```

C++20 进一步允许 lambda 模板参数：

```cpp
// C++20: 显式模板参数
auto add = []<typename T>(T a, T b) { return a + b; };
```

### 捕获的陷阱

**陷阱一：悬空引用**

```cpp
std::function<int()> CreateCounter() {
    int count = 0;
    return [&count]() { return ++count; };  // 灾难: count 是局部变量引用
    // 函数返回后 count 被销毁，lambda 中的引用悬空
}

// 正确做法：按值捕获
std::function<int()> CreateCounter() {
    int count = 0;
    return [count]() mutable { return ++count; };  // 拷贝一份，安全
}
```

**陷阱二：`mutable` 忘记加**

```cpp
int x = 10;
auto f = [x]() {
    // x += 1;  // 编译错误: lambda 的 operator() 默认是 const
    // 按值捕获的 x 是 const 副本，不能修改
};

auto f2 = [x]() mutable {
    x += 1;  // OK: mutable 移除了 const 限定
    return x;
};
```

---

## `std::function`：类型擦除的可调用包装器

### 为什么需要 `std::function`

不同的可调用实体——函数指针、函数对象、lambda——有**不同的类型**：

```cpp
bool Func1(int n) { return n > 0; }
struct Functor { bool operator()(int n) const { return n > 0; } };
auto lambda = [](int n) { return n > 0; };

// 三者行为相同，但类型完全不同
// Func1 的类型: bool(*)(int)
// Functor 的类型: Functor
// lambda 的类型: 编译器生成的 unnamed class
```

当需要存储、传递"任意签名的可调用对象"时（比如回调列表、策略容器），类型不统一就成了问题。`std::function` 通过**类型擦除**解决这个问题：

```cpp
#include <functional>
#include <vector>

int main() {
    // std::function 统一包装所有可调用对象
    std::vector<std::function<bool(int)>> predicates;

    predicates.push_back(Func1);                     // 函数指针
    predicates.push_back(Functor());                 // 函数对象
    predicates.push_back([](int n) { return n > 0; }); // lambda
    predicates.push_back(std::bind(std::greater<int>(), std::placeholders::_1, 0)); // bind 表达式

    for (auto& pred : predicates) {
        std::cout << pred(42) << std::endl;  // 全部输出 1
    }
    return 0;
}
```

### `std::function` 的开销

类型擦除不是免费的：

| 开销 | 说明 |
|---|---|
| 内存 | 可能堆分配（小对象优化范围内则栈上） |
| 调用 | 间接调用（虚函数机制），无法内联 |
| `sizeof` | 通常 32-48 字节（远大于函数指针的 8 字节） |

```cpp
// 对比大小
std::cout << sizeof(bool(*)(int)) << std::endl;           // 8
std::cout << sizeof(std::function<bool(int)>) << std::endl; // 32-48（取决于实现）
```

> **使用原则**：只有在需要**类型擦除**（存储异构可调用对象、跨 API 边界传递回调）时才用 `std::function`。能用模板参数或 `auto` 的地方就不要用 `std::function`——后者会引入不必要的间接调用。

### `std::function` vs 模板参数

```cpp
// 方案一：模板参数——零开销，编译时确定类型
template <typename Pred>
void Filter(std::vector<int>& v, Pred pred) {
    v.erase(std::remove_if(v.begin(), v.end(), pred), v.end());
}

// 方案二：std::function——有开销，但类型统一
void Filter(std::vector<int>& v, std::function<bool(int)> pred) {
    v.erase(std::remove_if(v.begin(), v.end(), pred), v.end());
}

int main() {
    std::vector<int> v = {1, 2, 3, 4, 5};

    // 方案一：lambda 类型直接传给模板，可以内联
    Filter(v, [](int n) { return n % 2 == 0; });  // 高效

    // 方案二：lambda 被包装为 std::function，间接调用
    Filter(v, std::function<bool(int)>([](int n) { return n % 2 == 0; }));  // 较慢
    return 0;
}
```

**何时选择哪个**：

| 场景 | 选择 |
|---|---|
| 算法内部使用回调（如 STL 算法） | 模板参数——零开销 |
| 类成员存储回调（如事件系统） | `std::function`——类型统一 |
| 跨编译单元传递回调 | `std::function`——隐藏实现细节 |
| 性能关键的循环中 | 模板参数——可内联 |

---

## `std::bind`：部分应用（Partial Application）

### 基本用法

`std::bind` 可以将一个可调用对象的某些参数预先绑定到固定值，生成一个新的可调用对象：

```cpp
#include <functional>

int Add(int a, int b, int c) {
    return a + b + c;
}

int main() {
    // 绑定第一个参数为 1，其余用占位符
    auto add_one = std::bind(Add, 1, std::placeholders::_1, std::placeholders::_2);
    std::cout << add_one(2, 3) << std::endl;  // 1 + 2 + 3 = 6

    // 绑定前两个参数
    auto add_partial = std::bind(Add, 10, 20, std::placeholders::_1);
    std::cout << add_partial(30) << std::endl;  // 10 + 20 + 30 = 60

    // 重排参数
    auto subtract = std::bind(Add, std::placeholders::_2, std::placeholders::_1, 0);
    std::cout << subtract(3, 5) << std::endl;  // 5 + 3 + 0 = 8（参数反序）
    return 0;
}
```

### `std::bind` 的问题

**问题一：可读性差**

占位符 `_1`、`_2` 的含义取决于上下文，不如 lambda 直观：

```cpp
// bind 版本——哪个参数对应什么？
auto f = std::bind(&Foo, std::placeholders::_2, 42, std::placeholders::_1);

// lambda 版本——语义清晰
auto f = [](auto&& a, auto&& b) { return Foo(b, 42, a); };
```

**问题二：占位符在嵌套时难以理解**

```cpp
// 这在做什么？
auto f = std::bind(std::plus<int>(),
    std::bind(std::multiplies<int>(), std::placeholders::_1, 2),
    1);
// 等价于: f(x) = x * 2 + 1

// lambda 版本——一目了然
auto f = [](int x) { return x * 2 + 1; };
```

**问题三：与 `std::function` 的交互问题**

`std::bind` 的返回类型是未指定的，且按值传递参数时可能导致意外的拷贝。

### 现代 C++：优先用 Lambda 替代 bind

C++14 之后，lambda 的初始化捕获完全覆盖了 `std::bind` 的功能：

```cpp
class Processor {
public:
    void Process(int x, const std::string& label) { /* ... */ }
};

int main() {
    Processor proc;

    // bind 方式
    auto bound = std::bind(&Processor::Process, &proc, std::placeholders::_1, "default");

    // lambda 方式——更清晰
    auto lambda = [&proc](int x) { proc.Process(x, "default"); };

    // C++14 移动捕获
    auto ptr = std::make_unique<Data>();
    auto bound_ptr = [p = std::move(ptr), &proc]() { proc.Process(*p); };
    // std::bind 无法移动捕获 unique_ptr
    return 0;
}
```

| 特性 | `std::bind` | Lambda |
|---|---|---|
| 可读性 | 差（占位符混淆） | 好（直观的代码） |
| 移动捕获 | 不支持 | C++14 初始化捕获 |
| 内联友好 | 较差 | 好 |
| 泛型 | 不支持 | C++14 `auto` 参数 |
| 调试 | 困难 | 容易 |

> **结论**：`std::bind` 在 C++14 之后基本没有存在的必要。新代码一律用 lambda。理解 `bind` 是为了阅读旧代码。

---

## 类型推导：让编译器帮你记住类型

### `auto`：最常用的类型推导

C++11 引入 `auto`，让编译器根据初始化表达式推导变量类型：

```cpp
auto x = 42;                    // int
auto pi = 3.14;                 // double
auto name = std::string("hi");  // std::string
auto it = v.begin();            // std::vector<int>::iterator
auto lambda = [](int n) { return n * 2; };  // 编译器生成的 unnamed class
```

`auto` 的核心价值不在于"少打几个字"，而在于：

**1. 避免写出冗长或无法写出的类型**

```cpp
// 没有 auto 的世界
std::unordered_map<std::string, std::vector<int>>::iterator it = map.find(key);

// 有 auto 的世界
auto it = map.find(key);

// lambda 的类型——你根本写不出来
auto f = [](int x) { return x * 2; };
// std::???  f = ...;  // lambda 类型是编译器内部生成的，没有公开的名字
```

**2. 重构时自动适应类型变化**

```cpp
// 如果 score 从 int 改为 double，所有 auto 自动适应
auto score = GetScore();  // 不需要改
```

**3. 保证类型正确性（避免意外截断）**

```cpp
std::unordered_map<std::string, int> m;
// 危险: std::pair<const std::string, int> vs std::pair<std::string, int>
for (std::pair<std::string, int>& p : m) { /* ... */ }  // 隐式拷贝！
for (auto& p : m) { /* ... */ }                          // 正确的引用类型
```

### `decltype`：查询表达式的类型

`decltype` 返回表达式的**精确类型**（包括引用和 const）：

```cpp
int x = 42;
decltype(x) a = x;         // int（和 x 的声明类型一致）
decltype((x)) b = x;       // int&（注意：双括号表示左值表达式）

const int& ref = x;
decltype(ref) c = x;       // const int&

int arr[5];
decltype(arr) d;           // int[5]（数组类型，非指针）
```

### 尾置返回类型（Trailing Return Type）

当函数返回类型依赖于参数类型时，`decltype` 和 `auto` 配合实现延迟推导：

```cpp
// C++11: 尾置返回类型
template <typename T, typename U>
auto Add(T a, U b) -> decltype(a + b) {
    return a + b;
}

// C++14: 自动推导返回类型
template <typename T, typename U>
auto Add(T a, U b) {
    return a + b;  // 编译器从 return 语句推导返回类型
}
```

### `auto` 与 `decltype(auto)`

C++14 引入 `decltype(auto)`，结合了两者的语义：

```cpp
template <typename Container>
decltype(auto) GetFirst(Container& c) {
    return c[0];  // 保留引用语义：如果 c[0] 返回引用，函数也返回引用
}

// 对比：
// auto 会去掉引用和 const
template <typename Container>
auto GetFirstBad(Container& c) {
    return c[0];  // 返回值的副本（auto 去掉了引用）
}
```

| 推导方式 | 处理引用 | 处理 const | 典型用途 |
|---|---|---|---|
| `auto` | 去掉引用 | 去掉顶层 const | 普通变量 |
| `auto&` | 保留引用 | 保留 const | 遍历容器 |
| `auto&&` | 万能引用 | 保留 | 转发 |
| `decltype(auto)` | 精确保留 | 精确保留 | 完美转发返回值 |

---

## 模板：泛型编程的基石

### 为什么需要模板

设想一个简单的 `Max` 函数：

```cpp
int Max(int a, int b) { return a > b ? a : b; }
double Max(double a, double b) { return a > b ? a : b; }
std::string Max(const std::string& a, const std::string& b) { return a > b ? a : b; }
// ... 每种类型重写一遍？
```

函数重载解决了签名不同的问题，但逻辑完全相同的代码写了多遍——违反 DRY 原则。模板让代码**对类型参数化**：

```cpp
template <typename T>
const T& Max(const T& a, const T& b) {
    return a > b ? a : b;
}

int main() {
    Max(1, 2);                        // T = int
    Max(3.14, 2.72);                  // T = double
    Max(std::string("a"), std::string("b"));  // T = std::string
    return 0;
}
```

### 模板的核心思想：鸭子类型 + 编译时多态

模板不要求类型继承自某个基类或实现某个接口。只要类型支持模板中使用的操作，就可以使用——这就是**鸭子类型**（duck typing）：

```cpp
template <typename T>
void Print(const T& val) {
    std::cout << val << std::endl;  // 只要求 T 支持 operator<<
}

Print(42);            // int: OK
Print(3.14);          // double: OK
Print(std::string("hi"));  // string: OK
// Print(std::vector<int>{});  // 编译错误: vector 不支持 operator<<
```

多态的时机不同：

| 多态方式 | 时机 | 机制 | 开销 |
|---|---|---|---|
| 虚函数 | 运行时 | vtable 间接调用 | 有 |
| 模板 | 编译时 | 代码实例化 | 零 |

模板生成的代码对每个具体类型都是"定制"的——编译器可以为每个类型做专门优化。

### 模板参数推导（C++17 `std::optional` 风格）

C++17 之前，类模板参数必须显式指定。C++17 允许从构造函数推导：

```cpp
// C++17 之前
std::pair<int, double> p(1, 3.14);
std::vector<int> v{1, 2, 3};

// C++17: 自动推导
std::pair p(1, 3.14);        // 推导为 std::pair<int, double>
std::vector v{1, 2, 3};      // 推导为 std::vector<int>
```

### SFINAE 与 `if constexpr`：条件编译

传统 SFINAE（Substitution Failure Is Not An Error）：

```cpp
// C++11: 通过类型特性选择重载——复杂且晦涩
template <typename T>
typename std::enable_if<std::is_integral<T>::value, T>::type
Abs(T val) { return val < 0 ? -val : val; }

template <typename T>
typename std::enable_if<std::is_floating_point<T>::value, T>::type
Abs(T val) { return std::fabs(val); }
```

C++17 的 `if constexpr`——编译时条件分支：

```cpp
template <typename T>
auto Abs(T val) {
    if constexpr (std::is_integral_v<T>) {
        return val < 0 ? -val : val;  // 整数: 只有这段被编译
    } else {
        return std::fabs(val);         // 浮点: 只有这段被编译
    }
}
```

C++20 Concepts——直接约束模板参数：

```cpp
// C++20: 对模板参数添加语义约束
template <typename T>
requires std::integral<T>
T Abs(T val) { return val < 0 ? -val : val; }

// 或更简洁的语法
template <std::integral T>
T Abs(T val) { return val < 0 ? -val : val; }
```

---

## 全景对比：可调用对象的选型

### 类型与性能对比

```cpp
// 同一个功能，四种实现方式
bool IsPositive(int n) { return n > 0; }  // 1. 普通函数

struct IsPositiveFunctor {                // 2. 函数对象
    bool operator()(int n) const { return n > 0; }
};

auto is_positive_lambda = [](int n) { return n > 0; };  // 3. Lambda

std::function<bool(int)> is_positive_fn = [](int n) { return n > 0; };  // 4. std::function 包装
```

| 方式 | 类型 | 能否携带状态 | 能否内联 | 大小 |
|---|---|---|---|---|
| 函数指针 | `bool(*)(int)` | ❌（只能用全局/静态变量） | 不确定 | 8 字节 |
| 函数对象 | 自定义 class | ✅ | ✅ | 取决于成员 |
| Lambda | 编译器生成 class | ✅（捕获列表） | ✅ | 取决于捕获 |
| `std::function` | `std::function<bool(int)>` | ✅ | ❌ | 32-48 字节 |

### 选择决策树

```
需要一个"可调用对象"？
├─ 只在一个地方使用 → Lambda（就地定义）
├─ 需要复用且有状态 → 函数对象（命名 class）
├─ 需要存储/传递异构可调用对象 → std::function（类型擦除）
├─ 需要 C 兼容回调 → 函数指针（+ void* userdata）
└─ 需要部分应用参数 → Lambda 初始化捕获（不要用 bind）
```

### 泛型选择的决策树

```
代码需要对多种类型工作？
├─ 类型在编译时已知且有限 → 函数重载
├─ 类型有公共基类 → 虚函数（运行时多态）
├─ 类型无公共基类但操作相同 → 模板（编译时多态）
└─ 类型在运行时动态决定 → std::function + 类型擦除
```

---

## 实战场景

### 场景一：事件系统（`std::function` + Lambda）

```cpp
#include <functional>
#include <unordered_map>
#include <vector>
#include <string>

class EventBus {
public:
    using Handler = std::function<void(const std::string&)>;

    void Subscribe(const std::string& event, Handler handler) {
        handlers_[event].push_back(std::move(handler));
    }

    void Publish(const std::string& event, const std::string& data) {
        auto it = handlers_.find(event);
        if (it != handlers_.end()) {
            for (auto& handler : it->second) {
                handler(data);
            }
        }
    }

private:
    std::unordered_map<std::string, std::vector<Handler>> handlers_;
};

int main() {
    EventBus bus;

    // Lambda 订阅——就地定义回调逻辑
    bus.Subscribe("user.login", [](const std::string& user) {
        std::cout << "User logged in: " << user << std::endl;
    });

    // 捕获上下文
    int login_count = 0;
    bus.Subscribe("user.login", [&login_count](const std::string& user) {
        login_count++;
        std::cout << "Total logins: " << login_count << std::endl;
    });

    bus.Publish("user.login", "Alice");
    bus.Publish("user.login", "Bob");
    return 0;
}
```

### 场景二：泛型算法管道（模板 + Lambda）

```cpp
#include <algorithm>
#include <vector>
#include <numeric>

template <typename Container, typename Predicate>
auto Filter(const Container& src, Predicate pred) {
    Container result;
    std::copy_if(src.begin(), src.end(), std::back_inserter(result), pred);
    return result;
}

template <typename Container, typename Transform>
auto Map(const Container& src, Transform fn) {
    using ResultType = std::decay_t<decltype(fn(*src.begin()))>;
    std::vector<ResultType> result;
    result.reserve(src.size());
    std::transform(src.begin(), src.end(), std::back_inserter(result), fn);
    return result;
}

int main() {
    std::vector<int> numbers = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    // 函数式管道: 过滤偶数 → 平方 → 求和
    auto evens = Filter(numbers, [](int n) { return n % 2 == 0; });
    auto squares = Map(evens, [](int n) { return n * n; });
    int sum = std::accumulate(squares.begin(), squares.end(), 0);
    // 4 + 16 + 36 + 64 + 100 = 220
    std::cout << sum << std::endl;
    return 0;
}
```

### 场景三：策略模式（模板 vs 虚函数）

```cpp
// 方案一：虚函数——运行时切换策略
class SortStrategy {
public:
    virtual ~SortStrategy() = default;
    virtual void Sort(std::vector<int>& data) = 0;
};

class QuickSortStrategy : public SortStrategy {
public:
    void Sort(std::vector<int>& data) override { /* 快排 */ }
};

// 方案二：模板——编译时确定策略，零开销
struct QuickSort { void operator()(std::vector<int>& data) { /* 快排 */ } };
struct MergeSort { void operator()(std::vector<int>& data) { /* 归并排序 */ } };

template <typename Strategy>
class Sorter {
public:
    void Sort(std::vector<int>& data) { strategy_(data); }
private:
    Strategy strategy_;
};

int main() {
    // 运行时策略 → 虚函数
    std::unique_ptr<SortStrategy> s = std::make_unique<QuickSortStrategy>();

    // 编译时策略 → 模板（更快，策略被内联）
    Sorter<QuickSort> fast_sorter;
    return 0;
}
```

### 场景四：自定义迭代器（模板 + 类型推导）

```cpp
template <typename Iterator, typename Predicate>
class FilterIterator {
public:
    // 类型别名——标准库迭代器接口要求
    using iterator_category = std::forward_iterator_tag;
    using value_type = typename std::iterator_traits<Iterator>::value_type;
    using difference_type = std::ptrdiff_t;
    using pointer = const value_type*;
    using reference = const value_type&;

    FilterIterator(Iterator current, Iterator end, Predicate pred)
        : current_(current), end_(end), pred_(pred) {
        AdvanceToValid();
    }

    reference operator*() const { return *current_; }
    FilterIterator& operator++() { ++current_; AdvanceToValid(); return *this; }
    bool operator!=(const FilterIterator& other) const { return current_ != other.current_; }

private:
    void AdvanceToValid() {
        while (current_ != end_ && !pred_(*current_)) {
            ++current_;
        }
    }
    Iterator current_;
    Iterator end_;
    Predicate pred_;
};

// 辅助函数——利用模板参数推导简化创建
template <typename Container, typename Pred>
auto MakeFilterRange(Container& c, Pred pred) {
    auto begin = FilterIterator<decltype(c.begin()), Pred>(c.begin(), c.end(), pred);
    auto end = FilterIterator<decltype(c.begin()), Pred>(c.end(), c.end(), pred);
    return std::make_pair(begin, end);
}

int main() {
    std::vector<int> v = {1, 2, 3, 4, 5, 6};
    auto [begin, end] = MakeFilterRange(v, [](int n) { return n > 3; });
    for (auto it = begin; it != end; ++it) {
        std::cout << *it << " ";  // 4 5 6
    }
    return 0;
}
```

---

## 思想总结：为什么要这么多层抽象

把所有概念放在一起看，它们的演进逻辑是清晰的：

```
问题 1: 需要把"行为"传给函数
  → 函数指针（C 时代）
    → 无法携带状态 → 函数对象（Functor）
      → 每次写 class 太繁琐 → Lambda（编译器生成 Functor）

问题 2: 不同可调用对象的类型不同
  → std::function（类型擦除，统一接口）

问题 3: 代码逻辑相同但类型不同
  → 模板（编译时泛型）
    → 模板参数类型太复杂 → auto / decltype（类型推导）
      → 模板约束不够清晰 → Concepts（C++20）
```

| 层次 | 解决的问题 | 代价 |
|---|---|---|
| 函数指针 | 基本回调 | 无状态、不可内联 |
| 函数对象 | 带状态的回调 | 需手写 class |
| Lambda | 简化函数对象写法 | 无（语法糖） |
| `std::function` | 统一可调用对象类型 | 间接调用、堆分配 |
| 模板 | 对类型泛化 | 编译时间、代码膨胀 |
| 类型推导 | 简化模板类型表达 | 无 |
| Concepts | 约束模板参数 | C++20 |

> **核心思想**：C++ 的这些机制不是重复发明，而是解决不同抽象层次的问题。函数指针是底层机制，lambda 是语法便利，`std::function` 是类型统一，模板是编译时泛化，类型推导是易用性改进。每一层都建立在前一层之上，组合使用才能发挥最大威力。

记住几条实践规则：

1. **默认用 lambda**——简洁、高效、就地定义。
2. **需要存储回调时用 `std::function`**——统一类型、运行时多态。
3. **不要用 `std::bind`**——lambda 完全替代它。
4. **算法和库用模板**——零开销、编译时多态。
5. **能用 `auto` 就用 `auto`**——减少冗余、避免意外类型不匹配。
6. **返回值用 `decltype(auto)`**——完美转发返回类型（包括引用语义）。
