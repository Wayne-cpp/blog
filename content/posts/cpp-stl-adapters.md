---
title: "C++ 容器适配器：stack、queue 与 priority_queue 的封装艺术"
date: 2021-08-12T10:00:00+08:00
tags: ["C++", "STL", "stack", "queue", "priority_queue", "容器适配器"]
categories: ["技术"]
summary: "深入剖析 C++ 三种容器适配器的设计原理与底层实现。从适配器模式出发，讲清楚 stack 的 LIFO 栈、queue 的 FIFO 队列、priority_queue 的堆实现，以及如何选择底层容器、自定义比较器和实际应用场景。"
ShowToc: true
---

C++ 标准模板库（STL）提供了丰富的容器，比如 `vector`、`deque`、`list`、`map`。但有些场景下，你不需要一个"万能"的容器，你只需要一个**受限的接口**。

比如，你想要一个栈，只允许从顶部压入和弹出。你想要一个队列，只允许从一端进、另一端出。你想要一个优先队列，每次只取"最大"的那个元素。

这就是**容器适配器（Container Adaptor）**的用武之地。

它们不是从零实现的新容器，而是在已有容器之上套了一层薄薄的壳，暴露出特定语义的接口。这种设计思路来自经典的**适配器模式（Adapter Pattern）**：不改变底层对象，只改变它对外呈现的"形状"。

STL 提供了三种容器适配器：`std::stack`、`std::queue` 和 `std::priority_queue`。这篇文章会把它们掰开揉碎，从设计原理到底层实现，从接口用法到实际场景，一次性讲清楚。

---

## 适配器模式：用限制换取清晰

在讲具体的适配器之前，先搞明白一个核心问题：为什么用适配器，而不是直接拿底层容器操作？

### 什么是适配器模式

适配器模式是一种结构型设计模式。它的思路很简单：**把一个类的接口转换成客户端期望的另一个接口**。你可以把它想象成一个转接头，比如把 USB-C 接口转成 USB-A。被转换的东西本身没变，只是对外呈现的形式变了。

在 STL 中，容器适配器做的事情是：**拿一个通用的序列容器（比如 `deque`），把它包装成一个具有特定语义的"新容器"（比如栈或队列）**。底层的数据存储完全交给被包装的容器，适配器只负责：

1. 限制可用的操作（比如栈只暴露 `push`、`pop`、`top`）
2. 重新命名操作以符合特定语义（比如 `push` 背后可能调用底层的 `push_back`）

### 为什么要"限制"接口

你可能会想，直接用 `deque` 不就行了？为什么要套一层壳？

原因有三：

**语义清晰。** 当你看到一个 `stack<int>`，你立刻知道它是后进先出的栈。如果你看到 `deque<int>`，你还得去翻代码，确认作者是不是只用它做了栈的操作。适配器让意图变得显式。

**防止误用。** `stack` 不提供迭代器，不提供随机访问，不提供在中间插入的方法。这意味着你不可能"不小心"在栈的中间插入一个元素。**限制本身就是一种保护。**

**可替换底层。** `stack` 的底层不一定是 `deque`，你也可以换成 `vector` 或 `list`。适配器屏蔽了底层差异，你只需要关心栈的操作。

> 规则：如果你的需求满足某一种容器适配器的语义，优先使用适配器，而不是直接使用底层容器。

---

## std::stack：后进先出的栈

`std::stack` 是最简单的容器适配器，实现了经典的**后进先出（LIFO, Last In First Out）**语义。最后放进去的元素，最先被拿出来。

想象一摞盘子：你总是把新盘子放在最上面，拿的时候也从最上面拿。没人会从中间抽盘子出来。

### 底层容器

`std::stack` 的定义如下：

```cpp
template<
    class T,
    class Container = std::deque<T>
>
class stack;
```

默认底层容器是 `std::deque<T>`。为什么选 `deque` 而不是 `vector`？因为 `deque` 在两端操作都是 O(1)，而 `vector` 只在尾部操作是 O(1)。虽然 `stack` 只在尾部操作，但 `deque` 的内存模型更适合作为默认选择（不需要连续内存，扩容代价更低）。

你可以换成 `vector` 或 `list`，只要底层容器支持以下操作：

