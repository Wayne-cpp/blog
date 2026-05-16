---
title: "C++ STL 概览与迭代器：标准模板库的设计哲学"
date: 2021-08-08T10:00:00+08:00
tags: ["C++", "STL", "迭代器", "标准库"]
categories: ["C++"]
summary: "系统介绍 C++ STL 的六大组件（容器、迭代器、算法、函数对象、适配器、分配器）的设计哲学与协作关系，深入剖析迭代器的分类、失效规则与自定义迭代器的实现方法。"
ShowToc: true
---

STL 的核心愿景很简单：**写一次算法，让它在任何类型上都能工作**。

这不是一句空话。C++ 标准模板库的设计者 Alexander Stepanov 用泛型编程（generic programming）的思想，把"类型"从算法中彻底剥离出来。你不需要为 `int` 写一个 `sort`，再为 `string` 写一个 `sort`，再为自定义结构体写一个 `sort`。一套 `std::sort`，只要你的类型满足特定要求，它就能用。

这一切的根基，是 STL 的六大组件之间精密的协作关系。其中，迭代器（iterator）扮演着最关键的"胶水"角色：它把容器和算法连接起来，让算法不需要知道容器的具体实现，容器也不需要知道算法的存在。

这篇文章会先鸟瞰 STL 的整体架构，然后把镜头聚焦到迭代器上，从分类、使用、失效规则，一路讲到自定义迭代器的完整实现。

---

## STL 的六大组件

STL 由六个互相配合的组件构成。每个组件只负责一件事情，但它们组合在一起，就能覆盖绝大多数数据结构和算法的需求。

### 容器（Container）

容器负责存储数据。STL 提供了两大类容器：

- **序列容器**（sequence container）：`std::vector`、`std::deque`、`std::list`、`std::forward_list`、`std::array`。数据按插入顺序排列。
- **关联容器**（associative container）：`std::map`、`std::set`、`std::multimap`、`std::multiset`。数据按键自动排序。
- **无序关联容器**（unordered associative container）：`std::unordered_map`、`std::unordered_set` 等。基于哈希表，不保证顺序。

容器只管存数据。它不提供排序、查找、变换等算法操作。

### 迭代器（Iterator）

迭代器是访问容器元素的统一接口。它是指针的泛化（generalization），提供了类似指针的操作（`*`、`++`、`->`），但能适用于任何容器。

迭代器是容器和算法之间的桥梁。算法通过迭代器访问数据，完全不需要知道底层的容器类型。

### 算法（Algorithm）

`<algorithm>` 头文件提供了几十个通用算法：`std::sort`、`std::find`、`std::copy`、`std::transform`、`std::accumulate` 等等。这些算法全部通过迭代器操作数据，与具体容器解耦。

### 函数对象（Functor）

函数对象是重载了 `operator()` 的类，可以像函数一样调用。它比普通函数指针更灵活：可以携带状态，可以被内联优化。STL 中的 `std::greater`、`std::less` 就是典型的函数对象。C++11 之后，lambda 表达式在大多数场景下取代了手写函数对象，但底层机制是一样的。

### 适配器（Adapter）

适配器是对现有组件的接口包装，改变其行为以满足不同需求：

- **容器适配器**：`std::stack`、`std::queue`、`std::priority_queue`。它们不是独立的容器，而是对 `deque` 或 `vector` 的封装。
- **迭代器适配器**：`std::reverse_iterator`、`std::back_insert_iterator`、`std::istream_iterator`。
- **函数适配器**：`std::not_fn`、`std::bind`。

### 分配器（Allocator）

分配器封装了内存分配和释放的细节。每个 STL 容器都有一个默认的分配器（`std::allocator`），你可以自定义分配器来实现内存池、共享内存等特殊策略。大多数情况下你不需要碰它，但它是容器和底层内存系统之间的抽象层。

### 六大组件的协作关系

下面这张 ASCII 图展示了六大组件之间的关系：

