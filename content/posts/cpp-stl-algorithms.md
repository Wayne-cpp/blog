---
title: "C++ STL 算法全景：从排序到数值计算的泛型力量"
date: 2021-08-15T10:00:00+08:00
tags: ["C++", "STL", "算法", "排序", "查找"]
categories: ["技术"]
summary: "系统梳理 C++ STL algorithm 和 numeric 提供的核心算法。从排序、查找、修改、数值计算到 C++17 的新算法，讲清楚算法与迭代器协作的设计思想、复杂度保证与实际运用场景。"
ShowToc: true
---

写 C++ 的人多少都听过一句话：**不要手写循环，优先用算法**。但很多人对 STL 算法的认知停留在 `std::sort` 和 `std::find`，对 `<algorithm>` 和 `<numeric>` 里一百多个函数缺乏全景了解。这篇文章系统梳理这些算法的设计思路、复杂度保证和使用场景。

## 算法与容器的分离：迭代器作为桥梁

STL 最核心的设计决策之一，是把**算法**和**容器**彻底分开。`std::sort` 不知道自己在排 `std::vector` 还是 `std::deque`，它只关心一件事：给我一对迭代器，告诉我元素范围，我来排。

这个设计靠**迭代器（iterator）**打通。迭代器是算法和容器之间的通用语言：

- 容器提供 `begin()` / `end()`，暴露迭代器
- 算法接受迭代器对 `[first, last)`，不碰容器本身
- 迭代器按能力分层：输入迭代器、前向迭代器、双向迭代器、随机访问迭代器（C++20 再加连续迭代器）

```cpp
#include <algorithm>
#include <vector>
#include <list>

// sort 要求随机访问迭代器，vector 满足
std::vector<int> vec = {5, 2, 8, 1, 9};
std::sort(vec.begin(), vec.end());

// list 只有双向迭代器，不能直接 sort
// std::list<int> lst = {5, 2, 8};
// std::sort(lst.begin(), lst.end());  // compile error
// list 提供自己的成员 sort
std::list<int> lst = {5, 2, 8, 1, 9};
lst.sort();
```

> **规则**：算法对迭代器类别有要求。随机访问迭代器才能用 `sort`，前向迭代器就能用 `find`。编译期间不符合会直接报错。

这种分离的好处是**算法的复用率极高**。同一个 `std::copy` 可以复制数组、`vector`、`list`、甚至 C 风格数组。你不需要为每种容器重写一套算法。

## 算法设计哲学：操作迭代器范围，而非容器

所有 STL 算法遵循几个统一约定：

1. **输入范围是半开区间 `[first, last)`**，包含 `first`，不包含 `last`
2. **算法不直接修改容器大小**。删除元素只是把"要保留的"移到前面，返回新的逻辑终点
3. **返回值通常是迭代器**，指向结果位置或查找结果
4. **复杂度有明确保证**，标准文档里每个算法都标注了复杂度

```cpp
// half-open range: [begin, end)
// end points one-past-the-last element
std::vector<int> data = {1, 2, 3, 4, 5};
// data.begin() points to 1
// data.end() points past 5 (do not dereference!)
auto it = std::find(data.begin(), data.end(), 3);
// it points to the element with value 3
```

理解这些约定后，你会发现 STL 算法的接口模式非常规律，学起来比想象中快。

---

## 排序算法家族

排序是 STL 算法里最丰富的一个家族。不同的场景该用不同的排序函数，选择依据是**你需要什么程度的有序**。

### sort、stable_sort、partial_sort、nth_element

| 算法 | 复杂度 | 是否稳定 | 用途 |
|------|--------|----------|------|
| `sort` | O(N log N) | 否 | 完全排序 |
| `stable_sort` | O(N log²N) | 是 | 保持相等元素原始顺序 |
| `partial_sort` | O(N log M) | 否 | 只排前 M 个，后面不管 |
| `nth_element` | O(N) 平均 | 否 | 只保证第 n 个位置放对了元素 |

