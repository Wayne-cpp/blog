---
title: "C++ 智能指针：从 RAII 到所有权语义的全景解析"
date: 2021-07-22T10:00:00+08:00
tags: ["C++", "智能指针", "RAII", "内存管理", "unique_ptr", "shared_ptr"]
categories: ["技术"]
summary: "系统梳理 C++ 智能指针的设计哲学与实现原理。从 RAII 机制出发，深入剖析 unique_ptr 的独占所有权、shared_ptr 的引用计数与控制块、weak_ptr 的观察者角色，以及自定义删除器、循环引用、性能开销等工程关键问题。"
ShowToc: true
---

C++ 把内存管理权交给了程序员，也把内存泄漏的风险一同奉上。裸指针（raw pointer）的 `new` 和 `delete` 看似简单，但在异常安全、多重返回路径、复杂数据结构面前，确保每一次 `new` 都有且仅有一次对应的 `delete` 远比想象中困难。智能指针（Smart Pointer）是 C++ 对这个问题的标准答案——它不是"更聪明的指针"，而是通过 RAII 机制将资源生命周期绑定到对象生命周期，让编译器替你管理资源。

这篇文章从 RAII 原则出发，逐一剖析 `unique_ptr`、`shared_ptr`、`weak_ptr` 的实现原理、使用场景和常见陷阱。

## RAII：智能指针的设计哲学

### 什么是 RAII

RAII（Resource Acquisition Is Initialization，资源获取即初始化）是 C++ 最核心的资源管理范式。它的思想很简单：

1. **获取资源**在对象的构造函数中完成
2. **释放资源**在对象的析构函数中完成
3. 对象的生命周期由**作用域**（scope）控制——离开作用域时析构函数自动调用

```cpp
class FileGuard {
public:
    explicit FileGuard(const char* path) : file_(std::fopen(path, "r")) {
        if (!file_) throw std::runtime_error("Failed to open file");
    }
    ~FileGuard() {
        if (file_) std::fclose(file_);
    }
    // 禁止拷贝
    FileGuard(const FileGuard&) = delete;
    FileGuard& operator=(const FileGuard&) = delete;
    FILE* Get() const { return file_; }
private:
    FILE* file_;
};

void Process(const char* path) {
    FileGuard f(path);  // 构造时打开文件
    // 使用 f.Get() 操作文件...
    std::fgetc(f.Get());
    // 无论正常返回、提前 return 还是抛出异常，析构函数都会关闭文件
}
```

智能指针是 RAII 在动态内存上的泛化应用——构造时获取内存，析构时释放内存。

### 为什么不用裸指针

裸指针的问题不在于它"不智能"，而在于它**不表达所有权**：

```cpp
void Foo(int* p);   // p 指向的对象归谁所有？Foo 需要负责 delete 吗？
void Bar(int& r);   // r 引用的对象归谁所有？生命周期是什么？

int* p = new int(42);
Foo(p);
// 现在 delete p 的责任在谁手里？调用者？Foo？都不清楚。
```

智能指针通过**类型系统**表达所有权语义：

```cpp
void Foo(std::unique_ptr<int> p);    // 转移所有权，Foo 负责 delete
void Bar(const std::unique_ptr<int>& p);  // 借用，不涉及所有权
void Baz(std::shared_ptr<int> p);    // 共享所有权，引用计数管理
```

> **核心原则**：类型即文档。智能指针的类型编码了所有权信息，让代码的读者不需要追踪调用链就能理解资源的生命周期。

---

## unique_ptr：独占所有权

### 基本用法

`std::unique_ptr` 是零开销的智能指针。它独占所指向的对象，不允许多个 `unique_ptr` 指向同一对象：

```cpp
#include <memory>
#include <iostream>

class Widget {
public:
    Widget() { std::cout << "Widget ctor" << std::endl; }
    ~Widget() { std::cout << "Widget dtor" << std::endl; }
    void DoWork() { std::cout << "Widget::DoWork" << std::endl; }
};

int main() {
    // 创建
    auto p = std::make_unique<Widget>();  // Widget ctor

    // 使用 —— 和裸指针一样
    p->DoWork();
    (*p).DoWork();

    // 离开作用域，自动 delete
    // Widget dtor
    return 0;
}
```

### 所有权转移

`unique_ptr` 不可拷贝，但可以**移动**（move）——所有权从源转移到目标：

