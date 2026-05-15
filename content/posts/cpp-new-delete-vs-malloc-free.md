---
title: "C++ new/delete 与 C malloc/free：从接口差异到 glibc 堆管理原理"
date: 2021-07-14T10:00:00+08:00
tags: ["C++", "内存管理", "malloc", "new", "glibc", "堆"]
categories: ["技术"]
summary: "C++ 的 new/delete 和 C 语言的 malloc/free 都用于动态内存管理，但它们分属不同的抽象层级。本文从接口语义出发，逐层深入到 new 如何调用 malloc、glibc ptmalloc2 的 chunk 结构与分配回收机制、以及 placement new 与 operator new 重载等进阶话题。"
ShowToc: true
---

动态内存管理是系统编程的核心命题。C 语言用 `malloc`/`free` 管理堆内存，C++ 在此基础上引入了 `new`/`delete`。很多程序员知道"new 会调用构造函数"，但仅停留在这一层远远不够——`new` 的内部如何调用 `malloc`？`malloc` 向操作系统申请内存时经历了什么？`delete` 释放的内存真的归还给操作系统了吗？这篇文章从最上层的接口差异一路向下，讲到 glibc 的堆管理实现。

## 接口层：快速对比

先从程序员视角看两者的区别。

### 基本用法

```cpp
// ===== C 风格 =====
int* pi = (int*)malloc(sizeof(int));       // 分配
*pi = 42;
free(pi);                                   // 释放

int* arr = (int*)malloc(10 * sizeof(int));  // 数组
arr[0] = 1;
free(arr);

// ===== C++ 风格 =====
int* pi2 = new int(42);      // 分配 + 初始化
delete pi2;                   // 释放

int* arr2 = new int[10]();   // 分配 + 值初始化（全部为 0）
arr2[0] = 1;
delete[] arr2;                // 注意：数组用 delete[]
```

### 核心差异一览

| 维度 | `malloc`/`free` | `new`/`delete` |
|---|---|---|
| 语言 | C（C++ 也可用） | C++ 专属 |
| 返回类型 | `void*`，需手动转换 | 自动返回正确类型 |
| 失败行为 | 返回 `NULL` | 抛出 `std::bad_alloc`（可配置为 `nothrow` 返回 `nullptr`） |
| 构造/析构 | 不调用 | 自动调用构造函数和析构函数 |
| 大小指定 | 必须手动计算字节数 | 自动推算 `sizeof(T)` |
| 内存不足处理 | 无 | 可设置 `std::new_handler` |
| 重载 | 不可重载 | 可重载 `operator new`/`operator delete` |
| 数组释放 | 同 `free` | 必须用 `delete[]`（非 `delete`） |
| 互用 | **绝对不可混用** | **绝对不可混用** |

> **铁律**：`malloc` 分配的内存必须用 `free` 释放，`new` 分配的必须用 `delete` 释放。混用是未定义行为。

### 失败处理

```cpp
// malloc: 检查返回值
int* p = (int*)malloc(sizeof(int) * 1000000000);
if (p == NULL) {
    // 处理分配失败
    fprintf(stderr, "malloc failed\n");
    return;
}

// new: 捕获异常
try {
    int* p2 = new int[1000000000];
} catch (const std::bad_alloc& e) {
    std::cerr << "new failed: " << e.what() << std::endl;
}

// new nothrow: 不抛异常，返回 nullptr
#include <new>
int* p3 = new(std::nothrow) int[1000000000];
if (p3 == nullptr) {
    std::cerr << "new failed (nothrow)" << std::endl;
}
```

---

## 语义层：new/delete 做了什么

### new 的三步操作

`new` 表达式并非一个简单的函数调用，它做了**三件事**：

```
T* p = new T(args);
```

等价于：

```cpp
// 第一步：调用 operator new 分配原始内存（类似 malloc）
void* raw = operator new(sizeof(T));

// 第二步：在该内存上调用构造函数（placement new）
T* p = ::new(raw) T(args);

// 第三步：返回类型正确的指针
return p;
```

