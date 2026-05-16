---
title: "C++ 无序关联容器：unordered_map 与 unordered_set 的哈希表实现"
date: 2021-08-14T10:00:00+08:00
tags: ["C++", "STL", "unordered_map", "unordered_set", "哈希表"]
categories: ["C++"]
summary: "从哈希表的冲突解决策略出发，深入剖析 std::unordered_map 和 std::unordered_set 的桶数组与链表结构、负载因子与自动扩容机制、自定义哈希函数与相等判断的方法，以及与有序容器的选择指南。"
ShowToc: true
---

## 一、引言：O(1) 查找的承诺

当我们谈论 C++ 标准模板库（STL）中的关联容器时，`std::map` 和 `std::set` 通常是最先被提及的。它们基于红黑树实现，提供了 O(log n) 的查找、插入和删除性能。但在很多场景下，我们并不需要元素的有序性，只关心能不能**快速找到**某个键。

这就是 `std::unordered_map` 和 `std::unordered_set` 的用武之地。它们基于**哈希表**（hash table）实现，在平均情况下能提供 **O(1)** 的查找、插入和删除操作。

> 从 C++11 开始，无序关联容器正式进入标准库。在此之前，许多项目使用 `std::tr1::unordered_map` 或 `boost::unordered_map` 作为替代。

"unordered" 这个前缀很直白：容器不保证元素的顺序。遍历时的顺序取决于哈希函数和内部桶的布局，而不是键的大小关系。如果你不需要顺序，那就没有理由为维护红黑树付出额外代价。

这篇文章会从哈希表的基本原理讲起，然后深入 `unordered_map` 和 `unordered_set` 的内部结构、常用操作、自定义哈希函数、性能特征，以及与有序容器的选择指南。

## 二、哈希表基础

### 2.1 核心思想

哈希表的核心思想很简单：用一个**哈希函数**（hash function）把键映射到一个数组的某个位置，然后直接访问那个位置。

```
key ──→ hash_function(key) ──→ index ──→ bucket_array[index]
```

理想情况下，不同的键映射到不同的位置，查找只需要计算哈希值、访问数组，时间复杂度 O(1)。

### 2.2 哈希冲突

现实没那么完美。哈希函数的输出空间通常远小于输入空间，不同的键很可能映射到同一个位置，这就叫**哈希冲突**（hash collision）。

解决冲突的主流策略有两种：

| 策略 | 方法 | 特点 |
|------|------|------|
| **链地址法**（separate chaining） | 每个桶维护一个链表，冲突的元素挂在链表上 | STL 采用此方法 |
| **开放寻址法**（open addressing） | 冲突时探测下一个空位 | `std::flat_map`（C++23）采用 |
| **再哈希法**（rehashing） | 冲突时用另一个哈希函数计算新位置 | 较少单独使用 |

C++ STL 的无序容器选择了**链地址法**。每个桶（bucket）是一个单链表，所有哈希值映射到同一个桶的元素都被串在链表里。

### 2.3 为什么选择链地址法？

链地址法有几个让 STL 实现者青睐的优点：

1. **实现简单**，不需要复杂的探测序列
2. **删除操作高效**，直接从链表中移除节点即可
3. **对哈希函数质量要求相对宽松**，即使冲突稍多也能正常工作
4. **指针稳定**，链表节点的地址在 rehash 时可以保持不变（某些实现）

当然，链地址法也有缺点：每个节点需要额外的指针开销，缓存局部性不如开放寻址法。

## 三、内部结构

### 3.1 桶数组与链表

下面是一个简化的内部结构示意图。假设有 8 个桶，当前存储了若干键值对：

```
bucket_array (size = 8)
┌─────┐
│  0  │ → [empty]
├─────┤
│  1  │ → ["apple"→3] → ["avocado"→7] → null
├─────┤
│  2  │ → [empty]
├─────┤
│  3  │ → ["banana"→5] → null
├─────┤
│  4  │ → [empty]
├─────┤
│  5  │ → ["cherry"→2] → ["coconut"→9] → ["citrus"→4] → null
├─────┤
│  6  │ → [empty]
├─────┤
│  7  │ → ["date"→6] → null
└─────┘

hash("apple")   % 8 = 1
hash("avocado") % 8 = 1
hash("banana")  % 8 = 3
hash("cherry")  % 8 = 5
hash("coconut") % 8 = 5
hash("citrus")  % 8 = 5
hash("date")    % 8 = 7
```

