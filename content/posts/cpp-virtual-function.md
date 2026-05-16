---
title: "C++ 虚函数、虚函数表与虚函数指针：多态的底层实现全解析"
date: 2021-07-20T10:00:00+08:00
tags: ["C++", "虚函数", "多态", "vtable", "面向对象"]
categories: ["C++"]
summary: "从编译器实现角度深入剖析 C++ 多态的核心机制：虚函数、虚函数表（vtable）与虚函数指针（vptr）。通过内存布局分析、代码示例和汇编视角，讲清楚动态绑定的底层原理，以及纯虚函数、多重继承、虚析构函数等关键场景的工程实践。"
ShowToc: true
---

C++ 的多态是面向对象编程的核心能力之一，而虚函数是实现运行时多态的唯一机制。大部分教程停留在"基类指针调用派生类方法"这个层面，但真正理解多态——理解它为什么能工作、代价是什么、边界在哪里——需要深入到虚函数表（vtable）和虚函数指针（vptr）。这篇文章从编译器的视角，把虚函数的底层机制一次性讲透。

## 多态：静态绑定 vs 动态绑定

### 什么是多态

多态（Polymorphism）指同一接口在不同对象上表现出不同行为。C++ 支持两种多态：

| 类型 | 时机 | 机制 | 示例 |
|---|---|---|---|
| 编译时多态（静态） | 编译期确定 | 函数重载、模板 | `max(1, 2)` vs `max(1.0, 2.0)` |
| 运行时多态（动态） | 运行期确定 | 虚函数 | 基类指针调用派生类方法 |

虚函数实现的是**运行时多态**——编译器在编译时不知道最终调用哪个函数，运行时根据对象的实际类型来决定。

### 为什么需要虚函数

先看一个不用虚函数的例子：

```cpp
class Animal {
public:
    void Speak() {
        std::cout << "Animal sound" << std::endl;
    }
};

class Dog : public Animal {
public:
    void Speak() {
        std::cout << "Woof!" << std::endl;
    }
};

int main() {
    Animal* a = new Dog();
    a->Speak();  // 输出 "Animal sound"，而非 "Woof!"
    delete a;
    return 0;
}
```

没有 `virtual` 修饰时，编译器根据指针的**静态类型**（`Animal*`）决定调用 `Animal::Speak`。这就是**静态绑定**（早绑定）。

加上 `virtual` 后：

```cpp
class Animal {
public:
    virtual void Speak() {
        std::cout << "Animal sound" << std::endl;
    }
};

class Dog : public Animal {
public:
    void Speak() override {
        std::cout << "Woof!" << std::endl;
    }
};

int main() {
    Animal* a = new Dog();
    a->Speak();  // 输出 "Woof!"
    delete a;
    return 0;
}
```

这次输出 `"Woof!"`。编译器根据指针指向对象的**实际类型**（`Dog`）来选择函数版本——**动态绑定**（晚绑定）。

> **核心区别**：静态绑定看指针类型，动态绑定看对象类型。`virtual` 关键字就是从静态绑定切换到动态绑定的开关。

---

## 虚函数表（vtable）：编译器的函数派发表

### 什么是 vtable

当类中存在虚函数时，编译器会为该类生成一个**虚函数表**（Virtual Function Table，简称 vtable）。它是一个**静态数组**，存储了该类所有虚函数的函数指针。

每个类（不是每个对象）只有一份 vtable，放在只读数据段中。

```cpp
class Base {
public:
    virtual void Foo() { std::cout << "Base::Foo" << std::endl; }
    virtual void Bar() { std::cout << "Base::Bar" << std::endl; }
    void Baz()      { std::cout << "Base::Baz" << std::endl; }
};
```

编译器为 `Base` 生成的 vtable 大致如下：

```
Base vtable:
  [0]: &Base::Foo   // 虚函数，进入 vtable
  [1]: &Base::Bar   // 虚函数，进入 vtable
  // Baz 不是虚函数，不在 vtable 中
```

