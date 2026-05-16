---
title: "C++ 分配器（Allocator）：STL 容器的内存管理抽象"
date: 2021-08-17T10:00:00+08:00
tags: ["C++", "STL", "分配器", "Allocator", "内存管理"]
categories: ["技术"]
summary: "深入剖析 C++ 分配器的设计哲学与实现机制。从默认的 std::allocator 出发，讲清楚 allocate/deallocate、construct/destroy 的演进，以及如何自定义分配器实现内存池、arena 分配等高性能策略。"
ShowToc: true
---

STL 容器之所以能灵活地管理各种数据结构，背后有一层关键抽象：**allocator**（分配器）。它把"从哪搞到内存"和"怎么在这块内存上构造对象"这两件事从容器逻辑里彻底剥离出来。大多数人日常写代码用不上自定义分配器，但理解它的设计，能让你在性能敏感场景下拥有真正的控制力。

这篇文章会从 `std::allocator` 的基本模型开始，逐步拆解分配器的接口要求、C++11/17 的演进，最后动手写两个自定义分配器。

## 为什么要把内存管理抽象出来

假设你在写一个 `vector`。你需要：申请一块连续内存、在上面构造对象、析构对象、释放内存。前两步和后两步做的事情性质完全不同。申请/释放内存是系统级操作（调用 `malloc` 或 `operator new`），而构造/析构是语言级操作（调用构造函数和析构函数）。

如果把 `new T[args...]` 直接硬编码在容器里，你就没法控制内存来源。游戏引擎可能想从一个预分配的大块内存里切分，嵌入式系统可能根本没有堆。分配器的存在，让容器不用关心这些细节。

换个角度说，分配器是一种**策略（strategy）模式**。容器负责数据结构的逻辑（插入、删除、遍历），分配器负责内存策略（从哪分配、怎么回收）。两者通过模板参数组合在一起。

## std::allocator 的基本模型

`std::allocator<T>` 是标准库提供的默认分配器。它做的事情很简单：把 C++ 的 `operator new` 和 `operator delete` 包装成一个类型安全的接口。

```cpp
template <typename T>
class allocator {
public:
    using value_type = T;

    // Allocate raw memory for n objects of type T
    // Does NOT call constructors
    T* allocate(std::size_t n);

    // Deallocate memory previously allocated by allocate()
    // Does NOT call destructors
    void deallocate(T* p, std::size_t n);

    // Construct an object at location p using args
    // (Deprecated in C++17, removed in C++20)
    template <typename U, typename... Args>
    void construct(U* p, Args&&... args);

    // Destroy the object at p (call its destructor)
    // (Deprecated in C++17, removed in C++20)
    template <typename U>
    void destroy(U* p);
};
```

四个操作，职责分明：

| 操作            | 做的事                     | 不做的事       |
| --------------- | -------------------------- | -------------- |
| `allocate`      | 分配原始内存（不调用构造函数） | 不构造对象     |
| `deallocate`    | 释放原始内存                 | 不调用析构函数 |
| `construct`     | 在已分配的内存上构造对象       | 不分配内存     |
| `destroy`       | 析构对象                     | 不释放内存     |

> **规则**：`allocate` 只管生肉（原始内存），`construct` 才是把肉做熟（调用构造函数）。两者必须分开调用。

看一个最基本的使用示例：

```cpp
#include <memory>
#include <string>

void BasicAllocatorUsage() {
    std::allocator<std::string> alloc;

    // Step 1: allocate raw memory for 3 strings
    std::string* p = alloc.allocate(3);

    // Step 2: construct objects in that memory
    std::allocator_traits<decltype(alloc)>::construct(alloc, p, "hello");
    std::allocator_traits<decltype(alloc)>::construct(alloc, p + 1, "world");
    std::allocator_traits<decltype(alloc)>::construct(alloc, p + 2, "!");

    // Use the strings
    for (int i = 0; i < 3; ++i) {
        // prints: hello world !
    }

    // Step 3: destroy objects
    for (int i = 0; i < 3; ++i) {
        std::allocator_traits<decltype(alloc)>::destroy(alloc, p + i);
    }

    // Step 4: deallocate raw memory
    alloc.deallocate(p, 3);
}
```

注意这里用了 `std::allocator_traits` 来调用 `construct` 和 `destroy`。C++11 之后，这是推荐做法，后面会解释原因。

## 容器如何使用分配器

每个 STL 容器的模板参数列表里，都有一个可选的分配器参数：