每个桶的索引由 `hash(key) % bucket_count` 决定。桶 5 里有三个元素，说明发生了冲突，它们被串在同一个链表里。查找时，先定位桶，再沿链表逐个比较键。

### 3.2 桶的结构细节

在典型实现（如 libstdc++、libc++）中，桶数组存放的是链表头指针。节点（node）结构大致如下：

```cpp
// Simplified node structure (conceptual)
template <typename Value>
struct HashNode {
    Value value;          // key-value pair or key only
    HashNode* next;       // pointer to next node in the bucket
};
```

对于 `unordered_map`，`Value` 是 `std::pair<const Key, T>`。对于 `unordered_set`，`Value` 就是 `Key` 本身。

> 标准并没有规定哈希表的内部实现细节，只规定了接口和复杂度要求。不同标准库实现可能有差异，但链地址法是最常见的选择。

## 四、负载因子与自动扩容

### 4.1 什么是负载因子

**负载因子**（load factor）定义为：

```
load_factor = size / bucket_count
```

其中 `size` 是容器中元素的数量，`bucket_count` 是桶的数量。负载因子越高，说明平均每个桶里的元素越多，冲突越严重，查找性能越接近 O(n)。

### 4.2 max_load_factor 与自动 rehash

STL 为每个无序容器维护一个 `max_load_factor`，默认值为 **1.0**。当插入操作导致 `load_factor >= max_load_factor` 时，容器会自动触发 **rehash**：

1. 分配一个更大的桶数组（通常是当前大小的两倍左右）
2. 重新计算每个元素的桶位置
3. 释放旧的桶数组

```cpp
std::unordered_map<std::string, int> word_count;

std::cout << "bucket count: " << word_count.bucket_count() << "\n";
std::cout << "max load factor: " << word_count.max_load_factor() << "\n";

// Insert elements and watch rehash
for (int i = 0; i < 100; ++i) {
    word_count["word_" + std::to_string(i)] = i;
    std::cout << "size=" << word_count.size()
              << " buckets=" << word_count.bucket_count()
              << " load_factor=" << word_count.load_factor() << "\n";
}
```

输出会显示桶数量在关键时刻跳跃增长，负载因子随之下降。

### 4.3 手动控制 rehash

你可以手动控制 rehash 的时机，避免在性能敏感的操作中意外触发：

```cpp
std::unordered_map<int, std::string> data;

// Pre-allocate buckets for at least 1000 elements
data.rehash(1000);

// Or use reserve (more intuitive)
data.reserve(1000);

// Check current state
std::cout << "buckets: " << data.bucket_count() << "\n";
std::cout << "load factor: " << data.load_factor() << "\n";
```

`rehash(n)` 确保桶数量足够容纳 `n` 个元素而不超过 `max_load_factor`。`reserve(n)` 功能类似，但语义更清晰：预留空间给 `n` 个元素。

> 如果你能预估元素数量，提前 `reserve` 可以避免多次 rehash，显著提升插入性能。

### 4.4 调整 max_load_factor

```cpp
std::unordered_map<int, int> cache;

// Lower max load factor: more buckets, less collisions, more memory
cache.max_load_factor(0.5f);

// Higher max load factor: fewer buckets, more collisions, less memory
cache.max_load_factor(2.0f);
```

降低 `max_load_factor` 会使用更多内存来换取更少的冲突，适合查找密集的场景。调高则相反。通常默认的 1.0 是个不错的平衡点。

## 五、std::unordered_map 详解

`std::unordered_map` 是最常用的无序关联容器，存储键值对（key-value pairs），键唯一。

### 5.1 基本操作

