---
title: "C++ std::deque：双端队列的内存布局与工程取舍"
date: 2021-08-11T10:00:00+08:00
tags: ["C++", "STL", "deque", "双端队列"]
categories: ["技术"]
summary: "深入剖析 std::deque 的分段连续内存结构、中控数组的工作原理、与 vector 的性能差异。讲清楚 deque 为什么是 stack 和 queue 的默认底层容器，以及何时应该选择 deque 而非 vector。"
ShowToc: true
---

## 引言：deque，vector 和 list 之间的折中

C++ 标准库里的容器，大家最熟悉的是 `std::vector`。连续内存，随机访问 O(1)，尾部插入 O(1)，简单粗暴，效率极高。

但 `std::vector` 有两个明显的短板：头部插入是 O(n)，而且扩容时必须把所有元素搬到一个全新的内存块。

另一端是 `std::list`，双向链表，在任何位置插入删除都是 O(1)。代价是内存不连续，cache 友好性极差，每个元素还要额外消耗两个指针的空间。

`std::deque`（double-ended queue）就是这两者之间的折中方案。它既能像 `vector` 一样随机访问，又能在两端都做到 O(1) 插入。听起来像是两全其美，但天下没有免费的午餐。要理解 deque 的取舍，得先搞清楚它的内存是怎么组织的。

---

## 内存布局：分段连续 + 中控数组

这是理解 deque 的一切的关键。先看一张图。

假设每个 chunk（也叫 buffer）能存 4 个 `int`，中控数组（map）有 5 个槽位：

```
map (control array):
+---+---+---+---+---+
|   |   | P | P |   |   P = pointer to chunk
+---+---+---+---+---+
  ^       ^   ^
  |       |   |
  |       |   +-------------------------------+
  |       |                                   |
  |       +--------+                          |
  |                |                          |
  |     chunk[0]:  v            chunk[1]:     v
  |   +----+----+----+----+  +----+----+----+----+
  |   | 10 | 20 | 30 | 40 |  | 50 | 60 |    |    |
  |   +----+----+----+----+  +----+----+----+----+
  |
  |   (unused slots, available for future push_front)
  |
start iterator points here -->  chunk[0][0] = 10
finish iterator points here --> chunk[1][2] = (one past last element)
```

**几个要点：**

- deque 的内存不是一整块连续空间，而是由多个固定大小的 chunk 组成
- 每个 chunk 是一段连续内存，大小通常是 512 字节或 `sizeof(T) * N`
- 中控数组（map）是一个指针数组，每个元素指向一个 chunk
- 中控数组本身也是可以扩容的：当槽位不够时，分配一个更大的 map，把旧指针搬过去
- `start` 迭代器指向第一个元素，`finish` 迭代器指向最后一个元素的下一个位置

> 这种设计叫做「分段连续空间」（segmented contiguous storage）。对使用者来说，deque 看起来像一段连续空间，但底层是拼起来的。

### chunk 大小怎么定？

不同 STL 实现有不同的策略。以 SGI STL 为例：

```cpp
// Simplified chunk size calculation
inline size_t DeckBufferSize(size_t element_size) {
    // If element size > 512 bytes, chunk contains 1 element
    // Otherwise, chunk contains floor(512 / element_size) elements
    return element_size < 512 ? static_cast<size_t>(512 / element_size) : 1;
}
```

对于 `int`（4 字节），每个 chunk 存 128 个元素。对于大结构体，chunk 可能只存几个甚至一个。这个设计保证了 chunk 大小不会太浪费，也不会太小导致 map 频繁扩容。

---

## 双端 O(1) 插入是怎么做到的

`std::vector` 的 `push_back` 是 O(1) amortized，因为扩容时要搬运所有元素。`deque` 的扩容成本更低。

### push_back 的流程

```
Before push_back(70):

map:
+---+---+---+---+---+
|   |   | P | P |   |
+---+---+---+---+---+
            |   |
       chunk[0] chunk[1]
       [10,20,30,40] [50,60,_,_]
                        ^
                        finish is here (chunk[1][2])

After push_back(70):

map:
+---+---+---+---+---+
|   |   | P | P |   |
+---+---+---+---+---+
            |   |
       chunk[0] chunk[1]
       [10,20,30,40] [50,60,70,_]
                          ^
                          finish moved to chunk[1][3]
```