对于数组 `new T[n]`，等价于：

```cpp
void* raw = operator new[](sizeof(T) * n + overhead);  // 可能有额外开销
// 逐个调用构造函数
for (size_t i = 0; i < n; ++i) {
    ::new(static_cast<char*>(raw) + overhead + i * sizeof(T)) T();
}
return static_cast<T*>(static_cast<char*>(raw) + overhead);
```

### delete 的两步操作

```
delete p;
```

等价于：

```cpp
// 第一步：调用析构函数
p->~T();

// 第二步：调用 operator delete 释放内存（类似 free）
operator delete(p);
```

### 调用关系图

```
new 表达式          malloc 函数
    |                   |
    v                   |
operator new  ──────────┘
    |                   |
    v                   v
构造函数          系统调用 (brk/mmap)
                        |
                        v
                    内核管理物理页面
```

关键认知：**`operator new` 的默认实现会调用 `malloc`**。也就是说，C++ 的 `new` 在内存获取这一层，最终还是走 C 运行时的 `malloc`。

```cpp
// glibc libstdc++ 的简化实现
void* operator new(std::size_t size) {
    void* p = malloc(size);
    if (p == nullptr) {
        // 调用 new_handler 或抛出 bad_alloc
        throw std::bad_alloc();
    }
    return p;
}

void operator delete(void* p) noexcept {
    free(p);
}
```

这就是为什么理解 `malloc`/`free` 的底层原理对 C++ 程序员同样重要——你的 `new` 最终调用的就是它。

---

## 底层原理：glibc malloc 如何管理堆

### 进程地址空间中的堆

在 Linux 进程的虚拟地址空间中，堆位于数据段（BSS）之上，向高地址增长：

```
高地址
┌───────────────┐
│    栈 (Stack)  │  ← 向低地址增长
│       ↓       │
├───────────────┤
│               │
│  ← 空闲区域 → │
│               │
├───────────────┤
│    堆 (Heap)   │  ← 向高地址增长 (通过 brk/sbrk)
│       ↑       │
├───────────────┤
│  BSS / Data    │
├───────────────┤
│  Text (代码段) │
└───────────────┘
低地址
```

堆内存的扩展有两种方式：

1. **`brk`/`sbrk`** — 移动进程数据段的末尾指针，用于较小的分配
2. **`mmap`** — 在进程地址空间中映射一块独立的内存区域，用于较大的分配（通常 ≥ 128KB）

### chunk：malloc 的基本管理单元

glibc 的 `malloc`（ptmalloc2 实现）将堆内存划分为**块（chunk）**。每个 chunk 由头部（metadata）和数据区（user data）组成：

```c
// 简化的 chunk 结构
struct malloc_chunk {
    size_t prev_size;   // 前一个 chunk 的大小（仅当前一个 chunk 空闲时有效）
    size_t size;        // 当前 chunk 的大小，包含头部
                         // 最低三位用作标志位：
                         //   bit 0: PREV_INUSE  — 前一个 chunk 是否在使用
                         //   bit 1: IS_MMAPPED   — 是否通过 mmap 分配
                         //   bit 2: NON_MAIN_ARENA — 是否属于非主 arena

    // 以下两个字段仅在空闲 chunk 中存在
    struct malloc_chunk* fd;  // 指向 bin 中下一个空闲 chunk（forward）
    struct malloc_chunk* bk;  // 指向 bin 中上一个空闲 chunk（backward）
};
```

内存布局：

```
已分配的 chunk:
┌──────────────────┐ ← chunk 起始地址
│  prev_size       │  (如果前一个 chunk 在用，此处可复用存用户数据)
├──────────────────┤
│  size | flags    │  ← chunk 头部
├──────────────────┤
│                  │
│  用户数据         │  ← malloc 返回的指针指向这里
│                  │
└──────────────────┘

空闲的 chunk:
┌──────────────────┐
│  prev_size       │
├──────────────────┤
│  size | flags    │
├──────────────────┤
│  fd              │  ← 指向同一 bin 中下一个空闲 chunk
├──────────────────┤
│  bk              │  ← 指向同一 bin 中上一个空闲 chunk
├──────────────────┤
│  (空闲空间)       │
└──────────────────┘
```