```
   +-----------+      Iterator      +-----------+
   |           | <================> |           |
   | Container |                    | Algorithm |
   |           | -----.             |           |
   +-----------+      |             +-----------+
        |             |                  ^
        v             |                  |
   +-----------+      |           +-----------+
   |           |      |           |           |
   | Allocator |      |           |  Functor  |
   |           |      |           |           |
   +-----------+      |           +-----------+
                      |                  ^
                      |                  |
                      v                  |
                +-----------+     +-----------+
                |           |     |           |
                |  Adapter  | --> | Container |
                |           |     |  Iterator |
                +-----------+     |  Functor  |
                                  +-----------+
```

核心数据流：**Container** 通过 **Iterator** 把数据暴露给 **Algorithm**，**Algorithm** 使用 **Functor** 定制行为，**Allocator** 在底层默默管理内存，**Adapter** 可以包装前三者来改变接口。

---

## 迭代器：容器与算法的桥梁

### 迭代器是什么

迭代器是指针的泛化。普通指针就是一种迭代器，但迭代器不一定是指针。

考虑这个场景：你要遍历一个容器中的元素。如果是数组，你用指针：

```cpp
int arr[] = {1, 2, 3, 4, 5};
for (int* ptr = arr; ptr != arr + 5; ++ptr) {
    std::cout << *ptr << " ";
}
```

如果是 `std::vector`，你用迭代器：

```cpp
std::vector<int> vec = {1, 2, 3, 4, 5};
for (auto it = vec.begin(); it != vec.end(); ++it) {
    std::cout << *it << " ";
}
```

如果是 `std::list`，你同样用迭代器：

```cpp
std::list<int> lst = {1, 2, 3, 4, 5};
for (auto it = lst.begin(); it != lst.end(); ++it) {
    std::cout << *it << " ";
}
```

三种不同的数据结构，三种不同的内存布局，但遍历的代码几乎一模一样。这就是迭代器的威力。算法只需要知道"怎么前进"（`++`）和"怎么读值"（`*`），完全不需要关心底层是连续内存还是链表节点。

> 迭代器的设计哲学：**提供统一的访问接口，隐藏容器的内部实现。**

### 迭代器的分类

不是所有迭代器都能做相同的事情。`std::list` 的迭代器能前进和后退，但不能像 `std::vector` 的迭代器那样随机跳跃。STL 把迭代器分成几个层次，能力从弱到强：

| 类别 | 能读 | 能写 | 能前进 | 能后退 | 能随机跳 | 能比较 |
|------|------|------|--------|--------|----------|--------|
| **InputIterator** | Yes | No | Yes | No | No | `==`/`!=` |
| **OutputIterator** | No | Yes | Yes | No | No | No |
| **ForwardIterator** | Yes | Yes | Yes | No | No | `==`/`!=` |
| **BidirectionalIterator** | Yes | Yes | Yes | Yes | No | `==`/`!=` |
| **RandomAccessIterator** | Yes | Yes | Yes | Yes | Yes | `==`/`!=`/`<`/`>` |
| **ContiguousIterator** (C++20) | Yes | Yes | Yes | Yes | Yes | `==`/`!=`/`<`/`>` |

> **ContiguousIterator** 是 C++20 新增的。它在 `RandomAccessIterator` 的基础上额外保证：元素在内存中是连续存放的。这意味着迭代器可以安全地转换为裸指针。`std::vector`、`std::string`、`std::array` 的迭代器都属于这一类。

这些分类之间是**递进关系**：后一种迭代器包含了前一种的所有能力。就像继承体系一样：

```
ContiguousIterator
  └── RandomAccessIterator
        └── BidirectionalIterator
              └── ForwardIterator
                    ├── InputIterator
                    └── OutputIterator
```

每种容器提供的迭代器类别是固定的。算法会根据自己的需要，在接口中声明要求的最低迭代器类别。比如：

- `std::find` 需要 **InputIterator**（只需要遍历和读取）
- `std::reverse` 需要 **BidirectionalIterator**（需要能后退）
- `std::sort` 需要 **RandomAccessIterator**（需要能随机跳跃来做快速排序）