| 要求的操作          | 对应 stack 操作 |
|:--------------------|:----------------|
| `push_back()`       | `push()`        |
| `pop_back()`        | `pop()`         |
| `back()`            | `top()`         |
| `empty()`           | `empty()`       |
| `size()`            | `size()`        |

### 核心操作

```cpp
#include <stack>
#include <iostream>

void StackBasicOps() {
    std::stack<int> stk;

    // Push elements onto the stack
    stk.push(10);
    stk.push(20);
    stk.push(30);

    // Check size and emptiness
    std::cout << "Size: " << stk.size() << "\n";   // 3
    std::cout << "Empty: " << stk.empty() << "\n";  // 0 (false)

    // Access the top element (does NOT remove it)
    std::cout << "Top: " << stk.top() << "\n";      // 30

    // Pop the top element (returns void)
    stk.pop();
    std::cout << "Top after pop: " << stk.top() << "\n";  // 20
}
```

几个要点：

- `top()` 返回栈顶元素的引用，但**不会移除它**。
- `pop()` 移除栈顶元素，但**返回值是 `void`**。这是一个有意的设计：如果 `pop()` 同时返回元素，在拷贝构造函数抛出异常时，元素就丢失了。所以 STL 把"访问"和"移除"分成两步。
- `push()` 等价于底层的 `push_back()`。

### 拷贝与移动

```cpp
std::stack<int> s1;
s1.push(1);
s1.push(2);

// Copy construction
std::stack<int> s2(s1);
std::cout << s2.top() << "\n";  // 2

// Move construction (C++11)
std::stack<int> s3(std::move(s1));
std::cout << s3.top() << "\n";  // 2
// s1 is now in a valid but unspecified state
```

### 交换操作

```cpp
std::stack<int> left;
left.push(1);
left.push(2);

std::stack<int> right;
right.push(99);

// Member swap
left.swap(right);

std::cout << left.top() << "\n";   // 99
std::cout << right.top() << "\n";  // 2
```

`swap()` 的复杂度取决于底层容器。对于 `deque`，是 O(1) 常数时间。

### 典型应用场景

#### 1. 表达式求值

用两个栈（操作数栈和运算符栈）来实现中缀表达式求值，经典的 Dijkstra 双栈算法：

```cpp
#include <stack>
#include <string>
#include <cassert>

// Evaluate a simple expression like "( 1 + ( 2 * 3 ) )"
// Tokens are separated by spaces, only + and * operators
double EvaluateExpression(const std::string& expr) {
    std::stack<double> operands;
    std::stack<char> operators;

    // Simplified parsing for demonstration
    // In production, use a proper tokenizer
    for (size_t i = 0; i < expr.size(); ++i) {
        char ch = expr[i];
        if (ch == ' ') continue;
        if (ch == '(') {
            continue;
        } else if (ch == '+' || ch == '*') {
            operators.push(ch);
        } else if (ch == ')') {
            char op = operators.top();
            operators.pop();
            double right = operands.top();
            operands.pop();
            double left = operands.top();
            operands.pop();
            if (op == '+') operands.push(left + right);
            else if (op == '*') operands.push(left * right);
        } else if (ch >= '0' && ch <= '9') {
            operands.push(static_cast<double>(ch - '0'));
        }
    }
    return operands.top();
}
```

#### 2. 深度优先搜索（DFS）

用栈模拟递归的调用栈，实现非递归的 DFS：

```cpp
#include <stack>
#include <vector>

void DfsWithStack(const std::vector<std::vector<int>>& graph,
                  int start) {
    int node_count = static_cast<int>(graph.size());
    std::vector<bool> visited(node_count, false);
    std::stack<int> stk;
    stk.push(start);

    while (!stk.empty()) {
        int current = stk.top();
        stk.pop();

        if (visited[current]) continue;
        visited[current] = true;

        // Process current node
        // ...

        // Push neighbors in reverse order
        // so that they are processed in order
        for (auto it = graph[current].rbegin();
             it != graph[current].rend(); ++it) {
            if (!visited[*it]) {
                stk.push(*it);
            }
        }
    }
}
```

#### 3. 撤销系统（Undo）

每次操作前，把状态压入栈。撤销时弹出恢复：