```cpp
#include <algorithm>
#include <vector>
#include <iostream>

void demo_sort() {
    std::vector<int> data = {9, 3, 7, 1, 5, 2, 8, 4, 6, 0};

    // Full sort
    std::sort(data.begin(), data.end());
    // data: 0 1 2 3 4 5 6 7 8 9

    // Custom comparator: descending
    std::sort(data.begin(), data.end(), std::greater<int>());
    // data: 9 8 7 6 5 4 3 2 1 0
}

void demo_stable_sort() {
    struct Employee {
        std::string name;
        int age;
    };

    std::vector<Employee> staff = {
        {"Alice", 30}, {"Bob", 25}, {"Carol", 30}, {"Dave", 25}
    };

    // Sort by age, preserve original order for same age
    std::stable_sort(staff.begin(), staff.end(),
        [](const Employee& a, const Employee& b) {
            return a.age < b.age;
        });
    // Bob(25) stays before Dave(25)
    // Alice(30) stays before Carol(30)
}

void demo_partial_sort() {
    std::vector<int> data = {9, 3, 7, 1, 5, 2, 8, 4, 6, 0};

    // Only guarantee the smallest 3 are sorted at the front
    std::partial_sort(data.begin(), data.begin() + 3, data.end());
    // data: 0 1 2 [rest in unspecified order]
    // Useful for "top-K" problems
}

void demo_nth_element() {
    std::vector<int> data = {9, 3, 7, 1, 5, 2, 8, 4, 6, 0};

    // Partition so element at position 4 is what it would be in sorted order
    // Elements before are <= it, elements after are >= it
    std::nth_element(data.begin(), data.begin() + 4, data.end());
    // data[4] == 4 (the median of 0-9)
    // Elements before data[4] are all <= 4 (not necessarily sorted)
    // Elements after data[4] are all >= 4 (not necessarily sorted)
}
```

### is_sorted 检查

C++11 引入了 `is_sorted` 和 `is_sorted_until`，用来检查或定位有序区间的边界：

```cpp
#include <algorithm>
#include <vector>

void check_sorted() {
    std::vector<int> data = {1, 2, 3, 5, 4, 6, 7};

    // Is the whole range sorted?
    bool sorted = std::is_sorted(data.begin(), data.end());
    // false, because 5 > 4 at position 3-4

    // Find where sorting breaks
    auto it = std::is_sorted_until(data.begin(), data.end());
    // it points to 4 (first out-of-order element)
    // distance from begin to it = 4 (elements 1,2,3,5 are sorted)
}
```

> **选择原则**：不需要完全有序就别用 `sort`。只需要前 K 个用 `partial_sort`，只需要中位数或分区用 `nth_element`。这是免费的性能优化。

---

## 查找算法

查找算法分两类：**线性查找**和**二分查找**。选择取决于数据是否有序。

### 线性查找：find 家族

```cpp
#include <algorithm>
#include <vector>
#include <string>

void demo_find() {
    std::vector<int> data = {10, 20, 30, 40, 50};

    // Find exact value
    auto it = std::find(data.begin(), data.end(), 30);
    if (it != data.end()) {
        // Found at index 2
    }

    // Find with predicate
    auto it2 = std::find_if(data.begin(), data.end(),
        [](int x) { return x > 35; });
    // it2 points to 40

    // Find first that does NOT satisfy predicate
    auto it3 = std::find_if_not(data.begin(), data.end(),
        [](int x) { return x < 30; });
    // it3 points to 30
}
```

线性查找复杂度 O(N)，数据量大时应该考虑用二分查找。

### 二分查找家族

二分查找要求数据**已经有序**，复杂度 O(log N)：

| 算法 | 返回值 | 含义 |
|------|--------|------|
| `binary_search` | `bool` | 值是否存在 |
| `lower_bound` | 迭代器 | 第一个 >= value 的位置 |
| `upper_bound` | 迭代器 | 第一个 > value 的位置 |
| `equal_range` | 迭代器对 | [lower_bound, upper_bound) |

```cpp
#include <algorithm>
#include <vector>
#include <iostream>

void demo_binary_search() {
    std::vector<int> data = {1, 2, 4, 4, 4, 7, 9, 10};

    // Does value exist?
    bool found = std::binary_search(data.begin(), data.end(), 4);
    // true

    // First position where element >= 4
    auto lower = std::lower_bound(data.begin(), data.end(), 4);
    // lower points to index 2 (first 4)

    // First position where element > 4
    auto upper = std::upper_bound(data.begin(), data.end(), 4);
    // upper points to index 5 (value 7)

    // Count of element 4
    int count = upper - lower;  // 3

    // Get the range of all elements equal to 4
    auto range = std::equal_range(data.begin(), data.end(), 4);
    // range.first == lower, range.second == upper
    int count2 = range.second - range.first;  // 3
}
```