如果你传了一个能力不足的迭代器给算法，编译器会报错。这种错误在编译期就能被捕获，不会拖到运行时。

### begin / end 家族

STL 提供了一组函数来获取容器的迭代器：

```cpp
std::vector<int> vec = {1, 2, 3, 4, 5};

// Forward iteration
for (auto it = vec.begin(); it != vec.end(); ++it) {
    std::cout << *it << " ";
}
// Output: 1 2 3 4 5

// Const forward iteration (read-only)
for (auto it = vec.cbegin(); it != vec.cend(); ++it) {
    std::cout << *it << " ";
    // *it = 10;  // Compile error: read-only
}

// Reverse iteration
for (auto it = vec.rbegin(); it != vec.rend(); ++it) {
    std::cout << *it << " ";
}
// Output: 5 4 3 2 1

// Const reverse iteration (read-only)
for (auto it = vec.crbegin(); it != vec.crend(); ++it) {
    std::cout << *it << " ";
}
// Output: 5 4 3 2 1
```

C++11 还引入了 `std::begin()` 和 `std::end()` 自由函数，它们不仅适用于 STL 容器，也适用于原生数组：

```cpp
int arr[] = {10, 20, 30, 40, 50};

// Works with plain arrays
auto it = std::begin(arr);  // equivalent to arr
auto end = std::end(arr);   // equivalent to arr + 5

while (it != end) {
    std::cout << *it << " ";
    ++it;
}
// Output: 10 20 30 40 50
```

下面是完整的迭代器获取函数对照表：

| 函数 | 方向 | 可变性 | 适用范围 |
|------|------|--------|----------|
| `c.begin()` / `std::begin(c)` | 正向 | 可读写 | STL 容器 + 原生数组 |
| `c.end()` / `std::end(c)` | 正向（末尾之后） | 可读写 | STL 容器 + 原生数组 |
| `c.cbegin()` | 正向 | 只读 | STL 容器 |
| `c.cend()` | 正向（末尾之后） | 只读 | STL 容器 |
| `c.rbegin()` | 反向 | 可读写 | 支持双向迭代器的容器 |
| `c.rend()` | 反向（起点之前） | 可读写 | 支持双向迭代器的容器 |
| `c.crbegin()` | 反向 | 只读 | 支持双向迭代器的容器 |
| `c.crend()` | 反向（起点之前） | 只读 | 支持双向迭代器的容器 |

> **注意**：`end()` 返回的迭代器指向的是最后一个元素**之后**的位置（past-the-end），不要对这个位置解引用。它是作为终止条件使用的，永远不要访问 `*end()`。

### 迭代器失效规则

迭代器失效是 C++ 编程中最容易踩的坑之一。当容器的内部结构发生变化时（比如插入、删除元素），之前获取的迭代器可能不再有效。使用失效的迭代器会导致**未定义行为**（undefined behavior），程序可能崩溃，可能看起来正常运行，也可能产生难以调试的间歇性错误。

> **铁律**：只要容器的结构发生了变化，就要假设之前的迭代器已经失效，除非你确切知道该容器在该操作下的失效规则。

下面是各主要容器的迭代器失效规则：

| 容器 | 插入操作 | 删除操作 |
|------|----------|----------|
| `std::vector` | 若引起 `capacity` 变化（重新分配），所有迭代器失效；否则仅插入点之后的迭代器失效 | 被删除元素及之后的迭代器失效 |
| `std::deque` | 在首尾插入：所有迭代器失效；在中间插入：所有迭代器失效 | 删除首元素：仅指向首元素的迭代器失效；删除尾元素：仅指向尾元素的迭代器失效（但过去的 `end()` 失效）；删除中间：所有迭代器失效 |
| `std::list` | 不会导致任何迭代器失效 | 仅指向被删除元素的迭代器失效 |
| `std::map` / `std::set` | 不会导致任何迭代器失效 | 仅指向被删除元素的迭代器失效 |
| `std::unordered_map` / `std::unordered_set` | 若引起 `rehash`，所有迭代器失效；否则不受影响 | 仅指向被删除元素的迭代器失效 |

