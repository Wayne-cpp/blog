---
title: "C++ 多线程编程：从 std::thread 到线程安全的全景解析"
date: 2021-08-01T10:00:00+08:00
tags: ["C++", "多线程", "std::thread", "互斥锁", "条件变量", "线程安全"]
categories: ["C++"]
summary: "系统梳理 C++ 多线程编程的核心机制。从 C++ 封装线程的设计动机出发，深入剖析 std::thread、互斥锁族、条件变量、std::atomic 的原理与用法，并通过生产者-消费者、线程池等实战场景，讲清楚线程安全的常见模式与陷阱。"
ShowToc: true
---

C++11 之前，C++ 标准库没有线程支持——要用多线程，就得直接调用 POSIX 线程（`pthread`）或 Windows API（`CreateThread`）。这意味着同一份代码无法跨平台编译，而且 C 的线程 API 完全不理解 C++ 的对象生命周期、异常安全和 RAII。C++11 引入 `<thread>`、`<mutex>`、`<condition_variable>`、`<atomic>` 等标准组件，不是简单地给 pthread 套个壳，而是把多线程编程纳入了 C++ 的类型系统和资源管理范式。

这篇文章从"为什么 C++ 要自己封装线程"这个问题出发，系统梳理多线程编程的核心机制。

## 为什么 C++ 需要自己的线程库

### C 时代：平台相关的线程 API

```c
// POSIX 线程 (Linux/macOS)
#include <pthread.h>

void* Worker(void* arg) {
    int* data = (int*)arg;
    // ... 干活
    return NULL;
}

int main() {
    pthread_t tid;
    int value = 42;
    pthread_create(&tid, NULL, Worker, &value);
    pthread_join(tid, NULL);
    return 0;
}

// Windows 线程
// #include <windows.h>
// CreateThread(NULL, 0, ThreadFunc, &value, 0, NULL);
```

问题显而易见：

| 问题 | 说明 |
|---|---|
| **平台依赖** | `pthread` 和 `CreateThread` API 完全不同，代码不能跨平台 |
| **无类型安全** | 参数是 `void*`，需要手动强转，容易出错 |
| **无异常安全** | C API 不理解 C++ 异常，线程中抛异常会导致 `std::terminate` |
| **无 RAII** | 锁的获取/释放是手动配对，忘记释放就死锁 |
| **无法与 C++ 特性组合** | 不能传 Lambda、不能用移动语义、不能和模板配合 |

### C++11 的设计目标

C++ 线程库的设计遵循 C++ 的核心哲学——**零开销抽象** + **RAII** + **类型安全**：

```cpp
#include <thread>
#include <iostream>
#include <string>

int main() {
    std::string message = "Hello from thread";

    // 对比 pthread: 有类型安全、支持 Lambda、支持移动语义
    std::thread t([&message]() {
        std::cout << message << std::endl;
    });

    t.join();  // RAII 思想：明确的生命周期管理
    return 0;
}
```

> **设计思想**：不是"封装 pthread"，而是"用 C++ 的方式重新思考并发编程"。线程是一等公民，锁是 RAII 对象，条件变量和类型系统协作，原子操作直接映射到硬件指令。

---

## std::thread：线程的创建与管理

### 基本用法

```cpp
#include <thread>
#include <iostream>

// 方式一：函数指针
void Worker(int id) {
    std::cout << "Thread " << id << " running" << std::endl;
}

// 方式二：Lambda
auto lambda_worker = [](int id) {
    std::cout << "Lambda thread " << id << std::endl;
};

// 方式三：函子
class FunctorWorker {
public:
    explicit FunctorWorker(int id) : id_(id) {}
    void operator()() const {
        std::cout << "Functor thread " << id_ << std::endl;
    }
private:
    int id_;
};

int main() {
    std::thread t1(Worker, 1);            // 函数指针 + 参数
    std::thread t2(lambda_worker, 2);     // Lambda + 参数
    std::thread t3(FunctorWorker(3));     // 函子
    std::thread t4([]{ std::cout << 4; }); // 内联 Lambda

    t1.join();
    t2.join();
    t3.join();
    t4.join();
    return 0;
}
```

