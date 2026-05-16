---
title: "C++ 有序关联容器：map 与 set 的红黑树实现全解析"
date: 2021-08-13T10:00:00+08:00
tags: ["C++", "STL", "map", "set", "红黑树", "关联容器"]
categories: ["C++"]
summary: "从红黑树的平衡原理出发，深入剖析 std::map、std::set、std::multimap、std::multiset 的底层实现、查找/插入/删除的时间复杂度、自定义比较器的方法，以及有序遍历、范围查询等独特能力。"
ShowToc: true
---

C++ 标准模板库（STL）提供了四类有序关联容器：`std::map`、`std::set`、`std::multimap`、`std::multiset`。它们的核心特征是**按键有序排列**，背后靠一棵**红黑树**（Red-Black Tree）来维持平衡。

这篇笔记从红黑树的基本原理开始，逐步拆解这四个容器的底层结构、常用操作、性能特征，以及实际工程中容易踩到的坑。

---

## 有序关联容器一览

先快速建立一个整体印象。四个容器的区别主要在两个维度：存的是**键值对还是单纯键**，是否**允许重复键**。

| 容器            | 存储内容   | 键是否唯一 | 底层结构  |
| --------------- | ---------- | ---------- | --------- |
| `std::map`      | key-value  | 是         | 红黑树    |
| `std::set`      | key        | 是         | 红黑树    |
| `std::multimap` | key-value  | 否         | 红黑树    |
| `std::multiset` | key        | 否         | 红黑树    |

不管哪一个，**查找、插入、删除**的时间复杂度都是 **O(log n)**。这棵树始终保持平衡，不会退化成链表。

和 `std::unordered_map` / `std::unordered_set` 相比，有序容器最大的优势是：**你可以按顺序遍历，可以做范围查询**。代价是常数因子稍大（哈希表平均 O(1)），而且每个节点要多存几个指针。

---

## 红黑树基础

### 为什么需要平衡二叉搜索树

普通的二叉搜索树（BST）在最坏情况下会退化成链表。想象一下按升序插入 `1, 2, 3, 4, 5`，每个新节点都是右孩子，树的高度变成 n，查找变成 O(n)。

红黑树通过给节点染色并附加几条规则，保证**最长路径不超过最短路径的两倍**。这样一来，树的高度始终保持在 O(log n)，所有操作的对数复杂度就有了保障。

### 红黑树的五条性质

> 红黑树的每条规则都在限制"偏斜"程度。

1. 每个节点要么是**红色**，要么是**黑色**。
2. 根节点是**黑色**的。
3. 所有叶子节点（NIL 节点）是**黑色**的。
4. 如果一个节点是红色的，则它的两个子节点都是**黑色**的。（不能有连续的红色节点。）
5. 从任一节点到其所有叶子节点的路径上，**黑色节点的数量相同**。（这个数量叫"黑高"。）

这五条性质共同约束了树的结构。性质 4 限制了红色节点的密度，性质 5 保证了平衡性。由此可以推导出：最长路径（红黑交替）最多是最短路径（全黑）的两倍。

### 旋转：维护平衡的基本操作

插入或删除节点后，红黑树的性质可能被破坏。恢复性质的手段是**旋转**（rotation）加**重新着色**。

旋转分两种：

- **左旋**：把右孩子提上来当父节点，原父节点变成左孩子。
- **右旋**：把左孩子提上来当父节点，原父节点变成右孩子。

```
    左旋 (x)              右旋 (y)
       y                     x
      / \                   / \
     x   C       <=>       A   y
    / \                       / \
   A   B                     B   C
```

旋转只改变指针关系，时间 O(1)。插入最多旋转两次，删除最多旋转三次。整体操作还是 O(log n)。

STL 的红黑树实现（在 GCC 中是 `_Rb_tree`）把插入后的调整逻辑封装在 `_Rb_tree_rebalance` 函数里，删除后的调整封装在 `_Rb_tree_rebalance_for_erase` 里。日常使用不需要关心这些细节，但知道它们的存在有助于理解为什么插入/删除不会导致迭代器大规模失效（后面会详述）。

