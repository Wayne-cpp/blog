---
title: "C++ std::vector 深度解析：最常用的容器背后做了什么"
date: 2021-08-09T10:00:00+08:00
tags: ["C++", "STL", "vector", "容器"]
categories: ["技术"]
summary: "从底层实现原理出发，深入剖析 std::vector 的内存管理策略、扩容机制、迭代器失效规则与性能特征。讲清楚 push_back、emplace_back、reserve、shrink_to_fit 等关键操作背后的设计决策，以及 vector 与其他序列容器的选择指南。"
ShowToc: true
---

如果你写 C++，你几乎一定用过 `std::vector`。它是 STL 中使用频率最高的容器，没有之一。Bjarne Stroustrup 本人多次建议："默认用 vector，除非你有充分的理由不用。"

这句话听起来简单，但 vector 背后的设计远比表面上复杂。为什么扩容倍数是 2？为什么 `push_back` 有时候会导致所有迭代器失效？`emplace_back` 到底比 `push_back` 快在哪里？`vector<bool>` 为什么是个特例？

这篇文章从底层实现出发，把这些问题一个个讲清楚。

## 内存布局：三根指针撑起一个容器

`std::vector` 的核心数据结构极其简洁：**三根指针**。

```cpp
template <typename T, typename Allocator = std::allocator<T>>
class vector {
public:
    // ... public interface
protected:
    iterator start;          // 指向已用内存的起始
    iterator finish;         // 指向已用内存的末尾（最后一个元素的下一位置）
    iterator end_of_storage; // 指向已分配内存的末尾
};
```

用 ASCII 图来表示：

```
start          finish     end_of_storage
  |               |             |
  v               v             v
+---+---+---+---+---+---+---+---+
| 0 | 1 | 2 | 3 | ? | ? | ? | ? |
+---+---+---+---+---+---+---+---+
  |<-  size()  ->|               |
  |<-        capacity()       ->|
```

从这三根指针可以推导出所有基本操作：

```cpp
size_type size() const {
    return size_type(finish - start);
}

size_type capacity() const {
    return size_type(end_of_storage - start);
}

bool empty() const {
    return start == finish;
}
```

没有额外的计数器，没有复杂的状态机。**减法就是全部**。这种设计带来了几个重要推论：

- `size()` 和 `capacity()` 都是 O(1)，而且几乎不可能出错
- 迭代器本质上就是原生指针（在大多数实现中）
- 内存是连续的，对 CPU 缓存极其友好

## 构造函数：六种创建 vector 的方式

C++11 之后，`std::vector` 提供了丰富的构造函数。理解每一种的使用场景，能帮你写出更清晰的代码。

```cpp
// 1. Default constructor: empty vector
std::vector<int> v1;

// 2. Fill constructor: n elements with value
std::vector<int> v2(10, 42);        // 10个42

// 3. Fill constructor: n default-constructed elements
std::vector<int> v3(10);            // 10个0

// 4. Range constructor: from iterators
std::list<int> lst = {1, 2, 3, 4, 5};
std::vector<int> v4(lst.begin(), lst.end());

// 5. Copy constructor
std::vector<int> v5(v4);

// 6. Move constructor
std::vector<int> v6(std::move(v5));  // v5 is now empty

// 7. Initializer list (C++11)
std::vector<int> v7 = {1, 2, 3, 4, 5};

// 8. From raw array
int arr[] = {10, 20, 30, 40};
std::vector<int> v8(std::begin(arr), std::end(arr));
```

一个容易忽略的细节：`vector<int> v(10)` 和 `vector<int> v{10}` 是**完全不同的**。前者创建 10 个默认值（0），后者创建 1 个元素（10）。花括号初始化会优先匹配 `initializer_list` 版本。

```cpp
std::vector<int> a(10);    // 10 elements, all 0
std::vector<int> b{10};    // 1 element: 10
std::vector<int> c(10, 5); // 10 elements, all 5
std::vector<int> d{10, 5}; // 2 elements: 10, 5
```