```cpp
#include <stack>
#include <string>

class TextEditor {
public:
    void Type(const std::string& text) {
        // Save current state before modifying
        undo_stack_.push(content_);
        content_ += text;
    }

    bool Undo() {
        if (undo_stack_.empty()) return false;
        content_ = undo_stack_.top();
        undo_stack_.pop();
        return true;
    }

    const std::string& Content() const {
        return content_;
    }

private:
    std::string content_;
    std::stack<std::string> undo_stack_;
};
```

#### 4. 括号匹配

检查一串括号是否配对：

```cpp
#include <stack>
#include <string>

bool IsParenthesesBalanced(const std::string& str) {
    std::stack<char> stk;
    for (char ch : str) {
        if (ch == '(' || ch == '[' || ch == '{') {
            stk.push(ch);
        } else if (ch == ')' || ch == ']' || ch == '}') {
            if (stk.empty()) return false;
            char top = stk.top();
            stk.pop();
            if (ch == ')' && top != '(') return false;
            if (ch == ']' && top != '[') return false;
            if (ch == '}' && top != '{') return false;
        }
    }
    return stk.empty();
}
```

---

## std::queue：先进先出的队列

`std::queue` 实现了**先进先出（FIFO, First In First Out）**语义。先排队的先被服务，就像超市的收银台。

### 底层容器

```cpp
template<
    class T,
    class Container = std::deque<T>
>
class queue;
```

默认底层也是 `std::deque<T>`。底层容器需要支持以下操作：

| 要求的操作          | 对应 queue 操作 |
|:--------------------|:----------------|
| `push_back()`       | `push()`        |
| `pop_front()`       | `pop()`         |
| `front()`           | `front()`       |
| `back()`            | `back()`        |
| `empty()`           | `empty()`       |
| `size()`            | `size()`        |

注意，底层容器必须同时支持 `push_back` 和 `pop_front`。`vector` 不支持 `pop_front`（它的 `pop_front` 是 O(n)），所以 `vector` **不能**作为 `queue` 的底层容器。能用的只有 `deque` 和 `list`。

### 核心操作

```cpp
#include <queue>
#include <iostream>

void QueueBasicOps() {
    std::queue<std::string> q;

    // Enqueue elements
    q.push("Alice");
    q.push("Bob");
    q.push("Charlie");

    std::cout << "Size: " << q.size() << "\n";    // 3
    std::cout << "Empty: " << q.empty() << "\n";   // 0 (false)

    // Access front and back (do NOT remove)
    std::cout << "Front: " << q.front() << "\n";   // Alice
    std::cout << "Back: " << q.back() << "\n";     // Charlie

    // Dequeue the front element
    q.pop();
    std::cout << "Front after pop: " << q.front() << "\n";  // Bob
}
```

和 `stack` 一样，`pop()` 是 `void` 返回。先 `front()` 获取，再 `pop()` 移除。

### 交换与比较

```cpp
std::queue<int> q1;
q1.push(1);
q1.push(2);

std::queue<int> q2;
q2.push(99);

q1.swap(q2);

std::cout << q1.front() << "\n";  // 99
std::cout << q2.front() << "\n";  // 1
```

`queue` 也支持 `==`、`!=`、`<`、`<=`、`>`、`>=` 等比较运算符，它们逐元素比较底层容器的内容。

### 典型应用场景

#### 1. 广度优先搜索（BFS）

BFS 是 `queue` 最经典的应用。按层遍历图或树：

```cpp
#include <queue>
#include <vector>

void BfsWithQueue(const std::vector<std::vector<int>>& graph,
                  int start) {
    int node_count = static_cast<int>(graph.size());
    std::vector<bool> visited(node_count, false);
    std::queue<int> q;

    q.push(start);
    visited[start] = true;

    while (!q.empty()) {
        int current = q.front();
        q.pop();

        // Process current node
        // ...

        for (int neighbor : graph[current]) {
            if (!visited[neighbor]) {
                visited[neighbor] = true;
                q.push(neighbor);
            }
        }
    }
}
```

#### 2. 任务调度

简单的先来先服务（FCFS）调度器：

```cpp
#include <queue>
#include <string>
#include <iostream>

struct Task {
    std::string name;
    int duration_ms;
};

class TaskScheduler {
public:
    void Submit(const Task& task) {
        task_queue_.push(task);
        std::cout << "Submitted: " << task.name << "\n";
    }

    void ProcessAll() {
        while (!task_queue_.empty()) {
            Task current = task_queue_.front();
            task_queue_.pop();
            std::cout << "Processing: " << current.name
                      << " (" << current.duration_ms << "ms)\n";
            // Simulate work...
        }
        std::cout << "All tasks done.\n";
    }

private:
    std::queue<Task> task_queue_;
};
```