```cpp
auto p1 = std::make_unique<Widget>();
// auto p2 = p1;              // 编译错误: unique_ptr 不可拷贝
auto p2 = std::move(p1);     // 所有权从 p1 转移到 p2
// 此后 p1 == nullptr，p2 指向对象
```

在函数间传递 `unique_ptr`：

```cpp
// 工厂函数：返回 unique_ptr 表示"生产者把所有权交给调用者"
std::unique_ptr<Widget> CreateWidget() {
    return std::make_unique<Widget>();  // 隐式 move（RVO / 返回值优化）
}

// 接管所有权：参数按值传递
void TakeOwnership(std::unique_ptr<Widget> p) {
    p->DoWork();
    // 离开作用域时自动销毁 Widget
}

// 借用：参数按引用传递，不涉及所有权
void BorrowWidget(const std::unique_ptr<Widget>& p) {
    p->DoWork();
    // 不销毁
}

int main() {
    auto w = CreateWidget();
    BorrowWidget(w);            // 借用，w 仍然持有所有权
    TakeOwnership(std::move(w)); // 转移所有权，此后 w == nullptr
    return 0;
}
```

> **参数传递规则**：按值传递 `unique_ptr` = 转移所有权；按 const 引用传递 = 借用不管理。函数签名的选择本身就是所有权的文档。

### 数组形式的 unique_ptr

`unique_ptr` 对数组和单个对象有不同的偏特化：

```cpp
// 单个对象 —— 调用 delete
auto p1 = std::make_unique<Widget>();
// delete p1;

// 数组 —— 调用 delete[]
auto p2 = std::make_unique<Widget[]>(10);
// delete[] p2;  为每个元素调用析构函数
p2[0].DoWork();  // 支持 operator[]
```

> **最佳实践**：优先用 `std::vector` 或 `std::array` 而非 `unique_ptr<T[]>`。数组形式的 `unique_ptr` 缺乏大小信息，功能远不如标准容器。只在对接 C API 等特殊场景使用。

### 零开销保证

`unique_ptr` 的设计目标之一是**零运行时开销**。在典型实现中：

| 操作 | `unique_ptr` | 裸指针 |
|---|---|---|
| 访问对象 | `p->` 直接解引用 | 相同 |
| 判空 | `if (p)` 直接检查 | 相同 |
| 析构释放 | `delete`（编译器内联） | 需手动 `delete` |
| `sizeof` | 等于一个裸指针（默认删除器） | 相同 |

`unique_ptr` 是一种**编译时机制**——所有权约束在编译期通过删除拷贝构造函数实现，运行时和裸指针完全等价（使用默认删除器时）。

---

## shared_ptr：共享所有权

### 引用计数机制

`std::shared_ptr` 允许多个指针共享同一个对象。内部使用**引用计数**（reference counting）跟踪有多少个 `shared_ptr` 指向该对象：

```cpp
auto p1 = std::make_shared<Widget>();  // 引用计数 = 1
{
    auto p2 = p1;   // 引用计数 = 2（拷贝构造）
    auto p3 = p1;   // 引用计数 = 3
}   // p2, p3 离开作用域，引用计数 = 1

// p1 仍有效，引用计数 = 1
// main 结束时 p1 析构，引用计数 = 0，Widget 被销毁
```

### 控制块（Control Block）

`shared_ptr` 的实现不仅包含指向对象的指针，还包含一个指向**控制块**的指针。控制块存储了：

```
shared_ptr 内部结构:
┌──────────────────┐
│ ptr_ ────────────────→ Widget 对象
│ ctrl_ ──────────────→ 控制块
└──────────────────┘

控制块:
┌──────────────────┐
│ strong_count     │  强引用计数 (shared_ptr 数量)
│ weak_count       │  弱引用计数 (weak_ptr 数量 + 1)
│ ptr              │  指向实际对象（可能和 shared_ptr 的 ptr 不同）
│ allocator        │  分配器（如果用了 allocate_shared）
│ deleter          │  删除器
└──────────────────┘
```

`sizeof(shared_ptr)` 通常是两个指针大小（16 字节 / 64 位系统），比 `unique_ptr` 多一倍。

### make_shared 的优势

```cpp
// 方式一：分开分配
auto p1 = std::shared_ptr<Widget>(new Widget);
// 两次内存分配: 一次 Widget，一次控制块

// 方式二：合并分配（推荐）
auto p2 = std::make_shared<Widget>();
// 一次内存分配: Widget 和控制块连续存放
```