> **提示**：在有序容器上做查找，用 `lower_bound` / `upper_bound` 而不是 `find`。`std::map` 和 `std::set` 有自己的成员函数版本，效率更高，优先用成员函数。

---

## 计数与判定

### count 与 count_if

```cpp
#include <algorithm>
#include <vector>

void demo_count() {
    std::vector<int> data = {1, 2, 3, 2, 4, 2, 5};

    // Count occurrences of value
    auto n = std::count(data.begin(), data.end(), 2);
    // n == 3

    // Count elements satisfying predicate
    auto even_count = std::count_if(data.begin(), data.end(),
        [](int x) { return x % 2 == 0; });
    // even_count == 4 (2, 2, 4, 2)
}
```

### 全量判定：all_of、any_of、none_of

C++11 引入的三个判定函数，语义非常直观：

```cpp
#include <algorithm>
#include <vector>

void demo_predicates() {
    std::vector<int> data = {2, 4, 6, 8, 10};

    bool all_even = std::all_of(data.begin(), data.end(),
        [](int x) { return x % 2 == 0; });
    // true: every element is even

    bool any_big = std::any_of(data.begin(), data.end(),
        [](int x) { return x > 20; });
    // false: no element > 20

    bool none_negative = std::none_of(data.begin(), data.end(),
        [](int x) { return x < 0; });
    // true: no negative elements
}
```

对空范围，`all_of` 返回 `true`，`any_of` 返回 `false`，`none_of` 返回 `true`。这是逻辑上的空真（vacuous truth）。

### equal 与 mismatch

比较两个序列的异同：

```cpp
#include <algorithm>
#include <vector>
#include <string>

void demo_compare() {
    std::vector<int> a = {1, 2, 3, 4, 5};
    std::vector<int> b = {1, 2, 3, 4, 5};
    std::vector<int> c = {1, 2, 9, 4, 5};

    // Are two ranges equal?
    bool same = std::equal(a.begin(), a.end(), b.begin());
    // true

    // Find first mismatch
    auto result = std::mismatch(a.begin(), a.end(), c.begin());
    // result.first points to a[2] (value 3)
    // result.second points to c[2] (value 9)
}
```

C++14 版本的 `equal` 支持双范围完整签名，更安全：

```cpp
// C++14: specify both ranges explicitly
bool same = std::equal(a.begin(), a.end(), b.begin(), b.end());
```

---

## 修改序列算法

修改类算法是最常用的一批。它们不会改变容器大小，只会改变元素的值或位置。

### copy 与 copy_if

```cpp
#include <algorithm>
#include <vector>

void demo_copy() {
    std::vector<int> src = {1, 2, 3, 4, 5};
    std::vector<int> dst(5);

    // Copy all
    std::copy(src.begin(), src.end(), dst.begin());

    // Copy only even numbers
    std::vector<int> evens;
    std::copy_if(src.begin(), src.end(), std::back_inserter(evens),
        [](int x) { return x % 2 == 0; });
    // evens: {2, 4}

    // Copy backwards (safe when ranges overlap with dst at lower addresses)
    std::vector<int> dst2(5);
    std::copy_backward(src.begin(), src.end(), dst2.end());
}
```

`std::back_inserter` 是个适配器，把赋值操作变成 `push_back`。在不知道目标大小时特别有用。

### transform

`transform` 分一元和二元两个版本：

```cpp
#include <algorithm>
#include <vector>
#include <string>
#include <cctype>

void demo_transform() {
    // Unary: apply function to each element
    std::string s = "hello world";
    std::transform(s.begin(), s.end(), s.begin(),
        [](unsigned char c) { return std::toupper(c); });
    // s: "HELLO WORLD"

    // Binary: apply function to pairs from two ranges
    std::vector<int> a = {1, 2, 3};
    std::vector<int> b = {10, 20, 30};
    std::vector<int> result(3);
    std::transform(a.begin(), a.end(), b.begin(), result.begin(),
        std::plus<int>());
    // result: {11, 22, 33}
}
```