**关键细节**：

- `malloc` 返回给用户的指针**不是** chunk 的起始地址，而是跳过头部之后的地址
- chunk 的大小有**最小限制**——在 64 位系统上，空闲 chunk 至少需要 32 字节（头部 16 字节 + fd/bk 各 8 字节），向上对齐到 16 字节
- `size` 字段的低三位用作标志位，因为 chunk 大小总是 8 或 16 字节对齐的，低三位永远是 0

### bins：空闲 chunk 的分类组织

`malloc` 将空闲 chunk 按大小分组存放在不同的**桶（bin）**中，加速查找：

| 类型 | 数量 | 大小范围 | 数据结构 |
|---|---|---|---|
| Fast bins | 10 个 | ≤ 160 字节（64 位） | 单向链表（LIFO） |
| Small bins | 62 个 | ≤ 1024 字节 | 双向链表（FIFO） |
| Large bins | 63 个 | > 1024 字节 | 双向链表（按大小排序） |
| Unsorted bin | 1 个 | 任意 | 双向链表 |

**分配时的查找顺序**：

```
1. Fast bin — 精确匹配（最快路径）
2. Small bin — 精确匹配
3. Unsorted bin — 遍历最近释放的 chunk
4. Large bin — 按 size 排序查找最接近的
5. Top chunk — 从堆顶分割
6. 扩展堆 — 通过 brk/mmap 向 OS 申请新内存
```

### 一次 malloc 的完整流程

以 `malloc(24)` 为例（64 位系统）：

```
1. 将请求大小对齐到 16 字节 → 实际需要 32 字节用户空间
2. 加上 chunk 头部 16 字节 → 总共需要 48 字节 chunk
3. 在 fast bin 中查找大小为 48 的空闲 chunk
   ├─ 找到 → 从 fast bin 中摘除，返回用户数据区指针
   └─ 没找到 → 进入下一步
4. 在 small bin 中查找
   ├─ 找到 → 摘除并返回
   └─ 没找到 → 进入下一步
5. 处理 unsorted bin 中的 chunk（尝试合并相邻空闲块）
6. 在 small/large bin 中查找最接近的大小
7. 从 top chunk 切割
   ├─ top chunk 足够大 → 切割，返回
   └─ top chunk 不够 → 调用 sbrk() 扩展堆
8. 返回用户数据区指针
```

### free 做了什么

`free(ptr)` 并不总是把内存还给操作系统：

1. 根据 `ptr` 向前偏移找到 chunk 头部
2. 检查 chunk 大小和标志位
3. **合并（coalesce）**：检查相邻 chunk 是否空闲，如果是，合并成一个更大的 chunk
4. 小 chunk 放入 fast bin（不合并，快速缓存）
5. 大 chunk 合并后放入 unsorted bin
6. 如果顶部的空闲 chunk 超过阈值（`trim_threshold`，默认 128KB），才通过 `brk` 把内存还给操作系统

> **重要认知**：`free` 之后内存不一定立即归还给操作系统。它可能只是被标记为"空闲"并放入 bin 中，等待下次 `malloc` 复用。这意味着进程的 RSS（常驻内存集）在 `free` 后可能不会立即下降。

---

## new[]/delete[] 的 cookie 问题

### 为什么不能用 delete 释放 new[] 分配的内存

```cpp
class Foo {
public:
    Foo()  { std::cout << "construct\n"; }
    ~Foo() { std::cout << "destruct\n"; }
};

Foo* arr = new Foo[3];
// 如果用 delete arr 而不是 delete[] arr：
// - 只调用第一个元素的析构函数
// - 传递给 operator delete 的地址可能不正确
// - 结果：内存泄漏 + 未定义行为
```

### 底层原因：数组 cookie