#### 3. 消息队列

生产者-消费者模型的基础：

```cpp
#include <queue>
#include <string>
#include <mutex>
#include <condition_variable>

class MessageQueue {
public:
    void Publish(const std::string& message) {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            messages_.push(message);
        }
        cv_.notify_one();
    }

    std::string Consume() {
        std::unique_lock<std::mutex> lock(mutex_);
        cv_.wait(lock, [this] { return !messages_.empty(); });
        std::string msg = messages_.front();
        messages_.pop();
        return msg;
    }

private:
    std::queue<std::string> messages_;
    std::mutex mutex_;
    std::condition_variable cv_;
};
```

---

## std::priority_queue：优先级最高的先出

`std::priority_queue` 是三个适配器中最复杂也最有趣的一个。它保证每次取出的元素都是当前队列中**优先级最高**的。默认情况下，"最大"的元素优先级最高。

### 堆：priority_queue 的灵魂

`priority_queue` 的底层是一个**堆（Heap）**，具体来说是**最大堆（Max-Heap）**。堆是一种特殊的完全二叉树，满足**堆性质**：每个节点的值都大于等于（或小于等于）其子节点的值。

STL 并没有单独的"堆"容器，而是通过算法来操作普通的序列容器（通常是 `vector`）：

- `std::make_heap` — 将一个范围调整为堆
- `std::push_heap` — 向堆中插入元素
- `std::pop_heap` — 移除堆顶元素
- `std::sort_heap` — 将堆排序为有序序列

`priority_queue` 就是把这些堆算法封装起来，提供了一个简洁的接口。

### 底层容器与模板参数

```cpp
template<
    class T,
    class Container = std::vector<T>,
    class Compare = std::less<typename Container::value_type>
>
class priority_queue;
```

三个模板参数：

- **T**：元素类型
- **Container**：底层容器，默认 `std::vector<T>`。必须支持随机访问迭代器、`front()`、`push_back()`、`pop_back()`。可以用 `vector` 或 `deque`。
- **Compare**：比较器，默认 `std::less<T>`，即**最大堆**。传入 `std::greater<T>` 则变成**最小堆**。

> 注意：`priority_queue` 默认是**最大堆**，`top()` 返回最大元素。如果你想取最小元素，需要自定义比较器。

### 核心操作

```cpp
#include <queue>
#include <iostream>
#include <vector>

void PriorityQueueBasicOps() {
    // Max-heap (default)
    std::priority_queue<int> max_heap;
    max_heap.push(30);
    max_heap.push(10);
    max_heap.push(50);
    max_heap.push(20);

    std::cout << "Top (max): " << max_heap.top() << "\n";  // 50

    max_heap.pop();
    std::cout << "Top after pop: " << max_heap.top() << "\n";  // 30

    std::cout << "Size: " << max_heap.size() << "\n";  // 3
    std::cout << "Empty: " << max_heap.empty() << "\n";  // 0
}
```

### 最小堆：自定义比较器

```cpp
#include <queue>
#include <vector>
#include <functional>
#include <iostream>

void MinHeapExample() {
    // Min-heap: use std::greater as comparator
    std::priority_queue<
        int,
        std::vector<int>,
        std::greater<int>
    > min_heap;

    min_heap.push(30);
    min_heap.push(10);
    min_heap.push(50);
    min_heap.push(20);

    std::cout << "Top (min): " << min_heap.top() << "\n";  // 10

    min_heap.pop();
    std::cout << "Top after pop: " << min_heap.top() << "\n";  // 20
}
```

原理：`std::less<T>` 比较两个元素 `a` 和 `b`，当 `a < b` 时返回 `true`。堆算法会把"更小"的元素往下沉，所以最大的元素浮到顶部。换成 `std::greater<T>` 后，"更大"的元素往下沉，最小的元素浮到顶部。

### 堆算法的内部实现

了解 `priority_queue` 的内部工作方式，有助于理解它的性能特征。以最大堆为例：

**push 操作（O(log n)）：**

1. 将新元素追加到 `vector` 末尾（`push_back`）
2. 调用 `std::push_heap`，新元素从叶子节点向上"冒泡"（percolate up / sift up），直到堆性质恢复

