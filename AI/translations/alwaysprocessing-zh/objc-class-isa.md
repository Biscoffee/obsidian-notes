# Objective-C Internals: The Many Uses of isa

> 原文：https://alwaysprocessing.blog/2023/01/19/objc-class-isa
> 发布：2023-01-19　作者：Brian T. Kelley
> 译者：MiMo；代码块保留英文原样　|　仅供个人学习，未公开发布

---

Objective-C 运行时通过将额外信息打包进对象的 isa 指针来优化性能。本文比较了不同的打包机制，并讨论了指针值中存储的各种字段值。

本系列的首篇文章介绍了 isa 指针：它是每个 Objective-C 对象中的一个实例变量（ivar），指向其类对象（class object），用于标识对象实例的类型。前一篇文章引用了该字段的内部定义（`char isa_storage[sizeof(isa_t)]`），并提到 isa 字段已被弃用。现在，我们将更详细地探讨运行时如何使用此字段，以及其被弃用的情况。

**背景**

在 Apple 的 iOS、macOS 和 tvOS 上的 32 位 Objective-C 运行时中，isa 字段仅仅是指向对象类对象的指针。以下来自 `objc-private.h`、`objc-object.h` 和 `objc-class.mm` 的代码展示了 `object_getClass()` 运行时函数的有效实现（针对这些平台），该函数用于从对象实例获取 isa 类指针值。（译注：此处描述的纯指针形式的 isa 实现是较早期的实现，现代 64 位运行时主要使用压缩的 isa。）

```
// objc-private.h
union isa_t {
private:
    Class cls;

public:
    Class getClass(bool authenticated);
};

// objc-object.h
Class objc_object::getIsa() {
    return ISA(); // bool argument defaulted to false
}

Class isa_t::getClass(bool) {
    return cls;
}

Class objc_object::ISA(bool) {
    return isa().getClass(false);
}

// objc-class.mm
Class object_getClass(id obj) {
    if (obj) return obj->getIsa();
    else return Nil;
}
```

**非指针isa（Non-Pointer isa）**

苹果的64位Objective-C运行时（在64位Intel处理器上的模拟器除外）以及Apple Watch的Objective-C运行时使用一种“非指针isa”机制，该机制将额外信息打包到未使用的指针位中。

在指针值中设置额外位会改变其引用的地址并使其值失效——新值可能并非进程地址空间中的有效地址，若被解引用可能导致非对齐内存访问等。因此得名“非指针isa”。

由于`isa`字段不再仅存储类指针值，它在`objc.h`公共头文件中被标记为已弃用（deprecated），以阻止可能导致未定义行为的直接使用。（可用性宏定义位于`objc-api.h`。）`object_getClass()`和`object_setClass()`函数是直接使用`isa`字段的官方替代方案。（译注：此描述基于历史版本，现代系统实现可能已变化。）

```
// objc-api.h
#if !defined(OBJC_ISA_AVAILABILITY)
#   define OBJC_ISA_AVAILABILITY  __attribute__((deprecated))
#endif

// objc.h
struct objc_object {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;
};
```

**非指针isa变体**

截至本文撰写时，非指针isa（non-pointer isa）共有三种实现：

- 紧凑型isa（Apple Silicon arm64-not-e 变体，64位 Intel）
- 带指针认证的紧凑型isa（Apple Silicon arm64e 变体）
- 索引型isa（Apple Watch）

下表展示了每种非指针isa变体中，填充于未使用的类指针位域的附加字段：

| 字段名               | 紧凑型isa | 带认证紧凑型isa | 索引型isa |
|----------------------|-----------|----------------|-----------|
| nonpointer           | ✓         | ✓              | ✓         |
| has_assoc            | ✓         | ✓              | ✓         |
| has_cxx_dtor         | ✓         | x              | ✓         |
| shiftcls             | ✓         | −              | −         |
| shiftcls_and_sig     | −         | ✓              | −         |
| indexcls             | −         | −              | ✓         |
| magic                | ✓         | ✓              | ✓         |
| weakly_referenced    | ✓         | ✓              | ✓         |
| has_sidetable_rc     | ✓         | ✓              | ✓         |
| extra_rc             | ✓         | ✓              | ✓         |

图例：
- ✓ 表示该变体拥有此字段
- − 表示该字段不适用于该变体
- x 表示该变体没有此字段

现在，让我们来探讨运行时中每个字段的用途，以及它们如何用于提升Objective-C运行时性能。

**nonpointer**

这是指针有效载荷（pointer payload）的最低有效位（因此对于指针值而言它始终为零）。如果该位被置位，则表明此isa值是非指针变体。此字段使运行时能够根据每个类的情况，在运行时选择使用传统的“isa值即类指针”行为，或是采用非指针isa优化，以维持兼容性。

在 macOS 上，针对 OS X 10.10 或更早版本链接的应用程序会禁用非指针 isa（non-pointer isa），因为直接使用 isa 直到 OS X 10.11 才被弃用。

```
if (dyld_get_active_platform() == PLATFORM_MACOS && !dyld_program_sdk_at_least(dyld_platform_version_macOS_10_11)) {
    DisableNonpointerIsa = true;
}
```

若主应用程序可执行文件中存在 `__DATA,__objc_rawisa` 节，运行时（runtime）将禁用非指针 isa（non-pointer isa）功能。可能加载在直接使用 isa 指针（direct isa usage）被弃用前已链接的插件（plug-ins）的应用程序，可使用此节以维持二进制兼容性。（此功能亦为 macOS 独有。）

```
for (EACH_HEADER) {
    if (hi->mhdr()->filetype != MH_EXECUTE) continue;
    unsigned long size;
    if (getsectiondata(hi->mhdr(), "__DATA", "__objc_rawisa", &size)) {
        DisableNonpointerIsa = true;
    }
    break;  // assume only one MH_EXECUTE image
}
```