### 参数传递：值拷贝 vs 引用

`std::thread` 的构造函数**按值拷贝**所有参数。如果需要传引用，必须用 `std::ref` 或 `std::cref`：

```cpp
void Increment(int& x) { ++x; }

int value = 0;

// std::thread t(Increment, value);      // 编译错误! 值拷贝，无法绑定到 int&
std::thread t(Increment, std::ref(value)); // OK: 显式传递引用

t.join();
std::cout << value << std::endl;  // 1
```

> **为什么这样设计**：线程函数可能在原线程修改参数之后才执行。如果默认传引用，很容易产生悬空引用。强制 `std::ref` 是一种"你确定知道自己在做什么"的显式标记。

### join vs detach

```cpp
std::thread t(SomeFunction);

// join: 等待线程完成（阻塞当前线程）
t.join();

// detach: 分离线程，让它在后台运行（不等待）
// t.detach();
```

| 操作 | 行为 | 风险 |
|---|---|---|
| `join()` | 阻塞等待线程结束 | 无（安全） |
| `detach()` | 分离，线程独立运行 | 线程可能访问已销毁的变量 |
| 不调任何 | 析构时调用 `std::terminate` | 程序崩溃 |

**最重要的规则**：每个 `std::thread` 对象在销毁前**必须**调用 `join()` 或 `detach()`，否则程序会 `terminate`。

### RAII 线程守卫

手动管理 `join` 容易遗漏。用 RAII 包装：

```cpp
class ThreadGuard {
public:
    explicit ThreadGuard(std::thread t) : thread_(std::move(t)) {}
    ~ThreadGuard() {
        if (thread_.joinable()) {
            thread_.join();
        }
    }
    // 禁止拷贝
    ThreadGuard(const ThreadGuard&) = delete;
    ThreadGuard& operator=(const ThreadGuard&) = delete;
    // 允许移动
    ThreadGuard(ThreadGuard&&) = default;
    ThreadGuard& operator=(ThreadGuard&&) = default;
private:
    std::thread thread_;
};

int main() {
    int result = 0;
    {
        ThreadGuard guard(std::thread([&result] {
            result = 42;
        }));
        // 离开作用域时自动 join
    }
    std::cout << result << std::endl;  // 42
    return 0;
}
```

C++20 的 `std::jthread` 内置了这个行为：

```cpp
#include <thread>

int main() {
    std::jthread t([] { /* ... */ });  // 析构时自动 join
    return 0;
}
```

---

## 互斥锁：保护共享数据

### std::mutex

互斥锁（Mutex，Mutual Exclusion）是最基本的同步原语。同一时刻只有一个线程可以持有锁：

```cpp
#include <mutex>
#include <iostream>
#include <thread>

int counter = 0;
std::mutex mtx;

void Increment(int times) {
    for (int i = 0; i < times; ++i) {
        mtx.lock();
        ++counter;
        mtx.unlock();
    }
}

int main() {
    std::thread t1(Increment, 100000);
    std::thread t2(Increment, 100000);
    t1.join();
    t2.join();
    std::cout << counter << std::endl;  // 200000（没有竞争）
    return 0;
}
```

### 手动 lock/unlock 的危险

上面的代码有一个严重问题：如果 `++counter` 抛异常，`unlock` 永远不会被调用——**死锁**。

```cpp
void Dangerous() {
    mtx.lock();
    DoSomething();     // 如果这里抛异常？
    mtx.unlock();      // 永远不会执行
}

// 更糟的情况
mtx.lock();
if (error) return;    // 提前返回，忘记 unlock
mtx.unlock();
```