---

## 节点结构

### map 的节点

`std::map<Key, Value>` 的每个树节点存储一个 `std::pair<const Key, Value>`。注意 `const`：**键一旦插入就不可修改**，否则会破坏树的有序性。

```
        [10, "hello"]           <-- 节点存储 pair<const Key, Value>
        /          \
   [5, "world"]    [15, "foo"]
   /     \              \
 NIL     NIL          NIL
```

简化后的节点内存布局：

```cpp
// Conceptual structure, not the exact STL source
struct RbTreeNode {
    RbTreeNode* left;
    RbTreeNode* right;
    RbTreeNode* parent;
    bool        is_red;
    std::pair<const Key, Value> data;
};
```

### set 的节点

`std::set<Key>` 的节点只存一个 `Key`，没有 value 部分。底层和 map 共享同一套红黑树代码，只是 value 类型变成了一个空类型（类似 `std::nullptr_t` 或者一个占位符），然后通过模板特化避免浪费空间。

```
        [10]               <-- 节点只存 Key
        /  \
      [5]  [15]
```

> 实际上，GCC 的实现中 `std::set` 依然使用 `_Rb_tree<Key, Key, ...>` 模板，value 字段存在但被设计成不占实际空间的空基类优化（EBO）。所以 `set` 的节点不比 `map` 的节点少一个指针，只是省掉了 value 的空间。

---

## std::map 详解

`std::map` 是最常用的有序关联容器。它维护一组有序的键值对，支持通过键快速查找、插入和删除。

### 基本操作一览

```cpp
#include <iostream>
#include <map>
#include <string>

int main() {
    std::map<std::string, int> age_map;

    // --- insertion ---
    age_map.insert({"Alice", 30});
    age_map.insert(std::make_pair("Bob", 25));
    age_map.emplace("Charlie", 35);    // construct in-place

    // --- operator[] ---
    // Read: returns value if key exists
    // Write: INSERTS default-constructed value if key does NOT exist!
    std::cout << age_map["Alice"] << "\n";   // 30
    age_map["David"] = 28;                    // inserts {David, 0}, then assigns 28

    // --- at() ---
    // Like operator[], but throws std::out_of_range if not found
    try {
        std::cout << age_map.at("Eve") << "\n";
    } catch (const std::out_of_range& e) {
        std::cout << "Eve not found\n";
    }

    // --- lookup ---
    auto it = age_map.find("Bob");
    if (it != age_map.end()) {
        std::cout << "Bob is " << it->second << "\n";  // 25
    }

    std::cout << age_map.count("Alice") << "\n";     // 1
    std::cout << age_map.count("Eve") << "\n";       // 0

    // C++20 contains()
    // if (age_map.contains("Alice")) { ... }

    // --- erase ---
    age_map.erase("Charlie");        // by key, returns number removed
    age_map.erase(age_map.begin());  // by iterator

    // --- iteration (sorted by key) ---
    for (const auto& [name, age] : age_map) {
        std::cout << name << ": " << age << "\n";
    }

    return 0;
}
```

### operator[] 的陷阱

`operator[]` 是 `std::map` 里最容易踩坑的地方。它的行为是：

- 如果键**存在**，返回对应 value 的引用。
- 如果键**不存在**，**插入一个默认构造的元素**，然后返回其引用。

这意味着：

```cpp
std::map<std::string, int> scores;
int val = scores["unknown"];  // silently inserts {"unknown", 0}!
```

> 如果你不希望自动插入，请用 `at()` 或 `find()`。

一个常见的错误是在只读场景下使用 `operator[]`，比如打印 map 内容：

```cpp
// BAD: const map cannot use operator[]
void PrintMap(const std::map<std::string, int>& m) {
    // m["key"];  // compile error! operator[] is not const
}

// GOOD: use find or at
void PrintMap(const std::map<std::string, int>& m) {
    auto it = m.find("key");
    if (it != m.end()) {
        std::cout << it->second << "\n";
    }
}
```

### insert vs emplace