这个坑在代码审查中见过不止一次。

## Capacity 与 Size：两个容易混淆的概念

`size()` 是当前实际存储了多少个元素。`capacity()` 是已经分配的内存能装多少个元素。两者之间的差值就是**空闲槽位**，不需要重新分配内存就能直接使用。

```cpp
std::vector<int> v;
std::cout << v.size() << ", " << v.capacity() << "\n";  // 0, 0

v.push_back(1);
std::cout << v.size() << ", " << v.capacity() << "\n";  // 1, 1

v.push_back(2);
std::cout << v.size() << ", " << v.capacity() << "\n";  // 2, 2

v.push_back(3);
std::cout << v.size() << ", " << v.capacity() << "\n";  // 3, 4 (capacity doubled)
```

### reserve：提前分配，避免反复搬家

如果你大致知道要装多少元素，提前 `reserve` 可以避免多次扩容带来的拷贝开销。

```cpp
std::vector<int> v;
v.reserve(1000);  // allocate space for 1000 elements

for (int i = 0; i < 1000; ++i) {
    v.push_back(i);  // no reallocation happens
}
```

> `reserve(n)` 只改变 `capacity()`，不改变 `size()`。它不会创建任何新元素，也不会调用任何构造函数。

### resize：改变元素数量

`resize` 和 `reserve` 的区别在于，`resize` 会真正改变 `size()`。

```cpp
std::vector<int> v = {1, 2, 3};

v.resize(6);       // size=6, new elements are 0
// v = {1, 2, 3, 0, 0, 0}

v.resize(2);       // size=2, last 4 elements destroyed
// v = {1, 2}

v.resize(5, 99);   // size=5, new elements are 99
// v = {1, 2, 99, 99, 99}
```

### shrink_to_fit：归还多余的内存

`shrink_to_fit` 是一个**非绑定请求**，意思是实现可以忽略它。但在主流实现中（GCC libstdc++、Clang libc++、MSVC），它都会真正释放多余的内存。

```cpp
std::vector<int> v(1000);
std::cout << v.capacity() << "\n";  // 1000

v.resize(10);
std::cout << v.capacity() << "\n";  // still 1000

v.shrink_to_fit();
std::cout << v.capacity() << "\n";  // 10 (or close to it)
```

注意，`shrink_to_fit` 通常会分配一块恰好够用的新内存，把元素搬过去，然后释放旧内存。所以它本身也不便宜，别在循环里调用。

---

## 扩容策略：为什么是 2 倍

当 `push_back` 发现 `size() == capacity()` 时，就需要扩容了。主流实现的策略是 **按 2 倍增长**（MSVC 用 1.5 倍，GCC 和 Clang 用 2 倍）。

扩容的具体步骤：

1. 分配一块更大的内存（通常 `new_capacity = max(old_capacity * 2, required_size)`）
2. 把旧元素搬到新内存（move 或 copy）
3. 释放旧内存
4. 更新三根指针

```cpp
void push_back(const T& value) {
    if (finish != end_of_storage) {
        // still have room, construct in place
        construct(finish, value);
        ++finish;
    } else {
        // need to reallocate
        insert_aux(end(), value);
    }
}
```

为什么偏偏选 2 倍而不是 3 倍或者 1.5 倍？这背后有一个数学上的考量。

假设我们连续 `push_back` 了 n 个元素，总的拷贝次数大约是：

```
1 + 2 + 4 + 8 + ... + n/2 + n ≈ 2n
```

也就是说，n 次 `push_back` 的均摊时间复杂度是 **O(1)**。均摊的意思是：单次操作可能很慢（触发扩容时），但把总开销均摊到每次操作上，就是常数时间。

至于 2 倍 vs 1.5 倍的争议：2 倍的增长速度更快，扩容次数更少，但内存浪费可能更大。Facebook 的 `folly::fbvector` 用了 1.5 倍，理由是在某些内存分配器下，1.5 倍能更好地复用之前释放的内存块。

