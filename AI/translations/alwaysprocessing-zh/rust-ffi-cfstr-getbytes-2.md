# Rust API Bindings: CFStringGetBytes Is Hard, Part 2

> 原文：https://alwaysprocessing.blog/2024/01/07/rust-ffi-cfstr-getbytes-2
> 发布：2024-01-07　作者：Brian T. Kelley
> 译者：MiMo；代码块保留英文原样　|　仅供个人学习，未公开发布

---

Rust API 绑定：`CFStringGetBytes` 之难，第二部分

运用 Rust 的语言特性，我们可以为 `CFStringGetBytes` 提供 API 绑定，在调用点阻止不受支持的参数组合，从而修复前文中指出的问题。

上一篇文章描述了我在为 `CFStringGetBytes` 构建 Rust 绑定时遇到的复杂情况。本文则分享我为第一层绑定所做设计决策背后的原理。这最底层的缓解了前文指出的四个问题。

我说“第一层绑定”，是因为本文讨论的方法构成了最多四个其他 Rust 接口用于调用 `CFStringGetBytes` 的基础。然而，为何需要这么多层次？正如《Rust API 指南》所言：

> 函数应暴露中间结果以避免重复工作

> 许多用于解答问题的函数也会计算出相关的有趣数据。如果这些数据可能对客户端有用，考虑将其暴露在 API 中。

因此，在构建我认为是最地道的 Rust API 时，我层层递进地暴露了更低层次的接口，以提供更高的可定制性（代价是增加了复杂性）。

首先，我们回顾一下 Core Foundation 的 C API：

```
CFIndex CFStringGetBytes(CFStringRef theString, CFRange range, CFStringEncoding encoding, UInt8 lossByte, Boolean isExternalRepresentation, UInt8 *buffer, CFIndex maxBufLen, CFIndex *usedBufLen);
```

并将其与在我的crate中实现的最直接的Rust接口进行比较：

```
impl String {
  pub fn get_bytes_unchecked(
    &self,
    range: impl RangeBounds<usize>,
    encoding: GetBytesEncoding,
    buf: Option<&mut [u8]>,
  ) -> GetBytesResult { /* ... */ }
}
```

`&self` 代表 `CFStringRef`（CF字符串引用）指针，这是面向对象接口绑定的典型方式。

`CFRange` 参数的等效类型是 `impl RangeBounds<usize>`，允许调用者使用 Rust 的范围表达式。例如，调用者可以传递 `..` 来指定字符串的完整范围。

`CFRange` 的字段类型为 `CFIndex`，这是一种有符号类型。本系列的第一篇文章讨论了实现无符号到有符号转换时的设计选择。

`GetBytesEncoding` 替代了 `CFStringEncoding`，并同时涵盖了 `lossByte` 和 `isExternalRepresentation` 参数。以下部分将提供更多细节。

`buf: Option<&mut [u8]>` 捕获了可选的 `UInt8 *buffer` 和 `CFIndex maxBufLen` 参数。`Option` 类型清晰地表达了缓冲区并非必需。如果提供了缓冲区，该切片将提供缓冲区的长度。

`GetBytesResult` 返回类型包含了 C API 的返回值（已转换的 UTF-16 代码单元数量）以及 C API 中的输出参数 `usedBufLen`。

`_unchecked` 后缀暗示此方法存在一个调用者必须处理的特殊问题。后续章节将详细阐述这一行为。

该方法实现了对前文所述第四个问题的修复检查：

如果编码为 `kCFStringEncodingUTF16`、`isExternalRepresentation` 为 true 且 `maxBufLen` 小于 2，Core Foundation 在写入 BOM（字节顺序标记）时将发生缓冲区溢出。（UTF-32 的 BOM 写入操作会验证缓冲区容量。）

若缺少此检查，该函数的使用可能导致不健全性（unsoundness），因此需要添加 `unsafe` 限定符。

**GetBytesEncoding**

`GetBytesEncoding` 结构体封装了 `CFStringEncoding`、`lossByte` 和 `isExternalRepresentation` 这三个参数。