`insert` 接收一个已经构造好的对象（或初始化列表），而 `emplace` 在容器内部直接构造对象，省去了一次临时对象的创建和移动。

```cpp
struct Person {
    std::string name;
    int age;
    Person(std::string n, int a) : name(std::move(n)), age(a) {}
};

std::map<int, Person> people;

// insert: constructs Person first, then moves/copies into the tree node
people.insert({1, Person("Alice", 30)});

// emplace: forwards arguments, constructs Person directly in the node
people.emplace(1, "Alice", 30);
```

两者都返回一个 `std::pair<iterator, bool>`，其中 `bool` 表示是否插入成功（键重复时返回 `false`）。

### 有序范围查询

`std::map` 提供了 `lower_bound` 和 `upper_bound`，利用红黑树的有序性做范围查询。

- `lower_bound(key)`：返回指向**第一个不小于 key** 的元素的迭代器。
- `upper_bound(key)`：返回指向**第一个大于 key** 的元素的迭代器。
- `equal_range(key)`：返回 `std::pair`，等价于 `{lower_bound(key), upper_bound(key)}`。

```cpp
#include <iostream>
#include <map>

int main() {
    std::map<int, std::string> data = {
        {10, "a"}, {20, "b"}, {30, "c"},
        {40, "d"}, {50, "e"}
    };

    // Find all entries with key >= 20 and key < 40
    auto lo = data.lower_bound(20);  // points to {20, "b"}
    auto hi = data.upper_bound(39);  // points to {40, "d"}

    for (auto it = lo; it != hi; ++it) {
        std::cout << it->first << ": " << it->second << "\n";
    }
    // Output:
    // 20: b
    // 30: c

    return 0;
}
```

`equal_range` 在 `std::map` 中最多返回一个元素（因为键唯一），但在 `std::multimap` 中非常有用，后面会看到。

### contains (C++20)

C++20 引入了 `contains` 成员函数，比 `find(...) != end()` 更直观：

```cpp
std::map<int, std::string> m = {{1, "one"}, {2, "two"}};

// Before C++20
if (m.find(3) != m.end()) { /* ... */ }

// C++20
if (m.contains(3)) { /* ... */ }
```

---

## std::set 详解

`std::set` 可以理解为只有键、没有值的 `std::map`。它维护一组**不重复的、有序的**元素。

### 核心操作

```cpp
#include <iostream>
#include <set>
#include <vector>

int main() {
    std::set<int> s;

    // --- insertion ---
    s.insert(10);
    s.insert(20);
    s.insert(10);   // duplicate, ignored
    s.insert(5);

    // duplicate check
    auto [iter, inserted] = s.insert(15);
    std::cout << "Inserted: " << inserted << "\n";  // 1 (true)

    auto [iter2, inserted2] = s.insert(10);
    std::cout << "Inserted: " << inserted2 << "\n"; // 0 (false)

    // --- erase ---
    s.erase(5);              // by value, returns 1 if removed
    s.erase(s.begin());      // by iterator

    // --- lookup ---
    if (s.count(20)) {
        std::cout << "20 is in the set\n";
    }

    auto it = s.find(20);
    if (it != s.end()) {
        std::cout << *it << "\n";   // 20
    }

    // --- sorted iteration ---
    for (int val : s) {
        std::cout << val << " ";
    }
    std::cout << "\n";

    // --- range construction ---
    std::vector<int> vec = {5, 3, 1, 4, 2, 3, 5};
    std::set<int> unique_sorted(vec.begin(), vec.end());
    // unique_sorted: {1, 2, 3, 4, 5}

    for (int v : unique_sorted) {
        std::cout << v << " ";
    }
    // Output: 1 2 3 4 5

    return 0;
}
```

### set 没有 operator[]

`std::set` 不提供 `operator[]` 或 `at()`。因为 set 里没有"值"可以返回，它自己就是键。想访问元素，只能通过迭代器或 `find`。

这也意味着不能用下标语法来插入元素，只能用 `insert` 和 `emplace`。

---

## std::multimap 与 std::multiset

