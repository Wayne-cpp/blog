---
title: "C/C++ memcpy 与 strcpy：语义差异、底层实现与 *cpy 家族全梳理"
date: 2021-07-15T10:00:00+08:00
tags: ["C++", "C", "memcpy", "strcpy", "字符串", "内存操作"]
categories: ["技术"]
summary: "memcpy 和 strcpy 都能复制一段内存，但语义截然不同——一个按字节拷贝，一个按字符串拷贝。本文从接口差异出发，深入 glibc 的底层实现（SSE/AVX 向量化拷贝），并梳理 strncpy、memmove、strcat 等 *cpy 系列函数各自的适用场景与常见陷阱。"
ShowToc: true
---

`memcpy` 和 `strcpy` 是 C 标准库中使用频率最高的两个拷贝函数。初学者经常困惑：都能复制内容，该用哪个？答案是——它们的语义完全不同：`memcpy` 按指定字节数复制原始内存，`strcpy` 按 `\0` 终止符复制字符串。搞混它们，轻则数据截断，重则缓冲区溢出。这篇文章从接口语义出发，讲到 glibc 底层的向量化实现，最后把整个 `*cpy` 家族一次讲清。

## 核心区别：一表看清

| 维度 | `memcpy` | `strcpy` |
|---|---|---|
| 所属头文件 | `<string.h>` | `<string.h>` |
| 拷贝依据 | **字节数**（由调用者指定） | **`\0` 终止符**（自动检测） |
| 适用数据 | 任意二进制数据 | 仅限 C 风格字符串 |
| 遇到 `\0` | **不停**，继续拷贝 | **停止**，拷贝包含 `\0` |
| 参数 | `dest, src, n` | `dest, src` |
| 长度由谁控制 | 调用者 | 字符串内容本身 |
| 源和目的重叠 | **未定义行为** | **未定义行为**（应使用 `memmove`） |
| 返回值 | `dest` 指针 | `dest` 指针 |

```cpp
char src[] = "hello\0world";  // 11 字节，中间有一个 \0

char buf1[16] = {0};
memcpy(buf1, src, 11);   // 拷贝全部 11 字节 → "hello\0world"
// buf1 内容：h e l l o \0 w o r l d \0 \0 \0 \0 \0

char buf2[16] = {0};
strcpy(buf2, src);       // 遇到第一个 \0 停止 → "hello"
// buf2 内容：h e l l o \0 0 0 0 0 0 0 0 0 0 0
```

关键区别一目了然：`strcpy` 碰到 `\0` 就停，`memcpy` 不管内容是什么，死板地拷贝你指定的字节数。

---

## 接口详解

### strcpy

```c
char* strcpy(char* dest, const char* src);
```

**行为**：将 `src` 指向的字符串（包含结尾的 `\0`）复制到 `dest`，直到遇到 `src` 中的第一个 `\0` 为止。

```cpp
char src[] = "hello";
char dest[10];
strcpy(dest, src);
// dest: "hello\0"（6 字节，包含 \0）

// 危险：如果 dest 不够大
char small[3];
strcpy(small, "hello");  // 缓冲区溢出！写入超出 small 边界
```

**致命缺陷**：`strcpy` 不接受目标缓冲区大小参数。如果 `src` 比预期长，就会写入越界——这是 C 历史上最常见的安全漏洞来源之一。这也是为什么很多编码规范（包括 CERT C、MISRA C）禁止使用 `strcpy`。

### memcpy

```c
void* memcpy(void* dest, const void* src, size_t n);
```

**行为**：从 `src` 复制 `n` 个字节到 `dest`，不关心内容是什么。

```cpp
// 复制字符串
char src[] = "hello";
char dest[10];
memcpy(dest, src, 6);  // 包含 \0，共 6 字节

// 复制整数数组
int arr1[] = {1, 2, 3, 4, 5};
int arr2[5];
memcpy(arr2, arr1, sizeof(arr1));  // 20 字节

// 复制结构体
struct Point { double x, y; };
Point a = {1.0, 2.0};
Point b;
memcpy(&b, &a, sizeof(Point));

// 复制任意二进制数据
uint8_t packet[1500];
uint8_t buffer[1500];
memcpy(buffer, packet, sizeof(packet));
```

`memcpy` 的通用性远强于 `strcpy`——它不关心数据类型，只关心字节数。

---

## 底层实现：glibc 如何加速 memcpy

### 朴素实现

最直观的 `memcpy` 实现就是一个字节一个字节地拷贝：

```c
// 朴素版——仅用于理解语义
void* memcpy_naive(void* dest, const void* src, size_t n) {
    char* d = (char*)dest;
    const char* s = (const char*)src;
    while (n--) {
        *d++ = *s++;
    }
    return dest;
}
```