> **规则**：永远不要手动调用 `lock()` / `unlock()`。用 RAII 锁守卫。

### std::lock_guard：最简单的 RAII 锁

```cpp
#include <mutex>

std::mutex mtx;

void Safe() {
    std::lock_guard<std::mutex> lock(mtx);  // 构造时 lock
    // 临界区：操作共享数据
    // 即使抛异常或提前 return，析构时也会自动 unlock
}
```

### std::unique_lock：灵活的 RAII 锁

`unique_lock` 提供更多控制——可以延迟加锁、手动解锁、和条件变量配合：

```cpp
std::mutex mtx;

void FlexibleLock() {
    std::unique_lock<std::mutex> lock(mtx);  // 构造时加锁

    // 操作共享数据...

    lock.unlock();  // 手动解锁，提前退出临界区

    // 做不需要锁的工作...

    lock.lock();    // 重新加锁

    // 再次操作共享数据...

    // 析构时自动 unlock（如果还持有锁的话）
}
```

| 特性 | `lock_guard` | `unique_lock` |
|---|---|---|
| 自动加锁/解锁 | ✅ | ✅ |
| 手动解锁/重新加锁 | ❌ | ✅ |
| 延迟加锁（defer_lock） | ❌ | ✅ |
| 移动语义 | ❌ | ✅ |
| 配合条件变量 | ❌ | ✅ |
| 开销 | 最小 | 略大（额外状态标志） |

> **选择规则**：默认用 `lock_guard`。需要条件变量、手动解锁或延迟加锁时用 `unique_lock`。

### 锁族一览

| 锁类型 | 用途 |
|---|---|
| `std::mutex` | 基本互斥锁 |
| `std::recursive_mutex` | 可重入锁——同一线程可多次加锁，需相同次数解锁 |
| `std::timed_mutex` | 支持超时的锁——`try_lock_for` / `try_lock_until` |
| `std::shared_mutex` | 读写锁（C++17）——允许多个读者或一个写者 |

### 读写锁（shared_mutex）

```cpp
#include <shared_mutex>
#include <mutex>
#include <thread>
#include <iostream>
#include <vector>

class ThreadSafeCache {
public:
    int Get(const std::string& key) {
        std::shared_lock lock(mutex_);  // 共享锁（读锁）
        return data_.at(key);
    }

    void Set(const std::string& key, int value) {
        std::unique_lock lock(mutex_);  // 独占锁（写锁）
        data_[key] = value;
    }
private:
    std::unordered_map<std::string, int> data_;
    std::shared_mutex mutex_;
};
```

读写锁的语义：

```
多个线程同时读   → 允许（共享锁）
一个线程正在写   → 排他（独占锁）
写的时候不能读   → 排他
读的时候不能写   → 排他
```

### std::scoped_lock：同时锁多个互斥量

C++17 的 `scoped_lock` 可以一次锁定多个互斥量，并且**避免死锁**：

```cpp
std::mutex mtx1, mtx2;

void Transfer(Account& from, Account& to, int amount) {
    // 同时锁定两个 mutex，内部使用 deadlock-avoidance 算法
    std::scoped_lock lock(mtx1, mtx2);

    from.balance -= amount;
    to.balance += amount;
    // 析构时按逆序释放两个锁
}
```

手动锁多个 mutex 的经典死锁：

```cpp
// 线程 A: lock(mtx1) → lock(mtx2)
// 线程 B: lock(mtx2) → lock(mtx1)
// 结果: A 持有 mtx1 等 mtx2，B 持有 mtx2 等 mtx1 → 死锁

// scoped_lock 内部对 mutex 地址排序，确保所有线程按相同顺序加锁
```

---

## 条件变量：等待与通知

### 为什么需要条件变量

互斥锁解决的是"互斥访问"的问题。但多线程还有一个更基本的需求：**等待某个条件成立**。