当你的业务逻辑允许多个元素拥有相同的键时，就需要 `std::multimap` 或 `std::multiset`。

### std::multimap

```cpp
#include <iostream>
#include <map>
#include <string>

int main() {
    std::multimap<std::string, int> scores;

    // Multiple values for the same key
    scores.insert({"Alice", 85});
    scores.insert({"Alice", 92});
    scores.insert({"Bob", 78});
    scores.insert({"Alice", 88});
    scores.insert({"Bob", 95});

    // count how many entries for Alice
    std::cout << "Alice has " << scores.count("Alice") << " scores\n";  // 3

    // --- equal_range: iterate all values for a key ---
    auto range = scores.equal_range("Alice");
    for (auto it = range.first; it != range.second; ++it) {
        std::cout << "  Alice: " << it->second << "\n";
    }
    // Output:
    //   Alice: 85
    //   Alice: 92
    //   Alice: 88

    // --- erase by key removes ALL entries ---
    scores.erase("Bob");   // removes both {Bob, 78} and {Bob, 95}

    // --- erase a single entry by iterator ---
    auto first_alice = scores.find("Alice");
    if (first_alice != scores.end()) {
        scores.erase(first_alice);  // only removes one entry
    }

    // Remaining entries
    for (const auto& [name, score] : scores) {
        std::cout << name << ": " << score << "\n";
    }

    return 0;
}
```

> `multimap` **没有 `operator[]`**。因为一个键可能对应多个值，`operator[]` 的语义无法定义。你需要用 `insert`、`find`、`equal_range` 来操作。

### std::multiset

```cpp
#include <iostream>
#include <set>
#include <vector>

int main() {
    std::multiset<int> ms;

    // Duplicates are allowed
    ms.insert(10);
    ms.insert(20);
    ms.insert(10);
    ms.insert(10);
    ms.insert(30);

    std::cout << ms.count(10) << "\n";   // 3

    // Sorted order with duplicates
    for (int v : ms) {
        std::cout << v << " ";
    }
    // Output: 10 10 10 20 30

    // --- equal_range for range of duplicates ---
    auto [lo, hi] = ms.equal_range(10);
    std::cout << "\n10 appears " << std::distance(lo, hi) << " times\n";

    // --- erase by value removes ALL copies ---
    ms.erase(10);   // all three 10s are gone

    // --- erase one copy by iterator ---
    auto it = ms.find(20);
    if (it != ms.end()) {
        ms.erase(it);  // only the single 20
    }

    for (int v : ms) {
        std::cout << v << " ";
    }
    // Output: 30

    return 0;
}
```

`multiset` 常见的用途是维护一个**有序的、允许重复的数据流**，比如实时排行榜（同分选手可以并列），或者中位数过滤器的滑窗。

---

## 自定义比较器

默认情况下，`std::map` 和 `std::set` 用 `std::less<Key>` 来比较键的大小。如果你想改变排序规则，可以在模板参数中指定自定义的比较器。

### 使用 std::greater 降序排列

```cpp
#include <iostream>
#include <map>
#include <set>

int main() {
    // map with keys in descending order
    std::map<int, std::string, std::greater<int>> desc_map = {
        {1, "one"}, {3, "three"}, {2, "two"}
    };

    for (const auto& [k, v] : desc_map) {
        std::cout << k << ": " << v << "\n";
    }
    // Output:
    // 3: three
    // 2: two
    // 1: one

    // set with descending order
    std::set<int, std::greater<int>> desc_set = {5, 2, 8, 1};
    for (int v : desc_set) {
        std::cout << v << " ";
    }
    // Output: 8 5 2 1

    return 0;
}
```

### 自定义结构体作为键

当你用自定义类型做 map 或 set 的键时，必须提供一个比较函数。标准库要求的是一个**严格弱序**（strict weak ordering）关系。