### 派生类如何覆盖 vtable

派生类继承基类的 vtable 副本，然后用自己的函数替换被覆盖的条目：

```cpp
class Derived : public Base {
public:
    void Foo() override { std::cout << "Derived::Foo" << std::endl; }
    // 没有覆盖 Bar
};
```

```
Derived vtable:
  [0]: &Derived::Foo  // 被覆盖，替换为派生类版本
  [1]: &Base::Bar     // 未覆盖，沿用基类版本
```

### vtable 的工作流程

当通过基类指针调用虚函数时：

```
a->Speak()
 ↓
1. 通过对象 a 找到 vptr（指向 vtable 的指针）
2. 通过 vptr 找到对象的实际 vtable
3. 在 vtable 中查找 Speak 对应的槽位
4. 通过函数指针调用正确的函数
```

这就是**动态绑定**的完整过程——一次额外的间接寻址。

---

## 虚函数指针（vptr）：每个对象的秘密成员

### 什么是 vptr

每个包含虚函数的对象内部都有一个隐藏的成员变量——**虚函数指针**（Virtual Pointer，简称 vptr）。它指向该对象所属类的 vtable。

vptr 在构造函数中初始化，指向当前类的 vtable。这也是为什么构造函数中调用虚函数不会发生多态——此时 vptr 还没指向派生类的 vtable。

### 对象的内存布局

```cpp
class Base {
public:
    virtual void Foo() {}
    virtual void Bar() {}
    int base_data_;
};

class Derived : public Base {
public:
    void Foo() override {}
    int derived_data_;
};
```

在典型的 64 位系统上（GCC/Clang），内存布局如下：

```
Base 对象 (sizeof = 16):
┌──────────────────┐
│ vptr             │  8 字节 - 指向 Base 的 vtable
├──────────────────┤
│ base_data_       │  4 字节
├──────────────────┤
│ padding          │  4 字节 (对齐)
└──────────────────┘

Derived 对象 (sizeof = 16):
┌──────────────────┐
│ vptr             │  8 字节 - 指向 Derived 的 vtable
├──────────────────┤
│ base_data_       │  4 字节
├──────────────────┤
│ derived_data_    │  4 字节
└──────────────────┘
```

可以验证：

```cpp
#include <iostream>

class NoVirtual {
    int data_;
};

class WithVirtual {
    virtual void Foo() {}
    int data_;
};

int main() {
    std::cout << sizeof(NoVirtual) << std::endl;    // 4
    std::cout << sizeof(WithVirtual) << std::endl;  // 16 (8 字节 vptr + 4 字节 data + 4 字节对齐)
    return 0;
}
```

> **代价**：每个含虚函数的对象多一个指针大小的开销（64 位系统上 8 字节），每个类多一张 vtable。这是 C++ 实现"零开销抽象"原则中少数有运行成本的特性。

---

## 构造与析构中的 vptr 行为

### 构造函数中 vptr 的变化过程

构造对象时，vptr 会随着继承层次逐层更新：

```cpp
class Base {
public:
    Base() {
        std::cout << "Base ctor: ";
        WhoAmI();  // 调用的是 Base::WhoAmI，不是 Derived::WhoAmI
    }
    virtual void WhoAmI() { std::cout << "Base" << std::endl; }
};

class Derived : public Base {
public:
    Derived() {
        std::cout << "Derived ctor: ";
        WhoAmI();  // 调用的是 Derived::WhoAmI
    }
    void WhoAmI() override { std::cout << "Derived" << std::endl; }
};

int main() {
    Derived d;
    // 输出:
    // Base ctor: Base       ← Base 构造时 vptr 指向 Base vtable
    // Derived ctor: Derived  ← Derived 构造时 vptr 指向 Derived vtable
    return 0;
}
```

构造过程的 vptr 轨迹：