```cpp
// 错误的做法：忙等待（busy waiting），浪费 CPU
std::mutex mtx;
bool ready = false;

void Waiter() {
    while (true) {
        std::lock_guard lock(mtx);
        if (ready) break;
        // 不断循环检查，消耗 100% CPU
    }
    std::cout << "Ready!" << std::endl;
}
```

条件变量让线程**高效地等待**——不消耗 CPU，直到另一个线程发出通知：

### std::condition_variable

```cpp
#include <mutex>
#include <condition_variable>
#include <thread>
#include <iostream>

std::mutex mtx;
std::condition_variable cv;
bool ready = false;

void Waiter() {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, [] { return ready; });  // 等待条件成立
    // 等价于:
    // while (!ready) { cv.wait(lock); }
    std::cout << "Waiter: proceeding" << std::endl;
}

void Notifier() {
    std::this_thread::sleep_for(std::chrono::seconds(1));
    {
        std::lock_guard<std::mutex> lock(mtx);
        ready = true;
    }  // 离开作用域，释放锁
    cv.notify_one();  // 唤醒一个等待的线程
}

int main() {
    std::thread t1(Waiter);
    std::thread t2(Notifier);
    t1.join();
    t2.join();
    return 0;
}
```

### wait 的工作流程

```
1. 线程调用 cv.wait(lock, predicate)
2. while (!predicate()):
   a. 自动释放锁
   b. 线程进入等待状态（不消耗 CPU）
   c. 收到 notify 后重新获取锁
   d. 检查 predicate，不成立则继续等待
3. predicate 成立，wait 返回，线程持有锁
```

> **关键**：`wait` 必须在循环中检查条件（或使用带谓词的重载）。这是因为**虚假唤醒**（spurious wakeup）——即使没有 `notify`，`wait` 也可能偶尔返回。带谓词的 `wait` 内部自动处理了这个问题。

### notify_one vs notify_all

```cpp
cv.notify_one();  // 唤醒一个等待线程
cv.notify_all();  // 唤醒所有等待线程
```

| 方法 | 适用场景 |
|---|---|
| `notify_one` | 只需唤醒一个线程（如往队列添加一个任务） |
| `notify_all` | 需要唤醒所有线程（如状态变更影响所有等待者） |

---

## 经典模式：生产者-消费者

```cpp
#include <queue>
#include <mutex>
#include <condition_variable>
#include <thread>
#include <iostream>

template <typename T>
class BlockingQueue {
public:
    explicit BlockingQueue(size_t capacity) : capacity_(capacity) {}

    void Push(T value) {
        std::unique_lock lock(mutex_);
        // 等待队列不满
        not_full_.wait(lock, [this] { return queue_.size() < capacity_; });
        queue_.push(std::move(value));
        not_empty_.notify_one();
    }

    T Pop() {
        std::unique_lock lock(mutex_);
        // 等待队列不空
        not_empty_.wait(lock, [this] { return !queue_.empty(); });
        T value = std::move(queue_.front());
        queue_.pop();
        not_full_.notify_one();
        return value;
    }

    bool Empty() const {
        std::lock_guard lock(mutex_);
        return queue_.empty();
    }
private:
    std::queue<T> queue_;
    size_t capacity_;
    mutable std::mutex mutex_;
    std::condition_variable not_full_;
    std::condition_variable not_empty_;
};

int main() {
    BlockingQueue<int> queue(10);

    // 生产者
    std::thread producer([&queue] {
        for (int i = 0; i < 20; ++i) {
            queue.Push(i);
            std::cout << "Produced: " << i << std::endl;
        }
        queue.Push(-1);  // 哨兵值，表示结束
    });

    // 消费者
    std::thread consumer([&queue] {
        int value;
        while ((value = queue.Pop()) != -1) {
            std::cout << "Consumed: " << value << std::endl;
        }
    });

    producer.join();
    consumer.join();
    return 0;
}
```

---