`make_shared` 把对象和控制块分配在同一块内存中：

```
make_shared 的内存布局:
┌──────────────────────────────┐
│ 控制块 (strong_count, etc.)  │
├──────────────────────────────┤
│ Widget 对象                   │
└──────────────────────────────┘
```

优势：
- **一次分配**而非两次，减少内存分配开销
- **缓存友好**，对象和控制块在相邻内存
- **异常安全**，避免 `new` 和构造 `shared_ptr` 之间抛异常导致的泄漏

> **最佳实践**：总是优先使用 `make_shared`，除非需要自定义删除器或分配器。

### 线程安全性

`shared_ptr` 的线程安全模型需要精确理解：

| 操作 | 线程安全？ |
|---|---|
| 不同 `shared_ptr` 指向同一对象，各自读/写自己的 `shared_ptr` 实例 | **安全** |
| 多线程同时读同一个 `shared_ptr` 实例 | **安全** |
| 多线程同时写同一个 `shared_ptr` 实例（拷贝、赋值、reset） | **不安全** |
| 多线程访问被管理的对象 | **不安全**（和裸指针一样） |

```cpp
// 安全：每个线程操作自己的 shared_ptr
std::shared_ptr<Widget> global_ptr = std::make_shared<Widget>();

void ThreadFunc() {
    // 拷贝是安全的——引用计数操作是原子的
    auto local = global_ptr;
    local->DoWork();
}

// 不安全：多线程同时修改同一个 shared_ptr 实例
void UnsafeModify(std::shared_ptr<Widget>& p) {
    p = std::make_shared<Widget>();  // 多线程同时执行 = 数据竞争
}
```

> **规则**：`shared_ptr` 的引用计数是原子操作，多个 `shared_ptr` 实例之间的拷贝/析构是线程安全的。但同一个 `shared_ptr` 实例的并发修改不是线程安全的。被管理对象的线程安全取决于对象本身。

### enable_shared_from_this

有时在成员函数中需要获取指向 `this` 的 `shared_ptr`。直接构造会导致灾难：

```cpp
class BadWidget {
public:
    std::shared_ptr<BadWidget> GetSelf() {
        return std::shared_ptr<BadWidget>(this);  // 灾难！创建了第二个控制块
    }
};

auto p1 = std::make_shared<BadWidget>();
auto p2 = p1->GetSelf();
// p1 和 p2 各有独立的控制块，引用计数各自为 1
// 任何一个析构都会 delete 对象，另一个变成悬空指针
```

正确做法是继承 `std::enable_shared_from_this`：

```cpp
class GoodWidget : public std::enable_shared_from_this<GoodWidget> {
public:
    std::shared_ptr<GoodWidget> GetSelf() {
        return shared_from_this();  // 共享同一控制块
    }
};

auto p1 = std::make_shared<GoodWidget>();
auto p2 = p1->GetSelf();
// p1 和 p2 共享控制块，引用计数 = 2，正确
```

`enable_shared_from_this` 的原理：`shared_ptr` 构造时会检查类型是否继承自 `enable_shared_from_this`，如果是，就把内部的 `weak_ptr` 成员初始化为指向自身的弱引用。`shared_from_this()` 只是把这个弱引用提升为 `shared_ptr`。

> **前提条件**：调用 `shared_from_this()` 时，对象必须已经被 `shared_ptr` 管理。在构造函数中调用是未定义行为。

---

## weak_ptr：观察者角色

### 为什么需要 weak_ptr

`shared_ptr` 的引用计数机制有一个经典陷阱——**循环引用**：

```cpp
class Node {
public:
    std::shared_ptr<Node> next;
    std::shared_ptr<Node> prev;  // 灾难：循环引用
    ~Node() { std::cout << "Node dtor" << std::endl; }
};

int main() {
    auto a = std::make_shared<Node>();
    auto b = std::make_shared<Node>();
    a->next = b;  // a → b，b 的引用计数 +1
    b->prev = a;  // b → a，a 的引用计数 +1
    // a 和 b 互相持有对方的 shared_ptr
    // 离开作用域时引用计数各减到 1，但都不归零
    // 结果：内存泄漏，析构函数不被调用
    return 0;
}
```

`weak_ptr` 解决这个问题——它观察 `shared_ptr` 管理的对象，但**不增加引用计数**：