```cpp
#include <iostream>
#include <string>
#include <unordered_map>

int main() {
    std::unordered_map<std::string, int> ages;

    // Insert: multiple methods
    ages.insert({"Alice", 30});
    ages.emplace("Bob", 25);
    ages["Charlie"] = 35;

    // operator[]: access or insert
    // If key exists, returns reference to value
    // If key does NOT exist, inserts default-constructed value
    ages["David"] = 28;        // inserts {David, 0}, then assigns 28
    int alice_age = ages["Alice"]; // returns 30

    // at(): access with bounds checking
    // Throws std::out_of_range if key not found
    try {
        int eve_age = ages.at("Eve");
    } catch (const std::out_of_range& e) {
        std::cout << "Key not found: " << e.what() << "\n";
    }

    // find(): returns iterator or end()
    auto it = ages.find("Bob");
    if (it != ages.end()) {
        std::cout << it->first << " is " << it->second << " years old\n";
    }

    // count(): 0 or 1 (keys are unique)
    if (ages.count("Alice") > 0) {
        std::cout << "Alice is in the map\n";
    }

    // contains() (C++20): cleaner boolean check
#if __cplusplus >= 202002L
    if (ages.contains("Frank")) {
        std::cout << "Frank is in the map\n";
    }
#endif

    // erase: by key, by iterator, or by range
    ages.erase("Charlie");             // by key, returns number removed
    auto erase_it = ages.find("David");
    if (erase_it != ages.end()) {
        ages.erase(erase_it);          // by iterator
    }

    // Iterate (order is NOT guaranteed)
    for (const auto& [name, age] : ages) {
        std::cout << name << ": " << age << "\n";
    }

    return 0;
}
```

### 5.2 operator[] vs at()

这两个访问方法的区别非常重要：

```cpp
std::unordered_map<std::string, int> scores;

// operator[] is PERMISSIVE:
// - If key exists: returns reference to value
// - If key missing: INSERTS default value, returns reference
scores["math"];  // Oops! Inserts {"math", 0} even if unintended

// at() is STRICT:
// - If key exists: returns reference to value
// - If key missing: throws std::out_of_range
int val = scores.at("physics");  // Throws! Key does not exist
```

> 如果只是想查询某个键是否存在或读取值，**不要用 `operator[]`**。它会在键不存在时默默插入一个默认值，可能污染你的数据。

### 5.3 insert vs emplace

```cpp
std::unordered_map<int, std::string> data;

// insert: creates the object first, then copies/moves it in
data.insert(std::pair<int, std::string>(1, "one"));

// emplace: constructs the object in-place, no temporary
data.emplace(2, "two");

// emplace with hint (C++17+)
auto hint = data.begin();
data.emplace_hint(hint, 3, "three");
```

`emplace` 通常更高效，因为它避免了临时对象的创建。返回值都是 `pair<iterator, bool>`，其中 `bool` 表示是否插入成功（键已存在时为 `false`）。

```cpp
auto [iter1, ok1] = data.emplace(1, "another one");
std::cout << "Inserted? " << std::boolalpha << ok1 << "\n";  // false

auto [iter2, ok2] = data.emplace(4, "four");
std::cout << "Inserted? " << std::boolalpha << ok2 << "\n";  // true
```

### 5.4 桶信息查询

无序容器提供了一组桶相关的接口，可以用来诊断性能问题：

```cpp
std::unordered_map<std::string, int> map_data = {
    {"alpha", 1}, {"bravo", 2}, {"charlie", 3},
    {"delta", 4}, {"echo", 5},  {"foxtrot", 6}
};

// Total number of buckets
std::cout << "bucket_count: " << map_data.bucket_count() << "\n";

// Which bucket does a key belong to?
std::size_t bucket_idx = map_data.bucket("alpha");
std::cout << "alpha is in bucket " << bucket_idx << "\n";

// How many elements in a specific bucket?
std::cout << "bucket " << bucket_idx << " has "
          << map_data.bucket_size(bucket_idx) << " elements\n";

// Current load factor
std::cout << "load_factor: " << map_data.load_factor() << "\n";

// Iterate over a specific bucket
std::cout << "Elements in bucket " << bucket_idx << ":\n";
for (auto local_it = map_data.begin(bucket_idx);
     local_it != map_data.end(bucket_idx); ++local_it) {
    std::cout << "  " << local_it->first << " -> " << local_it->second << "\n";
}
```

如果发现某个桶的 `bucket_size` 特别大，说明哈希冲突严重，可能需要优化哈希函数。

### 5.5 删除操作

```cpp
std::unordered_map<int, std::string> items = {
    {1, "one"}, {2, "two"}, {3, "three"}, {4, "four"}, {5, "five"}
};

// Erase by key: returns 0 or 1
std::size_t removed = items.erase(3);
std::cout << "Removed " << removed << " element(s)\n";

// Erase by iterator: returns iterator to next element
auto next = items.erase(items.find(2));

// Erase by range
items.erase(items.begin(), items.end());  // clears all

// clear(): remove everything
items.clear();
```

## 六、std::unordered_set 详解

`std::unordered_set` 是 `unordered_map` 的"只有键"版本。它存储唯一的键，不关联任何值。适合需要快速判断"某个元素是否存在"的场景。