```
1. 分配内存（此时是原始内存）
2. 进入 Base() 构造函数
   → vptr 设置为 &Base::vtable
   → 调用 Base::WhoAmI()
3. 进入 Derived() 构造函数
   → vptr 更新为 &Derived::vtable
   → 调用 Derived::WhoAmI()
```

### 析构函数中的行为：逆序回退

析构时 vptr 的变化与构造相反：

```cpp
class Base {
public:
    virtual ~Base() {
        std::cout << "Base dtor: ";
        WhoAmI();  // Base::WhoAmI
    }
    virtual void WhoAmI() { std::cout << "Base" << std::endl; }
};

class Derived : public Base {
public:
    ~Derived() override {
        std::cout << "Derived dtor: ";
        WhoAmI();  // Derived::WhoAmI
    }
    void WhoAmI() override { std::cout << "Derived" << std::endl; }
};

int main() {
    Base* p = new Derived();
    delete p;
    // 输出:
    // Derived dtor: Derived  ← Derived 析构时 vptr 指向 Derived vtable
    // Base dtor: Base         ← Base 析构时 vptr 已回退到 Base vtable
    return 0;
}
```

> **规则**：构造和析构过程中，对象是动态变化的。在基类的构造/析构函数中，对象就是基类类型——vptr 指向基类的 vtable，虚函数调用不会下放到派生类。这是 C++ 防止未定义行为的重要设计。

---

## 纯虚函数与抽象类

### 定义与语法

纯虚函数没有实现（C++11 以前），强制派生类必须覆盖：

```cpp
class Shape {
public:
    virtual double Area() const = 0;  // 纯虚函数
    virtual ~Shape() = default;
};

class Circle : public Shape {
public:
    explicit Circle(double r) : radius_(r) {}
    double Area() const override {
        return 3.14159265 * radius_ * radius_;
    }
private:
    double radius_;
};
```

含纯虚函数的类是**抽象类**，不能实例化：

```cpp
Shape s;           // 编译错误: 不能实例化抽象类
Shape* p = new Circle(1.0);  // 正确: 可以用指针/引用
```

### 纯虚函数可以有函数体

一个鲜为人知的事实：纯虚函数可以有实现，派生类可以显式调用：

```cpp
class Base {
public:
    virtual void Foo() = 0;
};

void Base::Foo() {
    std::cout << "Base::Foo pure virtual implementation" << std::endl;
}

class Derived : public Base {
public:
    void Foo() override {
        Base::Foo();  // 显式调用基类的纯虚函数实现
        std::cout << "Derived::Foo" << std::endl;
    }
};
```

这个特性常用于提供默认行为，同时强制派生类显式决定是否使用它。

### 纯虚析构函数必须提供函数体

这是一个特例——即使析构函数是纯虚的，也必须提供实现，因为派生类析构时会自动调用基类析构函数：

```cpp
class AbstractBase {
public:
    virtual ~AbstractBase() = 0;  // 声明纯虚析构函数
};

AbstractBase::~AbstractBase() {   // 必须提供实现
    std::cout << "AbstractBase dtor" << std::endl;
}
```

---

## 虚析构函数：多态释放的关键

### 为什么基类析构函数必须是虚函数

这是 C++ 中最重要的规则之一：

```cpp
class Base {
public:
    ~Base() { std::cout << "Base dtor" << std::endl; }  // 非虚析构函数
};

class Derived : public Base {
public:
    ~Derived() { std::cout << "Derived dtor" << std::endl; }
    std::vector<int> data_{1, 2, 3, 4, 5};
};

int main() {
    Base* p = new Derived();
    delete p;  // 只调用 Base::~Base()，Derived::~Derived() 未被调用
    // Derived::data_ 的内存泄漏了
    return 0;
}
```