```cpp
class Node {
public:
    std::shared_ptr<Node> next;
    std::weak_ptr<Node> prev;  // 用 weak_ptr 打破循环
    ~Node() { std::cout << "Node dtor" << std::endl; }
};

int main() {
    auto a = std::make_shared<Node>();
    auto b = std::make_shared<Node>();
    a->next = b;  // b 的强引用计数 = 2
    b->prev = a;  // a 的弱引用计数 +1，强引用不变
    // 离开作用域时：
    // a 的强引用归零，a 被销毁 → Node dtor
    // b 的强引用归零，b 被销毁 → Node dtor
    return 0;
}
```

### weak_ptr 的使用模式

`weak_ptr` 不能直接解引用——必须先通过 `lock()` 提升为 `shared_ptr`：

```cpp
auto sp = std::make_shared<Widget>();
std::weak_ptr<Widget> wp = sp;

// 使用前检查对象是否仍然存活
if (auto locked = wp.lock()) {
    // locked 是 shared_ptr，对象仍存活
    locked->DoWork();
} else {
    // 对象已被销毁
    std::cout << "Object expired" << std::endl;
}

// 或者直接检查
std::cout << wp.use_count() << std::endl;  // 强引用计数
std::cout << wp.expired() << std::endl;     // 等价于 use_count() == 0
```

### 典型应用场景

**缓存系统**：缓存持有对象的 `weak_ptr`，不阻止对象被回收

```cpp
class Cache {
public:
    std::shared_ptr<Resource> Get(const std::string& key) {
        auto it = cache_.find(key);
        if (it != cache_.end()) {
            if (auto locked = it->second.lock()) {
                return locked;  // 缓存命中，对象仍存活
            }
            cache_.erase(it);  // 对象已过期，清理条目
        }
        // 缓存未命中，加载新资源
        auto resource = LoadResource(key);
        cache_[key] = resource;  // 存入 weak_ptr
        return resource;
    }
private:
    std::unordered_map<std::string, std::weak_ptr<Resource>> cache_;
    static std::shared_ptr<Resource> LoadResource(const std::string& key);
};
```

**观察者模式**：Subject 持有 Observer 的 `weak_ptr`，避免阻止 Observer 被销毁

```cpp
class Observer {
public:
    virtual ~Observer() = default;
    virtual void OnEvent(int event) = 0;
};

class Subject {
public:
    void Subscribe(std::weak_ptr<Observer> obs) {
        observers_.push_back(std::move(obs));
    }
    void Notify(int event) {
        // 清理已过期的观察者，并通知存活的
        auto it = observers_.begin();
        while (it != observers_.end()) {
            if (auto locked = it->lock()) {
                locked->OnEvent(event);
                ++it;
            } else {
                it = observers_.erase(it);  // Observer 已销毁，移除
            }
        }
    }
private:
    std::vector<std::weak_ptr<Observer>> observers_;
};
```

---

## 自定义删除器

### 基本用法

智能指针的删除器不限于 `delete`——任何可调用对象都可以作为删除器：

```cpp
// 管理 FILE*
auto file_closer = [](FILE* f) { if (f) std::fclose(f); };
auto f = std::unique_ptr<FILE, decltype(file_closer)>(std::fopen("test.txt", "r"), file_closer);
if (f) std::fgetc(f.get());

// 管理 POSIX 文件描述符
auto fd_closer = [](int* fd) { if (fd && *fd >= 0) close(*fd); delete fd; };
auto fd = std::unique_ptr<int, decltype(fd_closer)>(new int(open("test.txt", O_RDONLY)), fd_closer);

// 管理 GLFWwindow
auto glfw_window_deleter = [](GLFWwindow* w) { if (w) glfwDestroyWindow(w); };
auto window = std::unique_ptr<GLFWwindow, decltype(glfw_window_deleter)>(
    glfwCreateWindow(800, 600, "Test", nullptr, nullptr), glfw_window_deleter);
```

### unique_ptr 和 shared_ptr 删除器的区别

```cpp
// unique_ptr: 删除器是类型的一部分
// 不同的删除器 = 不同的类型
auto deleter1 = [](int* p) { delete p; };
auto deleter2 = [](int* p) { delete p; };
std::unique_ptr<int, decltype(deleter1)> p1(new int, deleter1);
std::unique_ptr<int, decltype(deleter2)> p2(new int, deleter2);
// p1 和 p2 类型不同！（即使行为一样）

// shared_ptr: 删除器不是类型的一部分
// 不同的删除器仍是同一类型
auto sp1 = std::shared_ptr<int>(new int, [](int* p) { delete p; });
auto sp2 = std::shared_ptr<int>(new int, [](int* p) { delete p; std::cout << "deleted" << std::endl; });
// sp1 和 sp2 类型相同，可以放入同一个容器
std::vector<std::shared_ptr<int>> vec = {sp1, sp2};
```

