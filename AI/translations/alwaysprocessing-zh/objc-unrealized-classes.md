# Objective-C Internals: Unrealized Classes (and Toll-Free Bridging)

> 原文：https://alwaysprocessing.blog/2023/02/16/objc-unrealized-classes
> 发布：2023-02-16　作者：Brian T. Kelley
> 译者：MiMo；代码块保留英文原样　|　仅供个人学习，未公开发布

---

Objective-C中的“未实现的类”可能指代未来类或类存根。未来类（一种私有运行时特性）用于支持与CoreFoundation的无缝桥接（Toll-Free Bridging）。而类存根则由Swift编译器生成，以支持稳定的Swift ABI与Objective-C之间的互操作性。

一篇早期探讨Objective-C类实现的文章忽略了[1]用于识别元类（Metaclass）的函数中一个有趣的细节：未实现的类这一概念。

```
// Like isMetaClass, but also valid on un-realized classes
bool isMetaClassMaybeUnrealized() { /* ... */ }
```

一个未实现类（unrealized class）是部分初始化的元类（metaclass），其中仅类名已知。未实现的元类有两种类型：未来类（future classes）和类存根（class stubs）。

**未来类**

Mac OS X 10.5 中的 Objective-C 运行时引入了未来类（future classes）作为一种私有运行时功能，用于清理无缝桥接（toll-free bridging）的实现。令人意外的是，这实际上在 `objc_getFutureClass` 函数的文档中有记载：

> 供 CoreFoundation 的无缝桥接使用。请勿自行调用此函数。

**无缝桥接**

首先，我们简要了解一下无缝桥接的实现。Mac OS X Developer Preview 1 引入了 CoreFoundation 框架和“无缝桥接”（toll-free bridging）。简而言之，无缝桥接允许开发人员在已桥接的 CoreFoundation 类型和 Foundation 类型之间自由进行强制类型转换（例如，将 `CFArrayRef` 转换为/自 `NSArray *`），从而像使用另一种类型一样传递一种类型，而不会产生任何运行时开销。此外，无缝桥接的 Foundation 类的用户自定义子类也得到了完全支持。

对无缝桥接（toll-free bridging）的全面探讨超出了本文范围，但您可在此处阅读更多内容。我们只需强调两个关键实现细节，以便理解其对 Objective-C 运行时（Objective-C runtime）的运用。

首先，所有 CoreFoundation 对象均以其首成员包含一个兼容 Objective-C 的 isa 字段（isa field）。来自 CFRuntime.h：

```
typedef struct __CFRuntimeBase {
    void *_isa;
    // ...
} CFRuntimeBase;
```

其次，每个具有 Foundation 等价物的 CoreFoundation 函数（例如 `CFArrayGetCount()` 和 `-[NSArray count]`），如果其 `isa` 与桥接的 CoreFoundation 类型的 `isa` 不匹配，就会调用 Objective-C 实现。这为用户定义的 Foundation 子类提供了桥接支持。（CFInternal.h 中定义了 Objective-C 分派宏。）

```
CFIndex CFArrayGetCount(CFArrayRef array) {
    CF_OBJC_FUNCDISPATCH0(__kCFArrayTypeID, CFIndex, array, "count");
    __CFGenericValidateType(array, __kCFArrayTypeID);
    return __CFArrayGetCount(array);
}

#define CF_OBJC_FUNCDISPATCH0(typeID, rettype, obj, sel) \
    if (__builtin_expect(CF_IS_OBJC(typeID, obj), 0)) \
    {rettype (*func)(const void *, SEL) = (void *)__CFSendObjCMsg; \
    static SEL s = NULL; if (!s) s = sel_registerName(sel); \
    return func((const void *)obj, s);}

CF_INLINE int CF_IS_OBJC(CFTypeID typeID, const void *obj) {
    return (((CFRuntimeBase *)obj)->_isa != __CFISAForTypeID(typeID) && ((CFRuntimeBase *)obj)->_isa > (void *)0xFFF);
}
```

CoreFoundation如何获取与Foundation共享的isa指针是本节剩余部分的主题。