`delete p` 时，编译器看到 `p` 的静态类型是 `Base*`，而 `~Base()` 不是虚函数，于是**静态绑定**到 `Base::~Base()`——派生类的析构函数被跳过，资源泄漏。

修复方法：给基类析构函数加上 `virtual`：

```cpp
class Base {
public:
    virtual ~Base() = default;  // 虚析构函数
};

class Derived : public Base {
public:
    ~Derived() override { std::cout << "Derived dtor" << std::endl; }
    std::vector<int> data_{1, 2, 3, 4, 5};
};

int main() {
    Base* p = new Derived();
    delete p;  // 正确: 先调用 Derived::~Derived()，再调用 Base::~Base()
    return 0;
}
```

> **规则**：只要类会被多态使用（通过基类指针/引用操作），析构函数**必须**是虚函数。这不是建议，是要求。违反这条规则会导致资源泄漏和未定义行为。

### `override` 和 `final`：现代 C++ 的安全网

C++11 引入了两个关键字，让虚函数的意图更明确：

```cpp
class Base {
public:
    virtual void Foo() {}
    virtual void Bar() final {}  // 不允许派生类再覆盖
};

class Derived : public Base {
public:
    void Foo() override {}   // 编译器检查: Base 中确有虚函数 Foo
    void Bar() override {}   // 编译错误: Base::Bar 是 final
    void Baz() override {}   // 编译错误: Base 中没有虚函数 Baz
};
```

| 关键字 | 作用 | 检查时机 |
|---|---|---|
| `override` | 声明"我要覆盖基类的虚函数" | 编译时检查基类是否存在该虚函数 |
| `final` | 声明"此函数不可被覆盖"或"此类不可被继承" | 编译时阻止覆盖/继承 |

> **实践建议**：派生类中所有覆盖的虚函数都应加 `override`。它能防止拼写错误、签名不匹配等隐蔽 bug。

---

## 多重继承下的 vtable

### 多重继承带来多个 vptr

多重继承时，对象可能包含多个 vptr，每个对应一个基类子对象：

```cpp
class Base1 {
public:
    virtual void Foo() { std::cout << "Base1::Foo" << std::endl; }
    int b1_;
};

class Base2 {
public:
    virtual void Bar() { std::cout << "Base2::Bar" << std::endl; }
    int b2_;
};

class Derived : public Base1, public Base2 {
public:
    void Foo() override { std::cout << "Derived::Foo" << std::endl; }
    void Bar() override { std::cout << "Derived::Bar" << std::endl; }
    int d_;
};
```

```
Derived 对象内存布局 (64 位):
┌──────────────────┐
│ vptr1 → Derived vtable1 (对应 Base1 子对象)
├──────────────────┤
│ b1_              │
├──────────────────┤
│ vptr2 → Derived vtable2 (对应 Base2 子对象)
├──────────────────┤
│ b2_              │
├──────────────────┤
│ d_               │
└──────────────────┘
```

```cpp
int main() {
    std::cout << sizeof(Base1) << std::endl;   // 16 (vptr + int + padding)
    std::cout << sizeof(Base2) << std::endl;   // 16
    std::cout << sizeof(Derived) << std::endl; // 32 (两个 vptr + 三个 int + padding)
    return 0;
}
```

### 指针调整（Thunk）

当 `Base2*` 指针指向 `Derived` 对象时，编译器需要调整指针偏移：

```cpp
Derived d;
Base2* p2 = &d;  // p2 不指向 d 的起始地址，而是指向 Base2 子对象的起始地址
// 编译器实际做了: p2 = (Base2*)((char*)&d + sizeof(Base1 子对象))
```

这种调整通常通过 **thunk**（一小段跳转代码）实现。vtable 中会记录必要的 `this` 指针偏移量，确保虚函数调用时 `this` 指针指向正确的子对象。

### 虚继承（Virtual Inheritance）

菱形继承是多重继承的经典问题：