对大多数应用来说，这个差异可以忽略。

---

## 元素访问：五种方式

```cpp
std::vector<int> v = {10, 20, 30, 40, 50};
```

| 方法 | 边界检查 | 可用场景 | 失败行为 |
|------|---------|---------|---------|
| `v[i]` | 无 | 确定索引合法时 | 未定义行为（UB） |
| `v.at(i)` | 有 | 需要安全访问时 | 抛出 `std::out_of_range` |
| `v.front()` | 无 | 访问首元素 | 空 vector 上是 UB |
| `v.back()` | 无 | 访问末元素 | 空 vector 上是 UB |
| `v.data()` | 无 | 需要 C 兼容指针 | 空 vector 返回可能为 nullptr |

```cpp
// operator[] - no bounds check, fastest
int first = v[0];

// at() - throws std::out_of_range if out of bounds
try {
    int val = v.at(100);  // throws!
} catch (const std::out_of_range& e) {
    std::cerr << e.what() << "\n";
}

// front() and back() - undefined if empty
int head = v.front();  // equivalent to v[0]
int tail = v.back();   // equivalent to v[v.size() - 1]

// data() - raw pointer to the underlying array
int* ptr = v.data();
// can pass to C APIs expecting int*
```

> `data()` 在 C++11 被正式加入。在此之前，人们用 `&v[0]` 或 `&v.front()` 来获取底层指针。但空 vector 上 `&v[0]` 是 UB，而 `data()` 对空 vector 返回的是一个合法的（可能为 null 的）指针。

---

## 修改操作：增删改

### push_back 与 pop_back

```cpp
std::vector<std::string> words;

// push_back: copy or move an element to the end
std::string s = "hello";
words.push_back(s);             // copy s into vector
words.push_back(std::move(s));  // move s into vector, s is now empty

// pop_back: remove last element
words.pop_back();               // destroys the last element, no return value
```

`pop_back` 不返回被删除元素的值。这不是设计疏忽，而是**异常安全**的考量。如果 `pop_back` 返回值，而值拷贝时抛出异常，元素就丢了：既不在 vector 里，也没成功返回给调用者。所以 C++ 的做法是先访问、再删除：

```cpp
std::string last = words.back();  // get it first
words.pop_back();                 // then remove it
```

### insert 与 emplace

`insert` 在指定位置插入元素，`emplace` 在指定位置**原位构造**元素。

```cpp
std::vector<int> v = {10, 20, 30};

// insert single element
auto it = v.insert(v.begin() + 1, 15);
// v = {10, 15, 20, 30}

// insert multiple copies
v.insert(v.begin(), 3, 0);
// v = {0, 0, 0, 10, 15, 20, 30}

// insert range
std::vector<int> extra = {100, 200};
v.insert(v.end(), extra.begin(), extra.end());
// v = {0, 0, 0, 10, 15, 20, 30, 100, 200}

// emplace: construct in place
struct Point {
    int x, y;
    Point(int x_, int y_) : x(x_), y(y_) {}
};
std::vector<Point> pts;
pts.emplace(pts.begin(), 3, 4);  // calls Point(3, 4) in place
```

### erase

```cpp
std::vector<int> v = {1, 2, 3, 4, 5};

// erase single element
auto it = v.erase(v.begin() + 2);
// v = {1, 2, 4, 5}, it points to 4

// erase range
v.erase(v.begin(), v.begin() + 2);
// v = {4, 5}
```

`erase` 的经典用法是配合 `remove_if` 实现"删除满足条件的元素"，即 **erase-remove 惯用法**：

```cpp
std::vector<int> v = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

// remove all even numbers
v.erase(
    std::remove_if(v.begin(), v.end(),
                   [](int x) { return x % 2 == 0; }),
    v.end()
);
// v = {1, 3, 5, 7, 9}
```

为什么不直接在循环里 `erase`？因为 `erase` 会使后面的迭代器全部失效（后面会详细讲），而 `remove_if` 只做逻辑移动，不改变容器大小，最后一次性 `erase` 掉尾部元素。