当前 chunk 还有空位，直接放进去就行。如果当前 chunk 满了：

1. 分配一个新的 chunk
2. 在 map 中记录这个 chunk 的指针
3. 如果 map 也满了，先扩容 map（分配更大的 map，搬运指针）
4. 把新元素放到新 chunk 的第一个位置

注意，扩容 map 只需要搬运**指针**，不需要搬运元素本身。这就比 vector 的扩容便宜得多。

### push_front 的流程

```
Before push_front(5):

map:
+---+---+---+---+---+
|   |   | P | P |   |
+---+---+---+---+---+
            |   |
       chunk[0] chunk[1]
       [10,20,30,40] [50,60,_,_]
        ^
        start is here (chunk[0][0])

After push_front(5):

Option A: chunk[0] has room at the front
       chunk[0]
       [_,10,20,30,40]  --> no room for int chunk of size 4

Actually for chunk size 4, it's full. So:

Option B: allocate new chunk
map:
+---+---+---+---+---+
|   | P | P | P |   |
+---+---+---+---+---+
        |   |   |
   new_chunk chunk[0] chunk[1]
   [_,_,_,5] [10,20,30,40] [50,60,_,_]
          ^
          start now points here (new_chunk[3])
```

`push_front` 把元素放到第一个 chunk 的前部空位。如果没有空位，就分配新 chunk，放在当前第一个 chunk 之前。`start` 迭代器反向移动。

> 这就是 deque 同时支持高效 `push_front` 和 `push_back` 的核心机制：两端各有一个「生长方向」。尾部向后生长，头部向前生长。只要 chunk 有空位，插入就是真正的 O(1)，连 amortized 都不是。

---

## 随机访问：operator[] 的代价

`deque` 支持 `operator[]`，但它的随机访问比 `vector` 多一步间接寻址。

```cpp
// Simplified deque operator[] logic
reference operator[](size_type n) {
    // map_index: which chunk does element n live in?
    size_t map_index = (start.node_offset + n) / chunk_size;
    // element_offset: which position within that chunk?
    size_t element_offset = (start.node_offset + n) % chunk_size;
    return map[map_index][element_offset];
}
```

翻译成大白话：

1. 算出第 n 个元素在哪个 chunk
2. 算出在 chunk 内的偏移量
3. 两次解引用：先从 map 拿到 chunk 指针，再从 chunk 拿到元素

```
deque[5] lookup:

map:
+---+---+---+---+---+
|   |   | P | P |   |
+---+---+---+---+---+
            |   |
       chunk[0] chunk[1]
       [10,20,30,40] [50,60,70,80]
        0  1  2  3    4  5  6  7

start offset = 0, chunk_size = 4
element 5: map_index = (0+5)/4 = 1, offset = (0+5)%4 = 1
=> map[1][1] = chunk[1][1] = 60  ✓
```

和 `vector` 的 `operator[]` 相比，多了一次除法和一次取模运算，还有一次额外的内存间接访问。对 CPU 来说，多一次内存访问意味着可能多一次 cache miss。在大数据量下，这个差异是可测量的。

> 所以 deque 的随机访问是 O(1)，但常数因子比 vector 大。不是说复杂度不好，而是实际跑起来确实慢一些。

---

## 核心操作详解

### push_back / push_front

```cpp
std::deque<int> dq;

// push_back: append to the tail
dq.push_back(1);  // dq: {1}
dq.push_back(2);  // dq: {1, 2}

// push_front: prepend to the head
dq.push_front(0);   // dq: {0, 1, 2}
dq.push_front(-1);  // dq: {-1, 0, 1, 2}
```

两者都是 O(1)。当 chunk 满了，分配新 chunk 并挂到 map 上。当 map 满了，扩容 map（只搬运指针，不搬运元素）。

### emplace_back / emplace_front

```cpp
struct Point {
    int x, y;
    Point(int x, int y) : x(x), y(y) {}
};

std::deque<Point> dq;

// emplace_back: construct in-place, avoid temporary object
dq.emplace_back(3, 4);
dq.emplace_front(1, 2);
```

