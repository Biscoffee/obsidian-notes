# Objective-C Internals: Class Graph Implementation

> 原文：https://alwaysprocessing.blog/2023/01/10/objc-class-graph-impl
> 发布：2023-01-10　作者：Brian T. Kelley
> 译者：MiMo；代码块保留英文原样　|　仅供个人学习，未公开发布

---

简要浏览 Objective-C 运行时（runtime）源代码，聚焦于对象与类类型的定义，阐明继承（inheritance）的实现方式、根类（root class）的特殊处理，以及与元类（metaclass）查找相关的注意事项。

前文探讨了 Objective-C 类架构，并展示了类层次结构的对象图（object graph）。本文将在此基础上，通过检查类对象图实现（类、超类及元类）来深入探讨。

我们首先从一些关键类型的公开定义入手。在 Objective-C 中，`Class`（类）类型表示任何类类型，`id`（任意对象）类型表示任何类的实例。Objective-C 运行时头文件 `objc.h` 定义了这些类型：

```
/// An opaque type that represents an Objective-C class.
typedef struct objc_class *Class;

/// Represents an instance of a class.
struct objc_object {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;
};

/// A pointer to an instance of a class.
typedef struct objc_object *id;
```

如前文所述，Objective-C 的类本身也是对象，但这层关系在公开的类型定义中并未体现。然而，如果我们查阅内部的类型定义，便能发现这层关联。

首先，`objc-private.h` 文件包含了真正的 `objc_object` 定义。虽然内部定义包含许多非虚 C++ 成员函数，但其唯一的成员变量 `isa_storage` 对应着（已弃用的）`isa` 实例变量。（译注：`isa` 字段在较新的 Objective-C Runtime 中已改为使用 `isa_t` 联合体，并配合指针标记等技术。）我不清楚内部类型为何是字符数组，但若要我猜测，这可能是为了结合该字段的多种重载形式，防止意外直接使用。关于 `isa` 字段的更多讨论，可参见我之前的文章。

```
struct objc_object {
    char isa_storage[sizeof(isa_t)];
};
```

接下来，`objc-runtime-new.h` 包含了 `objc_class` 数据结构的定义。它拥有几个自身的成员变量，并且，就像 `objc_object` 一样，它也具有许多非虚 C++ 成员函数。

```
struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
};
```

没有专门针对元类的类型；元类（metaclass）只是一个类，尽管 `objc_class` 类型有内部机制来识别和操作元类。

这里，我们看到 `objc_class` 派生自 `objc_object`，因此继承了 isa（类对象指针）字段。所以，一个类对象就像任何其他对象类型一样实现。接下来是 superclass（父类指针）字段，它指向父类对象（如果存在）。（`cache` 和 `bits` 不是类图构造的一部分，所以我们将在未来探讨它们。）

这就是构建 Objective-C 类图所需的一切：两个数据结构（`objc_object` 和 `objc_class`）和两个字段（isa 和 superclass）！

**objc_class 成员函数**

接下来，让我们检查一些 `objc_class` 成员函数，以了解架构图中边的实现。

**根类**

```
bool isRootClass() {
    return getSuperclass() == nil;
}
```

**Root Metaclasses**

如果一个类没有超类（superclass），那么它就是根类（root class）。然而，这在实践中并不常见，因为几乎所有 Objective-C 对象都派生自 NSObject（或者，在少数情况下，派生自 NSProxy）。因此，请注意，根元类（root metaclass）并不是一个根类。

```
bool isRootMetaclass() {
    return ISA() == (Class)this;
}
```

根元类拥有一个自指的 `isa` 指针，这是运行时（runtime）识别根元类的方式。据我所知，这是类图中唯一的环。

**元类的同一性**

```
bool isMetaClass() const {
    return cache.getBit(FAST_CACHE_META);
}

// Like isMetaClass, but also valid on un-realized classes
bool isMetaClassMaybeUnrealized() {
    if (isStubClass())
        return false;
    return bits.flags() & RW_META;
}
```

编译器发出的一个位标志（bit flag）用于标识元类实例（metaclass instance），这是元类实例与类实例之间的主要区别。

未实现的类（unrealized classes），包括桩类（stub classes），在《Objective-C 内部机制：未实现的类（与无缝桥接）》一文中有更详细的描述。

**元类检索**

```
// NOT identical to this->ISA when this is a metaclass
Class getMeta() {
    if (isMetaClassMaybeUnrealized()) return (Class)this;
    else return this->ISA();
}
```

当从某个类实例获取 metaclass（元类）时，需要检查该实例是否已经是 metaclass。如果是 metaclass，它会返回自身。否则，该类实例会通过其 isa pointer（isa 指针）返回 metaclass。

**Compiler Output（编译器输出）**

`objc_class` 数据结构是 Objective-C ABI（应用程序二进制接口）的一部分，这意味着第三方程序知晓其大小和字段布局的详细信息，并将这些信息编码到它们的可执行二进制文件中。我们可以通过检查以下简单类定义的编译器输出来观察到这一点。

```
#import <Foundation/Foundation.h>

@interface MyObject: NSObject
@end

@implementation MyObject
@end
```

通过运行命令 `clang -S MyObject.m` 为上述 `MyObject.m` 文件生成汇编（assembly）代码，将产出一个包含以下代码片段（及更多内容）的汇编文件。

```
.section    __DATA,__objc_data
_OBJC_CLASS_$_MyObject:
    .quad   _OBJC_METACLASS_$_MyObject
    .quad   _OBJC_CLASS_$_NSObject
    .quad   __objc_empty_cache
    .quad   0
    .quad   __OBJC_CLASS_RO_$_MyObject
_OBJC_METACLASS_$_MyObject:
    .quad   _OBJC_METACLASS_$_NSObject
    .quad   _OBJC_METACLASS_$_NSObject
    .quad   __objc_empty_cache
    .quad   0
    .quad   __OBJC_METACLASS_RO_$_MyObject
```

在此，我们可以看到编译器生成的代码与我们从上一篇文章的架构图中得出的观察结果相吻合：

`MyClass` 类对象具有：
*   一个指向 `MyClass` 元类（metaclass）的 `isa` 变量。
*   一个指向 `NSObject` 类对象的 `super` 变量。

`MyClass` 元类具有：
*   一个指向 `NSObject`（根对象）元类的 `isa` 变量。
*   一个指向 `NSObject` 元类的 `super` 变量。

（如上所述，`cache` 和 `bits` 字段将是未来一篇文章的主题。）