### replace、fill、generate

```cpp
#include <algorithm>
#include <vector>

void demo_replace_fill() {
    std::vector<int> data = {1, 2, 3, 2, 4, 2};

    // Replace all 2 with 99
    std::replace(data.begin(), data.end(), 2, 99);
    // data: {1, 99, 3, 99, 4, 99}

    // Replace with predicate
    std::replace_if(data.begin(), data.end(),
        [](int x) { return x > 50; }, 0);
    // data: {1, 0, 3, 0, 4, 0}

    // Fill with a value
    std::vector<int> buf(10);
    std::fill(buf.begin(), buf.end(), 42);
    // buf: all 42s

    // Fill with generated values
    int counter = 0;
    std::generate(buf.begin(), buf.end(), [&counter]() { return counter++; });
    // buf: {0, 1, 2, 3, 4, 5, 6, 7, 8, 9}
}
```

### remove 与 unique

`remove` 不真正删除元素，只是把要保留的元素移到前面。这是 STL 设计里经常让人困惑的地方，后面单独讲。

```cpp
#include <algorithm>
#include <vector>
#include <iostream>

void demo_remove() {
    std::vector<int> data = {1, 2, 3, 2, 4, 2, 5};

    // "Remove" all 2s — actually shifts non-2 elements forward
    auto new_end = std::remove(data.begin(), data.end(), 2);
    // data: {1, 3, 4, 5, ?, ?, ?}
    // new_end points to data[4]
    // Elements after new_end are unspecified (not gone!)

    // Actually erase them
    data.erase(new_end, data.end());
    // data: {1, 3, 4, 5}
}

void demo_unique() {
    // Remove consecutive duplicates (sort first for global uniqueness)
    std::vector<int> data = {1, 1, 2, 2, 2, 3, 1, 1};
    auto new_end = std::unique(data.begin(), data.end());
    // data: {1, 2, 3, 1, ?, ?, ?, ?}
    data.erase(new_end, data.end());
    // data: {1, 2, 3, 1}
}
```

### reverse 与 rotate

```cpp
#include <algorithm>
#include <vector>
#include <iostream>

void demo_reverse_rotate() {
    std::vector<int> data = {1, 2, 3, 4, 5};

    // Reverse in place
    std::reverse(data.begin(), data.end());
    // data: {5, 4, 3, 2, 1}

    // Rotate: move [begin, mid) to after [mid, end)
    // Equivalent to left-rotating so mid becomes the new first
    std::vector<int> data2 = {1, 2, 3, 4, 5};
    std::rotate(data2.begin(), data2.begin() + 2, data2.end());
    // data2: {3, 4, 5, 1, 2}
    // Original [3,4,5] moved to front, [1,2] to back
}
```

`rotate` 在实现循环缓冲区、左移数组等场景下非常好用。它比手写的三重 `reverse` 技巧更清晰，效率一样。

---

## Remove-Erase 惯用法

上面提到 `remove` 不真正删除元素，这是 STL 里最需要理解的设计模式。

### 为什么 remove 不能删除元素

因为 `remove` 是 `<algorithm>` 里的函数，它只拿到迭代器，不知道容器的类型。`std::vector` 有 `erase`，`std::list` 也有，但 `std::array` 没有。算法拿不到容器，所以它不能调用容器的 `erase`。

它做的事情是：把不该删的元素挪到前面，返回新逻辑终点。之后的元素还在内存里，只是逻辑上不属于这个序列了。

### 正确写法

```cpp
#include <algorithm>
#include <vector>

// The canonical remove-erase idiom
void remove_erase() {
    std::vector<int> data = {1, 2, 3, 2, 4, 2, 5};

    // Correct: combine remove and erase
    data.erase(
        std::remove(data.begin(), data.end(), 2),
        data.end()
    );
    // data: {1, 3, 4, 5}

    // With predicate
    data.erase(
        std::remove_if(data.begin(), data.end(),
            [](int x) { return x < 3; }),
        data.end()
    );
    // data: {3, 4, 5}
}
```