```cpp
namespace std {
    template <typename T, typename Allocator = allocator<T>>
    class vector;

    template <typename T, typename Allocator = allocator<T>>
    class list;

    template <typename Key, typename T, typename Compare = less<Key>,
              typename Allocator = allocator<pair<const Key, T>>>
    class map;
}
```

容器内部持有分配器的一份拷贝（或者通过 EBO，Empty Base Optimization，不占空间）。当容器需要分配内存时，它调用分配器的 `allocate`。当需要在已分配内存上构造元素时，它调用 `construct`（C++11 之后通过 `allocator_traits`）。

```cpp
#include <vector>
#include <list>
#include <map>
#include <memory>

// Default: uses std::allocator<int>
std::vector<int> v1;

// Custom allocator as second parameter
std::vector<int, MyAllocator<int>> v2;

// map's allocator works on pair<const Key, T>
std::map<int, std::string, std::less<int>,
         MyAllocator<std::pair<const int, std::string>>> m;
```

> **注意**：`map` 的分配器类型是 `allocator<pair<const Key, T>>`，不是 `allocator<Key>`。因为 map 内部存储的节点包含完整的键值对。

容器通过 `get_allocator()` 成员函数暴露内部使用的分配器：

```cpp
std::vector<int> vec;
auto alloc = vec.get_allocator();  // returns std::allocator<int>
```

## C++11 的重大变化：std::allocator_traits

C++11 引入了 `std::allocator_traits`，这是分配器机制演进中最重要的一步。

### 问题在哪

在 C++11 之前，如果你想写一个自定义分配器，你必须提供一大堆类型别名和成员函数：`pointer`、`const_pointer`、`reference`、`const_reference`、`rebind`、`construct`、`destroy`、`max_size`、`address`……其中很多都是机械重复的代码。

```cpp
// Pre-C++11: a custom allocator had to provide ALL of these
template <typename T>
class OldAllocator {
public:
    using value_type = T;
    using pointer = T*;
    using const_pointer = const T*;
    using reference = T&;
    using const_reference = const T&;
    using size_type = std::size_t;
    using difference_type = std::ptrdiff_t;

    template <typename U>
    struct rebind { using other = OldAllocator<U>; };

    pointer address(reference x) const { return &x; }
    const_pointer address(const_reference x) const { return &x; }

    pointer allocate(size_type n, const void* hint = 0) {
        return static_cast<pointer>(::operator new(n * sizeof(T)));
    }

    void deallocate(pointer p, size_type n) {
        ::operator delete(p);
    }

    size_type max_size() const {
        return std::numeric_limits<size_type>::max() / sizeof(T);
    }

    void construct(pointer p, const T& val) {
        new (p) T(val);
    }

    void destroy(pointer p) {
        p->~T();
    }
};
```

### allocator_traits 的解决方案

`std::allocator_traits` 充当一层中间件。容器不再直接调用分配器的成员函数，而是通过 traits 间接调用。Traits 提供了合理的默认实现，自定义分配器只需要提供最核心的部分。

```cpp
template <typename Alloc>
struct allocator_traits {
    using allocator_type = Alloc;
    using value_type = typename Alloc::value_type;

    // pointer defaults to value_type*
    using pointer = typename Alloc::pointer;  // or value_type* if missing

    // size_type defaults to std::size_t
    using size_type = typename Alloc::size_type;  // or std::size_t if missing

    // allocate: forwards to alloc.allocate(n)
    static pointer allocate(Alloc& a, size_type n);

    // deallocate: forwards to alloc.deallocate(p, n)
    static void deallocate(Alloc& a, pointer p, size_type n);

    // construct: uses alloc.construct if available,
    // otherwise falls back to ::new (p) T(args...)
    template <typename T, typename... Args>
    static void construct(Alloc& a, T* p, Args&&... args);

    // destroy: uses alloc.destroy if available,
    // otherwise falls back to p->~T()
    template <typename T>
    static void destroy(Alloc& a, T* p);

    // select_on_container_copy_construction
    static Alloc select_on_container_copy_construction(const Alloc& a);
};
```

### 为什么这样做

1. **减少样板代码**：自定义分配器只需提供 `value_type`、`allocate`、`deallocate`，其他全部由 traits 提供默认值
2. **向后兼容**：如果分配器提供了自己的 `construct`/`destroy`，traits 会优先使用
3. **为未来留空间**：标准库可以在 traits 里添加新功能，不需要改动现有的分配器实现

## 自定义分配器的最低要求

