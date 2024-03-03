---
author: "weedge"
title: "译：掌握 RAPIDS libcudf 中的字符串转换"
date: 2023-11-07T15:00:23+08:00
tags: [
	"oneday", "gpu", "RAPIDS libcudf", "string"
]
categories: [
	"技术",
]


---

![img](https://github.com/weedge/mypic/raw/master/oneday/mastering-string-transformations-in-rapids-libcudf/1.png)

字符串数据的高效处理对于许多数据科学应用至关重要。为了从字符串数据中提取有价值的信息，[RAPIDS libcudf](https://github.com/rapidsai/cudf)提供了强大的工具来加速字符串数据转换。libcudf 是一个 C++ GPU DataFrame 库，用于加载、连接、聚合和过滤数据。 

在数据科学中，字符串数据代表语音、文本、基因序列、日志记录和许多其他类型的信息。在使用字符串数据进行机器学习和特征工程时，必须经常对数据进行规范化和转换，然后才能将其应用于特定用例。libcudf 提供通用 API 和设备端实用程序，以支持各种自定义字符串操作。

这篇文章演示了如何使用 libcudf 通用 API 巧妙地转换字符串列。您将获得有关如何使用自定义内核和 libcudf 设备端实用程序解锁峰值性能的新知识。这篇文章还向您介绍了如何最好地管理 GPU 内存和高效构建 libcudf 列以加速字符串转换的示例。

(**注**：从文件中获取数据到buffer中，都需要通过字符串处理操作，特别是split操作，如果是数值，需要atoi，atof操作进行数据分析，向量化操作等， 和数据处理打交道的 super 马里奥 应该学会这个工具，这里直接使用底层操作库libcudf；集成的其他语言有java(JNI)和python(cython), 主要是方便和现有 大数据生态打通(会有一些内存方面的性能损耗)，大多是离线处理场景，特别是LLM的预训练场景)

<!--more-->

## 引入字符串Arrow格式

[libcudf 使用Arrow 格式](https://arrow.apache.org/docs/format/Columnar.html#variable-size-binary-layout)将字符串数据存储在设备内存中，该格式将字符串列表示为两个子列：`chars and offsets` 图 1所示。

该`chars`列将字符串数据保存为连续存储在内存中的 UTF-8 编码字符字节。

该`offsets`列包含递增的整数序列，这些整数是标识 chars 数据数组中每个单独字符串的开头的字节位置。最后的偏移量元素是 chars 列中的字节总数。这意味着行中单个字符串的大小`i`定义为 ( `offsets[i+1]-offsets[i])`。

![显示字符串向量 {"this", "is", "a", "column", "of", "strings"} 及其表示为大小为 6 的字符串类型列的示意图，从而产生“偏移量” INT32 类型的子列和 INT8 类型的“字符”子列。](https://github.com/weedge/mypic/raw/master/oneday/mastering-string-transformations-in-rapids-libcudf/2.png)

*图 1. 示意图显示箭头格式如何表示带有`chars`子`offsets`列的字符串列*

## 字符串编辑功能示例

为了说明字符串转换示例，请考虑一个函数，该函数接收两个输入字符串列并生成一个经过编辑的输出字符串列。

输入数据具有以下形式：“name”列包含用空格分隔的名字和姓氏，以及包含“public”或“private”状态的“visibilities”列。 

我们提出了“redact”函数，该函数对输入数据进行操作以生成由姓氏的第一个首字母后跟空格和整个名字组成的输出数据。但是，如果相应的可见性列是“private”，则输出字符串应完全编辑为“X X”。 

![该表显示“编辑”字符串转换示例，该转换接收名称和可见性字符串列作为输入，并接收部分或完全编辑的数据作为输出。](https://github.com/weedge/mypic/raw/master/oneday/mastering-string-transformations-in-rapids-libcudf/3.png)

*表 1.“编辑”字符串转换示例，该转换接收名称和可见性字符串列作为输入，并接收部分或完全编辑的数据作为输出*

## 使用 libcudf API 转换字符串

首先，可以使用[libcudf strings API](https://docs.rapids.ai/api/libcudf/nightly/group__strings__apis.html)完成字符串转换。通用 API 是一个很好的起点，也是比较性能的良好基准。

**API 函数对整个字符串列进行操作，每个函数至少启动一个内核，并为每个字符串分配一个线程。每个线程在 GPU 上并行处理单行数据，并输出单行作为新输出列的一部分**。

要使用通用 API 完成 redact 示例函数，请按照以下步骤操作：

1. 使用以下命令将“visibilities”字符串列转换为布尔列`contains`
2. 每当布尔列中的相应行条目为“false”时，通过复制“X X”，从名称列创建一个新的字符串列
3. 将“redacted”列拆分为名字和姓氏列
4. 将姓氏的第一个字符切片作为姓氏首字母
5. 通过使用空格 (“ “) 分隔符连接最后一个姓名缩写列和第一个姓名列来构建输出列。

```c++
// convert the visibility label into a boolean
auto const visible = cudf::string_scalar(std::string("public"));
auto const allowed = cudf::strings::contains(visibilities, visible);

// redact names 
auto const redaction = cudf::string_scalar(std::string("X X"));
auto const redacted = cudf::copy_if_else(names, redaction, allowed->view());

// split the first name and last initial into two columns
auto const sv = cudf::strings_column_view(redacted->view())
auto const first_last  = cudf::strings::split(sv);
auto const first = first_last->view().column(0);
auto const last  = first_last->view().column(1);
auto const last_initial = cudf::strings::slice_strings(last, 0, 1);  

// assemble a result column
auto const tv = cudf::table_view({last_initial->view(), first});
auto result = cudf::strings::concatenate(tv, std::string(" "));
```

在具有 600K 行数据的 A6000 上，此方法大约需要 3.5 毫秒。此示例使用`contains`,`copy_if_else, split, slice_strings`和`concatenate`来完成自定义字符串转换。[Nsight Systems](https://developer.nvidia.com/nsight-systems)的分析显示该`split`函数花费的时间最长，其次是`slice_strings`和`concatenate`。

图 2 显示了来自 Nsight Systems 的 redact 示例的分析数据，显示了每秒高达约 6 亿个元素的端到端字符串处理。这些区域对应于与每个功能相关的 NVTX 范围。浅蓝色范围对应于 CUDA 内核运行的时间段。

![显示使用 libcudf strings API 实现的 redact 示例的分析数据的水平条形图。 时间线显示了 contains、copy_if_else、split、slice_strings 和 concatenate 在 600K 到 10M 的行数范围内运行。 时间线将内核执行与字符串 API 函数重叠。 ](https://github.com/weedge/mypic/raw/master/oneday/mastering-string-transformations-in-rapids-libcudf/4.png)

*图 2. 对来自 Nsight Systems 的 redact 示例的数据进行分析*

## 使用自定义内核转换字符串

libcudf strings API 是一个快速高效的字符串转换工具包，但有时性能关键的函数需要运行得更快。libcudf 字符串 API 中额外工作的一个关键来源是为每个 API 调用在全局设备内存中创建至少一个新字符串列，从而提供了将多个 API 调用组合到自定义内核中的机会。 

### 内核 malloc 调用的性能限制

首先，我们将构建一个自定义内核来实现编辑示例转换。在设计这个内核时，我们必须记住 libcudf 字符串列是不可变的。

字符串列无法就地更改，因为字符字节是连续存储的，并且对字符串长度的任何更改都会使偏移量数据无效。因此，`redact_kernel`自定义内核通过使用 libcudf 列工厂来构建新的字符串列`offsets`和`chars`子列。 

在第一种方法中，每行的输出字符串是使用内核内部的 malloc 调用在[动态设备内存(dynamic device memory)](https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#dynamic-global-memory-allocation-and-operations)中创建的。自定义内核输出是指向每行输出的设备指针向量，并且该向量用作字符串列工厂的输入。 

自定义内核接受 [`cudf::column_device_view`](https://docs.rapids.ai/api/libcudf/nightly/classcudf_1_1column__device__view.html)来访问字符串列数据，并使用该`element`方法返回[`cudf::string_view`](https://docs.rapids.ai/api/libcudf/nightly/classcudf_1_1string__view.html)表示指定行索引处的字符串数据。内核输出是一个向量类型`cudf::string_view`，它保存指向设备内存的指针，其中包含输出字符串以及该字符串的大小（以字节为单位）。 

该类`cudf::string_view`与  C++17 `std::string_view` 类类似，但专门为 libcudf 实现，并将固定长度的字符数据包装在设备内存中编码为 UTF-8。它具有许多与std相关函数相同的特性（例如`find`，`substr`功能）以及 限制（没有空终止符）。`cudf::string_view`表示存储在设备内存中的字符序列，因此我们可以在此处使用它来记录输出向量的 malloc 内存。

### Malloc内核

```c++
// note the column_device_view inputs to the kernel

__global__ void redact_kernel(cudf::column_device_view const d_names,
                              cudf::column_device_view const d_visibilities,
                              cudf::string_view redaction,
                              cudf::string_view* d_output)
{
  // get index for this thread
  auto index = threadIdx.x + blockIdx.x * blockDim.x;
  if (index >= d_names.size()) return;

  auto const visible = cudf::string_view("public", 6);

  auto const name = d_names.element<cudf::string_view>(index);
  auto const vis  = d_visibilities.element<cudf::string_view>(index);
  if (vis == visible) {
    auto const space_idx    = name.find(' ');
    auto const first        = name.substr(0, space_idx);
    auto const last_initial = name.substr(space_idx + 1, 1);
    auto const output_size  = first.size_bytes() + last_initial.size_bytes() + 1;
    
    char* output_ptr = static_cast<char*>(malloc(output_size));

    // build output string
    d_output[index]  = cudf::string_view{output_ptr, output_size};
    memcpy(output_ptr, last_initial.data(), last_initial.size_bytes());
    output_ptr += last_initial.size_bytes();
    *output_ptr++ = ' ';
    memcpy(output_ptr, first.data(), first.size_bytes());
  } else {
    d_output[index] = cudf::string_view{redaction.data(), redaction.size_bytes()};
  }
}

__global__ void free_kernel(cudf::string_view redaction, cudf::string_view* d_output, int count)
{
  auto index = threadIdx.x + blockIdx.x * blockDim.x;
  if (index >= count) return;

  auto ptr = const_cast<char*>(d_output[index].data());
  if (ptr != redaction.data()) free(ptr); // free everything that does match the redaction string
}
```

在测量内核性能之前，这似乎是一种合理的方法。这种方法在具有 60 万行数据的 A6000 上大约需要 108 毫秒，比上面使用 libcudf strings API 提供的解决方案慢了 30 倍以上。

```
redact_kernel         60.3ms
free_kernel           45.5ms
make_strings_column    0.5ms
```

主要瓶颈是`malloc/free`两个内核内部的调用。CUDA动态设备内存需要`malloc/free`同步内核中的调用，导致并行执行退化为顺序执行。 

（**注**：这个方法主要是为了对比 提前分配内存的消除核内分配内存的情况，以及后面的内存资源管理rmm）

### 预分配工作内存以消除瓶颈

通过在启动内核之前用预先`malloc/free`分配的工作内存替换内核中的调用`malloc/free`来消除瓶颈。

对于redact示例，此示例中每个字符串的输出大小不应大于输入字符串本身，因为逻辑仅删除字符。因此，可以使用与输入缓冲区大小相同的单个设备内存缓冲区。使用输入偏移量来定位每行位置。

访问字符串列的偏移量涉及用`cudf::strings_column_view`包装`cudf::column_view`并调用其` offsets_begin`方法。还可以使用`chars_size`方法访问`chars`子列的大小。然后在内核之前调用`rmm::device_uvector`预先分配内存，来存储字符输出数据。

```c++
auto const scv = cudf::strings_column_view(names);
auto const offsets = scv.offsets_begin();
auto working_memory = rmm::device_uvector<char>(scv.chars_size(), stream);
```

### 预分配内核

```c++
__global__ void redact_kernel(cudf::column_device_view const d_names,
                              cudf::column_device_view const d_visibilities,
                              cudf::string_view redaction,
                              char* working_memory,
                              cudf::offset_type const* d_offsets,
                              cudf::string_view* d_output)
{
  auto index = threadIdx.x + blockIdx.x * blockDim.x;
  if (index >= d_names.size()) return;

  auto const visible = cudf::string_view("public", 6);

  auto const name = d_names.element<cudf::string_view>(index);
  auto const vis  = d_visibilities.element<cudf::string_view>(index);
  if (vis == visible) {
    auto const space_idx    = name.find(' ');
    auto const first        = name.substr(0, space_idx);
    auto const last_initial = name.substr(space_idx + 1, 1);
    auto const output_size  = first.size_bytes() + last_initial.size_bytes() + 1;

    // resolve output string location
    char* output_ptr = working_memory + d_offsets[index];
    d_output[index]  = cudf::string_view{output_ptr, output_size};

    // build output string into output_ptr
    memcpy(output_ptr, last_initial.data(), last_initial.size_bytes());
    output_ptr += last_initial.size_bytes();
    *output_ptr++ = ' ';
    memcpy(output_ptr, first.data(), first.size_bytes());
  } else {
    d_output[index] = cudf::string_view{redaction.data(), redaction.size_bytes()};
  }
}
```

内核输出一个传递 `cudf::string_view` 给 **[`cudf::make_strings_column`](https://docs.rapids.ai/api/libcudf/nightly/group__column__factories.html#ga163234e4e6b8f95d7a8f1796a0c3c79d)** 工厂函数的对象向量。该函数的第二个参数用于识别输出列中的空条目。本文中的示例没有 null 条目，因此`cudf::string_view{nullptr,0}`使用 nullptr 占位符。

```c++
auto str_ptrs = rmm::device_uvector<cudf::string_view>(names.size(), stream);

redact_kernel<<<blocks, block_size, 0, stream.value()>>>(*d_names,
                                                         *d_visibilities,
                                                         d_redaction.value(),
                                                         working_memory.data(),
                                                         offsets,
                                                         str_ptrs.data());

auto result = cudf::make_strings_column(str_ptrs, cudf::string_view{nullptr,0}, stream);
```

这种方法在具有 60 万行数据的 A6000 上大约需要 1.1 毫秒，因此比基线快 2 倍以上。大致细分如下所示：

```shell
  redact_kernel            66us
  make_strings_column     400us
```

剩余时间花费在`cudaMalloc, cudaFree, cudaMemcpy,`管理临时`rmm::device_uvector`实例的典型开销中。如果保证所有输出字符串的大小等于或小于输入字符串，则此方法效果很好。

总体而言，使用 RAPIDS RMM 切换到批量工作内存分配是一项重大改进，也是自定义字符串函数的良好解决方案。 

### 优化列创建以缩短计算时间

有没有办法进一步改善这一点？现在的瓶颈是`cudf::make_strings_column` 工厂函数，它从 `cudf::string_view` 对象向量构建两个字符串列组件 `offsets` 和  `chars`。

在 libcudf 中，包含了许多工厂函数来构建字符串列。前面示例中使用的工厂函数获取`cudf::string_view`对象的`cudf::device_span`，然后通过对底层字符数据执行`gather`来构造列，以构建偏移量和字符子列。`rmm::device_uvector`可自动转换为`cudf::device_span`，而无需复制任何数据。

但是，如果直接构建字符向量和偏移向量，则可以使用不同的工厂函数，该函数只需创建字符串列，而不需要收集来复制数据。 

`sizes_kernel`首先传递输入数据，以计算每个输出行的确切输出大小：

### 优化内核：第 1 部分

```c++
__global__ void sizes_kernel(cudf::column_device_view const d_names,
                             cudf::column_device_view const d_visibilities,
                             cudf::size_type* d_sizes)
{
  auto index = threadIdx.x + blockIdx.x * blockDim.x;
  if (index >= d_names.size()) return;

  auto const visible = cudf::string_view("public", 6);
  auto const redaction = cudf::string_view("X X", 3);

  auto const name = d_names.element<cudf::string_view>(index);
  auto const vis  = d_visibilities.element<cudf::string_view>(index);

  cudf::size_type result = redaction.size_bytes(); // init to redaction size
  if (vis == visible) {
    auto const space_idx    = name.find(' ');
    auto const first        = name.substr(0, space_idx);
    auto const last_initial = name.substr(space_idx + 1, 1);

    result = first.size_bytes() + last_initial.size_bytes() + 1;
  }

  d_sizes[index] = result;
}
```

然后通过执行 in-place `exclusive_scan`将输出大小转换为偏移量。请注意，`offsets`向量是用`names.size()+1`元素创建的。最后一个条目将是字节总数（所有大小加在一起），而第一个条目将为 0。这些都由`exclusive_scan`调用处理。从`offsets`列的最后一个条目检索`chars`列的大小，以构建字符向量。

```c++
// create offsets vector
auto offsets = rmm::device_uvector<cudf::size_type>(names.size() + 1, stream);

// compute output sizes
sizes_kernel<<<blocks, block_size, 0, stream.value()>>>(
  *d_names, *d_visibilities, offsets.data());

thrust::exclusive_scan(rmm::exec_policy(stream), offsets.begin(), offsets.end(), offsets.begin());
```

`redact_kernel`逻辑仍然非常相同，只是它接受输出`d_offsets`向量来解析每行的输出位置：

### 优化内核：第 2 部分

```c++
__global__ void redact_kernel(cudf::column_device_view const d_names,
                              cudf::column_device_view const d_visibilities,
                              cudf::size_type const* d_offsets,
                              char* d_chars)
{
  auto index = threadIdx.x + blockIdx.x * blockDim.x;
  if (index >= d_names.size()) return;

  auto const visible = cudf::string_view("public", 6);
  auto const redaction = cudf::string_view("X X", 3);

  // resolve output_ptr using the offsets vector
  char* output_ptr   = d_chars + d_offsets[index];

  auto const name = d_names.element<cudf::string_view>(index);
  auto const vis = d_visibilities.element<cudf::string_view>(index);
  if (vis == visible) {
    auto const space_idx    = name.find(' ');
    auto const first        = name.substr(0, space_idx);
    auto const last_initial = name.substr(space_idx + 1, 1);
    auto const output_size  = first.size_bytes() + last_initial.size_bytes() + 1;

    // build output string
    memcpy(output_ptr, last_initial.data(), last_initial.size_bytes());
    output_ptr += last_initial.size_bytes();
    *output_ptr++ = ' ';
    memcpy(output_ptr, first.data(), first.size_bytes());
  } else {
    memcpy(output_ptr, redaction.data(), redaction.size_bytes());
  }
}
```

从`d_offsets`列的最后一个条目检索输出`d_chars`列的大小以分配字符向量。内核使用预先计算的偏移向量启动并返回填充的字符向量。最后，libcudf 字符串列工厂创建输出字符串列。

此[`cudf::make_strings_column`](https://docs.rapids.ai/api/libcudf/nightly/group__column__factories.html#ga86f7623f0d230c96491ef88d665385cc)工厂函数构建字符串列而不复制数据。`offsets`数据和` chars`数据已经采用正确的预期格式，该工厂只是从每个向量中移动数据并在其周围创建列结构。完成后，`offsets`和`chars`的`rmm::device_uvectors`为空，它们的数据已移动到输出列中。

```c++
cudf::size_type output_size = offsets.back_element(stream);
auto chars = rmm::device_uvector<char>(output_size, stream);

redact_kernel<<<blocks, block_size, 0, stream.value()>>>(
    *d_names, *d_visibilities, offsets.data(), chars.data());

// from pre-assembled offsets and character buffers
auto result = cudf::make_strings_column(names.size(), std::move(offsets), std::move(chars));
```

这种方法在具有 600K 行数据的 A6000 上大约需要 300 us (0.3 ms)，比之前的方法提高了 2 倍以上。您可能会注意到`sizes_kernel`和`redact_kernel`共享很多相同的逻辑：一次测量输出的大小，然后再次填充输出。

从代码质量的角度来看，将转换重构为由`sizes_kernel`和`redact_kernel`调用的设备函数是有益的。从性能角度来看，您可能会惊讶地发现转换的计算成本被支付了两倍。

内存管理和更高效的列创建的好处通常超过执行两次转换的计算成本。 

表 2 显示了本文讨论的四种解决方案的计算时间、内核计数和处理的字节数。“内核启动总数”反映了启动的内核总数，包括计算内核和辅助内核。“处理的总字节数”是累积的 DRAM 读取和写入吞吐量，“处理的最小字节数”是我们的测试输入和输出的平均每行 37.9 字节。理想的“内存带宽有限”情况假设带宽为 768 GB/s，这是 A6000 的理论峰值吞吐量。

![该表显示了本文讨论的四种解决方案的计算时间、内核计数和处理的字节数。](https://github.com/weedge/mypic/raw/master/oneday/mastering-string-transformations-in-rapids-libcudf/5.png)

*表 2. 本文讨论的四种解决方案的计算时间、内核计数和处理的字节数*

由于内核启动次数减少和处理的总字节数减少，“优化内核”提供了最高的吞吐量。借助高效的自定义内核，内核启动总数从 31 次减少到 4 次，处理的总字节数从输入加输出大小的 12.6 倍减少到 1.75 倍。

因此，定制内核的吞吐量比用于编辑转换的通用字符串 API 高 10 倍以上。

## 峰值性能分析

[RAPIDS 内存管理器 (RMM)](https://developer.nvidia.com/blog/fast-flexible-allocation-for-cuda-with-rapids-memory-manager/)中的池内存资源是另一个可用于提高性能的工具。上面的示例使用默认的“CUDA 内存资源”来分配和释放全局设备内存。然而，分配工作内存所需的时间会增加字符串转换步骤之间的显着延迟。RMM 中的“内存池资源”通过预先分配大量内存并在处理过程中根据需要分配子分配来减少延迟。 

使用 CUDA 内存资源，“优化内核”显示了 10 倍到 15 倍的加速，但由于分配大小的增加，加速在行数增加时开始下降（图 3）。使用池内存资源可以减轻这种影响，并比 libcudf strings API 方法保持 15-25 倍的加速。

![对于 600K 到 10M 的行计数范围，使用自定义内核与 libcudf 字符串 API 的加速效果的散点图。 加速数据包括 4 个条件：“预分配内核”和“优化内核”，具有 CUDA 内存资源和池内存资源。 在每种情况下，池内存资源的性能均优于 CUDA 内存资源。 “优化内核 + RMM 池”显示大约 12-25 倍加速，“预分配内核 + RMM 池”显示 5-10 倍加速。](https://github.com/weedge/mypic/raw/master/oneday/mastering-string-transformations-in-rapids-libcudf/6.png)

*图 3. 使用默认 CUDA 内存资源（实线）和池内存资源（虚线)的自定义内核“预分配内核”和“优化内核”的加速与使用默认 CUDA 内存资源的 libcudf 字符串 API 的加速*

利用池内存资源，证明了端到端内存吞吐量接近两遍算法的理论极限。使用输入大小加上输出大小和计算时间来测量，“优化内核”的吞吐量达到 320-340 GB/s（图 4）。

两遍方法首先测量输出元素的大小，分配内存，然后使用输出设置内存。给定两遍处理算法，“优化内核”中的实现的性能接近内存带宽限制。“端到端内存吞吐量”定义为输入加输出大小（以 GB 为单位）除以计算时间。RTX A6000 内存带宽 (768 GB/s)。

![散点图显示“优化内核”、“预分配内核”和“libcudf 字符串 API”的内存吞吐量与输入/输出行计数的函数关系。 “端到端内存吞吐量”定义为输入加输出大小（以 GB 为单位）除以计算时间。 使用池内存资源时，“libcudf strings API”饱和度约为 40 GB/s，“预分配内核”饱和度约为 150 GB/s，“优化内核”饱和度约为 340 GB/s。](https://github.com/weedge/mypic/raw/master/oneday/mastering-string-transformations-in-rapids-libcudf/7.png)

*图 4. “优化内核”、“预分配内核”和“libcudf 字符串 API”的内存吞吐量与输入/输出行计数的关系*

## 概要

这篇文章演示了在 [libcudf](https://docs.rapids.ai/api/libcudf/nightly/index.html) 中编写高效字符串数据转换的两种方法。libcudf 通用 API 对于开发人员来说快速、简单，并且提供良好的性能。libcudf 还提供了专为与自定义内核一起使用而设计的设备端实用程序，在本例中解锁了 10 倍以上的更快性能。

## Reference

1. https://developer.nvidia.com/blog/mastering-string-transformations-in-rapids-libcudf/
1. https://docs.rapids.ai/api/libcudf/nightly/index.html
1. https://github.com/rapidsai/cudf/tree/HEAD/cpp/examples/strings