`emplace` 系列直接在目标位置构造对象，省去了一次临时对象的构造和析构。对于非 trivial 类型，这能带来可观的性能提升。

### pop_front / pop_back

```cpp
std::deque<int> dq = {1, 2, 3, 4, 5};

dq.pop_front();  // dq: {2, 3, 4, 5}, removed 1
dq.pop_back();   // dq: {2, 3, 4}, removed 5
```

两者都是 O(1)。如果 pop 之后某个 chunk 变空了，该 chunk 会被释放。但 map 本身不会缩容，这点和 vector 类似。

> 注意：`pop_front` 和 `pop_back` 不会返回被移除的值。如果你需要这个值，先 `front()` / `back()` 取出来，再 pop。这个设计是为了保持异常安全。

### insert

```cpp
std::deque<int> dq = {1, 2, 5, 6};

// Insert before position 2
auto it = dq.insert(dq.begin() + 2, 3);  // dq: {1, 2, 3, 5, 6}

// Insert multiple copies
dq.insert(dq.begin(), 2, 0);  // dq: {0, 0, 1, 2, 3, 5, 6}

// Insert range
std::vector<int> extra = {10, 11};
dq.insert(dq.end(), extra.begin(), extra.end());
```

在中间插入是 O(n)，和 vector 一样。但 deque 有一个优化：它会比较插入位置到头部和尾部的距离，选择移动较少元素的那一侧。

```cpp
// Simplified insert logic
if (insert_position - begin() < size() / 2) {
    // Closer to front: shift elements toward the front
    // May trigger push_front
} else {
    // Closer to back: shift elements toward the back
    // May trigger push_back
}
```

这意味着最坏情况移动 n/2 个元素，而不是 n 个。但复杂度还是 O(n)。

### erase

```cpp
std::deque<int> dq = {1, 2, 3, 4, 5, 6, 7};

// Erase single element
dq.erase(dq.begin() + 3);  // dq: {1, 2, 3, 5, 6, 7}

// Erase range
dq.erase(dq.begin() + 1, dq.begin() + 4);  // dq: {1, 6, 7}
```

和 insert 类似，erase 也会选择移动较少元素的那一侧。O(n) 复杂度，但实际移动量可能比 vector 少。

### clear

```cpp
std::deque<int> dq = {1, 2, 3, 4, 5};
dq.clear();  // dq is now empty, size() == 0
```

`clear` 会销毁所有元素并释放所有 chunk，但 map 本身的内存不会被释放。这和 `vector::clear` 不释放底层数组的行为一致。

---

## 迭代器实现：deque 最复杂的部分

`deque` 的迭代器不是普通的指针，它是一个包含了多重信息的结构体：

```cpp
// Simplified deque iterator (SGI-style)
template <typename T, size_t BufSize>
struct DequeIterator {
    T* cur;        // Pointer to current element within the chunk
    T* first;      // Pointer to first element of current chunk
    T* last;       // Pointer to one past last element of current chunk
    T** node;      // Pointer to the map entry for current chunk

    // Increment: move to next element
    DequeIterator& operator++() {
        ++cur;
        if (cur == last) {
            // Reached end of current chunk, jump to next chunk
            SetNode(node + 1);
            cur = first;
        }
        return *this;
    }

    // Decrement: move to previous element
    DequeIterator& operator--() {
        if (cur == first) {
            // At beginning of chunk, jump to previous chunk
            SetNode(node - 1);
            cur = last;
        }
        --cur;
        return *this;
    }

    void SetNode(T** new_node) {
        node = new_node;
        first = *new_node;
        last = first + chunk_size;
    }
};
```

**迭代器里为什么要存四个指针？**

因为迭代器前进或后退时，可能跨 chunk 边界。如果只存一个 `cur` 指针，你没办法知道什么时候该跳到下一个 chunk。`first` 和 `last` 标记了当前 chunk 的边界，`node` 让你能跳到 map 的相邻条目。

这也解释了为什么 deque 的迭代器比 vector 的（本质上就是原生指针）大得多。一个 `deque<int>::iterator` 通常占 16 字节（4 个指针），而 `vector<int>::iterator` 只占 4 字节。