**pop 操作（O(log n)）：**

1. 调用 `std::pop_heap`，把堆顶元素和末尾元素交换，然后对新的堆顶执行"下沉"（percolate down / sift down）
2. 调用 `pop_back` 移除末尾的原堆顶元素

```cpp
// Simplified illustration of push_heap logic
template<typename RandomIt, typename Compare>
void SiftUp(RandomIt first, RandomIt last, Compare comp) {
    auto hole_index = std::distance(first, last) - 1;
    auto value = *(first + hole_index);
    auto parent = (hole_index - 1) / 2;

    while (hole_index > 0 && comp(*(first + parent), value)) {
        *(first + hole_index) = *(first + parent);
        hole_index = parent;
        parent = (hole_index - 1) / 2;
    }
    *(first + hole_index) = value;
}
```

### 自定义类型的优先队列

当元素是自定义类型时，你需要提供比较器。可以用函数对象（functor）或 lambda：

```cpp
#include <queue>
#include <string>
#include <vector>
#include <iostream>

struct Task {
    std::string name;
    int priority;
    int duration_ms;
};

// Functor: higher priority value = higher priority
struct TaskComparator {
    bool operator()(const Task& a, const Task& b) const {
        // Return true if a has LOWER priority than b
        // (so b goes to the top)
        return a.priority < b.priority;
    }
};

void CustomPriorityQueue() {
    std::priority_queue<Task, std::vector<Task>, TaskComparator> pq;

    pq.push({"Low", 1, 100});
    pq.push({"Critical", 10, 50});
    pq.push({"Medium", 5, 200});

    // Tasks come out in priority order
    while (!pq.empty()) {
        Task t = pq.top();
        pq.pop();
        std::cout << "[" << t.priority << "] " << t.name
                  << " - " << t.duration_ms << "ms\n";
    }
    // Output:
    // [10] Critical - 50ms
    // [5] Medium - 200ms
    // [1] Low - 100ms
}
```

比较器的规则：`comparator(a, b)` 返回 `true` 意味着 `a` 的优先级**低于** `b`，所以 `b` 排在 `a` 前面。

### 典型应用场景

#### 1. Dijkstra 最短路径

经典的单源最短路径算法，用最小堆优化后时间复杂度为 O((V + E) log V)：

```cpp
#include <queue>
#include <vector>
#include <limits>
#include <functional>

using Edge = std::pair<int, int>;  // (weight, target)
using Graph = std::vector<std::vector<Edge>>;

std::vector<int> Dijkstra(const Graph& graph, int source) {
    int node_count = static_cast<int>(graph.size());
    const int kInfinity = std::numeric_limits<int>::max();
    std::vector<int> distance(node_count, kInfinity);
    std::vector<bool> visited(node_count, false);

    // Min-heap: (distance, node)
    using State = std::pair<int, int>;
    std::priority_queue<State, std::vector<State>, std::greater<State>> pq;

    distance[source] = 0;
    pq.push({0, source});

    while (!pq.empty()) {
        auto [current_dist, current_node] = pq.top();
        pq.pop();

        // Skip already processed nodes
        if (visited[current_node]) continue;
        visited[current_node] = true;

        for (const auto& [weight, neighbor] : graph[current_node]) {
            int new_dist = current_dist + weight;
            if (new_dist < distance[neighbor]) {
                distance[neighbor] = new_dist;
                pq.push({new_dist, neighbor});
            }
        }
    }

    return distance;
}
```

#### 2. Top-K 问题

找出数据流中最大的 K 个元素：

```cpp
#include <queue>
#include <vector>
#include <iostream>
#include <algorithm>

std::vector<int> FindTopK(const std::vector<int>& data, int k) {
    // Min-heap of size k
    std::priority_queue<
        int,
        std::vector<int>,
        std::greater<int>
    > min_heap;

    for (int value : data) {
        if (static_cast<int>(min_heap.size()) < k) {
            min_heap.push(value);
        } else if (value > min_heap.top()) {
            min_heap.pop();
            min_heap.push(value);
        }
    }

    // Extract results
    std::vector<int> result;
    while (!min_heap.empty()) {
        result.push_back(min_heap.top());
        min_heap.pop();
    }
    std::sort(result.rbegin(), result.rend());
    return result;
}
```

---