**Mac OS X 10.0 - Mac OS X 10.4 "Tiger"**

免费桥接（toll-free bridging）的原始实现相当复杂。Foundation定义了公共的Objective-C类（例如 `NSObject`、`NSArray`、`NSMutableArray`）、类簇（class clusters）的私有实现（例如 `NSCFArray`），以及一个桥接占位符（bridging placeholder）（例如 `NSCFArray__`），它是桥接实现的一个简单子类。

当Foundation被加载时，它会调用CoreFoundation的 `__CFSetupFoundationBridging()` 函数，该函数执行两项操作：

调用 `__CFInitialize()`。最初的初始化步骤之一是为 `__CFRuntimeObjCClassTable` 中的每个条目分配内存，该表将CFTypeIDs映射到其桥接的Objective-C `Class`。这里分配的 `objc_class`（回想一下，`Class` 是 `objc_class *` 的typedef）将在下一步成为CoreFoundation和Foundation之间共享的免费桥接类实例。该指针值在初始化过程早期就被预留，以便在后续初始化步骤中分配的CoreFoundation对象可以使用，从而保证这些对象能正确桥接。

对于每个桥接类型，首先查找其占位子类，若存在则调用 `_CFRuntimeSetupBridging()`。

`_CFRuntimeSetupBridging()` 会将占位子类的 `objc_class` 结构体（译注：此为 Objective-C 运行时数据结构）进行按位拷贝，复制到上一步分配的内存空间中。

随后，CoreFoundation 会将其按位拷贝得到的占位类呈现为该桥接类型的最终实现。

```
void __CFSetupFoundationBridging(void *, void *, void *, void *) {
    // ...
    __CFInitialize();
    // ...
    Class aClass = objc_lookUpClass("NSCFArray__");
    if (arrayClass != Nil) {
        _CFRuntimeSetupBridging(CFArrayGetTypeID(), aClass->super_class, aClass);
    }
    aClass = objc_lookUpClass("NSCFDictionary__");
    if (aClass != Nil) {
        _CFRuntimeSetupBridging(CFDictionaryGetTypeID(), aClass->super_class, aClass);
    }
    // ...
}

void __CFInitialize(void) {
    // ...
    __CFRuntimeObjCClassTable[CFDictionaryGetTypeID()] = calloc(sizeof(struct objc_class), 1);
    __CFRuntimeObjCClassTable[CFArrayGetTypeID()] = calloc(sizeof(struct objc_class), 1);
    // ...
}

Boolean _CFRuntimeSetupBridging(CFTypeID typeID, struct objc_class *mainClass, struct objc_class *subClass) {
    void *isa = __CFISAForTypeID(typeID);
    memmove(isa, subClass, sizeof(struct objc_class));
    class_poseAs(isa, mainClass);
    return true;
}
```

**Mac OS X 10.5 "Leopard" 及以后版本，以及 iOS**

Posing 是 Objective-C 运行时（Runtime）中的一个已弃用特性（在 Objective-C 2 中已被移除），它允许一个子类有效地取代其父类的身份。执行 posing 的类会采用原始类的名称，为了保持名称的唯一性，运行时随后会在原始类的名称前添加一个 `%`，并在原始元类（metaclass）的名称前添加 `_%`。因此，继续以数组为例，`NSCFArray__` 就变成了 `NSCFArray`，而原始的 `NSCFArray` 则变成了 `%NSCFArray`。

被 pose 的类实例会接收到所有发送给原始类实例的消息。在上述桥接完成后，任何发送给 `NSCFArray` 类的消息（例如 `alloc`），都会被正在 pose 为 `NSCFArray` 的类（即 `NSCFArray__`）所接收。因此，这种重定向会导致 Foundation 使用被 pose 的类类型来创建新对象。随后，当将一个 Foundation 对象作为 `CFTypeRef` 传递给 Core Foundation 时，`CF_IS_OBJC` 会返回 false，因为该对象的 `isa` 将其标识为私有的桥接类型（因此不需要 Objective-C 分派到用户定义的子类）。由于它是一个私有类，Core Foundation 可以直接访问其内部实现。