对于所有平台，Objective-C 运行时会为 `OS_object` 类及其子类禁用非指针型 isa（non-pointer isa）特性，因为 libdispatch 也会将 isa 指针用作虚方法表（vtable）。

```
else if (!hackedDispatch  &&  0 == strcmp(ro->getName(), "OS_object")) {
    // hack for libdispatch et al - isa also acts as vtable pointer
    hackedDispatch = true;
    instancesRequireRawIsa = true;
}
```

**has_assoc, has_cxx_dtor, weakly_referenced, 以及 has_sidetable_rc**

这四个字段的主要作用是判断一个对象能否使用快速释放路径（fast deallocation path），该路径仅通过 `free()` 释放对象内存。否则，运行时必须在释放内存前执行额外的簿记工作（bookkeeping）。相关代码来自 `objc-object.h`：

```
void objc_object::rootDealloc() {
    if (isTaggedPointer()) return;

    if (isa().nonpointer                     &&
        !isa().weakly_referenced             &&
        !isa().has_assoc                     &&
#if ISA_HAS_CXX_DTOR_BIT
        !isa().has_cxx_dtor                  &&
#else
        !isa().getClass(false)->hasCxxDtor() &&
#endif
        !isa().has_sidetable_rc)
    {
        free(this);
    } else {
        object_dispose((id)this);
    }
}
```

**has_assoc**：若对象通过 `objc_setAssociatedObject()` 运行时 API 创建了关联对象（associated object），则该位被置位。当对象拥有一个或多个关联对象时，运行时必须在释放对象内存前从其侧表（side table）中移除相关条目。

**has_cxx_dtor**：若该类或其父类包含 `.cxx_destruct` 方法，则该位被置位。当 Objective-C 对象拥有一个或多个 C++ 类型的实例变量（ivar）时[1]，运行时会在对象分配期间（早于任何 `init` 方法）调用类的 `.cxx_construct` 实例方法以执行所有非平凡构造函数。在 `dealloc` 方法链完成后，运行时会调用类的 `.cxx_destruct` 实例方法执行所有非平凡析构函数，随后才释放对象内存。

当自动引用计数（Automatic Reference Counting, ARC）启用时，编译器会在类的 `.cxx_destruct` 方法中实现实例变量的释放，这会阻止直接调用 `free()` 的优化。`object_dispose()` 的代码路径会调用 `objc_destructInstance()`，该函数若可用则会利用非指针 isa（isa）中的位来规避不必要的清理操作。

注意：当启用指针认证（pointer authentication）时，此位不可用，但需要付出额外内存加载的代价，才能在类对象中获取该信息。

weakly_referenced：每当创建指向对象的弱引用（weak reference）[2]时设置。与关联对象类似，运行时必须在释放对象内存前移除其侧表中的条目。

has_sidetable_rc：如果保留计数（retain count）已溢出 extra_rc，侧表将存储额外的保留计数，运行时同样必须在释放对象内存前移除这些条目。

shiftcls 与 shiftcls_and_sig

这些字段为紧凑型 isa 变体存储类指针（class pointer）。类对象始终按 8 字节对齐（无论是在二进制镜像中的布局，还是运行时通过标准分配器实现），因此最低 3 位始终为 0。所以，“shift class”字段存储的类指针会移除最低 3 位。此字段还依赖于对虚拟内存系统所允许的最大指针值的了解，因为将该值存储于位字段时会截断其高位比特。（译注：现代系统的指针宽度可能已超出此设计假设）

在 Apple Silicon 上，运行时使用 Pointer Authentication（指针认证）对类指针进行签名（并将其存储在 `shiftcls_and_sig` 字段中）。除了有效指针值的下界和上界外，该字段还依赖于对指针认证所使用比特位的知识。

`indexcls`

Apple Watch 上的 Objective-C 运行时将类指针存储在一个数组中，并将类的数组索引存储在 `isa` 的 `indexcls` 字段中。索引在运行时被懒惰地分配，如果数组的容量（32,767 个条目）耗尽，运行时会回退到使用指针 `isa`。

Apple Watch 的 ABI（应用程序二进制接口）使用 32 位指针[3]，这些指针没有足够的未使用比特位来同时存储指针值和打包的比特位。使用数组存储类指针减少了类标识所需的比特位数量，以一些间接性为代价实现了非指针 `isa` 的性能优势。

`magic`

该字段未被运行时使用。运行时导出魔术掩码和魔术值的常量，调试器使用这些常数来标识具有非指针 `isa` 的对象实例，以启用 Objective-C 调试功能。

`extra_rc`

当平台或类层级不支持 non-pointer isa（非指针 isa）功能时，对象实例的 retain count（保留计数）会存储在一个 side table（侧表）中。然而，对于大量并发的保留或释放操作，使用 side table 可能成为性能瓶颈，因为每次操作都必须获取锁。non-pointer isa 功能通过将实例的 retain count 存储在该字段中，从而减少了这种竞争。

如果 retain count 溢出了该字段，一半的保留计数将被移动到 side table，另一半则保留在该字段中。仅移动一半的计数，使得未来的释放操作可以先递减该字段的值，然后再获取锁以访问 side table 中的计数。

该字段的大小因平台而异。下表列出了每个平台上该字段的大小及最大内联保留计数值。

**64 位 Intel 上的 Packed isa** &emsp; 8 &emsp; 255

**Apple Silicon arm64-not-e 变体上的 Packed isa** &emsp; 19 &emsp; 524,287

**Apple Silicon 上带指针认证（arm64e 变体）的 Packed isa** &emsp; 8 &emsp; 255

**Indexed isa（Apple Watch）** &emsp; 7 &emsp; 127