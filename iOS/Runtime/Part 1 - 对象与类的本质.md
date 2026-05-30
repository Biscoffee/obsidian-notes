# 【iOS】Runtime - Part 1 && 对象与类的本质

## 目录

- [Runtime 简介](#runtime-简介)
- [对象的本质：objc_object](#对象的本质objc_object)
  - 1.1 最小对象：只有一个 isa
  - 1.2 isa 不是裸指针：union isa_t（nonpointer isa）
  - 1.3 ISA_BITFIELD 逐位拆解（arm64 vs arm64e；951.1 的 PAC 变化 shiftcls_and_sig）
  - 1.4 实战(LLDB)：用 ISA_MASK 从 isa 还原 Class
- [对象的内存布局](#对象的内存布局)
  - 2.1 实例对象里装了什么：isa + 成员变量(ivar)
  - 2.2 对象多大：class_getInstanceSize（8 字节对齐）vs malloc_size（16 字节最小）
  - 2.3 实战：NSObject 实例为什么是 8 需求 / 16 实分
- [类的本质：objc_class](#类的本质objc_class)
  - 3.1 类本身也是对象（objc_class : objc_object）
  - 3.2 objc_class 四大件：isa / superclass / cache / bits
  - 3.3 bits 里的乾坤：class_data_bits_t → class_rw_t →（class_ro_t）
  - 3.4 ro 与 rw 的分离：编译期只读 vs 运行期可写（WWDC2020 dirty memory 优化）
- [元类 metaclass](#元类-metaclass)
  - 4.1 类方法存哪？→ 引出元类
  - 4.2 isa 的去向：对象 → 类 → 元类 → 根元类
  - 4.3 闭环：根元类的 isa 指向自己
- [isa 走位与继承链](#isa-走位与继承链)
  - 5.1 isa 走位图（Mermaid）
  - 5.2 superclass 继承链：类链与元类链平行
  - 5.3 实战：object_getClass / 打印元类地址，逐格验证整张图
- [At Last](#at-last)

# Runtime 简介

写 Objective-C 的人,每天都在敲这样的代码:

```objc
[person sayHello];
```

我们几乎是条件反射地把它读成「调用 person 的 sayHello 方法」。但这其实是一个被其他语言的思维带偏的误读。在 Objective-C 里,方括号语法真正的含义是:**向 person 这个对象,发送一条名为 `sayHello` 的消息。** 编译器最终会把这一行翻译成一次再普通不过的 C 函数调用:

```objc
objc_msgSend(person, @selector(sayHello));
```

到这里,一个平时被我们彻底忽略的问题就浮现了 : **到底是在哪个时刻、由谁来决定 person 真正去执行哪一段代码?**

如果是 C++,这个问题的答案大多在编译期就尘埃落定了——一次普通的成员函数调用,编译完地址就基本刻死在了二进制里。但 Objective-C 偏偏选了另一条路:它把「这个对象到底是什么类」「这条消息对应哪一份方法实现」这些决定,统统推迟到程序跑起来的那一刻,交给 `objc_msgSend` 现场查找、现场拍板。这种「推迟到运行时再决定」的能力,正是我们口中「动态」二字的真正含义。

而支撑起这套动态机制的,是一个始终活在你程序里的库——运行时(Runtime)。可以用一个粗糙但好记的等式来概括它:

> **Objective-C ≈ C 语言 + 一个运行时库。**

那些在别的语言里编译期就拍板的事,在 OC 里都被交给了这个运行时。这个系列要讲的,正是它究竟是如何工作的。

更具体一点：runtime 是一套用 C（和少量汇编）编写的 API 库，它是 OC「动态性」与「消息发送」机制得以成立的底座。程序启动时，正是它负责把 OC 的类结构注册起来、把分类(Category)的方法整合进宿主类——这些「编译期没做完、留到运行期才落定」的活儿，都由它接手。

我们平时用到的很多能力，底层其实都是 runtime 在支撑：

1. 消息发送与转发机制
2. 动态获取类的属性列表、方法列表等
3. 关联对象（给分类「加」属性）
4. 方法交换 Method Swizzling（俗称 iOS 黑魔法）
5. 分类(Category)的实现
6. KVO 的底层原理

这系列博客也会在后期对每个部分进行深入讲解

# 对象的本质：objc_object

我们首先打开源码工程，全局搜索 (`command + shift + F`)
 你会搜到两个结果，这正是本章的核心对比：

这里是入口：他说明OC对象底层至少有一个isa，isa用来找到对象对应的类
```objective-C
/// Represents an instance of a class.
struct objc_object {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;
};
```

```objc
**struct** objc_object {

**private**:

    **char** isa_storage[**sizeof**(isa_t)];

  

    isa_t &isa() { **return** ***reinterpret_cast**<isa_t *>(isa_storage); }

    **const** isa_t &isa() **const** { **return** ***reinterpret_cast**<**const** isa_t *>(isa_storage); }

  

**public**:

  

    // ISA() assumes this is NOT a tagged pointer object

    Class ISA(**bool** authenticated = **false**) **const**;

  

    // rawISA() assumes this is NOT a tagged pointer object or a non pointer ISA

    Class rawISA() **const**;

  

    // getIsa() allows this to be a tagged pointer object

    Class getIsa() **const**;

    uintptr_t isaBits() **const**;

  

    // initIsa() should be used to init the isa of new objects only.

    // If this object already has an isa, use changeIsa() for correctness.

    // initInstanceIsa(): objects with no custom RR/AWZ

    // initClassIsa(): class objects

    // initProtocolIsa(): protocol objects

    // initIsa(): other objects

    **void** initIsa(Class cls /*nonpointer=false*/);

    **void** initClassIsa(Class cls /*nonpointer=maybe*/);

    **void** initProtocolIsa(Class cls /*nonpointer=maybe*/);

    **void** initInstanceIsa(Class cls, **bool** hasCxxDtor);

  

    // changeIsa() should be used to change the isa of existing objects.

    // If this is a new object, use initIsa() for performance.

    Class changeIsa(Class newCls);

  

    **bool** hasNonpointerIsa() **const**;

    **bool** isTaggedPointer() **const**;

    **bool** isBasicTaggedPointer() **const**;

    **bool** isExtTaggedPointer() **const**;

    **bool** isClass() **const**;

  

    // object may have associated objects?

    **bool** hasAssociatedObjects() **const**;

    **void** setHasAssociatedObjects();

  

    // object may be weakly referenced?

    **bool** isWeaklyReferenced() **const**;

    **void** setWeaklyReferenced_nolock();

  

    // object may be uniquely referenced?

    **bool** isUniquelyReferenced() **const**;

  

    // object may have -.cxx_destruct implementation?

    **bool** hasCxxDtor() **const**;

  

    // Optimized calls to retain/release methods

    **id** retain();

    **void** release();

    **id** autorelease();

  

    // Implementations of retain/release methods

    **id** rootRetain();

    **bool** rootRelease();

    **id** rootAutorelease();

    **bool** rootTryRetain();

    **bool** rootReleaseShouldDealloc();

    uintptr_t rootRetainCount() **const**;

  

    // Implementation of dealloc methods

    **bool** rootIsDeallocating() **const**;

    **void** clearDeallocating();

    **void** rootDealloc();

  

**private**:

    **void** initIsa(Class newCls, **bool** nonpointer, **bool** hasCxxDtor);

  

    // Slow paths for inline control

    **id** rootAutorelease2();

  

#if SUPPORT_NONPOINTER_ISA

    // Controls what parts of root{Retain,Release} to emit/inline

    // - Full means the full (slow) implementation

    // - Fast means the fastpaths only

    // - FastOrMsgSend means the fastpaths but checking whether we should call

    //   -retain/-release or Swift, for the usage of objc_{retain,release}

    **enum** **class** RRVariant {

        Full,

        Fast,

        FastOrMsgSend,

    };

  

    // Unified retain count manipulation for nonpointer isa

    **inline** **id** rootRetain(**bool** tryRetain, RRVariant variant);

    **inline** **bool** rootRelease(**bool** performDealloc, RRVariant variant);

    **id** rootRetain_overflow(**bool** tryRetain);

    uintptr_t rootRelease_underflow(**bool** performDealloc);

  

    **void** clearDeallocating_slow();

  

    // Side table retain count overflow for nonpointer isa

    **struct** SidetableBorrow { size_t borrowed, remaining; };

  

    **void** sidetable_lock() **const**;

    **void** sidetable_unlock() **const**;

  

    **void** sidetable_moveExtraRC_nolock(size_t extra_rc, **bool** isDeallocating, **bool** weaklyReferenced);

    **bool** sidetable_addExtraRC_nolock(size_t delta_rc);

    SidetableBorrow sidetable_subExtraRC_nolock(size_t delta_rc);

    size_t sidetable_getExtraRC_nolock() **const**;

    **void** sidetable_clearExtraRC_nolock();

#endif  SUPPORT_NONPOINTER_ISA 

  

    // Side-table-only retain count

    **bool** sidetable_isDeallocating() **const**;

    **void** sidetable_clearDeallocating();

  

    **bool** sidetable_isWeaklyReferenced() **const**;

    **void** sidetable_setWeaklyReferenced_nolock();

  

    **id** sidetable_retain(**bool** locked = **false**);

    **id** sidetable_retain_slow(SideTable& table);

  

    uintptr_t sidetable_release(**bool** locked = **false**, **bool** performDealloc = **true**);

    uintptr_t sidetable_release_slow(SideTable& table, **bool** performDealloc = **true**);

  

    **bool** sidetable_tryRetain();

  

    uintptr_t sidetable_retainCount() **const**;

#if DEBUG

    **bool** sidetable_present() **const**;

#endif

  

    **void** performDealloc();

};
```

![isa_storage_memory_layout.svg](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/isa_storage_memory_layout.svg)

![CleanShot 2026-05-30 at 13.17.22@2x.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/CleanShot%202026-05-30%20at%2013.17.22%402x.png)


在看新版怎么收口之前，先把旧版长什么样摆出来对比。这个结构其实经历了三个时代：

```cpp
// ① 古早版（objc4-750 及更早，sunnyxx / draveness 那批老博客里的样子）
struct objc_object {
    isa_t isa;        // public：外部能直接 obj->isa 摸到
};
```

```cpp
// ② objc4-818
struct objc_object {
private:
    isa_t isa;        // 已经 private，但仍是一个「有类型的 isa_t 成员」
public:
    Class ISA(bool authenticated = false);   // 强制走方法
    ...
};
```

```cpp
// ③ objc4-951.1（本文这版）
struct objc_object {
private:
    char  isa_storage[sizeof(isa_t)];        // 退化成「一坨裸字节」
    isa_t &isa() { return *reinterpret_cast<isa_t *>(isa_storage); }  // 只能借方法戴上 isa_t 这副眼镜
public:
    Class ISA(bool authenticated = false) const;
};
```

我们可以看到， isa_storage是真正存isa的地方。旧版本的写法是直接暴露 `isa_t isa` 作为 public 成员，任何人都能直接读写。新版本改成：`**char** isa_storage[**sizeof**(isa_t)];`  使用一个char 数组来占位。arm64e上，isa里面那根指针是被签名过的，如果有一个公开有类型的 isa_t  isa，外部代码就能直接obj->isa.cls直接摸到那根带签名的指针
——读出来是“看着像乱码”的值，甚至可能绕过验证。修改后，想要接触这8个字节都要经过isa()访问器，isa_t里面的 cls又被设置为private，因此你必须`getClass()/setClass()`

- `char` 在 C++ 标准里是"字节类型"，用它做原始存储是合法的 type-punning，不触发 UB（Undefined Behavior）。直接在 union 成员之间互相访问在 C++ 里有严格限制，但通过 `char[]` 中转是标准允许的。

- `char[]` 不会触发任何构造函数或析构函数。`isa_t` 是一个 union，如果直接作为成员，某些编译器版本可能对 union 成员的初始化有额外的限制，而 `char[]` 完全透明、惰性，什么都不做。

- 给对象内部开一块内存，大小刚好等于isa_t，所以实际的内存布局也没有变化

总的来说，这是一次**封装重构**，内存布局和 isa_t 的位域语义完全没变，只是把裸字段换成了私有字节数组 + 私有访问器，防止外部绕过 Runtime 直接操作 isa，同时避免 C++ union 直接访问的潜在 UB 问题。

再回头看这两行，其实它们都指向同一个地方：

```cpp
isa_t &isa() { return *reinterpret_cast<isa_t *>(isa_storage); }  // 内部访问
Class ISA(bool authenticated = false) const;                     // 对外取类
```

不管是内部用的 `isa()`，还是对外的 `ISA()`，它们去读那 8 个裸字节时，都是指的同一个——`isa_t`。换句话说，`objc_object` 这个壳本身没几两肉，它把「对象到底属于哪个类、引用计数是多少、有没有关联对象、是否被弱引用」这些信息，**全都打包压进了 `isa_t` 这 8 个字节里**。

所以「对象的本质是什么」这个问题，到这里就收敛成了一个更具体、也更要命的问题：`isa_t`












# At Last

## 参考与感谢

本文在学习和整理 Objective-C Runtime 相关内容时，参考了以下优秀资料，在此表示感谢：

1. [Apple Developer Documentation - Objective-C Runtime](https://developer.apple.com/documentation/objectivec/objective-c-runtime)

2. [Apple Open Source - objc4 Runtime Source Code](https://opensource.apple.com/source/objc4/)

3. [WWDC 2020 - Advancements in the Objective-C runtime](https://developer.apple.com/videos/play/wwdc2020/10163/)

4. [Mike Ash - Friday Q&A 2009-03-20: Objective-C Messaging](https://www.mikeash.com/pyblog/friday-qa-2009-03-20-objective-c-messaging.html)

5. [Mike Ash - Friday Q&A 2017-06-30: Dissecting objc_msgSend on ARM64](https://www.mikeash.com/pyblog/friday-qa-2017-06-30-dissecting-objc_msgsend-on-arm64.html)

6. [Cocoa Samurai - Understanding the Objective-C Runtime](https://cocoasamurai.blogspot.com/2010/01/understanding-objective-c-runtime.html)

7. [Always Processing - Objective-C Internals](https://alwaysprocessing.blog/series/objc-internals)

8. [sunnyxx - 重识 Objective-C Runtime](https://blog.sunnyxx.com/2016/08/13/reunderstanding-runtime-0/)

9. [sunnyxx - 神经病院 Objective-C Runtime 入院第一天](https://blog.sunnyxx.com/2014/11/06/runtime-nuts/)

10. [sunnyxx - objc category 的秘密](https://blog.sunnyxx.com/2014/03/05/objc_category_secret/)

11. [Draveness - 深入解析 ObjC 中方法的结构](https://draveness.me/method-struct/)

12. [Draveness - 你真的了解 load 方法么？](https://draveness.me/load/)

13. [Draveness - 关联对象 AssociatedObject 完全解析](https://draveness.me/ao/)

14. [BOB's Blog - Objective-C Runtime 相关优化与底层分析](https://blog.devtang.com/)

15. [bestswifter - 深入理解 Objective-C Runtime](https://github.com/bestswifter/blog/blob/master/articles/objc-runtime.md)

16. [Garan no dou - Objective-C 中的类和对象](https://blog.ibireme.com/)

17. [Tenloy's Blog - ObjC Runtime 总结](https://tenloy.github.io/)

---

感谢以上作者和资料对 Objective-C Runtime、消息发送、isa、类与元类、方法缓存、Category、关联对象、`+load` 等内容的深入分析。

感谢 `Opus 4.7  GPT 5.5  Mimo 2.5 pro`模型在搭建环境、调试、解析方面不可忽视的贡献。

本文仅作为个人学习整理，若有理解不当之处，仍以 Apple 官方文档和 objc4 源码为准。

始于 2026.5.27
