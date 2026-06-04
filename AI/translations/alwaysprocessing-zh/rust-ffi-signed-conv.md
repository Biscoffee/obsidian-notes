# Rust API Bindings: Core Foundation Signed/Unsigned Conversion

> 原文：https://alwaysprocessing.blog/2023/12/27/rust-ffi-signed-conv
> 发布：2023-12-27　作者：Brian T. Kelley
> 译者：MiMo；代码块保留英文原样　|　仅供个人学习，未公开发布

---

**无缝桥接方法**

处理 `CFIndex`/`usize`（有符号/无符号）的转换，对于创建符合 Rust 风格的 API 绑定至关重要。我研究了 Apple 在 Core Foundation 与 Foundation 之间，以及 Foundation 与 Swift 之间处理此转换的方式，以帮助我为自己的 Core Foundation crate 确定方向。

Core Foundation 的典型索引类型和大小类型 `CFIndex` 是有符号的。Foundation 的典型索引类型和大小类型 `NSUInteger` 是无符号的，而 Rust 的典型类型 `usize` 同样也是无符号的。

处理 `CFIndex`/`usize`（有符号/无符号）的转换，对于创建符合 Rust 风格的 API 绑定至关重要。在为我的 crate 实施具体方案之前，我研究了 Apple 在处理 Core Foundation↔Foundation 和 Foundation↔Swift 之间的此问题时是如何权衡性能（不安全的位转换）与正确性（检测并处理符号变化）的。

许多Core Foundation和Foundation类型是可互换的。`CFIndex`在Core Foundation中被广泛使用，而Foundation对应的接口使用`NSUInteger`。我无法访问Foundation源代码，但通过查看汇编代码（下方有示例），似乎Foundation在`CFIndex`和`NSUInteger`之间转换时使用了无检查的位转换（unchecked bit cast）——值直接传递，未检测潜在的符号变化。因此，当二进制位在`1 << sizeof(size_t) * 8 - 1`处被设置为1时，该整数值在Core Foundation接口中为负数，但在Foundation接口中为正数。

据我所知，苹果文档中对类型不匹配的唯一承认体现在以下对`CFRange`和`NSRange`中`location`和`length`字段的描述（略有编辑）：

为了与系统其余部分的类型兼容，`LONG_MAX`是你应为`location`和`length`使用的最大值。

一个有趣的关联细节是 `kCFNotFound` 和 `NSNotFound` 的差异。尽管许多 Core Foundation 和 Foundation 类型可以互换使用，但表示“未找到”的语义值取决于具体接口。`kCFNotFound` 定义为 -1，而 `NSNotFound` 是 `NSIntegerMax`（即 `CFIndex` 的最大值，`LONG_MAX`）。因此，返回 `NSRange` 的可失败 `Foundation` 方法实际可寻址范围被限制在 `[0, LONG_MAX)`。

在底层 Core Foundation 实现中使用有符号类型，加上 `NSNotFound` 定义为有符号最大值，限制了 Foundation 利用无符号类型的完整范围。

**未经检查的位转换示例**

`-[NSString length]` 返回 `NSUInteger`，但其内部实现只是简单尾调用 Core Foundation 的 `_CFStringGetLength2`，而后者返回 `CFIndex`。

```
CoreFoundation`-[__NSCFString length]:
  b       _CFStringGetLength2
```

Objective-C中的每个方法都有两个隐含参数：

`self`：指向对象实例的指针  
`_cmd`：解析为该方法的选择子（SEL）

上述尾调用（tail call）之所以有效，是因为 `_CFStringGetLength2` 只有一个参数——指向对象实例的指针，因此Objective-C方法与Core Foundation的C函数具有相同的应用程序二进制接口（ABI），无需额外工作来转发参数。（`_CFStringGetLength2` 会忽略第二个参数。）

另一方面，`-[NSString getCharacters:range:]` 在调用Core Foundation的 `_CFStringCheckAndGetCharacters` 前后都需要执行一些操作：

Objective-C方法有四个参数（`self`、`_cmd`、`buffer` 和 `range`），而Core Foundation函数有三个参数（`str`、`range`、`buffer`）。移除 `_cmd` 需要移动其后续的参数。此外，在此接口中，最后两个参数的顺序是相反的，需要重新排序。

通常，Core Foundation 不进行边界检查，而 Foundation 会。Foundation 会调用一个Core Foundation的系统编程接口（SPI，System Programming Interface），该接口在范围越界时返回错误代码，以便Foundation抛出异常。

```
CoreFoundation`-[__NSCFString getCharacters:range:]:
  pacibsp                         ; insert PAC into LR using SP as the modifier and key B
  stp     x20, x19, [sp, #-0x20]! ; SP -= 0x20; SP[0x00] = x20; SP[0x08] = x19
  stp     x29, x30, [sp, #0x10]   ; SP[0x10] = FP; SP[0x18] = LR;
  add     x29, sp, #0x10          ; FP = SP + 0x10 (address of {FP, LR})
  mov     x8, x2                  ; x8 = buffer (temporary)
  mov     x19, x1                 ; x19 = _cmd
  mov     x20, x0                 ; x20 = self
  mov     x1, x3                  ; x1 = range.location (arg 1, part 1/2)
  mov     x2, x4                  ; x2 = range.length   (arg 1, part 2/2)
  mov     x3, x8                  ; x3 = buffer         (arg 2)
  bl      _CFStringCheckAndGetCharacters
  cbnz    w0, raiseException      ; goto raiseException if _CFStringCheckAndGetCharacters did not return 0
  ldp     x29, x30, [sp, #0x10]   ; FP = SP[0x10]; LR = SP[0x18]
  ldp     x20, x19, [sp], #0x20   ; x20 = SP[0x00]; x19 = SP[0x08]; SP += 0x20
  retab                           ; return using PAC with SP as the modifier and key B
raiseException:
  mov     x0, x20                 ; x0 = self
  mov     x1, x19                 ; x1 = _cmd
  bl      -[__NSCFString getCharacters:range:].cold.1
```