```cpp
#include <iostream>
#include <map>
#include <string>

struct Student {
    int class_id;
    int student_id;
    std::string name;
};

// Comparator: sort by class_id, then by student_id
struct StudentComparator {
    bool operator()(const Student& a, const Student& b) const {
        if (a.class_id != b.class_id) {
            return a.class_id < b.class_id;
        }
        return a.student_id < b.student_id;
    }
};

int main() {
    std::map<Student, double, StudentComparator> grades;

    grades.emplace(Student{1, 10, "Alice"}, 95.5);
    grades.emplace(Student{1, 5, "Bob"}, 88.0);
    grades.emplace(Student{2, 3, "Charlie"}, 92.0);

    for (const auto& [student, grade] : grades) {
        std::cout << "Class " << student.class_id
                  << " Student " << student.student_id
                  << " (" << student.name << "): "
                  << grade << "\n";
    }
    // Output:
    // Class 1 Student 5 (Bob): 88
    // Class 1 Student 10 (Alice): 95.5
    // Class 2 Student 3 (Charlie): 92

    return 0;
}
```

> 比较器必须满足严格弱序：不可自反（`comp(a, a) == false`）、非对称（如果 `comp(a, b)` 为真，则 `comp(b, a)` 必须为假）、可传递。违反这些规则会导致未定义行为。

### Lambda 作为比较器（C++11）

你也可以用 lambda 表达式作为比较器，但 lambda 的类型是匿名的，没法直接写在模板参数里。需要用 `decltype` 或者 `std::function`。

```cpp
#include <iostream>
#include <map>
#include <string>
#include <functional>

int main() {
    // Use decltype to capture lambda type
    auto cmp = [](const std::string& a, const std::string& b) {
        return a.size() < b.size();  // sort by string length
    };

    std::map<std::string, int, decltype(cmp)> length_map(cmp);
    length_map["hello"] = 1;
    length_map["hi"] = 2;
    length_map["world"] = 3;
    length_map["a"] = 4;

    for (const auto& [k, v] : length_map) {
        std::cout << "\"" << k << "\" (len=" << k.size() << "): " << v << "\n";
    }
    // Output (sorted by length):
    // "a" (len=1): 4
    // "hi" (len=2): 2
    // "hello" (len=5): 1
    // "world" (len=5): 3
    // Note: "hello" before "world" because "hello" was inserted first
    //       (equal keys per comparator, insertion order preserved in some impls)

    return 0;
}
```

> 当两个键在比较器看来"相等"（`!cmp(a,b) && !cmp(b,a)`），map 将它们视为同一个键。长度相同的字符串在这个比较器下是"等价的"，所以后插入的可能覆盖先插入的，或者被忽略，取决于实现。实际项目中要确保比较器能区分所有你想区分的键。

---

## 有序遍历：对比 unordered 的核心优势

`std::map` 和 `std::set` 的迭代器按照键的升序（或自定义的序）遍历元素。这是 `std::unordered_map` 做不到的。

```cpp
#include <iostream>
#include <map>
#include <unordered_map>

int main() {
    std::map<int, std::string> ordered = {
        {5, "five"}, {1, "one"}, {3, "three"}, {4, "four"}, {2, "two"}
    };

    std::unordered_map<int, std::string> unordered = {
        {5, "five"}, {1, "one"}, {3, "three"}, {4, "four"}, {2, "two"}
    };

    std::cout << "Ordered map traversal:\n";
    for (const auto& [k, v] : ordered) {
        std::cout << "  " << k << ": " << v << "\n";
    }
    // Always: 1, 2, 3, 4, 5

    std::cout << "Unordered map traversal:\n";
    for (const auto& [k, v] : unordered) {
        std::cout << "  " << k << ": " << v << "\n";
    }
    // Order depends on hash values, no guarantees

    return 0;
}
```

适用有序遍历的场景：

- 按字母序输出字典。
- 按时间戳遍历事件日志（用时间戳作为键）。
- 实现一致性哈希的环结构。
- 任何需要"前驱"和"后继"查询的场景。

---

## 范围查询实战

范围查询是红黑树的一大杀手级特性。`lower_bound` 和 `upper_bound` 利用树的有序性，在 O(log n) 时间内定位区间边界。

### 示例：按分数段筛选学生