## std::atomic：无锁的线程安全

### 为什么需要原子操作

互斥锁的代价：每次加锁/解锁涉及内核态切换（通常几百纳秒），还可能引起线程阻塞。对于简单的计数器或标志位，这太重了。

`std::atomic` 提供了**无锁的原子操作**——直接利用 CPU 的原子指令（如 `x86` 的 `LOCK` 前缀、`CMPXCHG` 等）：

```cpp
#include <atomic>
#include <thread>
#include <iostream>

std::atomic<int> counter(0);

void Increment(int times) {
    for (int i = 0; i < times; ++i) {
        ++counter;  // 原子递增，不需要锁
    }
}

int main() {
    std::thread t1(Increment, 100000);
    std::thread t2(Increment, 100000);
    t1.join();
    t2.join();
    std::cout << counter.load() << std::endl;  // 200000
    return 0;
}
```

### atomic 的操作

```cpp
std::atomic<int> x(0);

// 读取
int val = x.load();           // 原子读
int val2 = x;                 // 隐式 load

// 写入
x.store(42);                  // 原子写
x = 42;                       // 隐式 store

// 交换
int old = x.exchange(10);     // 原子交换，返回旧值

// 比较并交换（CAS - Compare-And-Swap）
int expected = 10;
bool success = x.compare_exchange_strong(expected, 20);
// 如果 x == expected，则 x = 20，返回 true
// 否则 expected = x，返回 false

// 算术操作
++x;                          // 原子递增
--x;                          // 原子递减
x.fetch_add(5);               // 原子加，返回旧值
x.fetch_sub(3);               // 原子减，返回旧值
```

### CAS：原子操作的基石

几乎所有原子操作的高级原语都基于 **CAS（Compare-And-Swap）**：

```cpp
// 用 CAS 实现原子加法（概念性代码）
int AtomicAdd(std::atomic<int>& x, int delta) {
    int old = x.load();
    while (!x.compare_exchange_weak(old, old + delta)) {
        // CAS 失败（其他线程改了 x），重试
        // old 已被更新为当前值
    }
    return old;
}
```

### 内存序（Memory Order）

`std::atomic` 允许指定内存序，控制操作对其他线程的可见性：

```cpp
std::atomic<int> x(0);
std::atomic<bool> ready(false);

// 线程 A
x.store(42, std::memory_order_relaxed);
ready.store(true, std::memory_order_release);

// 线程 B
while (!ready.load(std::memory_order_acquire)) {}
// 保证能看到 x == 42
assert(x.load(std::memory_order_relaxed) == 42);
```

| 内存序 | 保证 | 开销 |
|---|---|---|
| `memory_order_relaxed` | 原子性，无顺序保证 | 最低 |
| `memory_order_acquire` | 之后的读写不能重排到此操作之前 | 低 |
| `memory_order_release` | 之前的读写不能重排到此操作之后 | 低 |
| `memory_order_acq_rel` | acquire + release | 中 |
| `memory_order_seq_cst`（默认） | 全局一致顺序 | 最高 |

> **实践建议**：除非你深入理解内存模型且有性能分析数据支撑，否则**使用默认的 `seq_cst`**。它是最安全的，性能差异在绝大多数场景下可以忽略。

### atomic vs mutex

| 方面 | `std::atomic` | `std::mutex` |
|---|---|---|
| 机制 | CPU 原子指令 | 操作系统内核对象 |
| 开销 | 纳秒级 | 百纳秒到微秒级 |
| 适用数据 | 单个标量（int、pointer、bool） | 任意临界区 |
| 阻塞 | 不阻塞（无锁） | 可能阻塞 |
| 复杂操作 | 不支持（如多个变量的原子更新） | 支持 |

---

## 线程安全的设计模式

### 模式一：用锁保护不变量