未来类（future classes）大幅简化了无缝桥接（toll-free bridging）的实现。在 Mac OS X 10.5 及更高版本中，桥接配置仍然在 `__CFInitialize()` 中完成，但存在几个关键差异：

`__CFInitialize()` 在动态链接器加载框架时被调用，从而消除了对下游代码的任何初始化依赖。

对于每个桥接类型，CoreFoundation 会使用类名调用 `objc_getFutureClass()`，并用运行时返回的 `Class` 指针填充 `__CFRuntimeObjCClassTable`。

如果该类已被加载，运行时会返回其实例指针。

否则，运行时会分配一个具有给定名称的 `objc_class` 实例，并返回指向这个"未来类"的指针。然后，当进程随后加载具有该名称的类时，运行时会将类定义复制到先前分配的内存中（类似于早期版本中的 `_CFRuntimeSetupBridging()`），并从二进制映像加载的类定义重映射到先前分配的类定义（类似于 posing）。尽管存在相似之处，这种策略同时简化了 Objective-C 运行时和 CoreFoundation 的实现。

以下是 Leopard 系统中新版 Core Foundation 实现的近似描述。

```
static void __CFInitialize(void) __attribute__ ((constructor));
static void __CFInitialize(void) {
    // ...
    _CFRuntimeBridgeClasses(0x10/*CFDictionaryGetTypeID()*/, "NSCFDictionary");
    _CFRuntimeBridgeClasses(0x11/*CFArrayGetTypeID()*/, "NSCFArray");
    // ...
}

void _CFRuntimeBridgeClasses(CFTypeID cf_typeID, const char *objc_classname) {
    __CFRuntimeObjCClassTable[cf_typeID] = objc_getFutureClass(objc_classname);
}
```

未来类（future class）无需在进程生命周期内实例化。但如果未来类接收到任何消息，进程将会崩溃。我抽查了使用 Objective-C 运行时函数处理未来类的情况，测试的函数均能正常工作——尽管这纯属偶然。

**存根类（Stub Classes）**

macOS 10.15 和 iOS 13 的 Objective-C 运行时引入了存根类以支持稳定的 Swift ABI（应用程序二进制接口）。当满足以下条件时，Swift 编译器会生成存根类：

1. Swift 类型通过 `@objc` 特性声明为可在 Objective-C 中表示（包括所有直接或间接继承自 `NSObject` 的类）。
2. 上述第 (1) 条中的 Swift 类被编译进启用了库演进（Library Evolution）的动态库（包括框架）中（需使用 `-enable-library-evolution` 构建标志）。库演进也被称为弹性（resilience）或 ABI 稳定性。
3. 另一个模块中的 Swift 类导入了第 (2) 条中的模块，并继承了第 (1) 条中定义的类。当这个派生 Swift 类被编译时，编译器会将 Objective-C 类元数据（metadata）生成为类存根（class stub）。

我对这个场景为何需要存根进行了粗略调查，但未能得出结论。我想答案只能留给Swift内部原理系列文章了🙃。不过，我们可以看看Objective-C运行时（Runtime）如何处理类存根（stub class）。

一个isa值为1到15（含）的值会将一个`objc_class`实例标识为存根类。目前，除1以外的其他isa值均被保留。

```
bool isStubClass() const {
    uintptr_t isa = (uintptr_t)isaBits();
    return 1 <= isa && isa < 16;
}
```

若调用 `objc_getClassList()` 或 `objc_copyClassList()`，运行时（runtime）将按需初始化所有已加载到进程中的桩类（stub classes）。否则，桩类会在需要该类对象时按需初始化。

Swift 编译器生成的 Objective-C 头文件会为类接口添加 `__attribute__((objc_class_stub))` 属性，这指示 Clang 通过调用 `objc_loadClassref()` 来获取类对象，而不是直接引用类符号。该运行时函数会在首次访问时调用 Swift 初始化器来生成 `Class` 对象，并将结果存储在桩类中供后续访问使用。

我期待着有朝一日能撰写一篇文章，解释究竟是哪个 Swift 特性使得这种额外的间接寻址成为必要！