```cpp
#include <iostream>
#include <map>
#include <string>

int main() {
    // score -> student name (assume unique scores for simplicity)
    std::map<int, std::string> leaderboard = {
        {100, "Zhang"}, {95, "Li"}, {90, "Wang"}, {85, "Zhao"},
        {80, "Sun"},    {75, "Zhou"}, {70, "Wu"}, {65, "Zheng"},
        {60, "Feng"},   {55, "Chen"}
    };

    // Find students with scores in [70, 90]
    auto lo = leaderboard.lower_bound(70);   // first >= 70
    auto hi = leaderboard.upper_bound(90);   // first > 90

    std::cout << "Students scoring 70-90:\n";
    for (auto it = lo; it != hi; ++it) {
        std::cout << "  " << it->second << ": " << it->first << "\n";
    }
    // Output:
    //   Wu: 70
    //   Zheng: 65  <-- Wait, this is wrong. Let me fix.
    // Actually lower_bound(70) returns iterator to {70, "Wu"},
    // and we iterate up to (not including) upper_bound(90) = {95, "Li"}.

    // Correct output:
    //   Wu: 70
    //   Zhou: 75
    //   Sun: 80
    //   Zhao: 85
    //   Wang: 90

    return 0;
}
```

### 用 equal_range 简化范围查询

`equal_range` 是 `lower_bound + upper_bound` 的组合，返回一个迭代器对：

```cpp
#include <iostream>
#include <set>

int main() {
    std::multiset<int> data = {1, 2, 2, 3, 3, 3, 4, 4, 5};

    // Find all 3s
    auto [lo, hi] = data.equal_range(3);
    std::cout << "Range of 3s: ";
    for (auto it = lo; it != hi; ++it) {
        std::cout << *it << " ";
    }
    std::cout << "\n";
    // Output: Range of 3s: 3 3 3

    std::cout << "Count via equal_range: "
              << std::distance(lo, hi) << "\n";  // 3

    return 0;
}
```

---

## 性能对比表

### 时间复杂度

| 操作             | map / set | multimap / multiset | 说明                        |
| ---------------- | --------- | ------------------- | --------------------------- |
| 查找 find        | O(log n)  | O(log n)            | 红黑树搜索                  |
| 插入 insert      | O(log n)  | O(log n)            | 搜索 + 旋转平衡             |
| 删除 erase       | O(log n)  | O(log n)            | 搜索 + 旋转平衡             |
| operator[]       | O(log n)  | 不提供               | 搜索 + 可能插入              |
| count            | O(log n)  | O(log n + k)        | k 是匹配数量                 |
| lower_bound      | O(log n)  | O(log n)            | 有序结构的独有优势           |
| upper_bound      | O(log n)  | O(log n)            | 同上                        |
| equal_range      | O(log n)  | O(log n)            | 返回 [lower, upper) 区间    |
| 有序遍历         | O(n)      | O(n)                | 中序遍历，已排序             |

### 和 unordered 容器的对比

| 指标           | 有序 (map/set)        | 无序 (unordered)       |
| -------------- | --------------------- | ---------------------- |
| 平均查找       | O(log n)              | O(1)                   |
| 最坏查找       | O(log n)              | O(n)（哈希冲突）       |
| 有序遍历       | 支持                  | 不支持                 |
| 范围查询       | 原生支持              | 不支持                 |
| 内存开销       | 每节点 3 指针 + 1 色  | 每桶一个链表节点        |
| 键类型要求     | 定义严格弱序           | 提供 hash + equals      |
| 迭代器稳定性   | 插入/删除不影响其他   | rehash 失效全部        |

---

## 迭代器失效规则

红黑树的插入和删除只影响局部节点（通过旋转和着色维护平衡），不会大规模移动元素。因此迭代器失效规则非常友好：

> **插入操作**：不会使任何现有迭代器失效。
>
> **删除操作**：只会使指向被删除元素的迭代器失效，其他迭代器不受影响。