有了 `allocator_traits` 之后，一个合法的 C++11 分配器只需要提供以下内容：

```cpp
template <typename T>
class MinimalAllocator {
public:
    using value_type = T;

    // Required: default constructor
    MinimalAllocator() noexcept = default;

    // Required: rebind constructor
    template <typename U>
    MinimalAllocator(const MinimalAllocator<U>&) noexcept {}

    // Required: allocate memory for n objects
    T* allocate(std::size_t n) {
        return static_cast<T*>(::operator new(n * sizeof(T)));
    }

    // Required: deallocate memory
    void deallocate(T* p, std::size_t n) noexcept {
        ::operator delete(p);
    }

    // Required: equality operators
    bool operator==(const MinimalAllocator&) const noexcept { return true; }
    bool operator!=(const MinimalAllocator&) const noexcept { return false; }
};
```

就这么简单。`construct`、`destroy`、`max_size`、`address` 全部由 `allocator_traits` 提供默认实现。

### 可选的高级接口

如果你的分配器有状态（stateful），或者需要控制容器复制时的行为，你可以提供以下可选接口：

| 接口                                            | 作用                                                     |
| ----------------------------------------------- | -------------------------------------------------------- |
| `construct(p, args...)`                         | 自定义构造方式，traits 会优先调用它                       |
| `destroy(p)`                                    | 自定义析构方式，traits 会优先调用它                       |
| `select_on_container_copy_construction()`       | 容器拷贝构造时，返回给新容器的分配器实例                   |
| `propagate_on_container_copy_assignment`        | 容器拷贝赋值时，是否连带赋值分配器                        |
| `propagate_on_container_move_assignment`        | 容器移动赋值时，是否连带赋值分配器                        |
| `propagate_on_container_swap`                   | 容器 swap 时，是否交换分配器                              |
| `is_always_equal`                               | 声明分配器是否总是相等（无状态分配器可以设为 `true_type`） |

这些可选接口都是通过在分配器里定义嵌套的 `using` 别名来启用的：

```cpp
template <typename T>
class StatefulAllocator {
public:
    using value_type = T;

    // Tell containers: when I'm copy-assigned, propagate me too
    using propagate_on_container_copy_assignment = std::true_type;

    // Tell containers: when I'm move-assigned, propagate me too
    using propagate_on_container_move_assignment = std::true_type;

    // Tell containers: swap allocators when containers are swapped
    using propagate_on_container_swap = std::true_type;

    // Tell containers: I'm never equal to another instance
    // (because I own different memory pools)
    using is_always_equal = std::false_type;

    // ... rest of implementation
};
```

## 实战一：Arena（栈式）分配器

Arena 分配器（也叫 stack allocator 或 bump allocator）是最简单的自定义分配器之一。它从一个预分配的连续内存块中依次分配，只增不减。释放操作在析构时一次性回收整个 arena。

这种分配器适用于生命周期明确的批量操作，比如一帧之内的大量临时分配。