链表和关联容器（`list`、`map`、`set`）的迭代器非常稳定，因为它们的节点在内存中是独立分配的，插入和删除不会影响其他节点的位置。这也是为什么删除 `list` 中的元素可以用一个经典的循环模式：

```cpp
std::list<int> lst = {1, 2, 3, 4, 5, 6};

// Safe erase-while-iterating pattern for list
for (auto it = lst.begin(); it != lst.end(); ) {
    if (*it % 2 == 0) {
        it = lst.erase(it);  // erase returns next valid iterator
    } else {
        ++it;
    }
}
// lst is now {1, 3, 5}
```

但 `vector` 就不一样了：

```cpp
std::vector<int> vec = {1, 2, 3, 4, 5};

// WRONG! Undefined behavior after first erase
for (auto it = vec.begin(); it != vec.end(); ++it) {
    if (*it == 3) {
        vec.erase(it);  // it is now invalidated!
    }
}

// Correct approach: use the returned iterator
for (auto it = vec.begin(); it != vec.end(); ) {
    if (*it == 3) {
        it = vec.erase(it);  // erase returns iterator to next element
    } else {
        ++it;
    }
}
```

更推荐的做法是使用 **erase-remove idiom**：

```cpp
#include <algorithm>

std::vector<int> vec = {1, 2, 3, 4, 5};

// Erase-remove idiom (the standard way)
vec.erase(
    std::remove(vec.begin(), vec.end(), 3),
    vec.end()
);
// vec is now {1, 2, 4, 5}
```

> **建议**：如果你需要在遍历过程中频繁插入或删除元素，优先考虑 `std::list` 或 `std::map`，它们的迭代器更稳定。如果必须用 `std::vector`，确保理解失效规则，或使用 erase-remove idiom。

---

## 自定义迭代器的实现

理解了迭代器的分类和用法之后，我们来动手实现一个自定义迭代器。这是一个学习迭代器的最佳方式。

假设我们要实现一个简单的固定大小整数容器 `IntBuffer`，并为它提供正向迭代器支持。

### 第一步：容器骨架

先搭好容器的框架：

```cpp
#include <cstddef>
#include <stdexcept>
#include <iterator>
#include <iostream>
#include <algorithm>

// A simple fixed-size integer buffer
class IntBuffer {
public:
    static const std::size_t kMaxSize = 100;

    IntBuffer() : size_(0) {}

    void PushBack(int value) {
        if (size_ >= kMaxSize) {
            throw std::out_of_range("IntBuffer is full");
        }
        data_[size_] = value;
        ++size_;
    }

    std::size_t Size() const { return size_; }

    int& operator[](std::size_t index) { return data_[index]; }
    const int& operator[](std::size_t index) const { return data_[index]; }

    // Forward declaration of iterator
    class Iterator;
    class ConstIterator;

    // Will add begin/end later

private:
    int data_[kMaxSize];
    std::size_t size_;
};
```

### 第二步：迭代器实现

下面是完整的迭代器实现。我们按照 STL 的惯例，提供 `iterator_traits` 需要的所有类型别名：

```cpp
class IntBuffer::Iterator {
public:
    // Required type aliases for iterator_traits
    using iterator_category = std::forward_iterator_tag;
    using value_type        = int;
    using difference_type   = std::ptrdiff_t;
    using pointer           = int*;
    using reference         = int&;

    Iterator() : ptr_(nullptr) {}
    explicit Iterator(pointer ptr) : ptr_(ptr) {}

    // Dereference
    reference operator*() const { return *ptr_; }
    pointer operator->() const { return ptr_; }

    // Pre-increment
    Iterator& operator++() {
        ++ptr_;
        return *this;
    }

    // Post-increment
    Iterator operator++(int) {
        Iterator tmp = *this;
        ++ptr_;
        return tmp;
    }

    // Equality comparison
    bool operator==(const Iterator& other) const {
        return ptr_ == other.ptr_;
    }

    bool operator!=(const Iterator& other) const {
        return ptr_ != other.ptr_;
    }

private:
    pointer ptr_;
};
```