> **规则**：永远把 `remove` / `remove_if` 和 `erase` 写在一起。单独写 `remove` 然后忘记 `erase` 是最常见的 bug 之一。C++20 引入了 `std::erase` 和 `std::erase_if` 免费函数，可以直接一行搞定。

### C++20 的简化写法

```cpp
#include <vector>
#include <algorithm>

// C++20: one-liner erase
void modern_erase() {
    std::vector<int> data = {1, 2, 3, 2, 4, 2, 5};

    // Erase all 2s
    std::erase(data, 2);
    // data: {1, 3, 4, 5}

    // Erase with predicate
    std::erase_if(data, [](int x) { return x < 3; });
    // data: {3, 4, 5}
}
```

---

## 集合操作

集合操作要求输入范围**已经有序**。它们处理有序序列上的并、交、差等运算。

```cpp
#include <algorithm>
#include <vector>
#include <iterator>
#include <iostream>

void demo_set_operations() {
    std::vector<int> a = {1, 2, 3, 4, 5};
    std::vector<int> b = {3, 4, 5, 6, 7};

    // Merge two sorted ranges
    std::vector<int> merged;
    std::merge(a.begin(), a.end(), b.begin(), b.end(),
        std::back_inserter(merged));
    // merged: {1, 2, 3, 3, 4, 4, 5, 5, 6, 7}

    // Does a contain all elements of b?
    bool contains = std::includes(a.begin(), a.end(), b.begin(), b.end());
    // false: b has 6, 7 which a does not

    // Union: elements in a or b (duplicates kept once)
    std::vector<int> union_result;
    std::set_union(a.begin(), a.end(), b.begin(), b.end(),
        std::back_inserter(union_result));
    // union_result: {1, 2, 3, 4, 5, 6, 7}

    // Intersection: elements in both a and b
    std::vector<int> inter_result;
    std::set_intersection(a.begin(), a.end(), b.begin(), b.end(),
        std::back_inserter(inter_result));
    // inter_result: {3, 4, 5}

    // Difference: elements in a but not in b
    std::vector<int> diff_result;
    std::set_difference(a.begin(), a.end(), b.begin(), b.end(),
        std::back_inserter(diff_result));
    // diff_result: {1, 2}

    // Symmetric difference: elements in a or b but not both
    std::vector<int> sym_diff;
    std::set_symmetric_difference(a.begin(), a.end(), b.begin(), b.end(),
        std::back_inserter(sym_diff));
    // sym_diff: {1, 2, 6, 7}
}
```

这些算法在处理有序数据时比手写循环简洁得多。记住前提条件：输入必须有序，否则结果无定义。

---

## 堆操作

STL 提供了一组以堆（heap）为基础的算法。堆是一个完全二叉树，通常用数组表示，`std::priority_queue` 底层就靠这些函数实现。

```cpp
#include <algorithm>
#include <vector>
#include <iostream>

void demo_heap() {
    std::vector<int> data = {3, 1, 4, 1, 5, 9, 2, 6};

    // Build a max-heap
    std::make_heap(data.begin(), data.end());
    // data is now a max-heap: 9 at front (largest)
    std::cout << "Max: " << data.front() << "\n";  // 9

    // Push a new element
    data.push_back(8);
    std::push_heap(data.begin(), data.end());
    // 8 is now correctly placed in the heap

    // Pop the max element (moves it to the back)
    std::pop_heap(data.begin(), data.end());
    int max_val = data.back();  // The popped max
    data.pop_back();            // Actually remove it

    // Sort the heap (turns heap into sorted range)
    std::sort_heap(data.begin(), data.end());
    // data is now sorted ascending
}
```

> **注意**：STL 堆默认是**大顶堆**。如果需要小顶堆，传入 `std::greater<>()` 作为比较器。

```cpp
// Min-heap
std::make_heap(data.begin(), data.end(), std::greater<int>());
std::push_heap(data.begin(), data.end(), std::greater<int>());
std::pop_heap(data.begin(), data.end(), std::greater<int>());
```

一般情况直接用 `std::priority_queue` 更方便，但了解底层堆操作有助于理解其行为和定制。

---

## 数值算法（numeric 头文件）

`<numeric>` 头文件提供了一组数值计算算法。很多人只知道 `std::accumulate`，其实还有好几个利器。

### accumulate