```cpp
#include <cstddef>
#include <cstdlib>
#include <memory>
#include <new>
#include <cassert>
#include <algorithm>
#include <vector>
#include <iostream>

// The shared arena resource: owns the raw memory buffer
class ArenaResource {
public:
    explicit ArenaResource(std::size_t size)
        : buffer_(static_cast<char*>(::operator new(size))),
          capacity_(size),
          offset_(0) {}

    ~ArenaResource() {
        ::operator delete(buffer_);
    }

    // No copying
    ArenaResource(const ArenaResource&) = delete;
    ArenaResource& operator=(const ArenaResource&) = delete;

    // Allocate n bytes, bump the offset forward
    void* Allocate(std::size_t n, std::size_t alignment = alignof(std::max_align_t)) {
        // Align the offset
        std::size_t current = reinterpret_cast<std::size_t>(buffer_ + offset_);
        std::size_t aligned = (current + alignment - 1) & ~(alignment - 1);
        std::size_t padding = aligned - current;

        if (offset_ + padding + n > capacity_) {
            throw std::bad_alloc();
        }

        void* ptr = buffer_ + offset_ + padding;
        offset_ += padding + n;
        return ptr;
    }

    // Reset the arena: all previous allocations are invalidated
    // Objects must already be destroyed before calling this
    void Reset() {
        offset_ = 0;
    }

    std::size_t Used() const { return offset_; }
    std::size_t Capacity() const { return capacity_; }

private:
    char* buffer_;
    std::size_t capacity_;
    std::size_t offset_;
};

// The allocator type that hooks into STL containers
template <typename T>
class ArenaAllocator {
public:
    using value_type = T;

    // This allocator is NOT always equal to another instance
    // because each may point to a different arena
    using is_always_equal = std::false_type;

    // Don't propagate on copy/move — each container keeps its arena
    using propagate_on_container_copy_assignment = std::false_type;
    using propagate_on_container_move_assignment = std::false_type;
    using propagate_on_container_swap = std::false_type;

    explicit ArenaAllocator(ArenaResource& resource) : resource_(&resource) {}

    template <typename U>
    ArenaAllocator(const ArenaAllocator<U>& other) noexcept
        : resource_(other.Resource()) {}

    T* allocate(std::size_t n) {
        void* ptr = resource_->Allocate(n * sizeof(T), alignof(T));
        return static_cast<T*>(ptr);
    }

    void deallocate(T* /*p*/, std::size_t /*n*/) noexcept {
        // Arena allocator does nothing on individual deallocation
        // Memory is reclaimed when ArenaResource is destroyed or reset
    }

    ArenaResource* Resource() const { return resource_; }

    // Two allocators are equal iff they share the same arena
    bool operator==(const ArenaAllocator& other) const noexcept {
        return resource_ == other.resource_;
    }

    bool operator!=(const ArenaAllocator& other) const noexcept {
        return resource_ != other.resource_;
    }

private:
    ArenaResource* resource_;
};

// Rebind support
template <typename T>
struct std::allocator_traits<ArenaAllocator<T>> {
    // All the defaults are fine; just expose the rebind mechanism
    // The compiler handles rebind automatically through the templated
    // converting constructor
};
```

使用示例：

```cpp
void ArenaExample() {
    // Create a 64KB arena
    ArenaResource arena(64 * 1024);

    // Create a vector that allocates from the arena
    using Alloc = ArenaAllocator<int>;
    std::vector<int, Alloc> vec(Alloc(arena));

    // Fill it up
    for (int i = 0; i < 1000; ++i) {
        vec.push_back(i);
    }

    std::cout << "Arena used: " << arena.Used() << " / "
              << arena.Capacity() << " bytes" << std::endl;

    // When vec goes out of scope, destructors run but no individual
    // deallocation happens. Arena reclaims everything at once.
}
```

> **注意**：arena 分配器不处理单个对象的释放。所有内存的生命周期绑定在 `ArenaResource` 上。容器析构时会调用元素的析构函数，但不会释放底层内存。只有当 `ArenaResource` 被销毁或 `Reset()` 时，内存才被回收。

这意味着你不能随便在 arena 上创建对象然后期望它独立释放。所有分配在同一个 arena 上的对象必须具有相同或嵌套的生命周期。

## 实战二：固定大小内存池分配器

内存池（memory pool）分配器从一大块预分配内存中切出固定大小的块。每次分配取一个空闲块，释放时归还到空闲链表。

这特别适合 `std::list`、`std::map` 这类基于节点的容器，因为它们的每个节点大小相同。