### 6.1 基本操作

```cpp
#include <iostream>
#include <string>
#include <unordered_set>

int main() {
    std::unordered_set<std::string> visited;

    // Insert
    visited.insert("page_1");
    visited.insert("page_2");
    visited.emplace("page_3");

    // Duplicate insert: returns pair<iterator, bool>
    auto [iter, ok] = visited.insert("page_1");
    std::cout << "Inserted page_1 again? " << std::boolalpha << ok << "\n";  // false

    // find
    if (visited.find("page_2") != visited.end()) {
        std::cout << "page_2 was visited\n";
    }

    // count: 0 or 1
    std::cout << "page_4 count: " << visited.count("page_4") << "\n";  // 0

    // contains (C++20)
#if __cplusplus >= 202002L
    if (visited.contains("page_1")) {
        std::cout << "page_1 exists\n";
    }
#endif

    // No operator[] — set has no mapped value

    // Erase
    visited.erase("page_3");

    // Iterate
    for (const auto& page : visited) {
        std::cout << page << "\n";
    }

    return 0;
}
```

### 6.2 典型应用场景

`unordered_set` 特别适合以下场景：

```cpp
// Deduplication
std::vector<int> raw_data = {3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5};
std::unordered_set<int> unique_data(raw_data.begin(), raw_data.end());
// unique_data: {3, 1, 4, 5, 9, 2, 6} (order varies)

// Membership test (fast "is in collection?")
std::unordered_set<std::string> StopWords = {
    "the", "a", "an", "is", "are", "was", "were", "in", "on", "at"
};

std::string word = "the";
if (StopWords.contains(word)) {
    // Skip this word
}

// Intersection check
std::unordered_set<int> set_a = {1, 2, 3, 4, 5};
std::unordered_set<int> set_b = {4, 5, 6, 7, 8};
for (int val : set_a) {
    if (set_b.contains(val)) {
        std::cout << val << " is in both sets\n";
    }
}
```

### 6.3 unordered_multiset 与 unordered_multimap

如果你需要允许重复键，STL 也提供了对应的多键版本：

- **`std::unordered_multiset`**：允许重复元素，`count()` 可以大于 1，`equal_range()` 返回所有匹配元素
- **`std::unordered_multimap`**：允许重复键，没有 `operator[]` 和 `at()`，需要用 `equal_range()` 访问同一键的所有值

```cpp
#include <unordered_set>
#include <unordered_map>
#include <iostream>

void DemoMultiContainers() {
    std::unordered_multiset<int> multi_set;
    multi_set.insert(1);
    multi_set.insert(1);
    multi_set.insert(1);
    std::cout << "count of 1: " << multi_set.count(1) << "\n";  // 3

    std::unordered_multimap<std::string, int> multi_map;
    multi_map.emplace("key", 10);
    multi_map.emplace("key", 20);
    multi_map.emplace("key", 30);

    auto range = multi_map.equal_range("key");
    for (auto it = range.first; it != range.second; ++it) {
        std::cout << it->first << " -> " << it->second << "\n";
    }
}
```

## 七、自定义哈希函数

默认情况下，STL 为基本类型（`int`, `double`, `std::string`, 指针等）提供了 `std::hash` 特化。但如果你想在无序容器中使用自定义类型作为键，就必须提供自己的哈希函数。

### 7.1 std::hash 特化

第一种方式是为你的类型特化 `std::hash`：

```cpp
#include <functional>
#include <string>
#include <unordered_map>

struct Person {
    std::string name;
    int birth_year;
    std::string city;

    bool operator==(const Person& other) const {
        return name == other.name
            && birth_year == other.birth_year
            && city == other.city;
    }
};

// Specialize std::hash for Person
namespace std {
template <>
struct hash<Person> {
    std::size_t operator()(const Person& p) const {
        // Combine hashes of all fields
        std::size_t h1 = std::hash<std::string>{}(p.name);
        std::size_t h2 = std::hash<int>{}(p.birth_year);
        std::size_t h3 = std::hash<std::string>{}(p.city);
        return h1 ^ (h2 << 1) ^ (h3 << 2);
    }
};
} // namespace std

int main() {
    std::unordered_map<Person, std::string> person_notes;
    person_notes[{"Alice", 1990, "Beijing"}] = "Team lead";
    person_notes[{"Bob", 1985, "Shanghai"}] = "Architect";
    return 0;
}
```