## 为什么适配器不支持迭代器

`std::stack`、`std::queue` 和 `std::priority_queue` 都不提供迭代器。不能对它们使用范围 `for` 循环，也不能传给 `std::sort` 等 STL 算法。

这不是疏忽，而是**有意为之**的设计决策。

### 理由一：语义保护

迭代器意味着你可以遍历容器中的所有元素，甚至修改它们。这直接违反了适配器的设计初衷。栈的语义是"只能从顶部操作"，如果你能用迭代器从底部偷看或修改元素，栈的语义就被破坏了。

### 理由二：内部不变量

`priority_queue` 内部维护着堆的性质。如果用户通过迭代器随意修改某个元素的值，堆性质可能被破坏，后续的 `push` 和 `pop` 就会产生未定义行为。

### 理由三：内存布局

`priority_queue` 的底层 `vector` 虽然可以随机访问，但元素的排列顺序并不完全有序（它只保证堆性质）。如果提供迭代器，用户可能误以为遍历顺序是有序的，产生混淆。

> 规则：如果你需要遍历容器中的所有元素，说明你的需求已经超出了适配器的语义范围，应该直接使用底层容器。

如果你确实需要访问底层容器，C++11 提供了一个受保护的成员 `c`，你可以通过继承来访问（但这种做法不推荐，破坏了封装性）。

---

## 自定义底层容器

容器适配器允许你替换底层容器。方法是在模板参数中指定。底层容器只需要满足适配器要求的操作接口即可。

### stack 使用 vector

```cpp
#include <stack>
#include <vector>

// Use vector as underlying container for stack
std::stack<int, std::vector<int>> vector_stack;
vector_stack.push(1);
vector_stack.push(2);
std::cout << vector_stack.top() << "\n";  // 2
```

### stack 使用 list

```cpp
#include <stack>
#include <list>

std::stack<int, std::list<int>> list_stack;
list_stack.push(42);
```

### queue 使用 list

```cpp
#include <queue>
#include <list>

// deque is default; list also works since it has push_back/pop_front
std::queue<int, std::list<int>> list_queue;
list_queue.push(100);
list_queue.push(200);
std::cout << list_queue.front() << "\n";  // 100
```

### 何时更换底层容器

| 场景                       | 推荐                  | 原因                              |
|:---------------------------|:----------------------|:----------------------------------|
| 默认使用                   | `deque`               | 均衡的性能，分块内存避免扩容开销  |
| 元素少且连续访问           | `vector`              | 缓存友好，内存连续                |
| 大量插入/删除，不需要随机访问 | `list`             | 插入删除 O(1)，不会导致迭代器失效 |
| 需要稳定引用（引用不失效） | `deque`               | 两端操作不会使已有元素的引用失效  |

---

## 性能对比

### 操作复杂度一览

| 操作           | stack        | queue        | priority_queue |
|:---------------|:-------------|:-------------|:---------------|
| 插入 (push)    | O(1)         | O(1)         | O(log n)       |
| 删除 (pop)     | O(1)         | O(1)         | O(log n)       |
| 访问顶部/首部  | O(1)         | O(1)         | O(1)           |
| 访问尾部       | N/A          | O(1)         | N/A            |
| 查询大小       | O(1)         | O(1)         | O(1)           |
| 判空           | O(1)         | O(1)         | O(1)           |
| 构造           | O(n)         | O(n)         | O(n)           |

### 底层容器对性能的影响

| 操作              | vector 底层  | deque 底层   | list 底层    |
|:------------------|:-------------|:-------------|:-------------|
| stack::push       | O(1)*        | O(1)         | O(1)         |
| stack::pop        | O(1)         | O(1)         | O(1)         |
| queue::push       | N/A          | O(1)         | O(1)         |
| queue::pop        | N/A          | O(1)         | O(1)         |
| 内存局部性        | 极好         | 较好         | 差           |
| 扩容代价          | 可能触发拷贝  | 低           | 无           |

> `*vector` 的 `push_back` 均摊 O(1)，但单次操作可能触发扩容（重新分配并拷贝所有元素）。

### priority_queue 的构造

`priority_queue` 支持从一个已有的范围构造，内部会调用 `std::make_heap`：

