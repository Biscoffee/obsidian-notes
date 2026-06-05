# Objective-C Internals: Tagged Pointer Objects

> 原文：https://alwaysprocessing.blog/2023/03/19/objc-tagged-ptr
> 发布：2023-03-19　作者：Brian T. Kelley
> 译者：MiMo；代码块保留英文原样　|　仅供个人学习，未公开发布

---

标记指针对象（tagged pointer object，一项私有的运行时特性）通过将对象数据存储在其指针值中来优化性能，从而消除了对象的堆内存分配。本文将通过观察 `NSNumber` 的实现来揭示此优化的应用及其影响。

标记指针对象是 Objective-C 运行时（Objective-C runtime）的一个私有特性，Apple 使用它来优化一些核心的 Foundation 类（此处一语双关）。

**什么是标记指针？**

大多数内存分配器（memory allocator）都保证每次分配的最低对齐要求（alignment）。例如，在 Apple 平台上，`malloc()` 保证的对齐方式“可用于任何数据类型，包括与 AltiVec 和 SSE 相关的类型”。

实际上，所有分配都是 16 字节对齐的，因此任何指针的最低四位总是零。有时利用这一事实在那些比特位中存储额外信息是有利的（这需要更新所有使用该指针值的地方，以便正确处理额外的比特位）。当指针中未使用的比特位存储了额外信息时，该指针通常被称为标记指针（tagged pointer）。

**标记指针对象**

在Objective-C中，对象的`isa`指针可能被标记，其细节已在本系列前文《isa的多种用途》一文的“非指针isa”章节中讨论过。

Objective-C中的标记指针对象是一种特殊类型的对象指针。若对象指针被标记，则由该标记指针所表示的类实例所持有的数据将被完全编码到指针值本身之中。此过程不会发生堆分配。

消除堆分配可以显著降低成本，例如对于一个`NSNumber`数组（`NSArray`）。在堆上分配的`NSNumber`至少使用16字节（因为分配是16字节对齐的），外加其指针的8字节。然而，被编码到标记指针中的`NSNumber`仅使用其指针的8字节，从而通过不调用分配器来节省内存和时间。

**标记身份**

用于将指针识别为标记指针的位因平台而异。`objc-internal.h`为运行时实现提供了一个便捷函数，用于识别一个指针是否被标记。macOS和Intel处理器上的Catalyst应用使用第0位来标识标记指针对象，而其他所有平台则使用第63位。（译注：此规则针对不同处理器架构和系统版本可能有变化）

```
#if __arm64__
  // ARM64 uses a new tagged pointer scheme...
# define _OBJC_TAG_MASK (1UL<<63)
#elif (TARGET_OS_OSX || TARGET_OS_MACCATALYST) && __x86_64__
  // 64-bit Mac - tag bit is LSB
# define _OBJC_TAG_MASK 1UL
#else
  // Everything else - tag bit is MSB
# define _OBJC_TAG_MASK (1UL<<63)
#endif

static inline bool
_objc_isTaggedPointer(const void *ptr) {
  return ((uintptr_t)ptr & _OBJC_TAG_MASK) == _OBJC_TAG_MASK;
}
```

消除用于存储对象值的堆内存分配的同时，也消除了 isa 指针的存储空间。因此，标记指针对象（tagged pointer）方案保留了一些位来标识对象的类。

在撰写本文时，Objective-C 运行时有两种保留类标识位的方案：一种为具有 60 位有效载荷（payload）的对象保留 3 位，另一种为具有 52 位有效载荷的对象保留 11 位。

最多 7 个类类型可以使用 60 位有效载荷的变体（其中第八个类型是标识 52 位有效载荷变体的特殊情况）。而最多 256 个类类型可以使用 52 位有效载荷的变体。`objc-internal.h` 中有一个枚举，为各种类标识位值提供了一些符号化的标识。

当运行时需要一个对象的 isa 指针时，它会调用 `objc_object::getIsa()`[1]（在 `objc-object.h` 中定义）。该函数对于在堆上分配的对象返回其 isa 实例变量，对于标记指针对象则返回存储在某个标签类数组中的 isa 指针。

```
inline Class
objc_object::getIsa() {
  if (fastpath(!isTaggedPointer())) return ISA(/*authenticated*/true);

  extern objc_class OBJC_CLASS_$___NSUnrecognizedTaggedPointer;
  uintptr_t slot, ptr = (uintptr_t)this;
  Class cls;

  slot = (ptr >> _OBJC_TAG_SLOT_SHIFT) & _OBJC_TAG_SLOT_MASK;
  cls = objc_tag_classes[slot];
  if (slowpath(cls == (Class)&OBJC_CLASS_$___NSUnrecognizedTaggedPointer)) {
      slot = (ptr >> _OBJC_TAG_EXT_SLOT_SHIFT) & _OBJC_TAG_EXT_SLOT_MASK;
      cls = objc_tag_ext_classes[slot];
  }
  return cls;
}
```