> 特化 `std::hash` 时要小心命名空间。必须放在 `namespace std` 中，且只能对自定义类型（你控制的类型）进行特化。对标准库类型进行特化是未定义行为。

### 7.2 函数对象（Functor）

第二种方式是写一个函数对象，然后作为模板参数传入：

```cpp
struct PersonHash {
    std::size_t operator()(const Person& p) const {
        std::size_t h1 = std::hash<std::string>{}(p.name);
        std::size_t h2 = std::hash<int>{}(p.birth_year);
        std::size_t h3 = std::hash<std::string>{}(p.city);
        return h1 ^ (h2 << 1) ^ (h3 << 2);
    }
};

// Use as template argument
std::unordered_map<Person, std::string, PersonHash> person_map;
```

这种方式不需要侵入 `std` 命名空间，通常更安全、更推荐。

### 7.3 组合多个字段的哈希值

上面代码中使用的 `h1 ^ (h2 << 1) ^ (h3 << 2)` 是一种简单但不完美的哈希组合方式。Boost 提供了更好的 `hash_combine`。这里给出一个常见的手动实现：

```cpp
#include <functional>
#include <string>
#include <unordered_map>

// A portable hash_combine implementation
template <typename T>
void HashCombine(std::size_t& seed, const T& val) {
    seed ^= std::hash<T>{}(val) + 0x9e3779b9 + (seed << 6) + (seed >> 2);
}

// Variadic version for multiple fields
template <typename... Args>
std::size_t HashCombineAll(const Args&... args) {
    std::size_t seed = 0;
    (HashCombine(seed, args), ...);  // C++17 fold expression
    return seed;
}

struct Student {
    int student_id;
    std::string name;
    std::string department;

    bool operator==(const Student& other) const {
        return student_id == other.student_id
            && name == other.name
            && department == other.department;
    }
};

struct StudentHash {
    std::size_t operator()(const Student& s) const {
        return HashCombineAll(s.student_id, s.name, s.department);
    }
};

int main() {
    std::unordered_map<Student, double, StudentHash> gpas;

    gpas[{2021001, "Zhang San", "CS"}] = 3.8;
    gpas[{2021002, "Li Si", "EE"}] = 3.5;
    gpas[{2021003, "Wang Wu", "Math"}] = 3.9;

    auto it = gpas.find({2021001, "Zhang San", "CS"});
    if (it != gpas.end()) {
        std::cout << "GPA: " << it->second << "\n";
    }

    return 0;
}
```

这个 `HashCombineAll` 利用了 C++17 的折叠表达式，支持任意数量、任意类型的字段组合。

### 7.4 使用 lambda 作为哈希函数（C++20 之前）

在 C++20 之前，lambda 不能直接作为模板参数。需要一个包装：

```cpp
// Wrapper to use lambda as hasher
struct PersonHashLambda {
    auto operator()(const Person& p) const -> std::size_t {
        auto h = [](const Person& person) {
            std::size_t h1 = std::hash<std::string>{}(person.name);
            std::size_t h2 = std::hash<int>{}(person.birth_year);
            return h1 ^ h2;
        };
        return h(p);
    }
};
```

C++20 中无状态 lambda 可以直接用在未计算上下文中，但实际使用 functor 仍然是最清晰的方式。

## 八、自定义键相等判断

无序容器需要两个东西来处理键：哈希函数（把键映射到桶）和**相等判断**（在桶内判断两个键是否相同）。

默认使用 `operator==`。如果你不能修改类型的 `operator==`，或者需要不同的相等语义，可以自定义：