```cpp
#include <queue>
#include <vector>

// Build a max-heap from an existing vector
std::vector<int> raw_data = {3, 1, 4, 1, 5, 9, 2, 6};

// Range constructor: O(n) heap construction
std::priority_queue<int> pq(raw_data.begin(), raw_data.end());

std::cout << pq.top() << "\n";  // 9
```

范围构造的时间复杂度是 O(n)，比逐个 `push` 的 O(n log n) 更高效。`std::make_heap` 使用自底向上的堆化（heapify）策略，叶节点不需要操作，只有约一半的节点需要调整。

---

## 实战：带优先级的任务调度器

把前面学到的知识串起来，实现一个实用的任务调度器。支持提交任务、按优先级取出任务、查看队列状态。

```cpp
#include <queue>
#include <vector>
#include <string>
#include <iostream>
#include <chrono>
#include <functional>

// Task definition
struct ScheduledTask {
    std::string id;
    int priority;        // Higher value = higher priority
    int estimated_ms;
    std::function<void()> job;

    // Comparator: higher priority first,
    // ties broken by shorter estimated time
    bool operator<(const ScheduledTask& other) const {
        if (priority != other.priority) {
            return priority < other.priority;
        }
        return estimated_ms > other.estimated_ms;
    }
};

class PriorityTaskScheduler {
public:
    void Submit(const std::string& id,
                int priority,
                int estimated_ms,
                std::function<void()> job) {
        ScheduledTask task{id, priority, estimated_ms, std::move(job)};
        task_queue_.push(task);
        std::cout << "[Submit] " << id
                  << " (priority=" << priority
                  << ", est=" << estimated_ms << "ms)\n";
    }

    void RunNext() {
        if (task_queue_.empty()) {
            std::cout << "[RunNext] No tasks pending.\n";
            return;
        }

        ScheduledTask task = task_queue_.top();
        task_queue_.pop();

        std::cout << "[RunNext] Executing " << task.id
                  << " (priority=" << task.priority << ")\n";

        auto start = std::chrono::steady_clock::now();
        task.job();
        auto end = std::chrono::steady_clock::now();

        auto elapsed = std::chrono::duration_cast<
            std::chrono::milliseconds>(end - start).count();
        std::cout << "[RunNext] " << task.id
                  << " completed in " << elapsed << "ms\n";
    }

    void RunAll() {
        int count = 0;
        while (!task_queue_.empty()) {
            RunNext();
            ++count;
        }
        std::cout << "[RunAll] Executed " << count
                  << " task(s).\n";
    }

    int PendingCount() const {
        return static_cast<int>(task_queue_.size());
    }

private:
    std::priority_queue<ScheduledTask> task_queue_;
};

// Usage example
void DemoScheduler() {
    PriorityTaskScheduler scheduler;

    scheduler.Submit("backup_db", 2, 5000, []() {
        std::cout << "  -> Backing up database...\n";
    });

    scheduler.Submit("send_alert", 10, 100, []() {
        std::cout << "  -> Sending critical alert!\n";
    });

    scheduler.Submit("generate_report", 5, 3000, []() {
        std::cout << "  -> Generating weekly report...\n";
    });

    scheduler.Submit("cleanup_logs", 1, 2000, []() {
        std::cout << "  -> Cleaning up log files...\n";
    });

    scheduler.Submit("refresh_cache", 5, 500, []() {
        std::cout << "  -> Refreshing cache...\n";
    });

    std::cout << "\n--- Running all tasks by priority ---\n\n";
    scheduler.RunAll();
}
```

运行结果：

```
[Submit] backup_db (priority=2, est=5000ms)
[Submit] send_alert (priority=10, est=100ms)
[Submit] generate_report (priority=5, est=3000ms)
[Submit] cleanup_logs (priority=1, est=2000ms)
[Submit] refresh_cache (priority=5, est=500ms)

--- Running all tasks by priority ---

[RunNext] Executing send_alert (priority=10)
  -> Sending critical alert!
[RunNext] send_alert completed in 0ms
[RunNext] Executing refresh_cache (priority=5)
  -> Refreshing cache...
[RunNext] refresh_cache completed in 0ms
[RunNext] Executing generate_report (priority=5)
  -> Generating weekly report...
[RunNext] generate_report completed in 0ms
[RunNext] Executing backup_db (priority=2)
  -> Backing up database...
[RunNext] backup_db completed in 0ms
[RunNext] Executing cleanup_logs (priority=1)
  -> Cleaning up log files...
[RunNext] cleanup_logs completed in 0ms
[RunAll] Executed 5 task(s).
```