### clear 与 swap

```cpp
std::vector<int> v = {1, 2, 3, 4, 5};

// clear: destroy all elements, size becomes 0
// capacity is NOT changed
v.clear();
std::cout << v.size() << ", " << v.capacity() << "\n";  // 0, 5

// swap trick to truly release memory
std::vector<int>().swap(v);
// v is now completely empty, capacity is 0

// swap two vectors (O(1), just swaps internal pointers)
std::vector<int> a = {1, 2, 3};
std::vector<int> b = {4, 5};
a.swap(b);
// a = {4, 5}, b = {1, 2, 3}
```

`swap` 是 O(1) 的，因为它只交换三根指针。这也是"copy-and-swap"惯用法的基础。

---

## emplace_back vs push_back：到底快在哪

这是面试高频题，也是实际开发中容易用错的点。

`push_back` 接受一个已经构造好的对象（或者可以隐式转换的对象），然后把它拷贝或移动到 vector 中。`emplace_back` 接受构造参数，**直接在 vector 的内存中原位构造**对象。

```cpp
struct User {
    std::string name;
    int age;
    User(std::string n, int a) : name(std::move(n)), age(a) {
        std::cout << "construct: " << name << "\n";
    }
    User(const User& other) : name(other.name), age(other.age) {
        std::cout << "copy: " << name << "\n";
    }
    User(User&& other) noexcept
        : name(std::move(other.name)), age(other.age) {
        std::cout << "move: " << name << "\n";
    }
    ~User() {
        std::cout << "destruct: " << name << "\n";
    }
};

std::vector<User> users;

// push_back: construct temporary, then move into vector
users.push_back(User("Alice", 25));
// Output: construct: Alice, move: Alice, destruct: Alice

// emplace_back: construct directly in vector's memory
users.emplace_back("Bob", 30);
// Output: construct: Bob
```

对比一下：

- `push_back(User("Alice", 25))`：先构造临时对象（一次构造），再移动到 vector 中（一次移动），最后析构临时对象（一次析构）。总共三次操作。
- `emplace_back("Bob", 30)`：直接在 vector 的内存中调用构造函数。总共一次操作。

省了一次移动和一次析构。对于简单类型（如 `int`），这没区别。但对于包含动态内存的类型（如 `std::string`），差异可能很显著。

> 不过要注意：如果传入的参数本身就是vector元素类型的对象，两者没有性能差异。`v.push_back(x)` 和 `v.emplace_back(x)` 在这种情况下做的事完全一样。

还有一个容易忽略的区别：`emplace_back` 能调用**显式构造函数**，而 `push_back` 不能。

```cpp
struct ExplicitInt {
    explicit ExplicitInt(int val) : value(val) {}
    int value;
};

std::vector<ExplicitInt> v;
v.push_back(42);      // compile error: implicit conversion
v.emplace_back(42);   // OK: calls explicit constructor directly
```

---

## 迭代器失效：最容易踩的坑

vector 的迭代器本质上（在大多数实现中）就是原生指针。一旦发生**重新分配**，旧的指针就指向了已经释放的内存。这就是迭代器失效。

规则不算复杂，但需要记住：

| 操作 | 是否导致失效 | 条件 |
|------|-------------|------|
| `push_back` | **是** | 如果 `size() == capacity()`（触发扩容） |
| `push_back` | 否 | 如果 `size() < capacity()`（不扩容） |
| `emplace_back` | 同 `push_back` | 取决于是否扩容 |
| `insert` / `emplace` | **是** | 插入点之后的所有迭代器失效；如果触发扩容，全部失效 |
| `erase` | **是** | 被删除元素及其之后的所有迭代器失效 |
| `pop_back` | **是** | 仅指向末尾元素的迭代器失效 |
| `clear` | **是** | 全部失效 |
| `reserve` | **是** | 如果 `new_cap > capacity()`，全部失效 |
| `resize` | **是** | 如果触发扩容，全部失效 |
| `swap` | **是** | 两个 vector 的迭代器全部交换归属 |