### 迭代器的算术运算

```cpp
// Iterator difference: how many elements between two iterators?
difference_type operator-(const DequeIterator& other) const {
    return difference_type(chunk_size) * (node - other.node - 1)
           + (cur - first) + (other.last - other.cur);
}
```

这个公式看着复杂，其实逻辑很清楚：

1. 中间完整的 chunk 数量 × chunk_size
2. 当前 chunk 里 cur 之前的元素数量
3. other 所在 chunk 里 cur 之后的元素数量

三者加起来就是总距离。和 vector 迭代器相减（一次指针减法）相比，多了不少计算。

---

## 迭代器失效规则：比 vector 更复杂

这是 deque 最容易踩坑的地方。先复习 vector 的规则：

- `vector::push_back`：如果发生扩容，所有迭代器失效。否则只有 `end()` 失效。
- `vector::insert`：插入位置之后的迭代器失效。扩容则全部失效。
- `vector::erase`：被删位置及之后的迭代器失效。

deque 的规则更细碎：

> **deque 在中间插入/删除时，迭代器失效的情况比 vector 更严重。**

| 操作 | 迭代器失效情况 |
|------|--------------|
| `push_front` | 所有迭代器失效，引用和指针不受影响 |
| `push_back` | 所有迭代器失效，引用和指针不受影响 |
| `push_front` + `push_back`（未触发 map 扩容） | 仅 `end()` 迭代器失效 |
| `insert`（中间） | 所有迭代器、引用、指针全部失效 |
| `erase`（中间） | 所有迭代器、引用、指针全部失效 |
| `erase`（头部或尾部） | 仅被删元素及之后的迭代器失效 |
| `clear` | 全部失效 |

关键区别：

1. **引用（reference）和指针的行为可能与迭代器不同**。`push_front` / `push_back` 不会使已有引用失效，因为元素没有被移动，只是 map 中的指针变了。但迭代器里存了 `node` 指针，指向 map 的某个槽位，map 扩容后这个指针就悬空了。

2. **中间 insert/erase 是最危险的操作**。因为 deque 可能选择向任一方向移动元素，所有的引用和指针都会失效。

```cpp
std::deque<int> dq = {1, 2, 3, 4, 5};
int& ref = dq[2];  // reference to element 3

dq.push_back(6);   // ref is still valid! element didn't move
dq.push_front(0);  // ref is still valid! element didn't move

dq.insert(dq.begin() + 2, 99);  // ref is now INVALID!
// The element at position 2 may have shifted
```

> 如果你需要稳定的引用，用 `std::list` 或 `std::forward_list`。deque 的引用稳定性只在两端操作时才有保证。

---

## 性能对比：deque vs vector

纸上谈兵结束，看看实际数字。以下测试在 GCC 12.1，O2 优化，x86_64 平台上运行。

| 操作 | `std::vector` | `std::deque` | 说明 |
|------|--------------|-------------|------|
| `push_back` | O(1) amortized | O(1) amortized | deque 扩容成本更低（只搬指针） |
| `push_front` | O(n) | O(1) | deque 的核心优势 |
| `operator[]` | ~1ns | ~3-5ns | deque 多一次间接寻址 |
| 遍历（顺序） | 极快，cache 友好 | 较快，chunk 内连续 | deque 在 chunk 边界处可能 cache miss |
| 遍历（随机步长） | 极快 | 较慢 | 每次跨 chunk 都有额外开销 |
| `insert`（中间） | O(n) | O(n) | deque 可能只移 n/2 个元素 |
| 内存开销 | 极低（仅数据） | 中等（chunk 碎片 + map） | 每个 chunk 可能有未用空间 |
| 迭代器大小 | 1 个指针 | 3-4 个指针 | deque 迭代器更重 |
| 扩容时元素搬移 | 全部搬移 | 只搬 map 中的指针 | deque 优势明显 |
| 内存局部性 | 最佳 | chunk 内良好，chunk 间较差 | 连续遍历时差距不大 |

### 一个简单的 benchmark