注意 `refresh_cache` 和 `generate_report` 的优先级相同（都是 5），但 `refresh_cache` 的预估时间更短（500ms vs 3000ms），所以它先被执行。这就是我们在 `operator<` 中实现的二级排序逻辑。

---

## 三个适配器的快速选择指南

| 你的需求                              | 选择              | 原因                           |
|:--------------------------------------|:------------------|:-------------------------------|
| 后进先出，只需操作顶部                | `stack`           | LIFO 语义，接口最简单          |
| 先进先出，公平排队                    | `queue`           | FIFO 语义，操作两端            |
| 每次取优先级最高/最低的元素           | `priority_queue`  | 堆实现，O(log n) 插入和删除    |
| 需要遍历所有元素                      | 不用适配器        | 适配器不提供迭代器             |
| 需要中间插入或删除                    | 不用适配器        | 适配器只提供端部操作           |
| 需要随机访问                          | 不用适配器        | 适配器不支持随机访问           |

---

## 常见陷阱与注意事项

### 陷阱一：对空容器调用 top() 或 front()

```cpp
std::stack<int> empty_stack;
// Undefined behavior! No bounds checking.
int value = empty_stack.top();

// Safe pattern: always check before access
if (!empty_stack.empty()) {
    value = empty_stack.top();
}
```

`top()`、`front()`、`back()` 在空容器上的行为是**未定义的**。不会抛出异常，不会返回默认值，可能直接崩溃。始终先检查 `empty()`。

### 陷阱二：priority_queue 的比较器方向

```cpp
// This is a MAX-heap, NOT a min-heap!
// std::less means "a < b returns true",
// so b is considered "better" and placed on top
std::priority_queue<int> pq;

// If you want a min-heap, you MUST use std::greater
std::priority_queue<int, std::vector<int>, std::greater<int>> min_pq;
```

这是一个非常常见的错误。记住：`std::less` 对应最大堆，`std::greater` 对应最小堆。直觉上反了，因为比较器定义的是"谁排在后面"而不是"谁排在前面"。

### 陷阱三：pop() 不返回值

```cpp
// WRONG: pop() returns void
// int value = stack.pop();

// CORRECT: two-step pattern
int value = stk.top();
stk.pop();
```

### 陷阱四：priority_queue 中的元素修改

如果你修改了已经在 `priority_queue` 中的元素的值（比如通过指针或引用），堆性质会被破坏，后续操作结果未定义。解决方案是先 `pop` 出来，修改后再 `push` 回去。

```cpp
// Wrong: modifying through reference breaks heap
// auto& elem = some_reference_to_element;
// elem.priority = 100;  // Heap property violated!

// Right: pop, modify, push
auto task = pq.top();
pq.pop();
task.priority = 100;
pq.push(task);
```

---

## 总结

容器适配器的核心思想是**通过限制接口来表达语义**。`stack`、`queue` 和 `priority_queue` 都不是"新容器"，而是对现有容器的封装。

几个要点：

**`std::stack`** 实现后进先出（LIFO）。底层默认用 `deque`，也可以用 `vector` 或 `list`。所有操作都是 O(1)。适合表达式求值、DFS、撤销系统。

**`std::queue`** 实现先进先出（FIFO）。底层默认用 `deque`，也可以用 `list`，但不能用 `vector`（因为需要 `pop_front`）。所有操作都是 O(1)。适合 BFS、任务调度、消息队列。

**`std::priority_queue`** 实现优先级队列，底层是最大堆。默认用 `vector`，通过 `std::make_heap` 等算法维护堆性质。插入和删除是 O(log n)，取堆顶是 O(1)。支持自定义比较器来实现最小堆或自定义排序。适合 Dijkstra、Top-K 问题、带优先级的任务调度。

三个适配器都不提供迭代器。这不是缺陷，而是设计：迭代器会打破适配器维护的语义约束。如果你需要遍历，你应该直接使用底层容器。

选择适配器的原则很简单：**让接口反映意图**。当你需要一个栈，就用 `stack`。当你需要一个队列，就用 `queue`。当你需要按优先级取元素，就用 `priority_queue`。不要用 `deque` 然后假装它是栈，那样只会让代码更难读、更容易出错。