```cpp
#include <numeric>
#include <vector>
#include <string>

void demo_accumulate() {
    std::vector<int> data = {1, 2, 3, 4, 5};

    // Sum with initial value 0
    int sum = std::accumulate(data.begin(), data.end(), 0);
    // sum == 15

    // Product: note initial value is 1, not 0
    int product = std::accumulate(data.begin(), data.end(), 1,
        std::multiplies<int>());
    // product == 120

    // String concatenation
    std::vector<std::string> words = {"Hello", " ", "World"};
    std::string result = std::accumulate(words.begin(), words.end(),
        std::string());
    // result: "Hello World"
}
```

> **陷阱**：`accumulate` 的初始值类型决定了计算的类型。初始值写 `0` 就是 `int`，写 `0LL` 就是 `long long`，写 `0.0` 就是 `double`。不注意的话容易溢出或丢失精度。

### inner_product

点积运算，两个向量逐元素相乘再求和：

```cpp
#include <numeric>
#include <vector>

void demo_inner_product() {
    std::vector<int> a = {1, 2, 3};
    std::vector<int> b = {4, 5, 6};

    // Dot product: 1*4 + 2*5 + 3*6
    int dot = std::inner_product(a.begin(), a.end(), b.begin(), 0);
    // dot == 32

    // Custom operation: multiply becomes addition, add becomes max
    // Result: max(1+4, 2+5, 3+6) = max(5, 7, 9) = 9
    // Actually this computes step by step:
    // init=0, step1=max(0, 1+4)=5, step2=max(5,2+5)=7, step3=max(7,3+6)=9
    int custom = std::inner_product(a.begin(), a.end(), b.begin(), 0,
        [](int acc, int val) { return std::max(acc, val); },
        std::plus<int>());
}
```

### partial_sum 与 adjacent_difference

```cpp
#include <numeric>
#include <vector>
#include <iostream>

void demo_partial_sum() {
    std::vector<int> data = {1, 2, 3, 4, 5};
    std::vector<int> prefix(data.size());

    // Running sum: {1, 1+2, 1+2+3, ...}
    std::partial_sum(data.begin(), data.end(), prefix.begin());
    // prefix: {1, 3, 6, 10, 15}

    // Useful for range sum queries:
    // sum of [i, j] = prefix[j] - prefix[i-1]
}

void demo_adjacent_diff() {
    std::vector<int> data = {2, 4, 7, 11, 16};
    std::vector<int> diff(data.size());

    // Differences between consecutive elements
    std::adjacent_difference(data.begin(), data.end(), diff.begin());
    // diff: {2, 2, 3, 4, 5}
    // First element is copied, then data[i] - data[i-1]
}
```

`partial_sum` 和 `adjacent_difference` 互为逆运算。先做一次 `partial_sum`，再对结果做 `adjacent_difference`，就能还原回原始数据。

### iota

C++11 引入的 `iota`，用递增值填充一个范围：

```cpp
#include <numeric>
#include <vector>
#include <iostream>

void demo_iota() {
    std::vector<int> indices(10);
    std::iota(indices.begin(), indices.end(), 0);
    // indices: {0, 1, 2, 3, 4, 5, 6, 7, 8, 9}

    // Start from 1
    std::iota(indices.begin(), indices.end(), 1);
    // indices: {1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

    // Useful: generate index sequence for sorting by another array
    std::vector<std::string> names = {"Charlie", "Alice", "Bob"};
    std::vector<int> order(names.size());
    std::iota(order.begin(), order.end(), 0);
    // Sort indices by corresponding name
    std::sort(order.begin(), order.end(), [&names](int i, int j) {
        return names[i] < names[j];
    });
    // order: {1, 2, 0} — indices sorted by name alphabetically
}
```

`iota` 在生成索引序列、配合排序实现间接排序等场景下非常实用。

---

## C++17 新增算法

C++17 在 `<algorithm>` 和 `<numeric>` 里加了一批新算法，补上了不少日常缺的工具。

### sample：随机采样

```cpp
#include <algorithm>
#include <vector>
#include <random>
#include <iostream>

void demo_sample() {
    std::vector<int> population = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    std::vector<int> result(3);

    std::random_device rd;
    std::mt19937 gen(rd());

    // Pick 3 elements uniformly at random (stable: preserves order)
    std::sample(population.begin(), population.end(),
        result.begin(), 3, gen);
    // Could be {2, 5, 8} or any 3-element subset, in original order
}
```

