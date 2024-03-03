---
author: "weedge"
title: "Go tips-笔记: string 36-41 mistakes"
date: 2023-02-14T14:26:23+08:00
tags: [
	"Golang"
]
categories: [
	"技术",
    "Golang"
]


---



## 笔记

在 Go 中，string是一种不可变的数据结构，包含以下内容：

- 指向不可变字节序列的指针，指向一个byte类型的数组
- 此序列中的总字节数

string在Go中的内部结构是`reflect.StringHeader`位于`reflect/value.go`

```go
// StringHeader is the runtime representation of a string.
// It cannot be used safely or portably and its representation may
// change in a later release.
// Moreover, the Data field is not sufficient to guarantee the data
// it references will not be garbage collected, so programs must keep
// a separate, correctly typed pointer to the underlying data.
type StringHeader struct {
	Data uintptr
	Len  int
}
//uintptr  an unsigned integer large enough to store the uninterpreted bits of a pointer value
```

<!--more-->

已通过unsafe.Poniter显示的将 string转换成`reflect.StringHeader` 结构，进而可以获取结构中的Data指正，然后通过unsafe.Poniter显示转成数组，比如[5]byte， 数组大小不一定等于原始string长度，即使越界访问，因为是只读，如果写的话会出现panic，所以可以越界这样获取； 代码如下：

```go
func ViewStringStruct() {
	s := "hello"
	sh := (*reflect.StringHeader)(unsafe.Pointer(&s))
	fmt.Printf("0x%x\\n", sh.Data)
	fmt.Println(sh.Len) // 5
	ptr := unsafe.Pointer(sh.Data)
	//arrPtr := (*[]byte)(ptr) // panic
	//arrPtr := (*[3]byte)(ptr)
	//arrPtr := (*[100]byte)(ptr) // access violation, string just only read, so is ok
	arrPtr := (*[5]byte)(ptr)
	fmt.Println(*arrPtr) // [104 101 108 108 111]
	fmt.Printf("%s\\n", *arrPtr)
	//arrPtr[3] = 100 // panic, string cann't change
}
```

[]byte 和 string的相互转换，在读取字符串的场景下经常使用到：

```go
func String(b []byte) string {
	return *(*string)(unsafe.Pointer(&b))
}
func Str2Bytes(s string) []byte {
	x := (*[2]uintptr)(unsafe.Pointer(&s))
	h := [3]uintptr{x[0], x[1], x[1]}
	return *(*[]byte)(unsafe.Pointer(&h))
}
```

### 36.不理解rune的概念 （重要）

字符集和编码之间的区别：

- 字符集(charset)是一组字符。例如，Unicode 字符集包含 2^21 个字符。
- 编码(encoding)是字符列表的二进制转换。例如，UTF-8 是一种编码标准，能够将所有 Unicode 字符编码为可变字节数（从 1 到 4 字节）。

UTF-8 将字符编码为 1 到 4 个字节，因此最多为 32 位, rune 是 int32的别名；需要清楚以下概念：