最典型的 bug 是在循环中一边遍历一边删除：

```cpp
// WRONG: undefined behavior
std::vector<int> v = {1, 2, 3, 4, 5};
for (auto it = v.begin(); it != v.end(); ++it) {
    if (*it == 3) {
        v.erase(it);  // it is now invalid!
    }
    // ++it on invalid iterator: UB
}

// CORRECT: use erase's return value
for (auto it = v.begin(); it != v.end(); ) {
    if (*it == 3) {
        it = v.erase(it);  // erase returns iterator to next element
    } else {
        ++it;
    }
}
```

或者直接用 erase-remove：

```cpp
v.erase(std::remove(v.begin(), v.end(), 3), v.end());
```

另一个常见场景：`push_back` 导致扩容，之前的迭代器全部作废。

```cpp
std::vector<int> v = {1, 2, 3};
auto it = v.begin();
v.push_back(4);  // may trigger reallocation
std::cout << *it;  // UB! it may point to freed memory
```

> 如果你需要保留指向 vector 元素的引用或迭代器，先确保 `capacity()` 足够大（通过 `reserve`），或者在每次可能扩容的操作之后重新获取。

---

## 性能一览

把所有操作的时间复杂度放在一起看：

| 操作 | 时间复杂度 | 备注 |
|------|-----------|------|
| 随机访问 `v[i]` | O(1) | 等价于指针偏移 |
| `push_back` | 均摊 O(1) | 最坏情况 O(n)（扩容时） |
| `emplace_back` | 均摊 O(1) | 同上，但省去移动/拷贝 |
| `pop_back` | O(1) | 只析构，不释放内存 |
| `insert(pos)` | O(n) | pos 之后的元素需要移动 |
| `erase(pos)` | O(n) | pos 之后的元素需要移动 |
| `reserve(n)` | O(n) | 如果 n > 当前 capacity |
| `resize(n)` | O(n) | 如果需要构造或析构元素 |
| `clear` | O(n) | 析构所有元素 |
| `swap` | O(1) | 只交换三根指针 |
| `size()` / `capacity()` | O(1) | 指针减法 |
| `shrink_to_fit` | O(n) | 重新分配并搬移 |

关键洞察：**vector 的随机访问是 O(1)，但在中间插入/删除是 O(n)**。如果你需要频繁在头部插入，`std::deque` 或 `std::list` 可能更合适。

---

## vector\<bool\>：一个特殊的特化

`std::vector<bool>` 是标准库中一个争议极大的特化版本。为了节省内存，它把 8 个 `bool` 打包进一个字节里。

```cpp
std::vector<bool> bv = {true, false, true, true};

// each element is only 1 bit, not 1 byte
std::cout << sizeof(std::vector<bool>) << "\n";  // implementation-defined

// but you can't take address of an element
// bool* p = &bv[0];  // compile error!
```

这带来了一堆问题：

1. **`operator[]` 不返回 `bool&`**，而是返回一个代理对象 `std::vector<bool>::reference`。你无法对其取地址。
2. **不能和 C API 互操作**。`data()` 方法根本不存在。
3. **并发访问不安全**。修改相邻位会写到同一个字节，产生数据竞争。
4. **和泛型代码不兼容**。很多期望 `vector<T>` 行为一致的模板代码在 `T=bool` 时会出错。

```cpp
// This compiles for vector<int> but NOT for vector<bool>
template <typename T>
void ProcessFirst(std::vector<T>& v) {
    T& first = v[0];   // error when T=bool
    // ...
}
```

如果你的代码需要一个真正的 `bool` 数组，有两个替代方案：

```cpp
// Option 1: use deque (no specialization)
std::deque<bool> dv = {true, false, true};

// Option 2: use a wrapper type
struct Bool {
    bool value;
};
std::vector<Bool> bv = {{true}, {false}, {true}};

// Option 3: use vector<char> or vector<uint8_t>
std::vector<uint8_t> bv = {1, 0, 1};
```