当系统框架在进程启动时进行初始化时，它们会调用 `_objc_registerTaggedPointerClass()` 来为给定的标签值（tag value）设置 `Class` 对象。我们可以通过为该函数添加符号断点（symbolic breakpoint）并在其被调用时打印参数来观察这一过程：

```
(lldb) reg re x0 x1
      x0 = 0x0000000000000003
      x1 = 0x0000000203a994f0  (void *)0x0000000203a99518: __NSCFNumber
```

消息发送的剖析

任何操作指针值的代码都必须专门处理标记指针（tagged pointer），因为无条件解引用该指针几乎必然导致运行时崩溃。

`objc_msgSend()` 的第一步会检查是否为标记指针，以决定如何加载对象的 `isa` 指针来查找其类的方法，这与前文展示的 `objc_object:getIsa()` 实现类似。加载 `isa` 指针后，无论指针是否为标记，对剩余的消息发送逻辑而言都无关紧要。

当一个支持标记指针对象的 Objective-C 类接收到消息时，它必须检查其 `self` 指针是否为标记指针并妥善处理该情况。以下是我对 `-[NSNumber integerValue]` 的反汇编分析，部分展示了 NSNumber 标记指针的工作方式（基于我对 macOS 12.6.2 arm64 架构下相关函数反汇编代码的解读）。

```
@implementation __NSCFNumber
- (NSInteger)integerValue {
  return [self longValue];
}

- (long)longValue {
  long longValue;
  CFNumberGetValue((__bridge CFNumberRef)self, kCFNumberSInt64Type, &longValue);
  return longValue;
}
@end

Boolean CFNumberGetValue(CFNumberRef number, CFNumberType theType, void *valuePtr) {
  if (_objc_isTaggedPointer(number)) {
    if (_objc_getTaggedPointerTag(number) == OBJC_TAG_NSNumber) {
      long localMemory;
      valuePtr = valuePtr ?: (void *)&localMemory;

      uintptr_t value = _objc_getTaggedPointerValue(number);
      uintptr_t shift = (value & 0x08) ? 0x4 : 0x6;

      CFNumberType type = __CFNumberTypeTable[theType].canonicalType;
      if (type <= kCFNumberFloat64Type) {
        value = value >> shift;

        switch (type) {
        case kCFNumberSInt8Type:
          *(uint8_t *)valuePtr = (uint8_t)value;
          break;
        case kCFNumberSInt16Type:
          *(uint16_t *)valuePtr = (uint16_t)value;
          break;
        case kCFNumberSInt32Type:
          *(uint32_t *)valuePtr = (uint32_t)value;
          break;
        case kCFNumberSInt64Type:
          *(uint64_t *)valuePtr = (uint64_t)value;
          break;
        case kCFNumberFloat32Type:
          *(float *)valuePtr = (float)value;
          break;
        case kCFNumberFloat64Type:
          *(double *)valuePtr = (double)value;
          break;
        }
        return true;
      } else {
        return __CFNumberGetValueCompat(number, theType, valuePtr);
      }
    } else {
      theType = __CFNumberTypeTable[theType].canonicalType;
      return [(__bridge id)number _getValue:valuePtr forType:theType];
    }
  } else {
    // ...
  }
}
```

关于上述手动反编译代码的几点评论和观察：

在这个代码路径中，私有 `__NSCFNumber` 子类并未操作 `self` 的值，因此无需处理 tagged pointer（标记指针）情况。`-integerValue` 方法作为 `longValue` 方法的别名，后者会调用 CoreFoundation 内部实现来完成核心操作。由于 `CFNumberRef` 和 `NSNumber *` 是 toll-free bridged（无损桥接）关系，一种类型封装另一种类型的设计是合理的。

苹果的 `CFNumber` 实现几乎可以肯定包含了 `objc-internal.h`，以便使用简化 tagged pointer 对象操作的内部函数。

有效载荷的 4 或 6 位可能作为其他 `CFNumber` 特性的位标志使用，但无论其具体功能如何，此函数均未利用该特性。

`__CFNumberTypeTable` 将公开的类型值映射到对应的固定大小类型，从而简化了值提取逻辑。

苹果拥有一种私有类型 `kCFNumberSInt128Type`，该类型未被 switch 语句处理，因此针对此情况必须调用 `__CFNumberGetValueCompat()` 函数。（译注：`__CFNumberGetValueCompat` 等以双下划线开头的函数通常为未公开的内部实现，现代系统可能已变化）

具有非零小数值的浮点数不使用标记指针（tagged pointer）对象优化，这是因为 `switch` 语句中的逻辑未涵盖该场景。

我对 `-_getValue:forType:` 的调用很感兴趣。这意味着存在另一个私有子类使用了标记指针对象优化，但我没有跟踪各种代码路径来尝试识别它。