```cpp
#include <iostream>
#include <map>
#include <set>

int main() {
    std::map<int, std::string> m = {
        {1, "a"}, {2, "b"}, {3, "c"}, {4, "d"}, {5, "e"}
    };

    auto it = m.find(3);
    std::cout << "Before erase: " << it->second << "\n";  // "c"

    m.insert({6, "f"});   // does NOT invalidate it
    std::cout << "After insert: " << it->second << "\n";  // "c"

    m.erase(1);           // erasing OTHER elements does NOT invalidate it
    std::cout << "After erase(1): " << it->second << "\n";  // "c"

    // BUT: erasing the element the iterator points to DOES invalidate it
    auto it2 = m.find(2);
    m.erase(it2);
    // it2 is now INVALID. Do NOT use it.

    // Safe pattern: use the return value of erase()
    auto it3 = m.find(4);
    it3 = m.erase(it3);   // erase returns iterator to next element
    // it3 now points to {5, "e"}

    return 0;
}
```

这一点和 `std::unordered_map` 形成鲜明对比。`unordered_map` 发生 rehash 时，所有迭代器全部失效。所以在需要迭代器稳定性的场景下，有序容器更安全。

---

## 内存开销分析

红黑树每个节点除了存储用户数据，还需要额外的指针和标记：

| 字段           | 大小（64 位系统） |
| -------------- | ----------------- |
| left 指针      | 8 bytes           |
| right 指针     | 8 bytes           |
| parent 指针    | 8 bytes           |
| 颜色标记       | 通常 1 byte（实现中可能因对齐占 4-8 bytes） |
| **总开销**     | 约 24-32 bytes per node |

所以一个 `std::map<int, int>` 的每个节点实际占用：

- 键 `int`：4 bytes
- 值 `int`：4 bytes
- 指针和颜色：约 32 bytes
- **合计：约 40 bytes**

对比 `std::unordered_map<int, int>`：每个节点约 24-32 bytes（哈希表节点的指针开销），加上桶数组的开销。

对于大量小对象，红黑树的内存开销可能成为瓶颈。如果你的键本身就是整数且范围连续，用 `std::vector` 排序后做二分查找可能更省内存。

---

## 常见陷阱

### 陷阱一：operator[] 隐式插入

这是 `std::map` 的新手头号杀手。

```cpp
std::map<std::string, int> word_count;

// BAD: just want to check if key exists
if (word_count["hello"] == 0) {  // inserts {"hello", 0}!
    std::cout << "Not found\n";   // wrong logic
}

// GOOD
if (word_count.find("hello") == word_count.end()) {
    std::cout << "Not found\n";
}

// GOOD (C++20)
if (!word_count.contains("hello")) {
    std::cout << "Not found\n";
}
```

### 陷阱二：尝试修改键

`std::map` 的键类型是 `const Key`，直接修改编译不过。但如果你通过 `const_cast` 或某些 hack 绕过了 const 保护，树的有序性会被破坏，后续操作全部变成未定义行为。

```cpp
std::map<int, std::string> m = {{1, "a"}, {2, "b"}, {3, "c"}};

// This WON'T compile:
// m.begin()->first = 100;  // error: assignment of read-only member

// If you need to "change" a key, you must erase + re-insert:
auto node = m.extract(2);       // C++17: extract the node
node.key() = 5;                 // modify key safely
m.insert(std::move(node));      // re-insert

for (const auto& [k, v] : m) {
    std::cout << k << ": " << v << "\n";
}
// Now: {1, "a"}, {3, "c"}, {5, "b"}
```

`extract`（C++17）是修改键的正确姿势。它在节点级别操作，不触发任何内存分配或释放。

### 陷阱三：multimap 的 erase 删除所有同键元素

```cpp
std::multimap<int, std::string> mm = {
    {1, "a"}, {1, "b"}, {1, "c"}, {2, "d"}
};

// This erases ALL entries with key 1:
mm.erase(1);  // removes {1,"a"}, {1,"b"}, {1,"c"}

// If you only want to erase ONE:
auto it = mm.find(1);
if (it != mm.end()) {
    mm.erase(it);  // removes only {1, "a"}
}
```