```cpp
class BankAccount {
public:
    explicit BankAccount(int balance) : balance_(balance) {}

    void Deposit(int amount) {
        std::lock_guard lock(mutex_);
        balance_ += amount;
    }

    void Withdraw(int amount) {
        std::lock_guard lock(mutex_);
        if (balance_ < amount) throw std::runtime_error("Insufficient funds");
        balance_ -= amount;
    }

    int GetBalance() const {
        std::lock_guard lock(mutex_);
        return balance_;
    }
private:
    mutable std::mutex mutex_;
    int balance_;
};
```

### 模式二：用 atomic 实现无锁计数

```cpp
class RequestCounter {
public:
    void Record() {
        count_.fetch_add(1, std::memory_order_relaxed);
    }
    int64_t Get() const {
        return count_.load(std::memory_order_relaxed);
    }
    void Reset() {
        count_.store(0, std::memory_order_relaxed);
    }
private:
    std::atomic<int64_t> count_{0};
};
```

### 模式三：用条件变量实现一次性事件

```cpp
class OnceFlag {
public:
    void Wait() {
        std::unique_lock lock(mutex_);
        cv_.wait(lock, [this] { return ready_; });
    }
    void Set() {
        {
            std::lock_guard lock(mutex_);
            ready_ = true;
        }
        cv_.notify_all();
    }
private:
    std::mutex mutex_;
    std::condition_variable cv_;
    bool ready_ = false;
};
```

### 模式四：线程安全的单例（双重检查锁定）

```cpp
class Singleton {
public:
    static Singleton& Instance() {
        Singleton* tmp = instance_.load(std::memory_order_acquire);
        if (!tmp) {
            std::lock_guard<std::mutex> lock(mutex_);
            tmp = instance_.load(std::memory_order_relaxed);
            if (!tmp) {
                tmp = new Singleton();
                instance_.store(tmp, std::memory_order_release);
            }
        }
        return *tmp;
    }
private:
    Singleton() = default;
    static std::atomic<Singleton*> instance_;
    static std::mutex mutex_;
};

std::atomic<Singleton*> Singleton::instance_{nullptr};
std::mutex Singleton::mutex_;

// C++11 更推荐的方式：Meyers' Singleton（利用静态局部变量的线程安全初始化）
class BetterSingleton {
public:
    static BetterSingleton& Instance() {
        static BetterSingleton instance;  // C++11 保证线程安全
        return instance;
    }
private:
    BetterSingleton() = default;
};
```

> **推荐**：优先使用 Meyers' Singleton（静态局部变量），C++11 标准保证其线程安全初始化。

---

## 线程安全容器：为什么标准库没有

C++ 标准库没有 `ConcurrentVector` 或 `ConcurrentHashMap`。这不是遗漏，而是**有意为之**——"线程安全容器"的语义很难定义。

```cpp
// 假设有一个线程安全的 vector
ThreadSafeVector<int> vec;

// 问题 1: TOCTOU (Time-of-Check-to-Time-of-Use)
if (!vec.Empty()) {         // 检查时非空
    // 另一个线程可能在这里 Pop 了最后一个元素
    int val = vec.Back();   // 崩溃！vector 已空
}

// 问题 2: 组合操作不是原子的
vec.PushBack(1);
vec.PushBack(2);
// 其他线程可能在两次 Push 之间看到 vector
```

**解决方案**：把锁放在容器外部，由调用者管理操作的粒度：

```cpp
std::vector<int> data;
std::mutex data_mutex;

// 调用者控制锁的粒度
{
    std::lock_guard lock(data_mutex);
    if (!data.empty()) {
        int val = data.back();
        data.pop_back();
        Process(val);
    }
}
```

---

## 常见陷阱

### 陷阱一：死锁