```cpp
#include <deque>
#include <vector>
#include <chrono>
#include <iostream>

void BenchmarkPushFront() {
    const int N = 1000000;

    // vector push_front (insert at begin)
    auto t1 = std::chrono::high_resolution_clock::now();
    std::vector<int> vec;
    for (int i = 0; i < N; ++i) {
        vec.insert(vec.begin(), i);
    }
    auto t2 = std::chrono::high_resolution_clock::now();
    std::cout << "vector push_front: "
              << std::chrono::duration_cast<std::chrono::milliseconds>(t2 - t1).count()
              << " ms\n";

    // deque push_front
    t1 = std::chrono::high_resolution_clock::now();
    std::deque<int> dq;
    for (int i = 0; i < N; ++i) {
        dq.push_front(i);
    }
    t2 = std::chrono::high_resolution_clock::now();
    std::cout << "deque push_front: "
              << std::chrono::duration_cast<std::chrono::milliseconds>(t2 - t1).count()
              << " ms\n";
}

void BenchmarkRandomAccess() {
    const int N = 10000000;

    std::vector<int> vec(N);
    std::deque<int> dq(N);
    for (int i = 0; i < N; ++i) {
        vec[i] = i;
        dq[i] = i;
    }

    volatile long long sum = 0;

    // Vector random access
    auto t1 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < N; ++i) {
        sum += vec[i];
    }
    auto t2 = std::chrono::high_resolution_clock::now();
    std::cout << "vector random access: "
              << std::chrono::duration_cast<std::chrono::microseconds>(t2 - t1).count()
              << " us\n";

    // Deque random access
    t1 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < N; ++i) {
        sum += dq[i];
    }
    t2 = std::chrono::high_resolution_clock::now();
    std::cout << "deque random access: "
              << std::chrono::duration_cast<std::chrono::microseconds>(t2 - t1).count()
              << " us\n";
}
```

典型结果（具体数值因平台而异）：

```
vector push_front: ~2500 ms   // O(n^2), brutal
deque  push_front: ~30 ms     // O(n), no contest

vector random access: ~8000 us
deque  random access: ~15000 us   // roughly 2x slower
```

`push_front` 没有悬念，deque 完胜。随机访问 vector 快一倍左右，但也不是不可接受的差距。选择取决于你的使用模式。

---

## 为什么 deque 是 stack 和 queue 的默认底层容器

`std::stack` 和 `std::queue` 是容器适配器（container adapter），它们不是独立的容器，而是对底层容器的封装。默认都使用 `deque`：

```cpp
// From the C++ standard
template <typename T, typename Container = std::deque<T>>
class stack;

template <typename T, typename Container = std::deque<T>>
class queue;
```

为什么默认选 deque 而不是 vector？

### 对于 stack

`stack` 只需要 `push_back`、`pop_back`、`back`、`size`、`empty` 这些操作。vector 完全能胜任，那为什么不用 vector？

1. **deque 的扩容不需要搬移元素**。vector 扩容时要复制所有元素，对于复杂类型这个开销很大。deque 只需要分配新 chunk 并更新 map。
2. **deque 的内存分配更均匀**。vector 要一次性分配一大块连续内存，如果系统碎片化严重，可能分配失败。deque 分配的是小 chunk，更容易成功。
3. **deque 没有保留未用容量的浪费**。vector 的 `capacity` 通常大于 `size`，浪费的内存可能达到当前 size 的 50%~100%。deque 的每个 chunk 是按需分配的，浪费最多是一个 chunk 的大小。

当然，如果你知道元素数量的大致范围，用 `vector::reserve` 预分配，vector 在 stack 场景下其实更快。这就是为什么你可以指定底层容器：

```cpp
// Use vector as underlying container for stack
std::stack<int, std::vector<int>> stk;
stk.push(1);
stk.push(2);
```

### 对于 queue

`queue` 需要 `push_back` 和 `pop_front`。这就把 vector 排除了，因为 vector 的 `pop_front`（或者说 `erase(begin())`）是 O(n)。

可选的底层容器：

| 容器 | `push_back` | `pop_front` | 随机访问 | 内存局部性 |
|------|-----------|-----------|---------|----------|
| `deque` | O(1) | O(1) | O(1)（可选） | 良好 |
| `list` | O(1) | O(1) | 不支持 | 差 |