这个版本功能正确，但性能很差——每次只拷贝 1 字节，无法利用 CPU 的宽寄存器。

### glibc 的实际实现策略

glibc 的 `memcpy` 是高度优化的汇编代码，核心策略是**根据数据大小选择不同的拷贝路径**：

```
小尺寸（≤ 16 字节）
  → 直接用 mov 指令（1~2 条）一次性搬完

中等尺寸（17 ~ 几百字节）
  → 用 SSE/AVX 寄存器（128/256 位）每次拷贝 16/32 字节
  → 循环展开（loop unrolling）减少分支开销

大尺寸（> 几百字节）
  → AVX-512 或 rep movsb/movsq（取决于 CPU 特性）
  → 某些架构使用 "non-temporal" 写入避免污染 CPU 缓存

未对齐的情况
  → 先处理头部未对齐的几个字节
  → 然后切换到对齐的宽拷贝主循环
```

示意代码（简化）：

```c
// 简化版 glibc memcpy 策略
void* memcpy_optimized(void* dest, const void* src, size_t n) {
    if (n <= 16) {
        // 小块：用 mov 直接复制
        return copy_small(dest, src, n);
    }

    // 处理头部未对齐部分
    size_t head = (size_t)dest & 0xF;  // 对齐到 16 字节边界
    if (head) {
        copy_small(dest, src, head);
        dest += head;
        src  += head;
        n    -= head;
    }

    // 主循环：每次拷贝 16 字节（SSE）
    while (n >= 16) {
        __m128i data = _mm_loadu_si128((__m128i*)src);  // 加载 128 位
        _mm_storeu_si128((__m128i*)dest, data);          // 存储 128 位
        dest += 16;
        src  += 16;
        n    -= 16;
    }

    // 处理尾部剩余
    if (n > 0) {
        copy_small(dest, src, n);
    }

    return dest;
}
```

### ERMS：现代 CPU 的终极加速

Intel 从 Ivy Bridge 开始引入了 **ERMS**（Enhanced REP MOVSB）特性，使得 `rep movsb` 指令在内部由硬件以接近内存带宽极限的速度执行复制。glibc 在检测到 ERMS 支持后，对大块拷贝直接使用 `rep movsb`，无需手动展开循环。

```bash
# 查看 CPU 是否支持 ERMS
$ grep erms /proc/cpuinfo
```

这意味着在现代 CPU 上，`memcpy` 大块数据的性能已经非常接近理论峰值——软件优化的空间越来越小。

### strcpy 的底层

glibc 的 `strcpy` 实现：

```c
// 简化版
char* strcpy(char* dest, const char* src) {
    char* ret = dest;
    while ((*dest++ = *src++) != '\0')
        ;
    return ret;
}
```

实际 glibc 实现也做了向量化优化——先用 SSE 扫描 `src` 中的 `\0` 位置，确定长度后再调用类似 `memcpy` 的向量化拷贝。但从原理上，`strcpy` 必须先扫描找到 `\0`，然后才能开始拷贝，这比 `memcpy` 多了一次遍历（两次 pass）。

---

## *cpy 函数家族全梳理

### 总览

| 函数 | 拷贝依据 | 遇到 `\0` | 源/目重叠 | 适用场景 |
|---|---|---|---|---|
| `memcpy` | 字节数 | 不停 | **UB** | 通用内存复制 |
| `memmove` | 字节数 | 不停 | **安全** | 可能重叠的内存复制 |
| `strcpy` | `\0` 终止符 | 停止 | **UB** | 字符串复制（无长度保护） |
| `strncpy` | 字节数或 `\0` | 停止后补 `\0` | **UB** | 有长度限制的字符串复制 |
| `strcat` | `\0` 终止符 | 停止 | **UB** | 字符串追加 |
| `strncat` | 字节数或 `\0` | 停止后补 `\0` | **UB** | 有长度限制的字符串追加 |

### memcpy vs memmove

`memmove` 是 `memcpy` 的"安全版"——它**保证源和目的内存区域重叠时也能正确工作**。

```cpp
char str[] = "abcdefghij";

// memcpy: 重叠时是 UB，结果不可预测
memcpy(str + 2, str, 5);   // 可能得到 "ababcdehij"，也可能崩溃

// memmove: 重叠安全，结果确定
memmove(str + 2, str, 5);  // 保证得到 "ababcdehij"
```

**实现原理**：`memmove` 在拷贝前先检查 `dest` 和 `src` 的位置关系：
- 如果 `dest < src`：从前往后拷贝（正向）
- 如果 `dest > src`：从后往前拷贝（反向）

这样即使区域重叠，也不会出现源数据被覆盖后才被读取的情况。