```cpp
#include <cstddef>
#include <cstdlib>
#include <memory>
#include <new>
#include <cassert>
#include <vector>
#include <list>
#include <iostream>

class MemoryPool {
public:
    // Construct a pool with block_size bytes per block, and capacity blocks
    MemoryPool(std::size_t block_size, std::size_t capacity)
        : block_size_(block_size),
          capacity_(capacity),
          used_count_(0) {
        // Allocate the raw memory
        buffer_ = static_cast<char*>(::operator new(block_size * capacity));

        // Build the free list: each block starts with a pointer to the next free block
        free_list_ = nullptr;
        for (std::size_t i = capacity; i > 0; --i) {
            char* block = buffer_ + (i - 1) * block_size;
            auto next = reinterpret_cast<NodePtr>(block);
            next->next = free_list_;
            free_list_ = next;
        }
    }

    ~MemoryPool() {
        ::operator delete(buffer_);
    }

    MemoryPool(const MemoryPool&) = delete;
    MemoryPool& operator=(const MemoryPool&) = delete;

    // Allocate one block
    void* Allocate() {
        if (!free_list_) {
            throw std::bad_alloc();
        }

        NodePtr block = free_list_;
        free_list_ = free_list_->next;
        ++used_count_;
        return block;
    }

    // Return a block to the pool
    void Deallocate(void* p) {
        if (!p) return;

        auto block = static_cast<NodePtr>(p);
        block->next = free_list_;
        free_list_ = block;
        --used_count_;
    }

    std::size_t BlockSize() const { return block_size_; }
    std::size_t Capacity() const { return capacity_; }
    std::size_t UsedCount() const { return used_count_; }

private:
    // Intrusive free list node
    struct FreeNode {
        FreeNode* next;
    };
    using NodePtr = FreeNode*;

    char* buffer_;
    std::size_t block_size_;
    std::size_t capacity_;
    std::size_t used_count_;
    NodePtr free_list_;
};

template <typename T>
class PoolAllocator {
public:
    using value_type = T;
    using is_always_equal = std::false_type;
    using propagate_on_container_copy_assignment = std::false_type;
    using propagate_on_container_move_assignment = std::false_type;
    using propagate_on_container_swap = std::false_type;

    explicit PoolAllocator(MemoryPool& pool) : pool_(&pool) {}

    template <typename U>
    PoolAllocator(const PoolAllocator<U>& other) noexcept
        : pool_(other.Pool()) {}

    T* allocate(std::size_t n) {
        // This pool allocator is designed for single-object allocation
        // (node-based containers). For n > 1, fall back to global new.
        if (n == 1) {
            return static_cast<T*>(pool_->Allocate());
        }
        return static_cast<T*>(::operator new(n * sizeof(T)));
    }

    void deallocate(T* p, std::size_t n) noexcept {
        if (n == 1) {
            pool_->Deallocate(p);
        } else {
            ::operator delete(p);
        }
    }

    MemoryPool* Pool() const { return pool_; }

    bool operator==(const PoolAllocator& other) const noexcept {
        return pool_ == other.pool_;
    }

    bool operator!=(const PoolAllocator& other) const noexcept {
        return pool_ != other.pool_;
    }

private:
    MemoryPool* pool_;
};
```

使用示例：

```cpp
void PoolExample() {
    // std::list<int> nodes are typically ~24 bytes (prev + next + value)
    // We'll be generous and use 32 bytes per block
    MemoryPool pool(32, 10000);

    using Alloc = PoolAllocator<int>;
    std::list<int, Alloc> lst(Alloc(pool));

    for (int i = 0; i < 5000; ++i) {
        lst.push_back(i);
    }

    std::cout << "Pool usage: " << pool.UsedCount() << " / "
              << pool.Capacity() << " blocks" << std::endl;

    // Removing elements returns blocks to the pool
    lst.pop_front();
    lst.pop_front();

    std::cout << "After pops: " << pool.UsedCount() << " blocks in use"
              << std::endl;
}
```

> **关键点**：内存池分配器适合节点大小固定的容器。对 `std::vector` 意义不大，因为 vector 会一次性申请一大段连续内存。对 `std::list` 和 `std::map` 效果最好。

## 什么时候该写自定义分配器

不是所有场景都需要自定义分配器。大多数情况下，`std::allocator` 配合系统默认的 `malloc/free` 已经够用。以下场景值得考虑：

**1. 实时系统**

游戏、音频处理、控制系统对延迟极度敏感。`malloc` 的耗时不可预测，可能在最坏情况下卡住整个帧。arena 分配器保证 O(1) 的分配时间。

**2. 嵌入式系统**

内存总量有限，没有操作系统级别的堆管理器。你需要在一块确定的内存上自行管理分配。

**3. 高频分配/释放**

某些程序会在短时间内大量创建和销毁小对象（比如解析器 AST 节点、游戏实体组件）。内存池可以消除碎片化，减少系统调用的开销。

**4. 调试和统计**

你可以写一个统计分配次数和总量的分配器，用来定位内存瓶颈。或者写一个在释放后填充垃圾数据的分配器，帮你抓悬空指针。

```cpp
template <typename T>
class DebugAllocator {
public:
    using value_type = T;

    T* allocate(std::size_t n) {
        std::cout << "[ALLOC] " << n * sizeof(T)
                  << " bytes (" << n << " objects)" << std::endl;
        return static_cast<T*>(::operator new(n * sizeof(T)));
    }

    void deallocate(T* p, std::size_t n) noexcept {
        std::cout << "[FREE] " << n * sizeof(T) << " bytes" << std::endl;
        ::operator delete(p);
    }

    bool operator==(const DebugAllocator&) const noexcept { return true; }
    bool operator!=(const DebugAllocator&) const noexcept { return false; }
};
```

**5. NUMA 感知**

在多路服务器上，远端内存访问比本地内存慢得多。你可以写一个绑定到特定 NUMA 节点的分配器。