> Effective C++ 条款 18 建议避免使用 `vector<bool>`。如果你不需要位压缩，用 `deque<bool>` 或 `vector<char>` 是更安全的选择。

---

## 常见陷阱

### 1. 下标越界不报错

```cpp
std::vector<int> v = {1, 2, 3};
v[100] = 42;  // UB: silent corruption or crash
```

`operator[]` 不做边界检查。在 Debug 模式下（GCC 的 `_GLIBCXX_DEBUG`，MSVC 的 Debug 模式）可能会断言失败，但在 Release 模式下就是 UB。如果不确定索引合法，用 `at()`。

### 2. 空 vector 上调用 front/back

```cpp
std::vector<int> v;
int x = v.front();  // UB: dereferencing end iterator
```

### 3. 保存了 size() 再修改容器

```cpp
std::vector<int> v = {1, 2, 3};
size_t old_size = v.size();
v.push_back(4);
for (size_t i = old_size; i < v.size(); ++i) {
    // this is fine, but...
}

// The classic mistake:
for (size_t i = 0; i < v.size(); ++i) {
    if (v[i] == 2) {
        v.erase(v.begin() + i);
        // don't increment i! you just skipped an element
    }
}
```

### 4. 忽略返回值

```cpp
std::vector<int> v = {1, 2, 3, 4, 5};
v.reserve(100);  // OK, but don't assume size changed
// v.size() is still 5, v[10] is still UB

v.shrink_to_fit();  // non-binding request
// capacity might not equal size() exactly
```

### 5. 在 range-for 中修改容器

```cpp
std::vector<int> v = {1, 2, 3, 4, 5};

// WRONG: modifying container during range-for
for (auto x : v) {
    if (x == 3) {
        v.push_back(6);  // may invalidate the internal iterator
    }
}

// range-for is syntactic sugar for:
// for (auto it = v.begin(); it != v.end(); ++it) { ... }
// push_back may cause reallocation, begin/end change
```

### 6. reserve 代替 resize

```cpp
std::vector<int> v;
v.reserve(10);
// v[0] = 42;  // UB! reserve doesn't create elements

v.resize(10);
v[0] = 42;     // OK, element exists
```

### 7. 忘记移动语义

```cpp
std::vector<std::string> names;

std::string name = GetName();
names.push_back(name);              // copy
names.push_back(std::move(name));   // move, name is now empty

// if you don't need name afterwards, always move
names.emplace_back(GetName());      // even better: construct in place
```

---

## 总结

`std::vector` 之所以是"默认容器"，不是因为它什么都能做，而是因为它在**最常见的使用模式**（尾部追加、随机访问、顺序遍历）下性能最优。连续内存意味着 CPU 缓存友好，指针运算意味着随机访问 O(1)，而均摊分析保证了尾部插入也是常数时间。

选择容器的简单决策树：

| 需求 | 推荐容器 |
|------|---------|
| 尾部追加，随机访问 | `std::vector` |
| 头尾都需要追加 | `std::deque` |
| 频繁在中间插入/删除 | `std::list`（但先确认真的需要） |
| 频繁查找 | `std::unordered_set` / `std::unordered_map` |
| 需要有序 | `std::set` / `std::map` |
| 固定大小，栈上分配 | `std::array` |

几个要点回顾：

- 底层是**三根指针**，一切操作都建立在指针算术之上
- **2 倍扩容**保证 `push_back` 均摊 O(1)
- `emplace_back` 在元素类型含动态内存时有实际优势
- 迭代器失效的根本原因是**内存重新分配**和**元素移动**
- `vector<bool>` 是个坑，能用 `deque<bool>` 替代就替代
- 不确定索引合法就用 `at()`，不确定容器是否为空就先检查

把 vector 用对不难，但把它用对用好，需要对内存模型和对象生命周期有清晰的理解。希望这篇文章帮你建立了这种理解。