```cpp
#include <functional>
#include <string>
#include <unordered_map>
#include <algorithm>

struct CaseInsensitiveString {
    std::string data;

    explicit CaseInsensitiveString(std::string s) : data(std::move(s)) {}
};

// Custom hash: convert to lowercase before hashing
struct CaseInsensitiveHash {
    std::size_t operator()(const CaseInsensitiveString& s) const {
        std::string lower = s.data;
        std::transform(lower.begin(), lower.end(), lower.begin(),
                       [](unsigned char c) { return std::tolower(c); });
        return std::hash<std::string>{}(lower);
    }
};

// Custom equality: case-insensitive comparison
struct CaseInsensitiveEqual {
    bool operator()(const CaseInsensitiveString& a,
                    const CaseInsensitiveString& b) const {
        if (a.data.size() != b.data.size()) return false;
        return std::equal(a.data.begin(), a.data.end(), b.data.begin(),
                         [](unsigned char ca, unsigned char cb) {
                             return std::tolower(ca) == std::tolower(cb);
                         });
    }
};

int main() {
    std::unordered_map<CaseInsensitiveString, int,
                       CaseInsensitiveHash,
                       CaseInsensitiveEqual> word_freq;

    word_freq[CaseInsensitiveString("Hello")] = 1;
    word_freq[CaseInsensitiveString("hello")]++;  // Same key!
    word_freq[CaseInsensitiveString("HELLO")]++;  // Same key!

    std::cout << "count: " << word_freq.size() << "\n";         // 1
    std::cout << "Hello freq: " << word_freq[CaseInsensitiveString("hello")] << "\n";  // 3

    return 0;
}
```

> 自定义的哈希函数和相等判断必须保持一致：如果 `equal_to(a, b)` 为 `true`，那么 `hash(a) == hash(b)` 也必须为 `true`。反过来的方向不要求。

模板参数的完整顺序是：

```cpp
std::unordered_map<Key, T, Hash, KeyEqual, Allocator>
std::unordered_set<Key, Hash, KeyEqual, Allocator>
```

## 九、性能分析

### 9.1 时间复杂度

| 操作 | 平均情况 | 最坏情况 |
|------|----------|----------|
| 查找（`find`, `at`, `count`） | O(1) | O(n) |
| 插入（`insert`, `emplace`） | O(1) | O(n) |
| 删除（`erase`） | O(1) | O(n) |
| `operator[]` | O(1) | O(n) |
| `rehash` | - | O(n) |

平均 O(1) 的前提是**哈希函数质量良好**且**负载因子合理**。最坏 O(n) 发生在所有元素都哈希到同一个桶时（哈希函数极差或恶意构造的输入）。

### 9.2 与有序容器的对比

| 特性 | `unordered_map` / `unordered_set` | `map` / `set` |
|------|-----------------------------------|----------------|
| 底层结构 | 哈希表 | 红黑树 |
| 查找 | O(1) 平均 | O(log n) |
| 插入 | O(1) 平均 | O(log n) |
| 删除 | O(1) 平均 | O(log n) |
| 元素顺序 | 无序 | 按键升序 |
| 内存开销 | 桶数组 + 链表指针 | 红黑树节点（3个指针 + 颜色） |
| 对键的要求 | `Hash` + `Equal` | `Compare`（`operator<`） |
| 迭代器失效 | rehash 时全部失效 | 插入不影响，删除只失效被删元素 |
| 范围查询 | 不支持 | 支持（`lower_bound`, `upper_bound`） |
| `std::algorithm` 兼容 | 需注意迭代器类别 | 随机访问不行，双向迭代器可用 |

### 9.3 实际性能参考

用一个简单的基准测试来感受差异：

```cpp
#include <chrono>
#include <iostream>
#include <map>
#include <unordered_map>
#include <vector>
#include <random>

void BenchmarkLookup(int num_elements) {
    std::vector<int> keys(num_elements);
    for (int i = 0; i < num_elements; ++i) {
        keys[i] = i;
    }

    // Shuffle keys for realistic lookup pattern
    std::random_device rd;
    std::mt19937 gen(rd());
    std::shuffle(keys.begin(), keys.end(), gen);

    std::unordered_map<int, int> umap;
    std::map<int, int> omap;

    for (int k : keys) {
        umap[k] = k * 2;
        omap[k] = k * 2;
    }

    // Benchmark unordered_map lookup
    auto start = std::chrono::high_resolution_clock::now();
    volatile int sum = 0;  // volatile to prevent optimization
    for (int k : keys) {
        auto it = umap.find(k);
        if (it != umap.end()) {
            sum += it->second;
        }
    }
    auto end = std::chrono::high_resolution_clock::now();
    auto umap_time = std::chrono::duration_cast<std::chrono::microseconds>(end - start);

    // Benchmark map lookup
    start = std::chrono::high_resolution_clock::now();
    sum = 0;
    for (int k : keys) {
        auto it = omap.find(k);
        if (it != omap.end()) {
            sum += it->second;
        }
    }
    end = std::chrono::high_resolution_clock::now();
    auto omap_time = std::chrono::duration_cast<std::chrono::microseconds>(end - start);

    std::cout << "n=" << num_elements
              << " unordered_map: " << umap_time.count() << " us"
              << " map: " << omap_time.count() << " us"
              << " ratio: " << static_cast<double>(omap_time.count()) / umap_time.count()
              << "\n";
}
```