```cpp
std::mutex mtx1, mtx2;

// 线程 A
void A() {
    std::lock_guard lock1(mtx1);
    std::this_thread::sleep_for(std::chrono::milliseconds(1));
    std::lock_guard lock2(mtx2);  // 等待 mtx2
    // ...
}

// 线程 B
void B() {
    std::lock_guard lock2(mtx2);
    std::this_thread::sleep_for(std::chrono::milliseconds(1));
    std::lock_guard lock1(mtx1);  // 等待 mtx1
    // ...
}

// 死锁: A 持有 mtx1 等 mtx2，B 持有 mtx2 等 mtx1
```

**解决**：用 `std::scoped_lock` 同时锁多个 mutex，或确保所有线程按相同顺序加锁。

### 陷阱二：数据竞争

```cpp
int counter = 0;  // 非原子变量，无锁保护

void Increment() {
    for (int i = 0; i < 100000; ++i) {
        ++counter;  // 数据竞争: 未定义行为
    }
}
```

**解决**：用 `std::atomic<int>` 或 `std::mutex` 保护。

> **关键认知**：数据竞争是**未定义行为**（Undefined Behavior），不仅仅是"结果可能不对"。编译器有权基于"没有数据竞争"的假设做优化，可能导致远比"结果不对"更严重的后果。

### 陷阱三：捕获局部变量的悬空引用

```cpp
std::thread StartTask() {
    int local = 42;
    return std::thread([&local] {
        std::cout << local << std::endl;  // local 已销毁！悬空引用
    });
}
```

**解决**：按值捕获，或确保被捕获的引用生命周期覆盖线程的生命周期。

### 陷阱四：在持有锁时执行耗时操作

```cpp
void ProcessRequest() {
    std::lock_guard lock(mutex_);
    SaveToDatabase();   // 持有锁时做 I/O——其他线程全部阻塞
    SendNetworkPacket(); // 同上
}
```

**解决**：在锁的保护下复制数据，释放锁后再做耗时操作。

```cpp
void ProcessRequest() {
    Data snapshot;
    {
        std::lock_guard lock(mutex_);
        snapshot = shared_data_;  // 在锁内复制
    }
    SaveToDatabase(snapshot);     // 锁外做 I/O
    SendNetworkPacket(snapshot);
}
```

---

## 性能对比

| 同步机制 | 适用场景 | 典型开销 |
|---|---|---|
| `std::mutex` | 保护临界区 | 几十到几百纳秒（无竞争） |
| `std::shared_mutex` | 读多写少 | 读锁几十纳秒，写锁类似 mutex |
| `std::atomic` | 简单标量操作 | 几纳秒（无锁） |
| `std::condition_variable` | 等待/通知 | 上下文切换开销（微秒级） |
| `std::scoped_lock` | 多锁场景 | 等于锁多个 mutex |

---

## 总结

C++ 封装线程库的设计思想：

| 设计目标 | C++ 的解决方案 |
|---|---|
| 跨平台 | `std::thread` 统一 API，内部映射到平台实现 |
| 类型安全 | 模板参数推导，告别 `void*` |
| 异常安全 | RAII 锁守卫，异常路径自动释放 |
| 资源管理 | `lock_guard` / `unique_lock` / `scoped_lock` |
| 与 C++ 特性组合 | Lambda、移动语义、模板完美配合 |
| 性能 | `std::atomic` 直接映射硬件原子指令 |

核心实践规则：

1. **永远用 RAII 管理锁**——`lock_guard` / `unique_lock` / `scoped_lock`，不要手动 `lock`/`unlock`。
2. **锁的粒度尽量小**——在锁内只做必须的操作，I/O 和耗时计算放在锁外。
3. **条件变量用带谓词的 `wait`**——自动处理虚假唤醒。
4. **简单计数用 `atomic`**——比 mutex 轻量得多。
5. **多锁用 `scoped_lock`**——避免死锁。
6. **不要设计"完全线程安全"的容器**——让调用者管理锁的粒度。
7. **注意 Lambda 捕获的生命周期**——引用捕获 + detach 是悬空引用的温床。
