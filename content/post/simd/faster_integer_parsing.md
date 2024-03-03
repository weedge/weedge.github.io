---
author: "weedge"
title: "译：更快的字符串转整数"
date: 2023-11-30T10:26:23+08:00
tags: [
	"cpu", "simd", "integer parsing"
]
categories: [
	"技术",
]
---

​                        ![](https://github.com/weedge/mypic/raw/master/simd/faster_integer_parsing/0.jpeg)

## 导读

字符串转换成整数，或者浮点类型数据，是在编程中经常遇到的问题，各种语言的标准库中会有实现，本文通过一个常见问题场景，来研究优化如何使用cpu 硬件SIMD指令集，并结合编译器在 log(n) 时间内完成此类parse操作；由于最终的优化需要结合对应cpu arch的指令集，这里硬件平台cpu为Intel x86，整数类型以uint64_t为例，最大2^64-1 20个字符表示。目的：结合场景优化思路(以小见大)，熟悉下Intel cpu simd相关指令的使用。常见场景： [simdjson](https://github.com/simdjson/simdjson)  (PS: 不因过早优化，在对应场景下整体稳定性和优化成本/收益上折中)

<!--more-->

## 问题

假设有一些基于网络(socket)的传输协议字符串或包含微秒时间戳的文件。需要尽快解析这些时间戳。也许是 json，也许是 csv 文件，也许是其他定制的文件。它有 16 个字符长(8*16)，这也适用于信用卡号码。

```
timestamp,event_id
1585201087123567,a
1585201087123585,b
1585201087123621,c
```

实现类似这样的功能：

```c++
uint64_t parse(std::string_view s);//c++17
```

## 标准库中的方法

在c/c++中可以调用标准库方法进行解析，比如 c中 `atoll`相关函数； c++中 [`std::atoll`](https://en.cppreference.com/w/cpp/string/byte/atoi)一个从 C 继承的函数； [`std::stringstream`](https://en.cppreference.com/w/cpp/io/basic_stringstream) 流方式处理；标准C++17 引入的  [`<charconv>`](https://en.cppreference.com/w/cpp/header/charconv)头文件中的方法；以及boost库 [`boost::spirit::qi`](https://www.boost.org/doc/libs/1_73_0/libs/spirit/doc/html/spirit/qi/reference/basics.html)中的方法。比如 使用from_chars 方法

```c++
inline std::uint64_t parse_char_conv(std::string_view s) noexcept
{
  std::uint64_t result = 0;
  auto [ptr, ec] = std::from_chars(s.data(), s.data() + s.size(), result);
  if (ec != std::errc()) {} // I have an error !
  return result;
}

```



## 优化思路

### 常规线性遍历

将展开的解决方案中的操作绘制为一棵树，以将“1234”解析为 32 位整数的简化示例为例：

![](https://github.com/weedge/mypic/raw/master/simd/faster_integer_parsing/1.png)

*“1234”操作的展开解图*

可以看到，乘法和加法的数量与字符数量成线性关系。很难看出如何改进这一点，因为每次乘法都是通过不同的因子（所以不能“一次性”相乘），并且在一天结束时需要将所有中间结果相加。

```c++
inline std::uint64_t parse_naive(std::string_view s) noexcept
{
  std::uint64_t result = 0;
  for(char digit : s)
  {
    result += result * 10 + digit - '0';
  }
  return result;
}

inline std::uint64_t parse_unrolled(std::string_view s) noexcept
{
  std::uint64_t result = 0;

  result += (s[0] - '0') * 1000000000000000ULL;
  result += (s[1] - '0') * 100000000000000ULL;
  result += (s[2] - '0') * 10000000000000ULL;
  result += (s[3] - '0') * 1000000000000ULL;
  result += (s[4] - '0') * 100000000000ULL;
  result += (s[5] - '0') * 10000000000ULL;
  result += (s[6] - '0') * 1000000000ULL;
  result += (s[7] - '0') * 100000000ULL;
  result += (s[8] - '0') * 10000000ULL;
  result += (s[9] - '0') * 1000000ULL;
  result += (s[10] - '0') * 100000ULL;
  result += (s[11] - '0') * 10000ULL;
  result += (s[12] - '0') * 1000ULL;
  result += (s[13] - '0') * 100ULL;
  result += (s[14] - '0') * 10ULL;
  result += (s[15] - '0');

  return result;
}
```



---

### 字节交换(byteswap)

然而，它仍然非常有规律。一方面，字符串中的第一个字符乘以最大的因子，因为它是最高有效的数字。

> 在小端机器（如 x86）上，整数的第一个字节包含最低有效数字，而字符串中的第一个字节包含最高有效数字。

![](https://github.com/weedge/mypic/raw/master/simd/faster_integer_parsing/2.png)

*将字符串视为整数，可以通过更少的操作更接近最终的解析状态 - 十六进制表示(机器只识别是01)*

现在，要将字符串的字节重新解释为整数，需要使用 `std::memcpy`（[以避免严格别名违规](https://blog.regehr.org/archives/1307)），并且编译器本身`__builtin_bswap64`来交换一条指令中的字节。

```c++
template <typename T>
inline T get_zeros_string() noexcept;

template <>
inline std::uint64_t get_zeros_string<std::uint64_t>() noexcept
{
  std::uint64_t result = 0;
  constexpr char zeros[] = "00000000";
  std::memcpy(&result, zeros, sizeof(result));
  return result;
}

// 64 = 8*8
inline std::uint64_t parse_8_chars(const char* string) noexcept
{
  std::uint64_t chunk = 0;
  std::memcpy(&chunk, string, sizeof(chunk));
  chunk = __builtin_bswap64(chunk - get_zeros_string<std::uint64_t>());
  return chunk;
}
```

这里使用了内置的64位swap进行反转

------

### 分而治之

从上一步中，最终得到一个整数，其位表示形式将每个数字放置在单独的字节中。即，尽管一个字节8位最多可以表示 256 个值(0~2^8-1)，但整数的每个字节中都有值 0-9。它们也采用正确的小端顺序。现在只需要以某种方式将它们“粉碎”在一起即可。

知道线性执行会太慢，下一个可能性是什么？ **O(log(n))**！需要一步将每个相邻数字组合成一对，然后将每对数字组合成四个一组，依此类推，直到得到整个整数。

[reddit 上的 Sopel97](https://www.reddit.com/r/cpp/comments/gr18ig/faster_integer_parsing/frx9agb) 指出 byteswap 不是必需的。无论哪种方式都可以组合相邻数字 - 它们的顺序并不重要。我意识到它有助于我获得下一个见解，但可以在最终代码中省略。

> 关键是同时处理相邻的数字。这允许操作树在 O(log(n)) 时间内运行。

这涉及将偶数索引数字乘以 10 的幂并保留奇数索引数字。这可以通过位掩码来选择性地应用操作来完成

![](https://github.com/weedge/mypic/raw/master/simd/faster_integer_parsing/3.png)

通过使用位掩码，我们可以一次对多个数字应用运算，将它们组合成一个更大的组

让`parse_8_chars`通过使用这个掩码技巧来完成之前开始的函数。作为屏蔽的一个巧妙的副作用，不需要减去 `'0'`，因为它会被屏蔽掉。

```c++
inline std::uint64_t parse_8_chars(const char* string) noexcept
{
  std::uint64_t chunk = 0;
  std::memcpy(&chunk, string, sizeof(chunk));

  // 1-byte mask trick (works on 4 pairs of single digits)
  std::uint64_t lower_digits = (chunk & 0x0f000f000f000f00) >> 8;
  std::uint64_t upper_digits = (chunk & 0x000f000f000f000f) * 10;
  chunk = lower_digits + upper_digits;

  // 2-byte mask trick (works on 2 pairs of two digits)
  lower_digits = (chunk & 0x00ff000000ff0000) >> 16;
  upper_digits = (chunk & 0x000000ff000000ff) * 100;
  chunk = lower_digits + upper_digits;

  // 4-byte mask trick (works on pair of four digits)
  lower_digits = (chunk & 0x0000ffff00000000) >> 32;
  upper_digits = (chunk & 0x000000000000ffff) * 10000;
  chunk = lower_digits + upper_digits;

  return chunk;
}
```



------

### 组合

把它们放在一起，为了解析 16 位整数，将它分成两个 8 字节的块，运行`parse_8_chars`刚刚编写的代码，并对它进行基准测试！

```c++
inline std::uint64_t parse_trick(std::string_view s) noexcept
{
  std::uint64_t upper_digits = parse_8_chars(s.data());
  std::uint64_t lower_digits = parse_8_chars(s.data() + 8);
  return upper_digits * 100000000 + lower_digits;
}

static void BM_trick(benchmark::State& state) {
  for (auto _ : state) {
    benchmark::DoNotOptimize(parse_trick(example_stringview));
  }
}
```

还不错，将展开循环(unrolled)[基准测试](https://quick-bench.com/q/PJAjDeGoSS_OsTrSdtPq1alye34)降低了近 50%左右；（注：这个和编译器优化相关，本文采用的是gcc 9.0 版本，高版本中unrolled版本指令有所优化，autosimd）尽管如此，感觉就像正在手动执行一堆屏蔽和元素操作。也许可以让 CPU SIMD指令集来进一优化，将指令存入更宽的寄存器，较少指令执行次数和执行周期。

------

### SIMD

有以下主要优化：

- 同时组合数字组以实现 O(log(n)) 时间

还有一个 16 个字符或 128 位字符串需要解析 - 可以使用 SIMD 吗？当然可以！[SIMD 代表单指令多数据，](https://en.wikipedia.org/wiki/SIMD)Intel 和 AMD CPU 均支持 SSE 和 AVX 指令，并且它们通常适用于更宽的寄存器(128bit i)。

使用[Intel 内部函数指南](https://software.intel.com/sites/landingpage/IntrinsicsGuide/)来为正确的 SIMD CPU 指令找到正确的编译器内部函数。

首先设置 16 个字节中每个字节的数字：

```c++
inline std::uint64_t parse_16_chars(const char* string) noexcept
{
  //__m128i  
  // This intrinsic may perform better than _mm_loadu_si128 when the data crosses a cache line boundary
  auto chunk = _mm_lddqu_si128(reinterpret_cast<const __m128i*>(string));
  auto zeros =  _mm_set1_epi8('0');
  chunk = chunk - zeros;
  
  // ...
}
```

现在，最引人注目的是`madd`功能。这些 SIMD 函数的作用与使用位掩码技巧所做的完全一样 - 它们采用宽寄存器，将其解释为较小整数的向量，将每个乘数乘以给定的乘数，并将相邻的乘数加在一起形成更宽整数的向量。全部在一个指令中！

作为获取每个字节，将奇数乘以 10 并将相邻对加在一起的示例，可以使用 [`_mm_maddubs_epi16`](https://www.felixcloutier.com/x86/pmaddubsw)

```c++
// The 1-byte "trick" in one instruction
const auto mult = _mm_set_epi8(
  1, 10, 1, 10, 1, 10, 1, 10, 1, 10, 1, 10, 1, 10, 1, 10
);
chunk = _mm_maddubs_epi16(chunk, mult);
```

还有另一条用于 2 字节技巧的指令`_mm_maddubs_epi16`，但不幸的是找不到用于 4 字节技巧的指令`_mm_maddubs_epi32`木有 - 这需要两条指令。这是完成的`parse_16_chars`函数：

```C++
inline std::uint64_t parse_16_chars(const char* string) noexcept
{
  auto chunk = _mm_lddqu_si128(reinterpret_cast<const __m128i*>(string));
  auto zeros =  _mm_set1_epi8('0');
  chunk = chunk - zeros;

  {
    const auto mult = _mm_set_epi8(
      1, 10, 1, 10, 1, 10, 1, 10, 1, 10, 1, 10, 1, 10, 1, 10
    );
    chunk = _mm_maddubs_epi16(chunk, mult);
  }
  {
    const auto mult = _mm_set_epi16(1, 100, 1, 100, 1, 100, 1, 100);
    chunk = _mm_madd_epi16(chunk, mult);
  }
  {
    chunk = _mm_packus_epi32(chunk, chunk);
    const auto mult = _mm_set_epi16(0, 0, 0, 0, 1, 10000, 1, 10000);
    chunk = _mm_madd_epi16(chunk, mult);
  }

  return ((chunk[0] & 0xffffffff) * 100000000) + (chunk[0] >> 32);
}
```

**0.75纳秒**！哇哦。

在实际生产环境中，需要对输入验证或长度检查；例如如下naive代码：(加上__device__可在gpu kernel上运行)

```c
static int h_atoi(const char* src) {
    int s = 0;
    bool isMinus = false;

    while (*src == ' ') {
        src++;
    }

    if (*src == '+' || *src == '-') {
        if (*src == '-') {
            isMinus = true;
        }
        src++;
    } else if (*src < '0' || *src > '9') {
        s = 2147483647;
        return s;
    }

    while (*src != '\0' && *src >= '0' && *src <= '9') {
        s = s * 10 + *src - '0';
        src++;
    }
    return s * (isMinus ? -1 : 1);
}
```

本文介绍的方法适用于固定长度的整数场景。一方面，当知道整数很长时，这可以用作“快速路径”，而在其他情况下则可以回退到简单循环。其次，通过使用一些更聪明的 SIMD 指令，可以在 2 纳秒内运行一些东西(甚至更快)，并完成验证和长度检查。比如simdjson场景。

## 基准测试

使用[Google Benchmark](https://github.com/google/benchmark)来衡量性能，并获得基线，与将最终结果直接加载到寄存器中进行比较 - 即不涉及实际解析。

运行基准测试！代码在这里并不重要，它只是显示正在进行基准测试的内容。

最终结果： https://quick-bench.com/q/NlmsLut8ol_JGwurPfKG22n2Mbs

注： boost库不是标准库，未显示在结果中，可以在本机上运行基准测试。



# 使用 AVX-512 快速解析整数

最近的英特尔处理器有新的指令 AVX-512，它可以一次处理多个字节并进行屏蔽，以便您可以仅选择一系列数据。

我假设您知道数字序列的开头和结尾。以下带有 AVX-512 内在函数的代码执行以下操作：

1. 计算以字节为单位的跨度 (digit_count)，
2. 如果有超过 20 个字节，就知道该整数太大，无法容纳 64 位整数，
3. 计算一个“掩码”：一个 32 位值，只有最高有效的 digital_count 位设置为 1，
4. 将 ASCII 或 UTF-8 字符串加载到 256 位寄存器中，
5. 减去字符值“0”以获得 0 到 9 之间的值（数字值），
6. 检查某个值是否超过 9，在这种情况下有一个非数字字符。

```c++
size_t digit_count = size_t(end - start);
// if (digit_count > 20) { error ....}
const simd8x32 ASCII_ZERO = _mm256_set1_epi8('0');
const simd8x32 NINE = _mm256_set1_epi8(9);
uint32_t mask = uint32_t(0xFFFFFFFF) << (start - end + 32);
auto in = _mm256_maskz_loadu_epi8(mask, end - 32);
auto base10_8bit = _mm256_maskz_sub_epi8(mask, in, ASCII_ZERO);
auto nondigits = _mm256_mask_cmpgt_epu8_mask(mask, base10_8bit, NINE);
if (nondigits) {
    // there is a non-digit
}
```

这是使用 AVX-512 功能的关键步骤。之后，对于熟悉传统 x64 处理器上的高级 Intel 内在函数的人来说，可以使用“老式”处理……大多数情况下，只需乘以 10、乘以 100、乘以 100000 即可创建四个 32 位值：第一个对应于最低有效的 8 个 ASCII 字节，第二个到下一个最高有效的 8 个 ASCII 字节，以及最多 4 个最高有效字节。当数字为 8 位或更少时，只有其中一个单词相关；当数字为 16 位或更少时，前两个单词有意义。总是浪费一个由零组成的 32 位值。代码如下：

```c++
auto DIGIT_VALUE_BASE10_8BIT = _mm256_set_epi8(1, 10, 1, 10, 1, 10, 1, 10,
                                               1, 10, 1, 10, 1, 10, 1, 10,
                                               1, 10, 1, 10, 1, 10, 1, 10,
                                               1, 10, 1, 10, 1, 10, 1, 10);
auto DIGIT_VALUE_BASE10E2_8BIT = _mm_set_epi8(1, 100, 1, 100, 1, 100, 1, 100, 1, 100, 1, 100, 1, 100, 1, 100);
auto DIGIT_VALUE_BASE10E4_16BIT = _mm_set_epi16(1, 10000, 1, 10000, 1, 10000, 1, 10000);
auto base10e2_16bit = _mm256_maddubs_epi16(base10_8bit, DIGIT_VALUE_BASE10_8BIT);
auto base10e2_8bit = _mm256_cvtepi16_epi8(base10e2_16bit);
auto base10e4_16bit = _mm_maddubs_epi16(base10e2_8bit, DIGIT_VALUE_BASE10E2_8BIT);
auto base10e8_32bit = _mm_madd_epi16(base10e4_16bit, DIGIT_VALUE_BASE10E4_16BIT);
```

[c++代码实现](https://github.com/lemire/Code-used-on-Daniel-Lemire-s-blog/tree/master/2023/09/22)，并使用GCC12编译。在 Ice Lake 服务器上运行基准测试。使用随机 32 位整数进行测试。AVX-512 的速度是标准方法`std::from_chars`的两倍多。

| AVX-512         | 1.8GB/秒 | 57 指令/数量   | 17 周期/次数 |
| --------------- | -------- | -------------- | ------------ |
| std::from_chars | 0.8GB/秒 | 128条指令/数量 | 39 周期/次数 |

目前的比较并不完全公平，因为 AVX-512 函数假设它知道数字序列的开头和结尾。

假设正在循环内按顺序解析数字，可以通过使用内联函数来提高性能，使其达到 2.3 GB/s，性能提升 30%。

原始代码将返回奇特的 std::Optional 值，但 GCC 受到负面影响，因此我将函数签名更改为更常规。甚至，在我的测试中，与 GCC 相比，LLVM/clang 稍微快一些。

**NOTE:** 

这里没有进行传统的解析，这涉及从左到右（最重要到最不重要）的解析。使这项工作有效的原因是“右对齐”SIMD 寄存器中的数字（将最低有效数字放入 SIMD 寄存器的最高有效字节中）。

这与传统的从左到右的数字解析器形成对比：通过知道数字的最低有效数字在哪里，我们确切地知道每个数字的价值（最右边的是数字 10^0，倒数第二个数字是数字 10^1，等等） ……）。我们不再需要执行传统的“读取下一个数字，将总数乘以 10，读取下一个数字并将其添加”，这会产生数字之间的数据依赖性。

因为支持最多 20 个字符数字(uint64_t max 2^64-1)，20*8=160位，所以从支持 32 字节寄存器的SIMD指令集开始（尽管在步骤 2 中很快缩小到 16 字节寄存器，因为 2 位数字仍然可以容纳在一个字节中）：

```
String: 1234567890
SIMD step 1 (8-bit x 32): 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 1 2 3 4 5 6 7 8 9 0
SIMD step 2 (8-bit x 16): 00 00 00 00 00 00 00 00 00 00 00 12 34 56 78 90
SIMD step 3 (16-bit x 8): 0000 0000 0000 0000 0000 0012 3456 7890
SIMD step 4 (32-bit x 4): 00000000 00000000 00000012 34567890
```

之后，提取 32 位数字，将它们乘以 10^16、10^8 和 10^0，然后将它们相加以获得 64 位结果（在本例中为 1234567890）。

值得注意的一点是：使用 AVX-512，还可以同时解析 2 个数字（每个数字字符串可由32字节(256位)寄存器存放处理），而无需修改算法。（如果知道数字字符串都是 8 字节(64位)或更少，可以解析更多！）如果想要 32 位数字，您可以一次解析 4 个！16位，8个！

请注意，如果加载 32 个字节（假设填充，对齐操作），查找非数字，并使用它来查找末尾，然后使用“右对齐”所有数字，则可以在不知道数字的完整大小的情况下执行此操作字节移位/洗牌(shift/shuffle)。

## Reference

1. https://kholdstare.github.io/technical/2020/05/26/faster-integer-parsing.html
2. https://www.reddit.com/r/cpp/comments/gr18ig/faster_integer_parsing/
3. https://quick-bench.com/q/-E78g-dkbnDvKlGVkgZc0owGrn0  https://godbolt.org/z/czvqh6v1a
4. **http://0x80.pl/articles/simd-parsing-int-sequences.html** [github](https://github.com/WojciechMula/parsing-int-series)
5. https://lemire.me/blog/2023/09/22/parsing-integers-quickly-with-avx-512/
6. https://en.wikichip.org/wiki/x86/avx512_vnni
7. https://blog.regehr.org/archives/1307
8. https://lab.cs.tsinghua.edu.cn/hpc/doc/assignments/5.simd/
9. **https://en.algorithmica.org/hpc/simd/** https://www.youtube.com/watch?v=vIRjSdTCIEU
10. **https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html** [github](https://github.com/intel/optimization-manual)