在 n = 100,000 量级时，`unordered_map` 的查找通常比 `map` 快 3 到 10 倍。差距随 n 增大而增大，因为 O(1) 和 O(log n) 之间的差异会越来越明显。

## 十、迭代器失效

这是一个容易踩坑的地方。无序容器的迭代器失效规则与有序容器不同。

### 10.1 rehash 导致全部失效

当插入操作触发 rehash 时，**所有迭代器、引用和指针都会失效**。这是无序容器最危险的迭代器失效场景。

```cpp
std::unordered_map<int, std::string> data = {{1, "one"}, {2, "two"}};

// DANGEROUS: iterator may be invalidated by insert
auto it = data.find(1);
data[3] = "three";  // May trigger rehash!
// it is now INVALID — do NOT use it!

// SAFE: use the returned iterator from insert
auto [new_it, ok] = data.emplace(4, "four");
// new_it is valid, but all previous iterators might be invalid
```

### 10.2 删除只失效被删元素

`erase` 只会使指向被删除元素的迭代器失效，其他迭代器不受影响：

```cpp
std::unordered_map<int, int> values = {
    {1, 10}, {2, 20}, {3, 30}, {4, 40}
};

// Safe erase-while-iterating pattern
for (auto it = values.begin(); it != values.end(); ) {
    if (it->second < 25) {
        it = values.erase(it);  // erase returns next valid iterator
    } else {
        ++it;
    }
}
// values now contains {3, 30} and {4, 40}
```

### 10.3 安全的遍历删除模式

C++20 引入了 `std::erase_if`，让删除操作更简洁安全：

```cpp
#if __cplusplus >= 202002L
#include <unordered_map>

std::unordered_map<int, int> values = {
    {1, 10}, {2, 20}, {3, 30}, {4, 40}
};

// C++20: erase_if removes elements matching the predicate
std::erase_if(values, [](const auto& item) {
    return item.second < 25;
});
#endif
```

> 总结：插入可能导致全部迭代器失效（rehash），删除只失效被删元素的迭代器。在遍历中插入元素时要格外小心。

## 十一、常见陷阱

### 11.1 operator[] 的副作用

```cpp
std::unordered_map<std::string, int> word_freq;

// INTENDED: count occurrences
std::vector<std::string> words = {"hello", "world", "hello"};

for (const auto& w : words) {
    word_freq[w]++;  // OK: inserts 0 first time, then increments
}

// UNINTENDED: just checking existence
void CheckExistence() {
    std::unordered_map<std::string, int> config = {{"timeout", 30}};

    // BUG: this inserts "debug_mode" with value 0!
    if (config["debug_mode"]) {
        EnableDebug();
    }
    // config now has {"debug_mode", 0}

    // CORRECT: use find or contains
    if (auto it = config.find("debug_mode"); it != config.end()) {
        if (it->second) {
            EnableDebug();
        }
    }

    // C++20: use contains
    if (config.contains("debug_mode") && config.at("debug_mode")) {
        EnableDebug();
    }
}
```

### 11.2 哈希函数质量差

一个糟糕的哈希函数可以把哈希表退化为链表：

```cpp
// TERRIBLE hash function: everything goes to bucket 0
struct BadHash {
    std::size_t operator()(int val) const {
        return 0;  // All keys hash to the same value!
    }
};

std::unordered_map<int, std::string, BadHash> bad_map;
// Every lookup becomes O(n), no better than a linked list

// GOOD hash function: distributes keys uniformly
struct GoodHash {
    std::size_t operator()(int val) const {
        // std::hash<int> already does a decent job
        return std::hash<int>{}(val);
    }
};
```

> 如果你发现 `unordered_map` 的性能不理想，第一步应该检查哈希函数的分布质量。可以用 `bucket_size()` 来查看各桶的负载是否均匀。

### 11.3 自定义类型缺少 hash 或 ==

```cpp
struct Point {
    int x, y;
    // Forgot to define operator== and hash!
};

// This will NOT compile:
// std::unordered_map<Point, int> point_map;
// Error: no match for std::hash<Point>

// FIX: provide both hash and equality
struct PointHash {
    std::size_t operator()(const Point& p) const {
        return HashCombineAll(p.x, p.y);
    }
};

// Need operator== for default KeyEqual
bool operator==(const Point& a, const Point& b) {
    return a.x == b.x && a.y == b.y;
}

// Now it works
std::unordered_map<Point, int, PointHash> point_map;
point_map[{3, 4}] = 5;
```