```
src:  [a] [b] [c] [d] [e] ...
              ↓ dest 在 src 右侧
              ↓ 从后往前拷贝
src:  ... [c] [d] [e] [f] [g]
      ↓ 从前往后拷贝
      ↑ dest 在 src 左侧
```

> **实践建议**：如果不确定是否重叠，一律用 `memmove`。性能差异在现代 CPU 上极小（glibc 的 `memmove` 同样有向量化优化）。

### strncpy

```c
char* strncpy(char* dest, const char* src, size_t n);
```

行为比大多数人想象的更微妙：

1. 从 `src` 复制字符到 `dest`，直到遇到 `\0` 或已复制 `n` 个字符
2. 如果 `src` 的长度 < `n`：**用 `\0` 填充 `dest` 的剩余空间**
3. 如果 `src` 的长度 ≥ `n`：**`dest` 不会以 `\0` 结尾**

```cpp
char buf[8];

// 情况 1：src 比 n 短
strncpy(buf, "hi", sizeof(buf));
// buf: 'h' 'i' '\0' '\0' '\0' '\0' '\0' '\0'  ← 填充 \0

// 情况 2：src 长度刚好 = n
strncpy(buf, "12345678", sizeof(buf));
// buf: '1' '2' '3' '4' '5' '6' '7' '8'  ← 没有 \0！
// 后续用 strlen(buf) 或 printf(buf) 会越界读取
```

**strncpy 的常见陷阱**：

1. **不保证以 `\0` 结尾** — 这是最常见的误解。如果源字符串长度 ≥ n，`dest` 没有 null 终止符
2. **性能浪费** — 短字符串 + 大 `n` 时，会花大量时间填充 `\0`（对 `memcpy` 来说完全不需要）

```cpp
// strncpy 的性能陷阱
char buf[4096];
strncpy(buf, "hi", sizeof(buf));
// 内部实现会填充 4094 个 \0！
```

> **strncpy 的设计初衷**：它的原始设计用途是填充固定宽度的 UNIX 文件系统路径字段（如 `dirent.d_name`），不是作为"安全的 strcpy"。现代代码中，推荐用 `snprintf` 或 `strlcpy`（非标准但广泛可用）替代。

### snprintf：更安全的字符串拷贝

```c
int snprintf(char* dest, size_t n, const char* format, ...);
```

```cpp
char buf[64];

// 安全的字符串拷贝
snprintf(buf, sizeof(buf), "%s", src_string);
// 保证：buf 总是以 \0 结尾（即使截断）
// 返回值：如果不截断，字符串应有的长度

// 还支持格式化
snprintf(buf, sizeof(buf), "name=%s,age=%d", name, age);
```

`snprintf` 的优点：
- **保证 null 终止** — `dest` 的最后一个字节一定是 `\0`
- **不会过度填充** — 不像 `strncpy` 那样浪费性能填充 `\0`
- **返回截断信息** — 如果返回值 ≥ n，说明内容被截断

### strcat / strncat

```c
char* strcat(char* dest, const char* src);
char* strncat(char* dest, const char* src, size_t n);
```

```cpp
char buf[64] = "hello";

strcat(buf, " world");    // buf: "hello world"
strncat(buf, "!", 1);     // buf: "hello world!"
```

**注意**：`strncat` 的 `n` 是 `src` 中最多追加的字符数，**不包括 `\0`**。`strncat` 保证追加后 `dest` 以 `\0` 结尾。但和 `strcat` 一样，它不检查 `dest` 的总大小，仍然可能溢出。

> **现代替代**：用 `snprintf` 一次性格式化，避免多次 `strcat` 带来的长度计算负担。

---

## 什么时候用哪个

### 决策树

```
需要拷贝数据？
├─ 数据是 C 字符串（以 \0 结尾）？
│  ├─ 知道目标缓冲区大小？
│  │  ├─ 是 → snprintf(dest, sizeof(dest), "%s", src)
│  │  └─ 否 → 修复你的代码，加上大小参数
│  └─ 不知道大小？→ snprintf + 返回值检测截断
│
├─ 数据是任意二进制（可能包含 \0）？
│  ├─ 源和目的可能重叠？
│  │  ├─ 是 → memmove(dest, src, n)
│  │  └─ 否 → memcpy(dest, src, n)
│  └─ 不确定是否重叠？→ memmove（性能差异可忽略）
│
└─ 结构体/数组拷贝？
   └─ memcpy 或直接赋值（C++ 优先用赋值）
```

### C++ 中的最佳实践

在 C++ 中，以上大部分函数都可以用标准库类型替代，**从根本上消除缓冲区溢出的风险**：