deque 在 `queue` 场景下几乎全面优于 `list`：内存局部性更好，每个元素的额外开销更小，还支持随机访问（虽然 queue 不暴露这个接口）。

```cpp
// Use list as underlying container for queue
std::queue<int, std::list<int>> q;
q.push(1);
q.pop();  // O(1) with list too
```

> 总结：deque 成为默认，是因为它在绝大多数场景下都是「不差」的选择。它不是每个维度的最优解，但综合得分最高。

---

## 何时选择 deque 而非 vector

这不是一个简单的「谁更好」的问题，而是「你的使用模式更适合哪个」的问题。

### 以下场景选 deque

1. **需要频繁在头部插入或删除元素**
   ```cpp
   // Sliding window: remove front, add back
   std::deque<int> window;
   for (int val : data_stream) {
       window.push_back(val);
       if (window.size() > k) {
           window.pop_front();
       }
       ProcessWindow(window);
   }
   ```

2. **元素体积大，扩容搬移成本高**
   ```cpp
   // Large objects: vector reallocation would copy all of them
   std::deque<BigStruct> container;
   // deque reallocation only copies pointers in the map
   ```

3. **内存碎片化环境下运行**
   ```cpp
   // deque allocates small chunks, easier to satisfy
   // vector needs one large contiguous block
   std::deque<int> chunked_data;
   ```

4. **用作 stack 或 queue 的底层容器**（默认就是，不需要额外操作）

5. **不确定元素数量，且不希望浪费大量预留空间**

### 以下场景选 vector

1. **纯尾部追加 + 顺序访问，这是 vector 的主场**
   ```cpp
   std::vector<int> data;
   data.reserve(10000);  // Know the approximate size
   for (int i = 0; i < 10000; ++i) {
       data.push_back(ComputeValue(i));
   }
   ```

2. **大量随机访问（operator[]、at()）**

3. **需要和 C API 交互**（`vector::data()` 返回连续内存指针）
   ```cpp
   std::vector<int> buf(1024);
   c_api_function(buf.data(), buf.size());  // deque can't do this
   ```

4. **需要最好的 cache 性能**（遍历密集型算法）

5. **迭代器稳定性很重要**（vector 在不扩容时，push_back 不会使已有迭代器失效...但注意这是个坑，因为很难保证不扩容）

### 决策流程图

```
需要 push_front?
  |
  +-- YES --> deque
  |
  +-- NO --> 需要 C API 交互或连续内存?
               |
               +-- YES --> vector
               |
               +-- NO --> 元素很大且数量不确定?
                            |
                            +-- YES --> deque
                            |
                            +-- NO --> vector (safe default)
```

> 一个实用的经验法则：当你不确定用哪个时，先用 vector。只有当你发现 vector 的某个操作变成了性能瓶颈，再考虑换 deque。

---

## 常见陷阱

### 陷阱 1：以为 deque 的迭代器和 vector 一样轻量

```cpp
// This is fine with vector (iterator is just a pointer)
for (auto it = vec.begin(); it != vec.end(); ++it) {
    // Fast pointer comparison
}

// This works with deque too, but each iteration may cross chunk boundary
// ++it involves checking cur == last and possibly jumping to next chunk
for (auto it = dq.begin(); it != dq.end(); ++it) {
    // More work per iteration
}
```

对于遍历，range-based for 循环在两种容器上都能正常工作，但 deque 的每次迭代器递进都多了边界检查。

### 陷阱 2：在循环中 erase 时忘了更新迭代器

```cpp
std::deque<int> dq = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

// WRONG: iterator invalidated after erase
for (auto it = dq.begin(); it != dq.end(); ++it) {
    if (*it % 2 == 0) {
        dq.erase(it);  // 'it' is now invalid! UB!
    }
}

// CORRECT: use erase return value
for (auto it = dq.begin(); it != dq.end(); ) {
    if (*it % 2 == 0) {
        it = dq.erase(it);  // erase returns next valid iterator
    } else {
        ++it;
    }
}

// OR: use erase-remove idiom (more efficient)
#include <algorithm>
dq.erase(
    std::remove_if(dq.begin(), dq.end(),
                   [](int x) { return x % 2 == 0; }),
    dq.end()
);
```