```cpp
class Animal { public: virtual void Speak() {} int age_; };
class Mammal : public Animal {};
class Bird : public Animal {};
class Bat : public Mammal, public Bird {};  // Bat 有两份 Animal
```

```
Bat 对象 (无虚继承):
┌──────────────────┐
│ Mammal 子对象     │
│   └─ Animal 子对象│  ← 第一份 age_
├──────────────────┤
│ Bird 子对象       │
│   └─ Animal 子对象│  ← 第二份 age_ (二义性!)
└──────────────────┘
```

虚继承解决这个问题：

```cpp
class Animal { public: virtual void Speak() {} int age_; };
class Mammal : virtual public Animal {};
class Bird  : virtual public Animal {};
class Bat : public Mammal, public Bird {};  // 只有一份 Animal
```

虚继承通过额外的 **vbase offset**（记录虚基类子对象的偏移量）来实现共享。代价是更复杂的内存布局和一次额外的间接寻址来访问虚基类成员。

> **实践建议**：避免复杂的继承层次。能用组合就别用继承，能用单继承就别用多重继承。虚继承是最后的手段，不是首选。

---

## 性能考量：虚函数的代价

### 时间开销

虚函数调用的成本比普通函数多几步：

| 步骤 | 普通函数 | 虚函数 |
|---|---|---|
| 绑定时机 | 编译时 | 运行时 |
| 寻址方式 | 直接地址（可能被内联） | vptr → vtable → 函数指针（间接） |
| 是否可内联 | 通常可以 | 通常不可以（编译时不知道调用目标） |

具体来说，一次虚函数调用需要：

1. 读取对象的 vptr（可能触发缓存未命中）
2. 从 vtable 中读取函数指针（又一次内存访问）
3. 间接跳转到函数地址

### 空间开销

| 开销 | 每个类 | 每个对象 |
|---|---|---|
| vtable | 一份（虚函数数量 × 指针大小） | 无 |
| vptr | 无 | 一个指针（8 字节 / 64 位系统） |

### 什么时候虚函数的性能影响值得关注

**大多数情况下不需要关心。** 一次虚函数调用的额外开销大约是 2-3 纳秒。在现代 CPU 的分支预测器面前，间接调用的代价已经被大幅降低。

但在以下场景中需要谨慎：

```cpp
// Bad: 紧凑循环中的虚函数调用
for (int i = 0; i < 10000000; ++i) {
    obj->VirtualMethod();  // 每次迭代都有间接调用开销
}

// Good: 数据导向设计，减少虚函数调用频率
std::vector<ConcreteA> batch_a;
std::vector<ConcreteB> batch_b;
for (auto& item : batch_a) { item.Method(); }  // 直接调用，可内联
for (auto& item : batch_b) { item.Method(); }  // 直接调用，可内联
```

> **核心原则**：不要为了"性能"而避免虚函数——要为了清晰的设计而使用它。只有在性能分析（profiling）确认虚函数是瓶颈时，才考虑优化。

---

## 替代方案与 C++ 的演变

### `std::function`：类型擦除的多态

C++11 引入的 `std::function` 提供了不依赖继承的多态：

```cpp
#include <functional>
#include <iostream>

void ProcessEvent(const std::function<void(int)>& handler) {
    handler(42);
}

int main() {
    // 不需要公共基类，不需要虚函数
    ProcessEvent([](int val) {
        std::cout << "Lambda handler: " << val << std::endl;
    });
    return 0;
}
```

### `std::variant` + `std::visit`：闭集多态

C++17 的 `std::variant` 适用于类型集合已知且封闭的场景：

```cpp
#include <variant>
#include <iostream>

struct Circle { double radius; };
struct Rectangle { double width, height; };

using Shape = std::variant<Circle, Rectangle>;

double Area(const Shape& s) {
    return std::visit([](const auto& shape) -> double {
        using T = std::decay_t<decltype(shape)>;
        if constexpr (std::is_same_v<T, Circle>) {
            return 3.14159265 * shape.radius * shape.radius;
        } else {
            return shape.width * shape.height;
        }
    }, s);
}
```