```cpp
// 不用 strcpy，用 std::string
std::string s1 = "hello";
std::string s2 = s1;             // 自动管理内存
std::string s3 = s1 + " world";  // 不用 strcat

// 不用 strncpy，用 copy 或 assign
char buf[64];
std::string src = "hello";
size_t len = src.copy(buf, sizeof(buf) - 1);
buf[len] = '\0';

// 不用 memcpy 拷贝结构体，用赋值
struct Point { double x, y; };
Point a = {1.0, 2.0};
Point b = a;    // C++ 自动生成拷贝操作，比 memcpy 更安全

// 不用 memcpy 拷贝容器，用构造/赋值
std::vector<int> v1 = {1, 2, 3};
std::vector<int> v2 = v1;

// 仍需使用 memcpy 的场景：
// 1. 二进制协议/网络包的序列化
// 2. 与 C API 交互
// 3. 性能关键路径上已知大小的内存块拷贝
// 4. 平凡可拷贝类型的批量转移
```

### 仍然需要 memcpy 的场景

```cpp
// 1. 网络协议序列化
#pragma pack(push, 1)
struct PacketHeader {
    uint32_t magic;
    uint16_t version;
    uint16_t length;
};
#pragma pack(pop)

PacketHeader hdr;
hdr.magic   = 0xDEADBEEF;
hdr.version = 1;
hdr.length  = payload.size();

uint8_t send_buf[1500];
memcpy(send_buf, &hdr, sizeof(hdr));  // 将结构体转为字节流
memcpy(send_buf + sizeof(hdr), payload.data(), payload.size());

// 2. 零拷贝缓冲区交换
void SwapBuffers(std::vector<uint8_t>& a, std::vector<uint8_t>& b) {
    size_t min_size = std::min(a.size(), b.size());
    memcpy(a.data(), b.data(), min_size);
}

// 3. 类型擦除的泛型拷贝（序列化框架）
void CopyBytes(void* dest, const void* src, size_t n) {
    memcpy(dest, src, n);  // 类型未知，只能按字节拷贝
}
```

---

## 常见误区与陷阱

### 误区一："memcpy 比 strcpy 快，所以字符串也用 memcpy"

不一定。如果你需要先调用 `strlen` 获取长度再传给 `memcpy`，总开销是 `strlen`（一次遍历）+ `memcpy`（又一次遍历），而 `strcpy` 内部的向量化实现只需一次扫描+拷贝。

```cpp
// 可能更慢
memcpy(dest, src, strlen(src) + 1);  // strlen 遍历一次 + memcpy 遍历一次

// 可能更快
strcpy(dest, src);  // glibc 内部一次向量化扫描+拷贝
```

### 误区二："memcpy 可以处理重叠内存"

**错误。** C 标准明确规定，`memcpy` 的源和目的区域重叠时是**未定义行为**。即使某些实现恰好能工作，也不能依赖。

```cpp
char buf[] = "hello world";
memcpy(buf + 2, buf, 5);  // UB！应使用 memmove
memmove(buf + 2, buf, 5); // 正确
```

### 误区三："strncpy 是安全的 strcpy"

**错误。** `strncpy` 有两个严重问题：
1. 源字符串长度 ≥ n 时，**不追加 `\0`**
2. 源字符串很短时，**浪费大量时间填充 `\0`**

```cpp
char buf[4];
strncpy(buf, "hello", sizeof(buf));
// buf: 'h' 'e' 'l' 'l'  ← 没有 \0！
printf("%s", buf);  // 未定义行为：越界读取
```

### 误区四："C++ 有了 std::string 就不需要了解这些函数"

不完全是。`std::string` 解决了大部分字符串操作问题，但：
- 网络编程、二进制协议仍需 `memcpy`
- 调试第三方 C 库时必须理解 `strcpy`/`strncpy` 的行为
- 面试中这些是高频考点
- 遗留代码维护中大量存在

---

## 总结

| 函数 | 一句话定位 | 使用建议 |
|---|---|---|
| `memcpy` | 按字节数拷贝任意内存 | 源/目不重叠时的通用内存复制 |
| `memmove` | 安全的 `memcpy`（允许重叠） | 不确定是否重叠时首选 |
| `strcpy` | 按字符串拷贝（无长度保护） | **避免使用**，用 `snprintf` 替代 |
| `strncpy` | 有长度限制的字符串拷贝 | **避免使用**，行为不符合直觉 |
| `strcat` | 字符串追加（无长度保护） | **避免使用**，用 `snprintf` 替代 |
| `snprintf` | 格式化写入（保证 null 终止） | C 字符串拷贝的安全首选 |

记住核心原则：**二进制数据用 `memcpy`/`memmove`，字符串操作用 C++ 的 `std::string`。如果必须用 C 风格字符串，选 `snprintf` 而非 `strcpy`/`strncpy`。**