`sample` 使用 reservoir sampling 算法（流式输入）或选择采样（随机访问输入），保证每个元素被选中的概率相等。

### clamp：限制范围

```cpp
#include <algorithm>

void demo_clamp() {
    int value = 15;

    // Clamp value to [0, 10]
    int clamped = std::clamp(value, 0, 10);
    // clamped == 10

    int value2 = 5;
    int clamped2 = std::clamp(value2, 0, 10);
    // clamped2 == 5 (within range, no change)

    int value3 = -3;
    int clamped3 = std::clamp(value3, 0, 10);
    // clamped3 == 0

    // With custom comparator
    // std::clamp(value, lo, hi, comp)
}
```

以前写 `std::max(lo, std::min(value, hi))`，现在一行 `std::clamp` 搞定。

### reduce 与 transform_reduce

```cpp
#include <numeric>
#include <vector>
#include <execution>

void demo_reduce() {
    std::vector<int> data = {1, 2, 3, 4, 5};

    // reduce: like accumulate but can be parallelized
    // Default operation is addition, default initial value is 0
    int sum = std::reduce(data.begin(), data.end(), 0);
    // sum == 15

    // With custom operation
    int product = std::reduce(data.begin(), data.end(), 1,
        std::multiplies<int>());
    // product == 120
}

void demo_transform_reduce() {
    std::vector<int> a = {1, 2, 3};
    std::vector<int> b = {4, 5, 6};

    // Dot product in one call
    int dot = std::transform_reduce(
        a.begin(), a.end(), b.begin(), 0);
    // dot == 1*4 + 2*5 + 3*6 == 32

    // Count total string length
    std::vector<std::string> words = {"Hello", "World", "!"};
    std::size_t total = std::transform_reduce(
        words.begin(), words.end(),
        std::size_t(0),
        std::plus<>(),
        [](const std::string& s) { return s.size(); });
    // total == 5 + 5 + 1 == 11
}
```

`transform_reduce` 比 `inner_product` 更通用：支持一元变换加归约，也支持二元变换加归约。它还可以搭配执行策略并行执行。

### for_each_n

```cpp
#include <algorithm>
#include <vector>
#include <iostream>

void demo_for_each_n() {
    std::vector<int> data = {10, 20, 30, 40, 50};

    // Process first 3 elements
    auto it = std::for_each_n(data.begin(), 3,
        [](int x) { std::cout << x << " "; });
    // Output: 10 20 30
    // it points to data[3] (one past the processed range)
}
```

---

## 执行策略（C++17）

C++17 引入了**并行 STL**，大多数算法新增了一个可选的执行策略参数，放在其他参数前面：

| 策略 | 含义 |
|------|------|
| `std::execution::seq` | 顺序执行，和不用策略一样 |
| `std::execution::par` | 并行执行，可以多线程 |
| `std::execution::par_unseq` | 并行且允许向量化，可能重排或交错 |

```cpp
#include <algorithm>
#include <numeric>
#include <vector>
#include <execution>

void demo_execution_policy() {
    std::vector<int> data(1'000'000);
    std::iota(data.begin(), data.end(), 1);

    // Sequential sort (default)
    std::sort(std::execution::seq, data.begin(), data.end());

    // Parallel sort — may use multiple threads
    std::sort(std::execution::par, data.begin(), data.end());

    // Parallel + vectorized reduce
    long long sum = std::transform_reduce(
        std::execution::par_unseq,
        data.begin(), data.end(),
        0LL,
        std::plus<>(),
        [](int x) { return static_cast<long long>(x) * x; });
    // Sum of squares, potentially parallelized and vectorized
}
```

> **注意**：使用并行执行策略时，传入的可调用对象（lambda、函数对象）必须是**线程安全的**。不能在 lambda 里修改共享状态而不加锁。并行策略的复杂度保证也和顺序版本不同，具体看标准文档。

实际使用中，并行版本在数据量大时（通常百万级以上）才有明显收益。小数据量反而可能因为线程创建开销变慢。

---