### CRTP：编译时多态

奇异递归模板模式（Curiously Recurring Template Pattern）在编译时实现多态，零运行时开销：

```cpp
template <typename Derived>
class Base {
public:
    void Interface() {
        static_cast<Derived*>(this)->Implementation();
    }
};

class Concrete : public Base<Concrete> {
public:
    void Implementation() {
        std::cout << "Concrete implementation" << std::endl;
    }
};
```

| 方案 | 多态时机 | 开销 | 适用场景 |
|---|---|---|---|
| 虚函数 | 运行时 | vptr + 间接调用 | 开放类型层次，经典 OOP |
| `std::function` | 运行时 | 堆分配（小对象优化除外） | 回调、事件处理 |
| `std::variant` | 编译时 | 无运行时多态开销 | 类型集合封闭且已知 |
| CRTP | 编译时 | 零开销 | 性能敏感，类型在编译时确定 |

---

## 实战场景

### 场景一：插件架构

虚函数最常见的应用场景——定义接口，运行时加载不同实现：

```cpp
// 接口定义（头文件）
class IRenderer {
public:
    virtual ~IRenderer() = default;
    virtual bool Initialize() = 0;
    virtual void Draw(const Scene& scene) = 0;
    virtual void Shutdown() = 0;
};

// OpenGL 实现
class GlRenderer : public IRenderer {
public:
    bool Initialize() override { /* OpenGL 初始化 */ return true; }
    void Draw(const Scene& scene) override { /* OpenGL 渲染 */ }
    void Shutdown() override { /* 清理 OpenGL 资源 */ }
};

// Vulkan 实现
class VkRenderer : public IRenderer {
public:
    bool Initialize() override { /* Vulkan 初始化 */ return true; }
    void Draw(const Scene& scene) override { /* Vulkan 渲染 */ }
    void Shutdown() override { /* 清理 Vulkan 资源 */ }
};

// 使用方——无需知道具体实现
void RunApplication(IRenderer* renderer) {
    renderer->Initialize();
    Scene scene;
    renderer->Draw(scene);
    renderer->Shutdown();
}
```

### 场景二：策略模式

用虚函数实现运行时可替换的算法策略：

```cpp
class SortStrategy {
public:
    virtual ~SortStrategy() = default;
    virtual void Sort(std::vector<int>& data) = 0;
};

class QuickSort : public SortStrategy {
public:
    void Sort(std::vector<int>& data) override { /* 快排实现 */ }
};

class MergeSort : public SortStrategy {
public:
    void Sort(std::vector<int>& data) override { /* 归并排序实现 */ }
};

class Sorter {
public:
    explicit Sorter(SortStrategy* strategy) : strategy_(strategy) {}
    void SetStrategy(SortStrategy* strategy) { strategy_ = strategy; }
    void Sort(std::vector<int>& data) { strategy_->Sort(data); }
private:
    SortStrategy* strategy_;
};
```

### 场景三：模板方法模式

基类定义算法骨架，派生类填充具体步骤：

```cpp
class DataProcessor {
public:
    // 模板方法：定义算法骨架（非虚函数，不希望被覆盖）
    void Process() {
        ReadData();
        Validate();
        Transform();
        WriteResult();
    }

    virtual ~DataProcessor() = default;

protected:
    virtual void ReadData() = 0;
    virtual void Validate() = 0;
    virtual void Transform() = 0;
    virtual void WriteResult() = 0;
};

class CsvProcessor : public DataProcessor {
protected:
    void ReadData() override { /* 读取 CSV */ }
    void Validate() override { /* 校验 CSV 格式 */ }
    void Transform() override { /* CSV 数据转换 */ }
    void WriteResult() override { /* 写入结果 */ }
};
```

---

## 常见陷阱