### 陷阱四：在遍历中删除元素

```cpp
std::map<int, std::string> m = {
    {1, "a"}, {2, "b"}, {3, "c"}, {4, "d"}
};

// BAD: undefined behavior after erase
for (auto it = m.begin(); it != m.end(); ++it) {
    if (it->second == "b") {
        m.erase(it);  // it is invalidated!
    }
}

// GOOD: C++11 erase returns next iterator
for (auto it = m.begin(); it != m.end(); ) {
    if (it->second == "b" || it->second == "c") {
        it = m.erase(it);
    } else {
        ++it;
    }
}

// GOOD: C++20 erase_if
// std::erase_if(m, [](const auto& pair) {
//     return pair.second == "b" || pair.second == "c";
// });
```

### 陷阱五：比较器必须一致

一旦 map/set 构造完成，比较器对象被存储在容器内部。如果你传了一个 lambda，确保 lambda 的捕获列表不引用已经销毁的变量。

```cpp
#include <iostream>
#include <map>
#include <functional>

std::map<int, int, std::function<bool(int, int)>> GetThresholdMap(int threshold) {
    // BAD: captures threshold by reference, but threshold is local!
    // auto cmp = [&threshold](int a, int b) {
    //     return a < b && a > threshold;
    // };

    // GOOD: captures by value
    auto cmp = [threshold](int a, int b) {
        // Note: this is a simplified example.
        // A real comparator must define a total ordering.
        return a < b;
    };

    std::map<int, int, std::function<bool(int, int)>> m(cmp);
    m[10] = 100;
    return m;  // comparator is safely copied
}
```

---

## 有序 vs 无序：如何选择

| 场景                           | 推荐                   |
| ------------------------------ | ---------------------- |
| 需要按键顺序遍历               | map / set              |
| 需要范围查询                   | map / set              |
| 需要前驱/后继操作              | map / set              |
| 只做精确查找，不在乎顺序       | unordered_map / set    |
| 键类型没有自然的哈希函数       | map / set              |
| 键类型没有自然的排序关系       | unordered_map / set    |
| 对最坏情况延迟敏感             | map / set（O(log n) 保底） |
| 对平均性能极致追求             | unordered_map / set    |
| 需要迭代器稳定性               | map / set              |

简单总结：

- **需要顺序**，用有序容器，没得选。
- **不需要顺序**，数据量大，追求平均速度，用 `unordered`。
- **不需要顺序**，但害怕哈希冲突导致最坏情况，用有序容器。

---

## 总结

`std::map`、`std::set`、`std::multimap`、`std::multiset` 背后的红黑树提供了 **O(log n)** 的查找、插入、删除性能，同时保证元素按键有序。这赋予了它们两个独特能力：

1. **有序遍历**：迭代器按序输出，适合字典、排行榜、时间线。
2. **范围查询**：`lower_bound`、`upper_bound`、`equal_range` 在对数时间内定位区间。

代价是每个节点额外 24 到 32 字节的指针开销，以及常数因子比哈希表稍慢。

工程实践中的几个要点：

- `map` 的 `operator[]` 会自动插入，只读场景用 `at()` 或 `find()`。
- 键是 `const` 的，不要试图修改。需要改键时用 `extract`（C++17）。
- `multimap::erase(key)` 删除所有同键元素，删单个要用迭代器版本。
- 遍历时删除元素，记得用 `it = erase(it)` 模式。
- 比较器必须满足严格弱序，否则未定义行为。

理解红黑树的平衡原理，能帮你建立对这些容器性能特征的直觉。至于旋转、着色的具体细节，属于实现层面，日常编码不需要操心。标准库已经把脏活干完了。

---

> **参考资料**
>
> -《C++ Concurrency in Action》Anthony Williams
> - GCC libstdc++ source: `bits/stl_tree.h`
> - cppreference: [std::map](https://en.cppreference.com/w/cpp/container/map), [std::set](https://en.cppreference.com/w/cpp/container/set)
> -《算法导论》（CLRS）第 13 章：红黑树