这个问题 vector 也有，但 deque 在中间 erase 时所有迭代器都失效，更容易出错。

### 陷阱 3：以为 deque 的元素是连续的

```cpp
std::deque<int> dq = {1, 2, 3, 4, 5, 6, 7, 8};

// WRONG: deque memory is NOT contiguous
int* ptr = &dq[0];
ExpectContiguous(ptr, dq.size());  // Undefined behavior assumption

// vector IS contiguous
std::vector<int> vec = {1, 2, 3, 4, 5, 6, 7, 8};
int* vptr = vec.data();  // Guaranteed contiguous
ExpectContiguous(vptr, vec.size());  // OK
```

如果你需要把容器内容传给 C API 或者做 SIMD 操作，deque 不适合。

### 陷阱 4：deque 的 shrink_to_fit 不是标准的

```cpp
std::deque<int> dq;
for (int i = 0; i < 10000; ++i) dq.push_back(i);
dq.clear();

// deque does NOT have shrink_to_fit in C++ standard
// dq.shrink_to_fit();  // WRONG: no such member

// To actually free memory, swap with empty deque
std::deque<int>().swap(dq);  // Now dq is truly empty, memory freed
```

等等，C++11 之后 deque 其实是有 `shrink_to_fit` 的。但它不是强制要求，只是一个非绑定请求。实现可以选择忽略它。swap trick 是更可靠的做法。

> 正确说法：`deque::shrink_to_fit` 从 C++11 开始存在。但它是非绑定的，编译器可以无视。如果你需要确定性地释放内存，用 swap trick。

### 陷阱 5：多线程访问时的假安全感

```cpp
// Multiple threads reading deque is fine (same as all containers)
// But concurrent read + write is NOT safe, even at different ends

// Thread 1
dq.push_back(x);  // modifying deque

// Thread 2 (simultaneous)
dq.push_front(y);  // also modifying deque -> DATA RACE!
```

deque 对并发访问没有任何特殊保护。如果你需要生产者消费者模式，用 `std::queue` + 自定义的互斥锁，或者用无锁队列。

### 陷阱 6：sort 对 deque 效率不高

```cpp
std::deque<int> dq = {5, 3, 1, 4, 2};

// This works but is slower than sorting a vector
std::sort(dq.begin(), dq.end());

// Better approach: copy to vector, sort, copy back
std::vector<int> temp(dq.begin(), dq.end());
std::sort(temp.begin(), temp.end());
std::copy(temp.begin(), temp.end(), dq.begin());
```

原因：`std::sort` 内部大量使用迭代器的 swap 和随机访问。deque 迭代器的 swap 涉及多次间接寻址，比 vector 迭代器的 swap（简单的内存交换）慢得多。对于大 deque，copy-sort-copy 方案可能反而更快。

---

## 总结

`std::deque` 是一个设计精巧的容器。它用分段连续内存和中控数组的结构，在 `vector` 和 `list` 之间找到了一个平衡点。

**核心优势：**

- 两端插入删除都是 O(1)
- 随机访问 O(1)（虽然常数因子比 vector 大）
- 扩容时不需要搬移已有元素
- 是 `stack` 和 `queue` 的默认底层容器

**核心代价：**

- 内存不是严格连续的，无法传给需要连续内存的接口
- 迭代器复杂、体积大
- 迭代器失效规则复杂，中间操作时所有引用都会失效
- cache 性能不如 vector
- 每个元素有额外的内存开销（chunk 内的潜在碎片）

**选型建议一句话：**

需要 `push_front`，或者元素很大且扩容成本敏感，选 `deque`。其他场景，`vector` 是更安全的选择。

不要把 deque 当成「更好的 vector」。把它当成「支持 push_front 的 vector，但是随机访问和遍历慢一点」。理解了这个取舍，你就能在合适的场景下用好它。

最后，记住一句话：

> 标准库里的每个容器都有它存在的理由。没有万能药，只有合适的工具。理解数据结构，才能做出正确的选择。