| 特性 | `unique_ptr` | `shared_ptr` |
|---|---|---|
| 删除器存储 | 模板参数的一部分 | 控制块中（类型擦除） |
| 不同删除器 | 不同类型 | 同一类型 |
| 运行时开销 | 零（编译时确定） | 有（虚调用 / 类型擦除） |
| 更换删除器 | 创建新实例 | 创建新实例 |

---

## aliasing constructor：高级技巧

`shared_ptr` 有一个鲜为人知的构造函数——**别名构造函数**（aliasing constructor），允许一个 `shared_ptr` 指向某个对象，但拥有另一个对象的生命周期：

```cpp
struct Person {
    std::string name;
    int age;
};

auto person = std::make_shared<Person>();

// aliasing constructor: 拥有 person 的生命周期，但指向 person->name
std::shared_ptr<std::string> name_ptr(person, &person->name);

// name_ptr 和 person 共享引用计数
// person 不被销毁时，name_ptr 所指的 name 也不会失效
std::cout << *name_ptr << std::endl;
```

这在返回容器内部元素的指针时特别有用：

```cpp
class Registry {
public:
    std::shared_ptr<const Widget> Find(const std::string& id) const {
        auto it = widgets_.find(id);
        if (it == widgets_.end()) return nullptr;
        // 返回指向容器内部元素的 shared_ptr
        // 借用 self 的生命周期，确保容器不被销毁
        return std::shared_ptr<const Widget>(self_, &it->second);
    }
private:
    std::shared_ptr<Registry> self_;
    std::unordered_map<std::string, Widget> widgets_;
};
```

---

## 性能对比

### 开销总览

| 指标 | 裸指针 | `unique_ptr` | `shared_ptr` |
|---|---|---|---|
| `sizeof` | 8 字节 | 8 字节（默认删除器） | 16 字节 |
| 拷贝开销 | 无 | 禁止 | 原子引用计数递增 |
| 析构开销 | 需手动 `delete` | `delete`（内联） | 原子递减 + 条件 `delete` |
| 构造开销 | `new` | `new`（或 `make_unique`） | `new` + 控制块分配（或 `make_shared` 一次分配） |
| 缓存影响 | 取决于用法 | 相同 | 控制块可能在不同缓存行 |
| 线程安全 | 无 | 无（不需要） | 引用计数原子操作 |

### `make_shared` 的一个权衡

`make_shared` 虽然只分配一次内存，但有一个值得注意的副作用：

```cpp
struct BigObject {
    std::vector<char> data_;  // 假设占 1MB
};

auto sp = std::make_shared<BigObject>();

// 当最后一个 shared_ptr 被销毁时：
// - BigObject 的析构函数被调用（data_ 释放）
// - 但控制块和 BigObject 的内存不会被释放，直到最后一个 weak_ptr 也销毁
// - 这意味着 1MB 内存可能在对象析构后仍被占用
```

如果存在 `weak_ptr`，`make_shared` 的合并分配会导致大对象的内存在对象析构后仍无法归还给操作系统（因为控制块还活着）。在这种情况下，用 `shared_ptr<T>(new T)` 更合适：

```cpp
// 当对象很大且可能有 weak_ptr 引用时
auto sp = std::shared_ptr<BigObject>(new BigObject);
// 对象和控制块分开分配
// shared_ptr 归零时立即释放对象内存，控制块独立存活直到 weak_ptr 归零
```

---

## 选择指南

### 决策流程

```
需要管理动态内存吗？
├─ 否 → 不需要智能指针
└─ 是 → 所有权模型是什么？
    ├─ 唯一所有权 → unique_ptr
    │   └─ 总是优先选择 unique_ptr
    ├─ 共享所有权 → shared_ptr
    │   └─ 确认真的需要共享，而非设计问题
    └─ 需要观察但不拥有 → weak_ptr
        └─ 必须搭配 shared_ptr 使用
```

### 三种指针的职责划分