### 11.4 修改键的值

```cpp
std::unordered_map<std::string, int> data = {{"key", 42}};

// The key is const — you CANNOT modify it through the iterator
auto it = data.find("key");
// it->first = "new_key";  // COMPILE ERROR: first is const

// To "change" a key, you must erase and re-insert
int old_value = it->second;
data.erase(it);
data["new_key"] = old_value;
```

键的 `const` 限定是有序和无序容器共同的特性。如果允许修改键，会破坏内部数据结构（红黑树的排序关系或哈希表的桶分配）。

### 11.5 异常安全

```cpp
std::unordered_map<std::string, std::vector<int>> data;

// If vector copy throws, the map state is still valid
try {
    data["large_key"] = std::vector<int>(1000000, 42);
} catch (const std::bad_alloc& e) {
    // data is in valid state, but "large_key" may or may not exist
    // (operator[] already inserted it with default value)
}
```

## 十二、有序 vs 无序：选择指南

### 12.1 选择无序容器的情况

选择 `unordered_map` 或 `unordered_set`，当：

- 只需要快速查找，不关心元素顺序
- 查找操作远多于遍历操作
- 键类型有良好（或可以提供）的哈希函数
- 不需要范围查询（`lower_bound`, `upper_bound`）
- 追求最佳的单次查找性能

### 12.2 选择有序容器的情况

选择 `map` 或 `set`，当：

- 需要按键排序遍历元素
- 需要范围查询（"找出所有 key >= X 且 key < Y 的元素"）
- 需要稳定的迭代器（插入不导致迭代器失效）
- 键类型没有自然的哈希函数，但有自然的比较操作
- 需要用 `lower_bound`、`upper_bound` 等有序操作

### 12.3 决策流程图

```
需要范围查询或有序遍历？
├── 是 → map / set
└── 否
    └── 键类型有可用的哈希函数？
        ├── 是 → unordered_map / unordered_set
        └── 否
            └── 能为键定义 operator< 吗？
                ├── 是 → map / set
                └── 否 → 需要同时自定义 hash 和 equal，或重新考虑设计
```

### 12.4 实际场景对照

| 场景 | 推荐容器 | 原因 |
|------|----------|------|
| 字符串频率统计 | `unordered_map` | 纯查找，不需要顺序 |
| 配置项存储 | `unordered_map` | 按键查找配置值 |
| 字典 / 单词集合 | `unordered_set` | 快速判断单词是否存在 |
| 时间序列数据索引 | `map` | 需要按时间排序 |
| 优先级队列映射 | `map` | 需要有序遍历优先级 |
| LRU 缓存 | `unordered_map` + list | 快速查找 + 顺序维护 |
| 拼写检查器 | `unordered_set` | 大量查找，不需要顺序 |
| 区间查询（如日程管理） | `map` | 需要 `lower_bound`/`upper_bound` |
| IP 地址到地理位置 | `unordered_map` | 纯查找 |
| JSON 对象 | `unordered_map`（或有序版本） | 键值对存储 |

## 十三、总结

`std::unordered_map` 和 `std::unordered_set` 是 C++ 标准库中基于哈希表的高性能关联容器。理解它们的内部原理，能帮助你做出更好的设计决策。

核心要点回顾：

1. **底层数据结构是链地址法哈希表**。每个桶是一个链表，冲突元素串在同一桶中
2. **平均 O(1) 查找**，前提是哈希函数质量好、负载因子合理。最坏情况退化为 O(n)
3. **`operator[]` 会默默插入**。如果只想查询，用 `find()`、`at()` 或 `contains()`
4. **自定义类型需要 `Hash` 和 `Equal`**。两者必须一致：相等的对象必须产生相同的哈希值
5. **`rehash` 使所有迭代器失效**。在持有迭代器时插入元素要格外小心
6. **`reserve()` 是你的朋友**。如果知道元素数量，提前预留空间可以避免多次 rehash
7. **选择有序还是无序取决于需求**。需要顺序和范围查询用 `map`/`set`，只需要快速查找用 `unordered_map`/`unordered_set`

掌握这些知识，你就能在 C++ 项目中自信地选择和使用无序关联容器了。