## 最佳实践

### 优先使用算法而非手写循环

这是最核心的建议。原因有几个：

**表达意图更清晰**。`std::find` 一看就知道在查找，`std::sort` 一看就知道在排序。手写 `for` 循环你得读代码才能明白在干什么。

**减少 bug**。手写循环容易出错：off-by-one、边界条件、迭代器失效。算法经过充分测试，复杂度有保证。

**性能不差，经常更好**。算法内部用了优化技巧：循环展开、缓存友好访问模式、特殊化处理。手写循环很少做到这些。

**可并行化**。C++17 加了执行策略，算法一行代码改成并行。手写循环要并行化得自己管理线程。

```cpp
// Bad: hand-written loop to find and process
void process_loop(const std::vector<int>& data) {
    for (size_t i = 0; i < data.size(); ++i) {
        if (data[i] > 100) {
            // process data[i]
            break;
        }
    }
}

// Good: algorithm says what you mean
void process_algorithm(const std::vector<int>& data) {
    auto it = std::find_if(data.begin(), data.end(),
        [](int x) { return x > 100; });
    if (it != data.end()) {
        // process *it
    }
}
```

### 选择正确的算法

不是所有场景都用 `sort`。一个常见错误是对只需要前 K 个元素的场景用完全排序：

```cpp
// Bad: full sort just to get top 3
std::sort(data.begin(), data.end(), std::greater<int>());
// Use data[0], data[1], data[2]

// Good: partial_sort
std::partial_sort(data.begin(), data.begin() + 3, data.end(),
    std::greater<int>());

// Best (if you don't need them sorted among themselves):
std::nth_element(data.begin(), data.begin() + 3, data.end(),
    std::greater<int>());
```

### 注意迭代器失效

算法本身不修改容器，但如果传入的迭代器在算法执行期间失效（比如其他线程修改了容器），就是未定义行为。在单线程场景下这不是问题，但在多线程环境里要格外小心。

### 善用 lambda

C++11 以后 lambda 让算法的可用性大幅提升。自定义比较、自定义谓词都可以内联写：

```cpp
std::sort(employees.begin(), employees.end(),
    [](const Employee& a, const Employee& b) {
        if (a.department != b.department)
            return a.department < b.department;
        return a.salary > b.salary;
    });
```

### 熟悉算法分类

心里有个分类框架，用的时候就知道该去哪找：

| 类别 | 典型算法 | 头文件 |
|------|----------|--------|
| 排序 | `sort`, `stable_sort`, `partial_sort`, `nth_element` | `<algorithm>` |
| 查找 | `find`, `binary_search`, `lower_bound`, `upper_bound` | `<algorithm>` |
| 计数判定 | `count`, `all_of`, `any_of`, `equal` | `<algorithm>` |
| 修改 | `copy`, `transform`, `replace`, `remove`, `unique` | `<algorithm>` |
| 集合 | `merge`, `set_union`, `set_intersection` | `<algorithm>` |
| 堆 | `make_heap`, `push_heap`, `pop_heap` | `<algorithm>` |
| 数值 | `accumulate`, `iota`, `partial_sum` | `<numeric>` |
| C++17 新增 | `sample`, `clamp`, `reduce`, `transform_reduce` | `<algorithm>` / `<numeric>` |

---

## 小结

STL 算法库是 C++ 标准库里设计得最优雅的部分之一。算法和容器通过迭代器解耦，一百多个算法覆盖了日常开发的绝大部分需求。

几个要点：

- **排序家族**按需选择：完全排序用 `sort`，前 K 个用 `partial_sort`，只需要分区用 `nth_element`
- **查找**区分线性和二分：无序用 `find` 系列，有序用 `lower_bound` 系列
- **remove-erase 惯用法**必须掌握：`remove` 不删除元素，要搭配 `erase`
- **数值算法**别忘 `<numeric>`：`accumulate`、`partial_sum`、`iota` 都很实用
- **C++17** 带来了 `reduce`、`transform_reduce`、`sample`、`clamp` 等好东西
- **并行执行策略**让算法一行代码就能并行化，但要注意线程安全

写 C++ 的时候，遇到循环先想一下：有没有现成的算法可以用？如果有，用它。代码更短、意图更清楚、bug 更少。