## 有状态分配器与容器语义

这是最容易踩坑的地方。

默认的 `std::allocator` 是无状态的。两个 `std::allocator<int>` 实例永远是"相等"的，因为它们都调用全局的 `operator new`。但自定义分配器往往是有状态的，比如持有指向 arena 或内存池的指针。

### 核心问题

当你拷贝、移动、交换容器时，分配器怎么办？

```cpp
ArenaResource arena1(4096);
ArenaResource arena2(4096);

using Alloc = ArenaAllocator<int>;
std::vector<int, Alloc> v1(Alloc(arena1));
std::vector<int, Alloc> v2(Alloc(arena2));

v1 = v2;       // Copy assignment: should v1 now use arena2?
v1 = std::move(v2);  // Move assignment: what about v1's elements in arena1?
std::swap(v1, v2);   // Swap: should the allocators swap too?
```

答案取决于分配器里定义的 **propagate** 标志：

### 三种 propagate 标志

| 标志                                               | 拷贝赋值时的行为                             |
| -------------------------------------------------- | -------------------------------------------- |
| `propagate_on_container_copy_assignment = true_type`  | 分配器也跟着赋值，旧分配器的内存需要先释放   |
| `propagate_on_container_copy_assignment = false_type` | 分配器不变，元素被拷贝到旧分配器的内存中     |
| （未定义）                                          | 等同于 `false_type`，traits 提供默认值       |

移动赋值和 swap 同理，各有对应的标志。

### 选择策略

- **无状态分配器**：全部设为 `false_type`（或不设，用默认值）。`is_always_equal = true_type`。
- **arena 分配器**：通常设为 `false_type`。容器拷贝时，元素被复制到目标 arena 中，源 arena 不受影响。
- **独占型内存池**：如果移动语义应该转移所有权，设 `move_assignment = true_type`。

> **陷阱**：如果 `propagate_on_container_swap = true_type`，但两个分配器不相等，`swap` 会触发未定义行为。标准要求在 propagate_swap 为 true 时，交换的两个容器必须拥有相等的分配器。

```cpp
// DANGER: undefined behavior if allocators are not equal
// but propagate_on_container_swap is true_type
using Alloc = PoolAllocator<int>;
MemoryPool pool1(32, 100);
MemoryPool pool2(32, 100);

std::vector<int, Alloc> a(Alloc(pool1));
std::vector<int, Alloc> b(Alloc(pool2));

// pool1 != pool2, so allocators are not equal
// If propagate_on_container_swap is true, this is UB:
std::swap(a, b);
```

解决办法：要么设 `propagate_on_container_swap = false_type`（元素被逐个移动到对方的内存池），要么确保只交换共享同一分配器的容器。

## C++17 的 PMR：多态内存资源

前面展示的自定义分配器都是基于模板的。你在编译期就确定了分配策略。如果你想在运行时切换分配策略（比如根据配置文件选择 arena 或 pool），模板方案就不太灵活了。

C++17 引入了 **PMR（Polymorphic Memory Resource）** 来解决这个问题。

### 核心类型

```cpp
namespace std::pmr {

// Abstract base class for memory resources
class memory_resource {
public:
    void* allocate(std::size_t bytes, std::size_t alignment = alignof(max_align_t));
    void deallocate(void* p, std::size_t bytes,
                    std::size_t alignment = alignof(max_align_t));
    bool is_equal(const memory_resource& other) const noexcept;

    virtual ~memory_resource() = default;

private:
    virtual void* do_allocate(std::size_t bytes, std::size_t alignment) = 0;
    virtual void do_deallocate(void* p, std::size_t bytes,
                               std::size_t alignment) = 0;
    virtual bool do_is_equal(const memory_resource& other) const noexcept = 0;
};

// pmr::vector uses polymorphic allocation through memory_resource*
template <typename T>
using vector = std::vector<T, polymorphic_allocator<T>>;

// pmr::string, pmr::list, pmr::map, etc.
using string = std::basic_string<char, std::char_traits<char>,
                                 polymorphic_allocator<char>>;

}  // namespace std::pmr
```

`polymorphic_allocator<T>` 内部持有一个 `memory_resource*` 指针。所有分配请求通过虚函数派发到具体的内存资源。代价是多一次间接跳转（虚函数调用），好处是运行时灵活性。

### 三种标准内存资源

#### monotonic_buffer_resource

和前面写的 arena 分配器原理相同。只增不减，一次性释放。