- 字符集是一组字符，而编码描述了如何将字符集转换为二进制。
- 在 Go 中，string引用任意字节的不可变切片。
- Go 源代码使用 UTF-8 编码。因此，所有字符串文字都是 UTF-8 字符串。但是因为字符串可以包含任意字节，如果它是从其他地方（不是源代码）获得的，则不能保证它是基于 UTF-8 编码的。
- rune对应于 Unicode 码位(code point)的概念，请参考：[code point](https://en.wikipedia.org/wiki/Code_point)，由单个值表示。
- 使用 UTF-8，可以将 Unicode 码位(code point)编码为 1 到 4 个字节。
- 在 Go 中使用`len`字符串返回字节数，而不是rune数。

如果想准确获取到有符文(rune)字符串的长度，可以使用`utf8.RuneCountInString` 函数

### 37.不准确的字符串迭代

```go
for i := range s {
    fmt.Printf("position %d: %c\\n", i, s[i])
}
```

以上代码，没有遍历每个rune，而是迭代rune的每个起始索引；

如果想遍历字符串的符文(rune)，可以使用`range`直接在字符串上循环，必须记住，**索引对应的不是符文索引，而是符文字节序列的起始索引；**因为一个符文可以由多个字节组成，如果要访问符文本身，应该使用 的值变量`range`，而不是字符串中的索引;

如果想获取第i个字符串的符文(rune)，在大多数情况下应该将字符串转换为一段 runes。

### 38.滥用 trim 函数

Go中的strings包，开发者可能经常混淆使用TrimRight 和 TrimSuffix， 或者 TrimLeft 和 TrimPrefix

TrimRight向后遍历每个符文；如果符文是提供的集合的一部分，则该函数将其删除。如果不是，该函数将停止迭代并返回剩余的字符串。TrimLeft 向前遍历同理。  Trim 函数 两边遍历也是一样。

如果想匹配整体的字符串进行删除的话， 应该使用 TrimSuffix 和 TrimPrefix，如果两边移除分别调用这两个方法。

### 39.优化不足的字符串连接

如果使用 s += str 的方式连接字符串，会有性能问题，因为字符的数据是不可变的，每次字符串 + 连接操作都会重新分配一次内存空间(allocator)；

应该使用 strings.Builder 结构来拼接字符串， Builder结构中有一个byte切片 buf []byte用于数据拼接，WriteString 内部使用append来操作，前面提到append操作会出发自动扩容，这样不用每次拼接的时候分配一次新的内存空间，提高了性能； 如果可以获取到拼接字符串的长度，那就可以直接通过 Grow函数来一次初始化拼接buf byte切片内存空间，这样性能可以得到进一步的提升(与 21. 切片初始化效率低下 分析一样)。类似WriteString方法对应string 类型，还有一下三总方法：

- 字节切片使用`Write`
- 单字节使用`WriteByte`
- 单个符文使用`WriteRune`

当然如果拼接的短字符串就那么几个，则没有必要使用strings.Builder 结构来拼接字符串，性能提升可以忽略，但是代码量可读性方面就降低了，可读性不如使用运算符`+=`或`fmt.Sprintf`。

### 40.无用的字符串转换

比如 string 转[]byte,  []byte 转string， 如果是直接 []byte(string) 或者 string([]byte)，转换都会有额外的内存分配，而且转换后的string是不可变的； 所以在进行字符串操作的时候，尽量都使用[]byte类型，避免转换string带来的额外操作，`strings`包也有替代品包`bytes`，大多数 I/O Buffer 都是操作 `[]byte`，字符串的拼接Builder结构也是对[]byte的操作。

### 41.子字符串和内存泄漏

在 Go 中使用子字符串操作时，字符串的结构可知：

1. 提供的间隔是基于字节数，而不是符文数。
2. 子字符串操作可能会导致内存泄漏，因为生成的子字符串将与初始字符串共享相同的底层数组。

防止这种情况发生的解决方案是手动执行字符串复制，或者从 Go 1.18 开始引入`strings.Clone`。

## 概括

- 了解符文对应于 Unicode 码位(code point)的概念，它可以由多个字节组成，应该是 Go 开发人员准确处理字符串的核心知识的一部分。
- 使用`range`运算符迭代字符串会迭代符文，其索引对应于符文字节序列的起始索引。要访问特定的符文索引（例如第三个符文），请将字符串转换为`[]rune`.
- `strings.TrimRight`/`strings.TrimLeft`删除给定集合中包含的所有向后/向前符文返回，而`strings.TrimSuffix`/`strings.TrimPrefix`删除提供后缀/前缀的字符串`返回`。
- 应该连接字符串列表`strings.Builder`以防止在每次迭代期间分配新字符串。
- 记住`bytes`包提供与包相同的操作`strings`可以帮助避免额外的字节/字符串转换。
- 使用副本而不是子字符串可以防止内存泄漏，因为子字符串操作返回的字符串将由相同的字节数组支持。