`new T[n]` 分配内存时，实现需要在某处记录数组长度 `n`（以便 `delete[]` 知道要调用多少次析构函数）。这个记录被称为**数组 cookie**。

```
new Foo[3] 的内存布局:

┌──────────────┐
│  cookie (n=3) │  ← operator new[] 记录元素数量
├──────────────┤
│  Foo[0]      │
├──────────────┤
│  Foo[1]      │
├──────────────┤
│  Foo[2]      │
└──────────────┘
   ↑
   arr 指向这里（cookie 之后）
```

`delete[] arr` 的流程：

1. 从 `arr` 向前偏移读取 cookie，得到元素数量 `n`
2. 从 `arr[0]` 到 `arr[n-1]` 逐个调用析构函数
3. 向前偏移回 cookie 的位置，调用 `operator delete[]` 释放整块内存

如果错误地使用 `delete arr`（非 `delete[]`），它只会调用一次析构函数，并且传给 `operator delete` 的是 `arr` 的位置而非 chunk 的真正起始位置。

> 对于**平凡类型**（trivial type，如 `int`、POD struct），不需要调用析构函数，有些实现可能不生成 cookie。但永远不要依赖这个行为——始终配对使用 `new[]`/`delete[]`。

---

## placement new：在指定内存上构造对象

`placement new` 是 `new` 的一种特殊形式，它**不分配内存**，只在已有的内存地址上调用构造函数：

```cpp
#include <new>

// 方式一：在栈上构造
alignas(std::string) char buf[sizeof(std::string)];
std::string* ps = new(buf) std::string("hello");
std::cout << *ps << std::endl;  // hello
ps->~std::string();              // 必须手动调用析构函数

// 方式二：在自定义内存池上构造
class MemoryPool {
public:
    void* Allocate(size_t size) { /* ... */ }
    void Deallocate(void* p) { /* ... */ }
};

MemoryPool pool;
void* raw = pool.Allocate(sizeof(std::string));
std::string* ps2 = new(raw) std::string("world");
ps2->~std::string();
pool.Deallocate(raw);
```

**注意**：`placement new` 构造的对象不能用 `delete` 释放——因为内存不是 `operator new` 分配的。必须**手动调用析构函数**，然后由原始的内存管理器释放内存。

### placement new 的典型用途

- **内存池**（memory pool）：预分配大块内存，按需构造对象
- **嵌入式/实时系统**：避免动态内存分配的不确定性
- **自定义分配器**：STL 容器的 allocator 底层使用 placement new

---

## operator new/delete 的重载

C++ 允许重载 `operator new` 和 `operator delete`，实现自定义内存管理策略：

### 全局重载

```cpp
void* operator new(std::size_t size) {
    std::cout << "global new: " << size << " bytes\n";
    void* p = malloc(size);
    if (!p) throw std::bad_alloc();
    return p;
}

void operator delete(void* p) noexcept {
    std::cout << "global delete\n";
    free(p);
}
```

> **警告**：全局重载 `operator new` 影响整个程序的所有 `new` 表达式，包括标准库内部。慎用。

### 类级别重载

```cpp
class GameObject {
public:
    void* operator new(std::size_t size) {
        // 使用自定义内存池
        return pool_.Allocate(size);
    }

    void operator delete(void* p) {
        pool_.Deallocate(p);
    }

private:
    static MemoryPool pool_;
};
```

类级别的重载只影响该类（及其派生类）的 `new`/`delete`，不影响其他类型。

---

## 内存泄漏检测与工具

### 原理：malloc 的调试钩子

glibc 提供了 `__malloc_hook` 等钩子函数，可以在每次 `malloc`/`free` 时触发回调，用于追踪内存分配：

```c
// 仅用于调试，glibc 2.34+ 已弃用
#include <malloc.h>

static void* (*old_malloc_hook)(size_t, const void*);
static void my_malloc_hook(size_t size, const void* caller) {
    __malloc_hook = old_malloc_hook;  // 恢复原钩子，避免递归
    void* result = malloc(size);
    fprintf(stderr, "malloc(%zu) = %p from %p\n", size, result, caller);
    __malloc_hook = my_malloc_hook;   // 重新安装钩子
    return result;
}

void InstallHooks() {
    old_malloc_hook = __malloc_hook;
    __malloc_hook = my_malloc_hook;
}
```