```cpp
#include <memory_resource>
#include <vector>
#include <iostream>

void MonotonicExample() {
    // Use a stack buffer as backing memory
    char buffer[4096];
    std::pmr::monotonic_buffer_resource mbr(
        buffer, sizeof(buffer), std::pmr::get_default_resource());

    {
        // All allocations come from the stack buffer
        std::pmr::vector<int> vec(&mbr);
        for (int i = 0; i < 100; ++i) {
            vec.push_back(i);
        }

        std::pmr::vector<std::pmr::string> strs(&mbr);
        strs.emplace_back("hello");
        strs.emplace_back("world");

        std::cout << "Used: " << mbr.bytes_used() << std::endl;
    }

    // Reset: all memory is "freed" at once
    // Individual deallocation is a no-op
    std::cout << "After scope exit, still used: "
              << mbr.bytes_used() << std::endl;

    mbr.release();  // Now it's actually reset
    std::cout << "After release: " << mbr.bytes_used() << std::endl;
}
```

#### unsynchronized_pool_resource

线程不安全的内存池。每个大小类别维护一个空闲链表。适合单线程场景，比 synchronized 版本快。

```cpp
void UnsyncPoolExample() {
    std::pmr::unsynchronized_pool_resource pool;

    std::pmr::vector<int> vec(&pool);
    for (int i = 0; i < 10000; ++i) {
        vec.push_back(i);
    }

    // Fast: individual deallocation returns blocks to the pool
    vec.clear();

    // Blocks are available for reuse without system calls
    vec.resize(10000);
}
```

#### synchronized_pool_resource

线程安全的内存池。内部使用互斥锁保护。多个线程可以安全地共享同一个 pool 实例。

```cpp
#include <thread>
#include <mutex>

void SyncPoolExample() {
    std::pmr::synchronized_pool_resource pool;

    auto worker = [&pool]() {
        std::pmr::vector<int> vec(&pool);
        for (int i = 0; i < 1000; ++i) {
            vec.push_back(i);
        }
    };

    // Safe: multiple threads share the same pool
    std::thread t1(worker);
    std::thread t2(worker);
    t1.join();
    t2.join();
}
```

### pool_options

两种 pool 资源都可以接受 `pool_options` 来调优：

```cpp
std::pmr::pool_options opts;
opts.max_blocks_per_chunk = 1024;  // Max blocks to allocate at once
opts.largest_required_pool_block = 1024;  // Max block size to pool

std::pmr::unsynchronized_pool_resource pool(opts);
```

`max_blocks_per_chunk` 控制每次从上游资源（通常是系统堆）获取多少个空闲块。值越小，内存越省，但分配频率越高。

`largest_required_pool_block` 设定池化的最大块大小。超过这个大小的分配请求直接转发给上游资源，不走池。

### 自定义 memory_resource

你可以继承 `memory_resource`，实现自己的分配策略：

```cpp
class TrackingResource : public std::pmr::memory_resource {
public:
    explicit TrackingResource(std::pmr::memory_resource* upstream =
                                  std::pmr::get_default_resource())
        : upstream_(upstream), total_allocated_(0), total_deallocated_(0) {}

    std::size_t TotalAllocated() const { return total_allocated_; }
    std::size_t TotalDeallocated() const { return total_deallocated_; }
    std::size_t BytesInUse() const {
        return total_allocated_ - total_deallocated_;
    }

private:
    void* do_allocate(std::size_t bytes, std::size_t alignment) override {
        void* p = upstream_->allocate(bytes, alignment);
        total_allocated_ += bytes;
        return p;
    }

    void do_deallocate(void* p, std::size_t bytes,
                       std::size_t alignment) override {
        upstream_->deallocate(p, bytes, alignment);
        total_deallocated_ += bytes;
    }

    bool do_is_equal(const memory_resource& other) const noexcept override {
        return this == &other;
    }

    std::pmr::memory_resource* upstream_;
    std::size_t total_allocated_;
    std::size_t total_deallocated_;
};
```

```cpp
void TrackingExample() {
    TrackingResource tracker;

    {
        std::pmr::vector<int> vec(&tracker);
        vec.reserve(1000);

        std::cout << "After reserve(1000): "
                  << tracker.BytesInUse() << " bytes in use" << std::endl;

        vec.clear();
        std::cout << "After clear: "
                  << tracker.BytesInUse() << " bytes in use" << std::endl;
        // Note: clear() doesn't free capacity
    }

    std::cout << "After destruction: "
              << tracker.BytesInUse() << " bytes in use" << std::endl;
}
```