### 陷阱一：在构造/析构中调用虚函数

```cpp
class Base {
public:
    Base() { Init(); }  // 不会多态！调的是 Base::Init
    virtual void Init() { std::cout << "Base Init" << std::endl; }
};

class Derived : public Base {
public:
    void Init() override { std::cout << "Derived Init" << std::endl; }
};
```

在基类构造函数中调用虚函数，绑定的是基类版本。这是**设计决策**，不是 bug——派生类部分尚未初始化，调用派生类的虚函数可能访问未初始化的成员。

**解决方案**：使用 `init()` 方法，构造完成后由调用方显式调用，或使用工厂模式。

### 陷阱二：默认参数与虚函数

```cpp
class Base {
public:
    virtual void Foo(int x = 10) {
        std::cout << "Base::Foo " << x << std::endl;
    }
};

class Derived : public Base {
public:
    void Foo(int x = 20) override {
        std::cout << "Derived::Foo " << x << std::endl;
    }
};

int main() {
    Base* p = new Derived();
    p->Foo();  // 输出 "Derived::Foo 10"，不是 20！
    delete p;
    return 0;
}
```

**默认参数是静态绑定的**——编译器根据指针的静态类型（`Base*`）决定默认值。函数体是动态绑定的，但默认参数是静态绑定的。

> **最佳实践**：虚函数不要使用默认参数。如果必须区分行为，使用重载。

### 陷阱三：隐藏基类函数

```cpp
class Base {
public:
    virtual void Foo(int x) { std::cout << "Base::Foo(int)" << std::endl; }
};

class Derived : public Base {
public:
    void Foo(int x) override { std::cout << "Derived::Foo(int)" << std::endl; }
    void Foo(double x) { std::cout << "Derived::Foo(double)" << std::endl; }
    // 注意: 上面这行隐藏了 Base::Foo(int) 的所有重载版本
};

int main() {
    Derived d;
    d.Foo(42);     // Derived::Foo(int) — 正确
    d.Foo(3.14);   // Derived::Foo(double) — 正确

    Base* p = &d;
    p->Foo(42);    // Derived::Foo(int) — 多态，正确
    // p->Foo(3.14); // 也是 Base::Foo(int)，3.14 截断为 3
    return 0;
}
```

> **最佳实践**：派生类中覆盖虚函数时，使用 `override` 关键字。需要暴露基类重载时，使用 `using Base::Foo;`。

---

## 总结

| 概念 | 一句话 |
|---|---|
| 虚函数 | 用 `virtual` 声明的成员函数，支持通过基类指针/引用实现运行时多态 |
| vtable | 编译器为含虚函数的类生成的函数指针数组，每个类一份 |
| vptr | 每个含虚函数的对象内部的隐藏指针，指向所属类的 vtable |
| 动态绑定 | 运行时通过 vptr → vtable → 函数指针的间接调用机制 |
| 虚析构函数 | 多态使用时基类析构函数必须为虚，否则派生类析构不会被调用 |
| 纯虚函数 | `= 0` 声明，使类变为抽象类，强制派生类实现 |
| `override` | 编译期检查是否正确覆盖基类虚函数 |
| `final` | 阻止函数被进一步覆盖或类被进一步继承 |

虚函数是 C++ 实现运行时多态的核心机制，理解 vtable 和 vptr 的工作原理有助于写出更健壮的代码，也有助于在性能敏感场景做出正确的设计决策。记住几个核心原则：

1. **多态使用的类，析构函数必须是虚函数**——没有例外。
2. **派生类覆盖的虚函数，一律加 `override`**——让编译器帮你检查。
3. **构造和析构中不要调用虚函数期望多态**——此时对象还不完整。
4. **虚函数的默认参数是静态绑定的**——避免在虚函数中使用默认参数。
5. **先设计，再考虑性能**——虚函数的开销在绝大多数场景下可以忽略。