虽然需要一些代码来将 Foundation 接口适配到 Core Foundation，但对于范围值从无符号到有符号的转换，其中没有任何验证。

**Swift 的处理方式**

对于 Apple 的框架（例如 AppKit、Foundation、UIKit 等），Swift 编译器将 `NSUInteger` 导入为 `Int`。为了使 C 和 Objective-C 接口对 Swift 代码可见，Swift 编译器会“导入”一个 Clang 模块来构建一个 Swift 模块，其中“导入”是 Swift 编译器的一个过程，它为 Clang 模块中存在的兼容的声明、定义和类型构建一个 Swift 原生的表示形式。

通常，`NSUInteger` 会被导入为 `UInt`。但是，在系统模块中（即，在由 `-isystem` 标志指定的目录下找到的模块），除非 `NSUInteger` 声明的是一个枚举的类型，否则 Swift 会静默地将其重新类型化（retype）为 `Int`。这种重新类型化操作发生在 Swift 模块的构建过程中。编译器不会记录任何关于此变更的元数据，并且该操作对编译器的其他部分不可见。因此，编译器为便于跨越语言边界而生成的任何桥接代码（thunk），都不会检查潜在的符号（sign）变化。

我查阅了 Swift 编译器的实现，以了解在什么情况下会发生从无符号到有符号的类型转换，以及这是如何实现的。

Swift 的 `ClangImporter` 的 `shouldAllowNSUIntegerAsInt` 成员函数包含了决定编译器是否应将 `NSUInteger` 的类型更改为 `Int` 的主要逻辑。它返回 `true` 的条件是：

该声明来自系统模块（system module）。

并且该声明的名称不包含 `Unsigned` 或 `unsigned`，这是一个特殊情况，用于为 `+[NSNumber numberWithUnsignedInteger:]`、`-[NSNumber initWithUnsignedInteger:]` 和 `-[NSNumber unsignedIntegerValue]` 保留 `NSUInteger` 类型。

有两种方式可以将 Clang 模块标识为系统模块：

1.  其模块映射（module map）具有 `[system]` 属性。
2.  编译器从系统目录加载了该模块映射。

    在这种情况下，模块映射的加载发生在解析头文件搜索路径（header search paths）期间。

模块从包含其模块映射的目录继承 `IsSystem` 标志。

任何非用户目录都是系统目录。

Clang 将非用户目录标识为其某个系统目录组（system directory groups）的一部分。

编译器调用参数（compiler invocation arguments）指定了哪些目录属于系统目录组。

**我的 crate 的做法**

在为我的 Core Foundation Rust API 绑定考虑各种有符号/无符号转换方法时，我评估了 Foundation 和 Swift 的做法（透明重类型化），并考虑了 Rust 的行为被定义为未定义行为的情况。虽然 Foundation 和 Swift 的做法可能导致意外的符号变化，但按照 Rust 的定义，它们不被视为不安全。唯一可能相关的未定义行为是“使用错误的调用 ABI 调用函数”，但有符号性通常不被视为 C ABI 的一部分。

我引入了两个 trait（特征）来方便有符号/无符号转换：

`ExpectFrom` 执行有符号/无符号转换，如果转换失败则 panic。其实现是 `<T as TryFrom>::try_from(value).expect("")` 的便利封装，虽然这很简单，但减少了绑定代码中临时 expect 的数量，提高了可读性。该 trait 主要用于将符合 Rust 惯用法的类型转换为原生的 Core Foundation 类型。如果 `CFIndex` 无法表示某个索引或大小，它会提供一个用户可见的信号。

`FromUnchecked` 执行有符号/无符号转换，并假设结果是正确的，这模仿了 Foundation 和 Swift 的透明重类型转换方式。此特征（trait）主要便于将原生的 Core Foundation 类型转换为地道的 Rust 类型，前提是假设值处于合理范围内。

如果符号变化未被检测到，安全的 Rust 代码将会 panic（恐慌）。不安全的代码必须确保所有值都处于给定域（domain）的边界内，这样，即使符号变化未被检测到也不会带来额外负担——前提是符号变化会导致该值超出边界。