## 性能对比

不同分配策略的性能特征差异很大。下面这张表概括了主要差异：

| 特性                     | `std::allocator` (malloc) | Arena 分配器          | 内存池分配器          |
| ------------------------ | ------------------------- | --------------------- | --------------------- |
| 分配速度                 | 不确定，通常 100-500ns    | O(1)，约 1-5ns        | O(1)，约 5-20ns       |
| 释放速度                 | 不确定，通常 50-300ns     | 不支持单个释放        | O(1)，约 5-20ns       |
| 内存碎片                 | 可能产生                  | 无                    | 极少                  |
| 适合容器类型             | 全部                      | vector, string 批量   | list, map, set 节点   |
| 线程安全                 | 取决于 malloc 实现        | 不安全，需外部同步     | 取决于实现             |
| 内存利用率               | 高（按需分配）            | 低（预分配，可能浪费） | 中（固定块大小）       |
| 缓存友好性               | 一般                      | 极好（连续内存）       | 较好（相邻块连续）     |
| 可否复用已释放的内存     | 可以                      | 不可以                 | 可以                   |
| 实现复杂度               | 无（用默认的）            | 低                     | 中                     |

实际数字因平台和负载而异，但相对关系基本如此。

做一个简单的 benchmark 感受一下差距：

```cpp
#include <chrono>
#include <list>
#include <iostream>

void BenchmarkListPush() {
    const int N = 100000;

    // 1. Default allocator
    {
        auto start = std::chrono::high_resolution_clock::now();
        std::list<int> lst;
        for (int i = 0; i < N; ++i) {
            lst.push_back(i);
        }
        auto end = std::chrono::high_resolution_clock::now();
        std::cout << "Default allocator: "
                  << std::chrono::duration_cast<std::chrono::microseconds>(
                         end - start)
                         .count()
                  << " us" << std::endl;
    }

    // 2. Pool allocator
    {
        MemoryPool pool(32, N + 100);
        auto start = std::chrono::high_resolution_clock::now();
        std::list<int, PoolAllocator<int>> lst(PoolAllocator<int>(pool));
        for (int i = 0; i < N; ++i) {
            lst.push_back(i);
        }
        auto end = std::chrono::high_resolution_clock::now();
        std::cout << "Pool allocator: "
                  << std::chrono::duration_cast<std::chrono::microseconds>(
                         end - start)
                         .count()
                  << " us" << std::endl;
    }
}
```

在典型的 x86_64 Linux 平台上，pool 版本通常快 2-5 倍，因为省去了大量 `malloc` 调用。

## 实践建议

总结一下实际工作中的选择策略。

**1. 不要过早优化**

如果性能不是瓶颈，就用 `std::allocator`。自定义分配器增加了代码复杂度，也让调试更困难。

**2. 先 profile，再优化**

在引入自定义分配器之前，用工具（Valgrind、ASan、instruments）确认内存分配确实是瓶颈。很多时候瓶颈在别的地方。

**3. 优先用 PMR**

C++17 的 PMR 比手写模板分配器更灵活，也更容易测试。如果你的项目用 C++17 或更新标准，优先考虑 `std::pmr`。

```cpp
// Clean and flexible: choose allocation strategy at runtime
std::pmr::monotonic_buffer_resource arena(1024 * 1024);
std::pmr::vector<int> vec(&arena);
```

**4. Arena 适合临时批量操作**

帧级别的临时数据、请求处理中的中间结果，这些场景下 arena 分配器能省掉大量 `malloc/free` 开销。记住 arena 的限制：不支持单个释放。

**5. Pool 适合节点容器**

`std::list`、`std::map`、`std::unordered_map` 这些基于节点的容器，每次分配的节点大小固定。内存池天生适合这种模式。

**6. 注意 propagate 语义**

有状态分配器一定要想清楚 `propagate_on_container_*` 的取值。不设的话，默认都是 `false_type`，大多数时候这是对的，但 swap 场景下可能出现意料之外的行为。

**7. 测试时用 DebugAllocator**

在开发阶段，用一个带日志的分配器帮你发现异常的分配模式（比如频繁分配释放、过大的单次分配）。

---

分配器是 STL 中被严重低估的机制。它不只是 `new` 的封装，而是一套完整的内存策略抽象。理解它，你就能在面对性能瓶颈时多一张底牌。从默认分配器开始，在需要时切换到 PMR 或自定义实现，这是比较务实的路径。