```
pub enum GetBytesEncoding {
  CharacterSet {
    character_set: CharacterSet,

    /// **Note:** Core Foundation will process surrogate pairs as two individual lossy code
    /// points, so the number of output code points will equal the number of input code units.
    loss_byte: Option<NonZeroU8>,
  },
  Utf8,
  Utf16 {
    byte_order: GetBytesByteOrder,
  },
  Utf32 {
    byte_order: GetBytesByteOrder,
    loss_byte: Option<NonZeroU8>,
  },
}
```

`CharacterSet` 枚举涵盖了所有非 Unicode 编码。（其对应的字符串创建函数 `CFStringCreateWithBytes` 同样具有特殊性。字符串构建绑定同样特别处理 UTF-8、UTF-16 和 UTF-32 编码，并通过 `CharacterSet` 枚举处理所有非 Unicode 表示。）

绑定中的 `lossByte` 类型是 `Option<NonZeroU8>`，这旨在通过类型系统表达：若使用有损字节，其值必须为非零。

遗憾的是，对于前文中指出的第二个问题，我唯一能找到的缓解措施是使用注释：

一个代码点（code point）若编码为代理对（surrogate pair），对于非 Unicode 编码而言将变成两个有损代码点。

`Utf8` 编码没有有损字节，这缓解了在阅读调用处和文档时可能出现的意外行为——即前文中第一个问题 a 部分指出的情况：

即使调用者提供了 `lossByte`，该函数也不会将该代码单元（code unit）作为有损转换来处理。

缺少转换回退机制可能暗示 UTF-8 不会像 UTF-16 那样失败，这是该方法使用 `_unchecked` 后缀的一个因素——直接调用的行为可能与默认预期不符。

UTF-16 有一个 `GetBytesByteOrder` 字段（下文讨论），它实现了 `isExternalRepresentation` 参数的功能。

UTF-32 与 UTF-16 类似，也有一个 `GetBytesByteOrder` 字段，并且与 `CharacterSet` 相似，它有一个损失字节（loss byte）用于处理无效的代理对（surrogates）（说明 UTF-32 是唯一实现损失字节支持的 Unicode 目标编码，正如上一篇文章中第一个问题的 b 部分所述）。

UTF-16 和 UTF-32 使用 16 位和 32 位整数标量（scalars），它们可能采用大端序（big endian）或小端序（little endian）的字节顺序。`GetBytesByteOrder` 枚举了支持的选项。

```
pub enum GetBytesByteOrder {
  BigEndian,
  HostNative {
    include_bom: bool,
  },
  LittleEndian,
}
```

Core Foundation仅支持在使用主机原生字节序时写入字节顺序标记（BOM）（isExternalRepresentation = true）。此枚举通过防止调用者指定不支持的组合，从而避免出现意外结果。

GetBytesEncoding::Utf8与GetBytesByteOrder中isExternalRepresentation实现的组合，共同解决了前文指出的第三个问题：

尽管注释中有所暗示，isExternalRepresentation并未为UTF-8编码BOM。

GetBytesResult

此函数返回一个结构体，其中包含C语言API的返回值及输出参数。

```
pub struct GetBytesResult {
  pub buf_len: usize,
  pub remaining: Option<Range<usize>>,
}
```

如果 `buf` 为 `Some`，则 `buf_len` 字段包含写入切片的字节数；否则，它包含转换处理范围所需的字节数。

如果调用已转换整个输入范围，则 `remaining` 字段为 `None`；否则，它包含调用期间未转换的输入范围部分。返回一个 `Range` 而非已转换的 UTF-16 码元数量，能通过提供明确的“完成”信号和下次调用的范围参数值来简化转换循环的实现。

如果调用方未提供损失字节，任何转换（UTF-16 转换除外）都可能失败。由于此函数是“未检查的”（unchecked），调用方有责任检查是否取得前进进展，否则将面临永不终止的风险。

下一篇博文将讨论 `get_bytes` 方法，该方法明确处理有损转换并防止潜在的无限循环。