然后是只读版本的 `ConstIterator`：

```cpp
class IntBuffer::ConstIterator {
public:
    // Required type aliases for iterator_traits
    using iterator_category = std::forward_iterator_tag;
    using value_type        = int;
    using difference_type   = std::ptrdiff_t;
    using pointer           = const int*;
    using reference         = const int&;

    ConstIterator() : ptr_(nullptr) {}
    explicit ConstIterator(pointer ptr) : ptr_(ptr) {}

    // Implicit conversion from Iterator to ConstIterator
    ConstIterator(const Iterator& other) : ptr_(other.operator->()) {}

    // Dereference (read-only)
    reference operator*() const { return *ptr_; }
    pointer operator->() const { return ptr_; }

    // Pre-increment
    ConstIterator& operator++() {
        ++ptr_;
        return *this;
    }

    // Post-increment
    ConstIterator operator++(int) {
        ConstIterator tmp = *this;
        ++ptr_;
        return tmp;
    }

    // Equality comparison
    bool operator==(const ConstIterator& other) const {
        return ptr_ == other.ptr_;
    }

    bool operator!=(const ConstIterator& other) const {
        return ptr_ != other.ptr_;
    }

private:
    pointer ptr_;
};
```

### 第三步：给容器添加 begin/end

回到 `IntBuffer` 类，添加迭代器获取方法：

```cpp
class IntBuffer {
public:
    // ... previous code ...

    // Iterator accessors
    Iterator begin() { return Iterator(data_); }
    Iterator end() { return Iterator(data_ + size_); }

    ConstIterator begin() const { return ConstIterator(data_); }
    ConstIterator end() const { return ConstIterator(data_ + size_); }

    ConstIterator cbegin() const { return ConstIterator(data_); }
    ConstIterator cend() const { return ConstIterator(data_ + size_); }

    // ... rest of the class ...
};
```

### 第四步：使用自定义迭代器

现在我们的 `IntBuffer` 可以和 STL 算法一起使用了：

```cpp
int main() {
    IntBuffer buf;
    buf.PushBack(5);
    buf.PushBack(2);
    buf.PushBack(8);
    buf.PushBack(1);
    buf.PushBack(9);

    // Range-based for loop works!
    std::cout << "Original: ";
    for (int val : buf) {
        std::cout << val << " ";
    }
    std::cout << "\n";
    // Output: Original: 5 2 8 1 9

    // STL algorithms work!
    std::sort(buf.begin(), buf.end());

    std::cout << "Sorted: ";
    for (int val : buf) {
        std::cout << val << " ";
    }
    std::cout << "\n";
    // Output: Sorted: 1 2 5 8 9

    // std::find works!
    auto it = std::find(buf.begin(), buf.end(), 5);
    if (it != buf.end()) {
        std::cout << "Found 5 at index: "
                  << std::distance(buf.begin(), it) << "\n";
    }
    // Output: Found 5 at index: 2

    // std::accumulate works!
    int total = std::accumulate(buf.begin(), buf.end(), 0);
    std::cout << "Sum: " << total << "\n";
    // Output: Sum: 25

    // Const iteration
    const IntBuffer& const_ref = buf;
    std::cout << "Const iteration: ";
    for (auto it = const_ref.begin(); it != const_ref.end(); ++it) {
        std::cout << *it << " ";
    }
    std::cout << "\n";
    // Output: Const iteration: 1 2 5 8 9

    return 0;
}
```

关键点在于：我们没有为 `IntBuffer` 单独实现任何算法，但 `std::sort`、`std::find`、`std::accumulate` 全部可以直接使用。只要迭代器满足算法要求的最低类别，一切就能运转。

### 自定义迭代器的要点总结

实现自定义迭代器时，必须注意以下几点：

