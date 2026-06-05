# Rust API Bindings: CFStringGetBytes Is Hard, Part 1

> 原文：https://alwaysprocessing.blog/2023/12/29/rust-ffi-cfstr-getbytes-1
> 发布：2023-12-29　作者：Brian T. Kelley
> 译者：MiMo；代码块保留英文原样　|　仅供个人学习，未公开发布

---

为 `CFStringGetBytes` 设计一个“默认正确”的接口出奇地复杂，因为它的许多行为都依赖于编码方式。

当我开始为 Core Foundation 构建 Rust API 绑定时，我认为使用 `CFCopyDescription` 实现 `Debug` trait（特征）将是一个很好的起点，它将有助于验证实现该 crate 时的增量进展。

由于 CFString 规范采用 UTF-16 编码，而 Rust 使用 UTF-8，因此需要使用 `CFStringCopyBytes` 来将 `CFCopyDescription` 返回的字符串值写入调试格式化器（Debug Formatter）。这最终成为了我第一个编写 Rust API 绑定的函数。

在编写 API 绑定时，我倾向于预先覆盖 100% 的 API。不完整的绑定层曾让我吃亏，所以我尽量避免在逻辑等效的接口中创建任何歧义行为。API 覆盖不足还可能导致绑定层出现设计缺口，而在积累了许多依赖项后，这种缺口可能难以弥补。

API绑定是一个需要首先缓慢推进的领域，通过将整个API表面区域完整、一致地纳入绑定设计中，才能使产品开发得以快速推进——避免陷入修复绑定缺陷和弥补绑定功能缺口的困境。

在所有接口设计中，我有两个常见目标：

接口应当符合语言惯例——熟练掌握该语言的开发者会觉得命名、设计模式、集成点等元素直观且熟悉。

不应存在可表示无效状态的情况。特别是，断言（assertion）、前置条件（precondition）等机制将变得多余，因为编译器会拒绝包含无效状态或无效控制流的代码。

我原本打算在今天的文章中讨论我的Rust API设计，但在最终审查时发现设计中存在一个潜在的无效状态，目前仍在解决中。这对博客写作倒是恰到好处——我们可以将问题和解决方案分成独立文章来呈现！

**CFStringGetBytes复杂度**

这个API看起来并不那么复杂：

```
/* The primitive conversion routine; allows you to convert a string piece at a time
       into a fixed size buffer. Returns number of characters converted.
   Characters that cannot be converted to the specified encoding are represented
       with the byte specified by lossByte; if lossByte is 0, then lossy conversion
       is not allowed and conversion stops, returning partial results.
   Pass buffer==NULL if you don't care about the converted string (but just the convertability,
       or number of bytes required).
   maxBufLength indicates the maximum number of bytes to generate. It is ignored when buffer==NULL.
   Does not zero-terminate. If you want to create Pascal or C string, allow one extra byte at start or end.
   Setting isExternalRepresentation causes any extra bytes that would allow
       the data to be made persistent to be included; for instance, the Unicode BOM. Note that
       CFString prepends UTF encoded data with the Unicode BOM <http://www.unicode.org/faq/utf_bom.html>
       when generating external representation if the target encoding allows. It's important to note that
       only UTF-8, UTF-16, and UTF-32 define the handling of the byte order mark character, and the "LE"
       and "BE" variants of UTF-16 and UTF-32 don't.
*/
CF_EXPORT
CFIndex CFStringGetBytes(CFStringRef theString, CFRange range, CFStringEncoding encoding, UInt8 lossByte, Boolean isExternalRepresentation, UInt8 *buffer, CFIndex maxBufLen, CFIndex *usedBufLen);
```

但是，在为我的绑定库编写测试时，我发现了以下在注释或文档中未曾明确说明的行为：

range 可能起始于或终止于代理对（surrogate pair）的中间，或者 CFString 可能包含无效的 UTF-16（它不会验证从 UTF-16 缓冲区创建的字符串，并且允许删除代理码元而不删除其配对码元）。对无效代理的处理方式取决于编码：

- 对于 UTF-8，转换会停止。即使调用者提供了损失字节（lossByte），该函数也不会将该码元作为有损转换来处理。

推论：如果代码假定 UTF-8 转换不会失败，并且字符串包含无效的 UTF-16 而循环未验证转换是否产生了进展，则会陷入无穷循环。

- 对于 UTF-32，代理码元会变成一个有损转换的码元。

- 对于所有其他编码（包括 UTF-16），则没有可观测的效果。

- 对于非 Unicode 编码，一个用代理对表示的码点会变成两个有损码点。

尽管注释中有暗示，但 isExternalRepresentation 对于 UTF-8 并不会编码 BOM。

如果 encoding 等于 kCFStringEncodingUTF16，isExternalRepresentation 为 true，且 maxBufLen 小于 2，那么 Core Foundation 在写入字节顺序标记（BOM，Byte Order Mark）时将会**溢出缓冲区**。（UTF-32 BOM 的写入则会验证缓冲区容量。）

在文档或代码中，我并未看到任何关于缓冲区对齐要求的说明。`buffer` 的类型 `UInt8 *` 暗示 `CFStringGetBytes` 支持非对齐缓冲区。这种支持是合理的，例如，为了便于调用方将字节写入一种可能会因打包而产生非对齐偏移的持久化格式。但据我所知，该实现依赖于未定义行为（将缓冲区指针强制转换为对齐要求更严格的类型）来实现对非对齐指针的支持。虽然处境岌岌可危但幸运的是，编译器选择的指令集恰好支持非对齐写入。

在大多数情况下，这些都是无足轻重的边缘案例：

尽管许多处理 UTF-16 的代码需要更好地处理代理对（surrogate pair），但我怀疑在某个范围的开头或结尾处发生代理对分割的情况并不多见。

非 Unicode 编码的主要转换方向是转化为 Unicode，因此从 Unicode 转换时因代理对（surrogate pairs）产生的代码点膨胀（code point inflation）现象可能较为罕见。

在实践中，BOM（字节顺序标记）的使用频率很低，尤其是在 UTF-8 编码中。

没有任何生产代码会使用单字节大小的缓冲区。

幸运的是，几层抽象设计就能简化大部分此类复杂性。敬请关注后续内容，我将概述如何设计一个 Rust API 来确保正确且可预测的行为。