| 智能指针 | 所有权语义 | 开销 | 典型场景 |
|---|---|---|---|
| `unique_ptr` | 独占，不可共享 | 零 | 工厂返回值、Pimpl 惯用法、RAII 包装 |
| `shared_ptr` | 共享，引用计数 | 中 | 图数据结构、异步回调、缓存 |
| `weak_ptr` | 不拥有，仅观察 | 低（配合 shared_ptr） | 打破循环引用、缓存、观察者 |

### 常见反模式

**反模式一：所有东西都用 `shared_ptr`**

```cpp
// Bad: 无脑使用 shared_ptr
class Engine {
    std::shared_ptr<Scene> scene_;
    std::shared_ptr<Renderer> renderer_;
    std::shared_ptr<InputHandler> input_;
    std::shared_ptr<AudioSystem> audio_;
    // ... 所有东西都是 shared_ptr，所有权完全混乱
};

// Good: 明确所有权
class Engine {
    std::unique_ptr<Scene> scene_;         // Engine 独占 Scene
    std::unique_ptr<Renderer> renderer_;   // Engine 独占 Renderer
    Renderer& GetRenderer() { return *renderer_; }  // 传引用给别人
};
```

`shared_ptr` 不是银弹。过度使用会让所有权关系变得模糊，引入不必要的原子操作开销，增加循环引用的风险。

**反模式二：`shared_ptr` 保存 `this`**

```cpp
class BadWidget {
public:
    void Register() {
        registry_.push_back(std::shared_ptr<BadWidget>(this));  // 新控制块，必崩
    }
};
```

正确做法是 `enable_shared_from_this` + `shared_from_this()`，如前文所述。

**反模式三：忽略 Pimpl 的删除器**

```cpp
// widget.h
class Widget {
public:
    Widget();
    ~Widget();  // 必须声明并在 .cpp 中定义
private:
    struct Impl;
    std::unique_ptr<Impl> impl_;  // Impl 是不完整类型
};

// widget.cpp
struct Widget::Impl { /* ... */ };
Widget::Widget() : impl_(std::make_unique<Impl>()) {}
Widget::~Widget() = default;  // 必须在 Impl 完整定义可见处实现
```

如果 `~Widget()` 在头文件中隐式生成（或用 `= default`），编译器看到的是不完整的 `Impl` 类型，无法生成正确的析构函数，会导致编译错误。

---

## 与 C 语言接口的桥接

### 处理 C API 返回的裸指针

```cpp
// C API 返回需要 free 的资源
extern "C" char* strdup(const char* s);

auto deleter = [](char* p) { std::free(p); };
auto str = std::unique_ptr<char, decltype(deleter)>(strdup("hello"), deleter);
```

### 使用 `unique_ptr` 的 `release()`

有时需要把智能指针管理的对象交给 C API（之后由 C API 负责释放）：

```cpp
auto p = std::make_unique<Widget>();
p->Initialize();

// 把所有权交给 C API，C API 负责之后释放
c_api_take_widget(p.release());  // release() 返回裸指针，unique_ptr 不再管理
```

> **注意**：`release()` 后 `unique_ptr` 变为空，不再负责释放。确保接收方有正确的释放机制。

### 用 `get()` 传递借用指针

```cpp
auto p = std::make_unique<Widget>();

// C API 只是借用，不获取所有权
c_api_use_widget(p.get());  // get() 返回裸指针，unique_ptr 仍管理对象
```

---

## 总结

| 智能指针 | 核心原则 | 一句话 |
|---|---|---|
| `unique_ptr` | 独占所有权，零开销 | 默认选择，用 `make_unique` 创建 |
| `shared_ptr` | 共享所有权，引用计数 | 仅在真正需要共享时使用 |
| `weak_ptr` | 不拥有，仅观察 | 打破 `shared_ptr` 的循环引用 |

核心原则：

1. **默认用 `unique_ptr`**——它零开销、语义明确，覆盖 80% 以上的场景。
2. **`shared_ptr` 是显眼的设计声明**——用它的地方意味着"这里需要共享所有权"，值得仔细审视。
3. **`weak_ptr` 是 `shared_ptr` 的补充**——用于观察和缓存，不单独存在。
4. **总是用 `make_unique` / `make_shared` 创建智能指针**——异常安全且更高效。
5. **裸指针只用于非所有权场景**——观察指针、函数参数（传引用更好）。
6. **Pimpl 中用 `unique_ptr` 时注意析构函数的位置**——必须在实现文件中定义。