1. **定义 `iterator_traits` 所需的五个类型别名**：
   - `iterator_category`：迭代器类别标签
   - `value_type`：元素的类型
   - `difference_type`：两个迭代器之间的距离类型（通常是 `std::ptrdiff_t`）
   - `pointer`：指向元素的指针类型
   - `reference`：元素的引用类型

2. **实现必要的操作符**：根据你声明的迭代器类别，至少实现对应的操作。`ForwardIterator` 需要 `*`、`++`（前置和后置）、`==`、`!=`。

3. **提供从 `Iterator` 到 `ConstIterator` 的隐式转换**：这样 `cbegin()` 返回的迭代器才能正确工作，也允许把非 const 容器的迭代器传给接受 const 迭代器的函数。

4. **提供 `begin()` 和 `end()`**：这是 range-based for 循环和 STL 算法的前提条件。

---

## 迭代器 vs 指针

迭代器是指针的泛化，但它们之间有几个关键区别：

### 指针能做的，迭代器不一定能做

裸指针是 C++ 中最强大的"迭代器"。它满足 **ContiguousIterator** 的所有要求，可以做任何迭代器能做的事情，包括算术运算：

```cpp
int arr[] = {10, 20, 30, 40, 50};
int* ptr = arr;

// Pointer arithmetic
std::cout << *(ptr + 2) << "\n";   // 30
std::cout << ptr[3] << "\n";       // 40
std::cout << (ptr + 4) - ptr << "\n";  // 4
```

但 `std::list<int>::iterator` 做不到这些：

```cpp
std::list<int> lst = {10, 20, 30, 40, 50};
auto it = lst.begin();

// it + 2;    // Compile error! List iterator is only bidirectional.
// it[3];     // Compile error!
// it += 2;   // Compile error!

// Can only do:
++it;          // Move forward one step
--it;          // Move backward one step
std::advance(it, 2);  // Move forward N steps (O(n) for list!)
```

### 迭代器有"失效"的概念

指针指向栈变量或静态存储时，不存在"失效"问题（只要变量还活着）。但迭代器指向容器元素时，容器的修改操作可能让迭代器失效。这一点在前面已经详细讨论过。

### 迭代器可以是"聪明的"

裸指针就是地址，没有额外逻辑。但迭代器可以在内部维护状态。比如 `std::vector<bool>` 的迭代器，由于 `vector<bool>` 用位压缩存储，它的迭代器实际上是一个代理对象（proxy），`*it` 返回的不是 `bool&`，而是一个临时的引用代理。这种设计让迭代器的行为和普通指针产生了差异。

```cpp
std::vector<bool> bv = {true, false, true};

// *it is a proxy, not a real bool reference
auto it = bv.begin();
bool value = *it;  // OK: copy the value
// bool& ref = *it;  // Compile error! Cannot bind proxy to bool&

// But this works fine:
*it = false;  // Writes through the proxy
```

### 对比总结

| 特性 | 裸指针 | 迭代器 |
|------|--------|--------|
| 本质 | 内存地址 | 泛化的访问抽象 |
| 适用范围 | 连续内存 | 任意数据结构 |
| 随机访问 | 天然支持 | 仅 RandomAccessIterator 支持 |
| 失效风险 | 手动管理 | 跟随容器规则 |
| 与 STL 算法配合 | 可以（如果指向连续内存） | 天然配合 |
| 状态维护 | 无 | 可以有内部状态 |
| 类型安全 | 弱 | 强（编译期检查类别） |

> 经验法则：能用迭代器就用迭代器，不要退回到指针操作。迭代器提供了更好的抽象和安全性。

---

## C++20 Ranges：迭代器的进化

C++20 引入了 Ranges 库（`<ranges>`），它在迭代器的基础上提供了更高层的抽象。Ranges 的核心思想是：**把"范围"本身当作一等公民来操作**，而不是每次都传递 `begin()` 和 `end()`。

### 从迭代器对到 Range

在 C++20 之前，算法接受的是迭代器对：

```cpp
std::vector<int> vec = {1, 2, 3, 4, 5};
std::sort(vec.begin(), vec.end());
auto it = std::find(vec.begin(), vec.end(), 3);
```