> 注意：`__malloc_hook` 在 glibc 2.34 中已被移除。现代替代方案包括 GCC 的 `-fsanitize=address`、Valgrind 等。

### 实用工具

| 工具 | 原理 | 用途 |
|---|---|---|
| Valgrind | 模拟执行，跟踪每次 alloc/free | 检测泄漏、越界、未初始化读取 |
| AddressSanitizer | 编译器插桩 + shadow memory | 检测越界、use-after-free、泄漏 |
| `mtrace` | glibc 内置的 alloc/free 日志 | 简单的泄漏检测 |
| `mallinfo()`/`mallinfo2()` | 查询 malloc 内部统计 | 了解堆使用情况 |

```cpp
// AddressSanitizer 使用：编译时加 -fsanitize=address
// g++ -g -fsanitize=address -o test test.cpp
// 运行时自动检测内存错误

// mtrace 使用
#include <mcheck.h>
int main() {
    mtrace();  // 开始追踪
    int* p = new int(42);
    // 忘记 delete p; → 运行 MALLOC_TRACE 环境变量指定的日志文件中会记录泄漏
    return 0;
}
```

---

## 现代 C++：还需要直接用 new/delete 吗

### 智能指针

C++11 引入的智能指针在 RAII 模式下封装了 `new`/`delete`，几乎消除了手动管理内存的需要：

```cpp
// unique_ptr：独占所有权
auto p1 = std::make_unique<int>(42);       // 内部调用 new
auto arr = std::make_unique<int[]>(100);   // 内部调用 new[]
// 离开作用域自动 delete，无需手动管理

// shared_ptr：共享所有权
auto p2 = std::make_shared<std::string>("hello");
// 引用计数归零时自动 delete
```

### 何时仍然需要了解底层

1. **性能优化**：理解 `malloc` 的 bin 结构有助于减少内存碎片
2. **自定义分配器**：高性能场景（游戏引擎、数据库）经常重载 `operator new`
3. **调试崩溃**：堆损坏（heap corruption）、double free、use-after-free 的根因分析
4. **嵌入式/实时系统**：确定性内存管理，避免 `malloc` 的不确定延迟
5. **理解异常安全**：`new` 失败时的行为、构造函数异常时的内存回收

### new 失败时的异常安全

```cpp
class Widget {
public:
    Widget() {
        // 如果这里抛出异常：
        // - operator new 分配的内存会被自动回收（operator delete 被调用）
        // - 不会泄漏
        data_ = new int[1000000000];
    }
    ~Widget() { delete[] data_; }
private:
    int* data_;
};

// C++ 保证：new 表达式中，如果构造函数抛出异常，
// operator new 分配的内存会被 operator delete 自动回收。
// 这称为 "unwinding guarantee"。
```

---

## 总结

| 维度 | `malloc`/`free` | `new`/`delete` |
|---|---|---|
| 抽象层级 | 底层内存分配器 | 语言级对象构造/析构 |
| 职责 | 分配/释放原始字节 | 分配内存 + 构造/析构对象 |
| 底层实现 | 直接调用系统调用（brk/mmap） | `operator new` → `malloc` → 系统调用 |
| 类型安全 | 无（`void*`） | 有（返回具体类型指针） |
| 错误处理 | 返回 `NULL` | 抛出 `std::bad_alloc` |
| 可定制性 | 钩子函数（已弃用） | 重载 `operator new/delete` |
| 适合场景 | C 代码、纯字节数据 | C++ 对象（尤其是非平凡类型） |

**一句话**：`new/delete` 是 C++ 在 `malloc/free` 之上构建的面向对象的内存管理层。理解 `malloc` 的底层机制（chunk、bin、brk/mmap），才能在遇到性能问题或内存故障时真正看懂发生了什么。