C++20 中，算法可以直接接受 Range：

```cpp
#include <ranges>
#include <algorithm>

std::vector<int> vec = {1, 2, 3, 4, 5};
std::ranges::sort(vec);
auto it = std::ranges::find(vec, 3);
```

### Views：惰性组合操作

Ranges 最强大的特性是 **Views**，它允许你用管道操作符（`|`）把多个变换串联起来，而且所有操作都是**惰性求值**的：

```cpp
#include <ranges>
#include <vector>
#include <iostream>

int main() {
    std::vector<int> nums = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    // Pipeline: filter even numbers -> square them -> take first 3
    auto result = nums
        | std::views::filter([](int n) { return n % 2 == 0; })
        | std::views::transform([](int n) { return n * n; })
        | std::views::take(3);

    for (int val : result) {
        std::cout << val << " ";
    }
    std::cout << "\n";
    // Output: 4 16 36

    return 0;
}
```

这段代码不会创建任何中间容器。`filter` 不会复制数据，`transform` 不会分配新数组，`take` 只是提前终止遍历。所有操作在实际遍历时才依次执行。

### Ranges 与迭代器的关系

Ranges 并没有取代迭代器。它的底层完全基于迭代器。一个 `Range` 本质上就是一个能提供 `begin()` 和 `end()` 的对象。`Views` 是一种特殊的 Range，它通过自定义迭代器来实现惰性求值。

换句话说，Ranges 是迭代器之上的**高层抽象**。你写的是数据变换的意图，而不是迭代器的操作细节。

> 如果你还在用 C++17 或更早的版本，可以关注 [range-v3](https://github.com/ericniebler/range-v3) 库，它是 C++20 Ranges 的前身，接口基本一致。

---

## 总结

### 迭代器类别与容器对照表

| 迭代器类别 | 支持的容器 |
|------------|------------|
| **ContiguousIterator** (C++20) | `std::vector`、`std::array`、`std::string`、`std::basic_string` |
| **RandomAccessIterator** | `std::deque` |
| **BidirectionalIterator** | `std::list`、`std::forward_list`(仅 Forward)、`std::map`、`std::set`、`std::multimap`、`std::multiset` |
| **ForwardIterator** | `std::forward_list`、`std::unordered_map`、`std::unordered_set`（单向链表桶结构） |
| **InputIterator** | `std::istream_iterator` |
| **OutputIterator** | `std::ostream_iterator`、`std::back_insert_iterator`、`std::front_insert_iterator` |

> 注意：`std::unordered_map` 和 `std::unordered_set` 的迭代器在 C++17 标准中是 **ForwardIterator**（至少），具体实现可能提供 BidirectionalIterator，但标准只保证 Forward。

### 核心要点回顾

**STL 的设计哲学**：六大组件各司其职，通过迭代器这个统一的接口协同工作。容器管存储，算法管计算，迭代器管访问，三者完全解耦。

**迭代器的本质**：指针的泛化。它定义了"遍历和访问数据"的抽象接口，使得算法可以在不了解容器内部结构的前提下操作数据。

**迭代器分类**：从弱到强六个层次，能力递增。算法根据需要声明最低要求，编译器在编译期检查兼容性。

**迭代器失效**：不同容器在不同操作下的失效规则不同。链表和关联容器的迭代器最稳定，`vector` 和 `deque` 在结构变化时可能导致大量迭代器失效。

**自定义迭代器**：只要正确定义了 `iterator_traits` 所需的类型别名和必要的操作符，任何自定义容器都能无缝接入 STL 算法体系。

**Ranges 的未来**：C++20 Ranges 在迭代器之上提供了更高级的抽象，用管道式组合替代了手动的迭代器传递，同时保持了惰性求值的高效性。

STL 迭代器的设计已经历了超过二十年的考验。它证明了泛型编程的思想在实践中是可行的：通过精心设计的抽象层次，我们可以在保持零成本抽象的同时，获得极高的代码复用性。理解迭代器，就是理解 STL 的灵魂。
