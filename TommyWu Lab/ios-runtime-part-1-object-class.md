---
title: "【iOS】Runtime - Part 1 && 对象与类的本质"
published: 2026-05-31
description: "从 objc_object、isa_t、Tagged Pointer、对象内存布局一路拆到类与对象的本质，梳理 Objective-C Runtime 中对象如何关联到类。"
tags: ["iOS", "Objective-C", "Runtime", "isa", "objc4"]
category: "iOS"
series: "iOS Runtime 系列"
seriesSlug: "ios-runtime"
seriesOrder: 1
draft: false
---

## 目录

- [Runtime 简介](#runtime-简介)
- [对象的本质：objc_object](#对象的本质objc_object)
  - objc_object：对象的骨架
  - isa_t
  - ISA_BITFIELD：isa 的位布局
  - getClass：从 isa 里取出 Class
  - Tagged Pointer 优化
- [对象的内存布局](#对象的内存布局)
  - 实例里装了什么：isa + 成员变量
  - 对象多大：从「需求」到「实分」的三个数
  - 实战：NSObject 为什么是「8 需求 / 16 实分」
- [类的本质：objc_class](#类的本质objc_class)
  - 类本身也是对象
  - 类的四大件：isa / superclass / cache / bits
  - bits：class_rw_t → class_ro_t
  - ro 与 rw 的分离：clean memory vs dirty memory
- 元类 metaclass（待写）
- isa 走位与继承链（待写）
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

## objc_object：对象的骨架

我们首先打开源码工程，全局搜索 (`command + shift + F`)
 你会搜到两个结果，这正是本章的核心对比：

这里是入口：他说明OC对象底层至少有一个isa，isa用来找到对象对应的类
```objc
/// Represents an instance of a class.
struct objc_object {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;
};
```

```objc
struct objc_object {

private:

    char isa_storage[sizeof(isa_t)];



    isa_t &isa() { return *reinterpret_cast<isa_t *>(isa_storage); }

    const isa_t &isa() const { return *reinterpret_cast<const isa_t *>(isa_storage); }



public:



    // ISA() assumes this is NOT a tagged pointer object

    Class ISA(bool authenticated = false) const;



    // rawISA() assumes this is NOT a tagged pointer object or a non pointer ISA

    Class rawISA() const;



    // getIsa() allows this to be a tagged pointer object

    Class getIsa() const;

    uintptr_t isaBits() const;



    // initIsa() should be used to init the isa of new objects only.

    // If this object already has an isa, use changeIsa() for correctness.

    // initInstanceIsa(): objects with no custom RR/AWZ

    // initClassIsa(): class objects

    // initProtocolIsa(): protocol objects

    // initIsa(): other objects

    void initIsa(Class cls /*nonpointer=false*/);

    void initClassIsa(Class cls /*nonpointer=maybe*/);

    void initProtocolIsa(Class cls /*nonpointer=maybe*/);

    void initInstanceIsa(Class cls, bool hasCxxDtor);



    // changeIsa() should be used to change the isa of existing objects.

    // If this is a new object, use initIsa() for performance.

    Class changeIsa(Class newCls);



    bool hasNonpointerIsa() const;

    bool isTaggedPointer() const;

    bool isBasicTaggedPointer() const;

    bool isExtTaggedPointer() const;

    bool isClass() const;



    // object may have associated objects?

    bool hasAssociatedObjects() const;

    void setHasAssociatedObjects();



    // object may be weakly referenced?

    bool isWeaklyReferenced() const;

    void setWeaklyReferenced_nolock();



    // object may be uniquely referenced?

    bool isUniquelyReferenced() const;



    // object may have -.cxx_destruct implementation?

    bool hasCxxDtor() const;



    // Optimized calls to retain/release methods

    id retain();

    void release();

    id autorelease();



    // Implementations of retain/release methods

    id rootRetain();

    bool rootRelease();

    id rootAutorelease();

    bool rootTryRetain();

    bool rootReleaseShouldDealloc();

    uintptr_t rootRetainCount() const;



    // Implementation of dealloc methods

    bool rootIsDeallocating() const;

    void clearDeallocating();

    void rootDealloc();



private:

    void initIsa(Class newCls, bool nonpointer, bool hasCxxDtor);



    // Slow paths for inline control

    id rootAutorelease2();



#if SUPPORT_NONPOINTER_ISA

    // Controls what parts of root{Retain,Release} to emit/inline

    // - Full means the full (slow) implementation

    // - Fast means the fastpaths only

    // - FastOrMsgSend means the fastpaths but checking whether we should call

    //   -retain/-release or Swift, for the usage of objc_{retain,release}

    enum class RRVariant {

        Full,

        Fast,

        FastOrMsgSend,

    };



    // Unified retain count manipulation for nonpointer isa

    inline id rootRetain(bool tryRetain, RRVariant variant);

    inline bool rootRelease(bool performDealloc, RRVariant variant);

    id rootRetain_overflow(bool tryRetain);

    uintptr_t rootRelease_underflow(bool performDealloc);



    void clearDeallocating_slow();



    // Side table retain count overflow for nonpointer isa

    struct SidetableBorrow { size_t borrowed, remaining; };



    void sidetable_lock() const;

    void sidetable_unlock() const;



    void sidetable_moveExtraRC_nolock(size_t extra_rc, bool isDeallocating, bool weaklyReferenced);

    bool sidetable_addExtraRC_nolock(size_t delta_rc);

    SidetableBorrow sidetable_subExtraRC_nolock(size_t delta_rc);

    size_t sidetable_getExtraRC_nolock() const;

    void sidetable_clearExtraRC_nolock();

#endif  SUPPORT_NONPOINTER_ISA 



    // Side-table-only retain count

    bool sidetable_isDeallocating() const;

    void sidetable_clearDeallocating();



    bool sidetable_isWeaklyReferenced() const;

    void sidetable_setWeaklyReferenced_nolock();



    id sidetable_retain(bool locked = false);

    id sidetable_retain_slow(SideTable& table);



    uintptr_t sidetable_release(bool locked = false, bool performDealloc = true);

    uintptr_t sidetable_release_slow(SideTable& table, bool performDealloc = true);



    bool sidetable_tryRetain();



    uintptr_t sidetable_retainCount() const;

#if DEBUG

    bool sidetable_present() const;

#endif



    void performDealloc();

};
```

![isa_storage_memory_layout.svg](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/isa_storage_memory_layout.svg)

![CleanShot 2026-05-30 at 13.17.22@2x.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/CleanShot%202026-05-30%20at%2013.17.22%402x.png)

在看新版之前，先把旧版长什么样摆出来对比。这个结构其实经历了三个时代：

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

<iframe src="/posts/ios-runtime-part-1-object-class/isa-storage-to-isa-t-steps.html" title="isa_storage 到 isa_t 的解读过程" loading="lazy" style="width:100%;min-height:520px;border:1px solid var(--line-divider);border-radius:18px;background:#07110f;overflow:hidden;"></iframe>

所以「对象的本质是什么」这个问题，到这里就收敛成了：`isa_t` 里到底装了什么？


## isa_t

```objc
union isa_t {

    isa_t() { } //默认构造函数

    isa_t(uintptr_t value) : bits(value) { }  // 带参数的构造函数

    uintptr_t bits;   //isa_t 里面真正存的是一个和指针一样大的无符号整数(64)。



private:

    // Accessing the class requires custom ptrauth operations, so

    // force clients to go through setClass/getClass by making this

    // private.
   

    Class cls;
    // 这段放在 private 中，让你不能从外部通过isa.cls直接访问，注释的意思是说：访问 `Class` 指针时，可能需要做 **ptrauth 指针认证**。在 Apple 的 arm64e 架构上，指针可能带有签名，不能像普通地址一样随便读出来用。Runtime 需要通过专门逻辑去认证、解码、还原。



public:

#if defined(ISA_BITFIELD)
// 如果当前平台定义了 ISA_BITFIELD，就启用 isa 位域结构。
    struct {
        ISA_BITFIELD;  // defined in isa.h
    };


// 当前平台的 isa 是否支持把一部分引用计数直接存在 isa 里面。
#if ISA_HAS_INLINE_RC

    bool isDeallocating() const {

        return extra_rc == 0 && has_sidetable_rc == 0;

	//extra_rc：存在 isa 里的额外引用计数
	//has_sidetable_rc：是否还有引用计数存在 SideTable 里
    }

    void setDeallocating() {

        extra_rc = 0;

        has_sidetable_rc = 0;

    }

#endif // ISA_HAS_INLINE_RC



#endif  defined(ISA_BITFIELD) 



    void setClass(Class cls, objc_object *obj);

    Class getClass(bool authenticated) const;

    Class getDecodedClass(bool authenticated) const;

};
```

如上代码为 isa_t 联合体本体，**union 里所有成员，起始地址相同，共享同一块内存。bits、cls、ISA_BITFIELD struct 都是这块内存的**成员

<iframe src="/posts/ios-runtime-part-1-object-class/isa-t-three-views.html" title="isa_t 的三种视角" loading="lazy" style="width:100%;min-height:620px;border:1px solid var(--line-divider);border-radius:18px;background:#07110f;overflow:hidden;"></iframe>


## ISA_BITFIELD：isa 的位布局

我们接下来看看`ISA_BITFIELD`：
```objc
/*
 * isa 的位布局随架构而不同，共 4 套。下面把每个架构的 ISA_BITFIELD
 * 直接展开成位域列表（省去 #define 和续行符 \，方便阅读），并标注每段
 * 的 bit 范围。挑你的目标架构看即可：真机 A12+ 与模拟器看 ①。
 */

// ===== ① arm64e（真机 A12+ / 模拟器，__has_feature(ptrauth_calls)）=====
//   ISA_MASK = 0x007ffffffffffff8ULL    ·    无独立 has_cxx_dtor 位（移到 cache flags）
uintptr_t nonpointer        : 1;    // bit0     nonpointer isa 开关（0=纯类指针）
uintptr_t has_assoc         : 1;    // bit1     有无关联对象
uintptr_t weakly_referenced : 1;    // bit2     有无被 __weak 弱引用
uintptr_t shiftcls_and_sig  : 52;   // bit3-54  类指针 + PAC 签名（合并）
uintptr_t has_sidetable_rc  : 1;    // bit55    RC 是否溢出到 SideTable
uintptr_t extra_rc          : 8;    // bit56-63 内联引用计数（retainCount-1）

// ===== ② arm64（非 e，不开 PAC）=====
//   ISA_MASK = 0x0000000ffffffff8ULL    ·    保留 has_cxx_dtor / magic 位
uintptr_t nonpointer        : 1;    // bit0
uintptr_t has_assoc         : 1;    // bit1     有无关联对象
uintptr_t has_cxx_dtor      : 1;    // bit2     类/父类有无 C++ 析构（arm64e 已移除）
uintptr_t shiftcls          : 33;   // bit3-35  类指针（无签名，故名 shiftcls）
uintptr_t magic             : 6;    // bit36-41 调试期识别 isa 的魔数（arm64e 已移除）
uintptr_t weakly_referenced : 1;    // bit42    有无被 __weak 弱引用
uintptr_t unused            : 1;    // bit43    保留位
uintptr_t has_sidetable_rc  : 1;    // bit44    RC 是否溢出到 SideTable
uintptr_t extra_rc          : 19;   // bit45-63 内联引用计数（位数比 arm64e 多）

// ===== ③ x86_64（Intel Mac / 旧模拟器）=====
//   ISA_MASK = 0x00007ffffffffff8ULL
uintptr_t nonpointer        : 1;    // bit0
uintptr_t has_assoc         : 1;    // bit1
uintptr_t has_cxx_dtor      : 1;    // bit2
uintptr_t shiftcls          : 44;   // bit3-46  类指针
uintptr_t magic             : 6;    // bit47-52 魔数
uintptr_t weakly_referenced : 1;    // bit53
uintptr_t unused            : 1;    // bit54
uintptr_t has_sidetable_rc  : 1;    // bit55
uintptr_t extra_rc          : 8;    // bit56-63 内联引用计数

// ===== ④ armv7k / arm64_32（Apple Watch 等 32 位，索引式 isa）=====
//   存「类表索引 indexcls」而非类指针；32 位地址空间小，用索引更省位
uintptr_t nonpointer        : 1;    // bit0
uintptr_t has_assoc         : 1;    // bit1
uintptr_t indexcls          : 15;   // bit2-16  类表索引（不是指针！）
uintptr_t magic             : 4;    // bit17-20
uintptr_t has_cxx_dtor      : 1;    // bit21
uintptr_t weakly_referenced : 1;    // bit22
uintptr_t unused            : 1;    // bit23
uintptr_t has_sidetable_rc  : 1;    // bit24
uintptr_t extra_rc          : 7;    // bit25-31 内联引用计数
```


## getClass：从 isa 里取出 Class

刚才我们了解了，isa_t 长什么样、位怎么分布，接下来我们看看 Runtime是怎么从isa_t 里拿到 Class 的？

```objc
// 从 isa 里把 Class 指针抠出来。authenticated 决定要不要做 PAC 指针认证：
// 多数调用方对安全不敏感，默认 false 跳过认证换性能；只有 msgSend / 填缓存
// 这类安全攸关的路径才会传 true。
inline Class
isa_t::getClass(MAYBE_UNUSED_AUTHENTICATED_PARAM bool authenticated) const {
#if SUPPORT_INDEXED_ISA
    // 索引式 isa（armv7k / arm64_32）：cls 本身就是裸类指针，原样返回
    return cls;
#else
    uintptr_t clsbits = bits;          // 先拿到完整的 64 位 isa

#   if __has_feature(ptrauth_calls)    // ===== arm64e：类指针被 PAC 签过名 =====
#       if ISA_SIGNING_AUTH_MODE == ISA_SIGNING_AUTH
    if (authenticated) {
        // 要认证：先用 ISA_MASK 保留「类指针 + 签名」那 52 位（shiftcls_and_sig）
        clsbits &= ISA_MASK;
        if (clsbits == 0)
            return Nil;
        // 再用 ptrauth_auth_data 验签 + 还原出真正能用的指针；鉴别子由
        // this(本 isa 的地址) 和固定常量混合而成，地址不对就还原失败
        clsbits = (uintptr_t)ptrauth_auth_data((void *)clsbits, ISA_SIGNING_KEY,
                      ptrauth_blend_discriminator(this, ISA_SIGNING_DISCRIMINATOR));
    } else {
        // 不认证：直接用运行期算好的 objc_debug_isa_class_mask 把签名/标志/RC 抹掉
        clsbits &= objc_debug_isa_class_mask;
    }
#       else
    clsbits &= objc_debug_isa_class_mask;   // 编译配置不要求认证，同样走快路
#       endif

#   else                               // ===== arm64(非e) / x86_64：指针无签名 =====
    clsbits &= ISA_MASK;               // 一把 ISA_MASK 抹掉低位标志 + 高位 RC 即可
#   endif

    return (Class)clsbits;             // 剩下的就是一根干净的 Class 指针
#endif
}
```

## Tagged Pointer 优化

到这里我们讲的「对象」，都是堆上一块内存、开头一根 `isa` 的普通对象。但其实还有一类「对象」根本没有 `isa`——它就是 **Tagged Pointer（标记指针）**。

对一个 `NSNumber *n = @5` 这种**又小又高频**的值类型来说，为了一个 `5` 去 `malloc` 一块堆内存、维护 `isa`、再管引用计数，实在太奢侈。Tagged Pointer 的思路很直接：**干脆不分配内存，把「类型标记 + 数据本身」直接塞进那 8 字节的指针里。** 这个「指针」根本不指向任何地址，**它本身就是数据**——`@5` 里的 `5`，就藏在这根「指针」的二进制位里。

### 怎么判定一个指针是不是 Tagged Pointer

判定逻辑只有一行（`objc-internal.h`）：

```objc
static inline bool
_objc_isTaggedPointer(const void * _Nullable ptr)
{
    // 标记位被置 1 的，就不是真指针，而是 Tagged Pointer
    return ((uintptr_t)ptr & _OBJC_TAG_MASK) == _OBJC_TAG_MASK;
}
```

关键就在 `_OBJC_TAG_MASK` 这个「标记位」放在哪，随平台不同：

```objc
#if OBJC_SPLIT_TAGGED_POINTERS        // arm64（iOS 真机 / Apple Silicon）
#   define _OBJC_TAG_MASK (1UL<<63)   // 看最高位 bit63
#elif OBJC_MSB_TAGGED_POINTERS        // 非 arm64、非 x86 Mac 的其它配置
#   define _OBJC_TAG_MASK (1UL<<63)
#else                                 // x86_64 Mac
#   define _OBJC_TAG_MASK 1UL         // 看最低位 bit0
#endif
```

这就解释了一个常见现象：真机上打印一个 Tagged 的 `NSNumber`，地址常是 `0xb000...` / `0x8000...` 这种「最高位为 1 的怪地址」——因为那根本不是地址，是被置了标记位的数据。

### 它标记的是哪些类

指针里除了标记位，还存了一个 `tag`，用来标识「这是哪个类的对象」。`tag` 的取值来自一张枚举表（`objc-internal.h`，这里节选）：

```objc
enum objc_tag_index_t : uint16_t
{
    // tag 0..6：60 位载荷
    OBJC_TAG_NSAtom            = 0,
    OBJC_TAG_NSString          = 2,
    OBJC_TAG_NSNumber          = 3,
    OBJC_TAG_NSIndexPath       = 4,
    OBJC_TAG_NSManagedObjectID = 5,
    OBJC_TAG_NSDate            = 6,
    OBJC_TAG_RESERVED_7        = 7,   // 保留

    // tag 8..263：52 位扩展载荷（NSColor / UIColor / NSIndexSet ...）
    OBJC_TAG_Photos_1          = 8,
    OBJC_TAG_NSColor           = 16,
    OBJC_TAG_UIColor           = 17,
    // ...
};
```

所以 arm64 上那 8 个字节大致是这样分的：

```
bit63        标记位（=1 表示这是 Tagged Pointer）
低位若干 bit   tag —— 它是哪个类（NSNumber? NSString? NSDate?）
中间 bit      payload —— 真正的数据
```

tag 0–6 有 60 位载荷，扩展 tag 有 52 位载荷。**装不下就退回普通堆对象**——比如一个很大的整数、一个很长的字符串，就不会做成 Tagged Pointer。

### 没有 isa，它怎么找到类

这一点正好和前面 `getClass` 那节对照着看。普通对象靠 `isa.bits & mask` 拿到 `Class`；而 Tagged Pointer 根本没有 `isa`，于是 `getIsa()` 走了另一条路（`objc-object.h`）：

```objc
inline Class
objc_object::getIsa() const
{
    // 普通对象：还是走 isa
    if (fastpath(!isTaggedPointer())) return ISA(/*authenticated*/true);

    // Tagged Pointer：从指针里抠出 tag 当下标，去全局表里查类
    uintptr_t slot = ((uintptr_t)this >> _OBJC_TAG_SLOT_SHIFT) & _OBJC_TAG_SLOT_MASK;
    Class cls = objc_tag_classes[slot];   // 即 objc_debug_taggedpointer_classes[]
    // ...（扩展 tag 再查 ext 表）
    return cls;
}
```

它把指针里的 `tag` 当作下标，去 `objc_debug_taggedpointer_classes[]` 这张「tag → Class」的全局表里查出类。

这也顺带解释了前面 `objc_object` 源码里那一堆 `ASSERT(!isTaggedPointer())`——像 `ISA()` 开头就断言「我不是 tagged」，因为 `ISA()` 只处理真有 `isa` 的对象，**Tagged Pointer 必须改走 `getIsa()`**。

### 为什么内存里看到的值像「乱码」

iOS 8.3 之后，Tagged Pointer 的真实布局在存进指针前，会先和一个全局混淆值 `objc_debug_taggedpointer_obfuscator` 做一次**异或加扰**：

```objc
uintptr_t value = (obfuscator ^ ptr);   // 编码/解码都要异或这个值
```

目的是**防止开发者去硬编码、依赖它的内部位布局**（Apple 保留随时改布局的权利）。所以你直接在内存里看到的 Tagged 指针位是被混淆过的，要 decode 才能还原出真正的 tag 和 payload。

### 它带来的好处

- **零分配**：不 `malloc`、不进堆，创建和销毁几乎零成本；
- **引用计数免费**：Tagged 对象的 `retain / release` 直接是 no-op——源码里 `rootRetain` 开头就是 `if (isTaggedPointer()) return (id)this;` 原样返回；
- **访问快**：数据就在指针里，无需解引用去堆上读。

> **几点边界**
> 1. 不是所有 `NSNumber / NSString` 都是 Tagged——超出载荷容量（大数、长字符串）会退回普通堆对象。
> 2. Tagged 对象 `object_getClass` 能拿到类（如 `__NSCFNumber` / `NSTaggedPointerString`），但它**没有 isa 字段**，行为和普通对象不完全一样（retain 免费、布局被混淆）。
> 3. 布局随架构 / 系统版本会变（arm64 看 bit63、x86 Mac 看 bit0），别把某个具体 bit 位当成「永远如此」写死。


<iframe src="/posts/ios-runtime-part-1-object-class/isa-t-layout.html" title="isa_t 位布局" loading="lazy" style="width:100%;min-height:760px;border:1px solid var(--line-divider);border-radius:18px;background:#0b0b0f;overflow:hidden;"></iframe>


# 对象的内存布局

除了 isa，一个实例在内存里到底还装了什么、整个占多大？这一章就把这块内存量清楚。

## 实例里装了什么：isa + 成员变量

一个普通实例对象在堆上的内存，结构非常朴素：

```
偏移 +0     isa（8 字节）
偏移 +8     第 1 个成员变量(ivar)
偏移 ...     第 2 个成员变量
            ...（按声明顺序、各自对齐规则排布）
```

`isa` 永远在 `+0`，紧接着是这个类（含父类）的所有成员变量，按声明顺序、各自的对齐要求依次排下去。换句话说——

> **实例对象 = 一根 isa + 一串成员变量。** 它身上没有方法，方法都存在「类」里（下一章讲）。

这也正是为什么 `NSObject` 的实例「最小」：它一个自定义成员都没有，身上就只有那 8 字节的 isa。

## 对象多大：从「需求」到「实分」的三个数

「一个对象多大」这个问题，其实有**三个互不相同**的答案。一个对象从「源码声明」走到「躺在堆上」，大小要过三道关：

```
① 编译期需求    class_ro_t.instanceSize  = isa(8) + 各 ivar 字节数
       │
       ▼ word_align —— 向上取整到 8 的倍数（64 位下字长 = 8）
② 理论大小      class_getInstanceSize()  = alignedInstanceSize()   ← 注意：不含 16 下限
       │
       ▼ alloc 时 instanceSize()：if (size < 16) size = 16   （CF 要求最小 16 字节）
③ 实际申请      malloc_instance(size) → malloc 还有自己的 16 字节分桶粒度
       ▼
     malloc_size()  = 堆真正给出的块大小（16 的倍数）
```

第①步的 `instanceSize` 是编译器在编译期就算好、存进类的只读数据 `class_ro_t` 里的（`isa` 加上所有成员变量的字节数）。第②步把它按字长（64 位下是 8 字节）向上对齐，这就是 `class_getInstanceSize` 返回的值：

```objc
// objc-class.mm:817 —— class_getInstanceSize 只到「对齐后的 ivar 需求」，没有 16 下限
size_t class_getInstanceSize(Class cls) {
    if (!cls) return 0;
    cls->realizeIfNeeded();
    return cls->alignedInstanceSize();   // = word_align(ro->instanceSize)，按 8 字节对齐
}
```

但**真正 alloc 时用的不是它**，而是 `instanceSize()`——这里才加上了「最小 16 字节」的下限：

```objc
// objc-runtime-new.h:3144 —— runtime 真正分配时用的大小
inline size_t instanceSize(size_t extraBytes) const {
    if (fastpath(cache.hasFastInstanceSize(extraBytes)))   // 缓存里有算好的就走快路
        return cache.fastInstanceSize(extraBytes);

    size_t size = alignedInstanceSize() + extraBytes;
    if (size < 16) size = 16;   // CF requires all objects be at least 16 bytes.
    return size;
}
```

最后在创建实例的核心函数里，把这个 `size` 交给 `malloc` 去要内存：

```objc
// objc-runtime-new.mm:9309 —— _class_createInstance_realized 节选
size = cls->instanceSize(extraBytes);     // 拿到「抬到 ≥16」之后的大小
id obj = objc::malloc_instance(size, cls); // 向堆申请
// ...
obj->initInstanceIsa(cls, hasCxxDtor);     // 把 isa 写进对象开头那 8 字节
```

而 `malloc` 自己还有**分桶粒度**（在 64 位上以 16 字节为单位），所以最终 `malloc_size()` 拿到的实际块，往往会再被向上取整到 16 的倍数。

## 实战：NSObject 为什么是「8 需求 / 16 实分」

把上面三个数套到具体的类上，就一目了然了：

| 类 | ① isa + ivar | ② `class_getInstanceSize`（对齐后） | ③ `malloc_size`（堆实分） |
|---|---|---|---|
| `NSObject`（只有 isa） | 8 | **8** | **16** |
| `Person { int age }` | 8 + 4 = 12 | 16（对齐到 8 的倍数） | 16 |
| `Person { NSString *name; int age }` | 8 + 8 + 4 = 20 | **24** | **32** |

- **`NSObject` 的「8 需求 / 16 实分」**：`class_getInstanceSize` 只返回 8（刚够装一根 isa），但真分配时 `instanceSize()` 把它抬到 16，`malloc` 也按 16 给——所以 `malloc_size` 是 16。这就是那句经典面试题「NSObject 对象占多少内存」的完整答案：**理论需求 8，实际占用 16。**
- **第三个例子最能说明「三个数互不相等」**：成员变量需求 20，对齐成 24，`malloc` 按 16 分桶再抬到 32。

到这里，「对象的本质」就讲完整了：**一根 isa + 一串成员变量，在堆上占着一块向上对齐到 16 倍数的内存。** 那 isa 指向的、成员变量大小所记录在的那个「类」，本身又是什么？下一章揭晓——**类，其实也是一个对象。**




# 类的本质：objc_class

上一章结尾我们埋了个钩子：`isa` 指向的、实例大小所记录的那个「类」，它本身又是什么？这一章就来拆开看。答案会有点反直觉——**类，其实也是一个对象。**

## 类本身也是对象

打开 `objc_class` 的定义，第一行就说明了一切：

```objc
// objc-runtime-new.h:2635
struct objc_class : objc_object {
    // Class ISA;               // 继承自 objc_object，指向元类
    Class superclass;
    cache_t cache;              // formerly cache pointer and vtable
    class_data_bits_t bits;     // class_rw_t * plus custom rr/alloc flags

    Class getSuperclass() const;               // 读父类（arm64e 下验签，见下文 superclass 小节）
    void  setSuperclass(Class newSuperclass);  // 写父类（arm64e 下签名）
    class_rw_t *data() const { return bits.data(); }
    void setData(class_rw_t *newData) { bits.setData(newData); }
    const class_ro_t *safe_ro() const { return bits.safe_ro(); }
    // ... isMetaClass()/getMeta()/instanceSize() 等大量方法（元类相关留 §4）
};
```

`objc_class` 继承自 `objc_object`，这意味着**类也是一个对象**，它同样从 `objc_object` 那里继承了一根 `isa`。所以前面讲对象时的那套（`isa`、`isa_t`、`getClass`）对类一样成立——只不过类的 `isa` 指向的不是普通类，而是**元类（metaclass）**，这个留到下一章。

## 类的四大件：isa / superclass / cache / bits

从上面的定义可以看出，一个类身上就四样东西：

1. **`isa`**（继承自 `objc_object`）→ 指向它的**元类**
2. **`superclass`** → 它的父类（`NSObject` 的 superclass 为 `nil`，这条链是「继承」的物理体现）
3. **`cache`** → 方法缓存。一条消息查到对应的实现后，会缓存在这里，下次同样的消息直接命中、跳过慢速查找。
4. **`bits`** → 类的「数据仓库」。方法列表、属性、协议、成员变量、实例大小……全藏在它身后。

### superclass：父类指针（arm64e 上同样被签名）

`superclass` 就是一根指向父类的 `Class` 指针。但在 arm64e 上，它和 `isa` 一样会被 PAC 签名，所以读写都得走专门的访问器，不能直接当地址用：

```objc
// objc-runtime-new.h:2641 / 2645 / 2671
Class superclass;

Class getSuperclass() const {
#if __has_feature(ptrauth_calls)
#   if ISA_SIGNING_AUTH_MODE == ISA_SIGNING_AUTH
    if (superclass == Nil) return Nil;
#if SUPERCLASS_SIGNING_TREAT_UNSIGNED_AS_NIL
    void *stripped = ptrauth_strip((void *)superclass, ISA_SIGNING_KEY);
    if ((void *)superclass == stripped) {
        void *resigned = ptrauth_sign_unauthenticated(stripped, ISA_SIGNING_KEY,
            ptrauth_blend_discriminator(&superclass, ISA_SIGNING_DISCRIMINATOR_CLASS_SUPERCLASS));
        if ((void *)superclass != resigned) return Nil;
    }
#endif
    void *result = ptrauth_auth_data((void *)superclass, ISA_SIGNING_KEY,
        ptrauth_blend_discriminator(&superclass, ISA_SIGNING_DISCRIMINATOR_CLASS_SUPERCLASS));
    return (Class)result;
#   else
    return (Class)ptrauth_strip((void *)superclass, ISA_SIGNING_KEY);
#   endif
#else
    return superclass;
#endif
}

void setSuperclass(Class newSuperclass) {
#if ISA_SIGNING_SIGN_MODE == ISA_SIGNING_SIGN_ALL
    superclass = (Class)ptrauth_sign_unauthenticated((void *)newSuperclass, ISA_SIGNING_KEY,
        ptrauth_blend_discriminator(&superclass, ISA_SIGNING_DISCRIMINATOR_CLASS_SUPERCLASS));
#else
    superclass = newSuperclass;
#endif
}
```

可以看到，这和前面 `isa` 的 `getClass` 是同一套路数：arm64e 上凡是存在结构体里的关键指针（isa、superclass）都被签了名，读写必须经访问器验签 / 签名。`NSObject` 的 `superclass` 为 `nil`，整条 `superclass` 链就是「继承关系」的物理体现。

### cache：方法缓存

```objc
// objc-runtime-new.h:337
struct cache_t {
private:
	// 缓存的真实存储，可以理解为：_bucketsAndMaybeMask -> bucket_t数组 —> 每个 bucket 存一组 SEL -> IMP
    explicit_atomic<uintptr_t> _bucketsAndMaybeMask;   // 桶数组指针（可能拼着 mask）
    union {
    //  普通情况下，当成struct看
        struct {
#if CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_OUTLINED && !__LP64__
            explicit_atomic<mask_t>    _mask; // 缓存容量掩码，用来算 bucket 下标
            uint16_t                   _occupied;  //  当前用了多少个 bucket

#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_OUTLINED && __LP64__
            explicit_atomic<mask_t>    _mask;
            uint16_t                   _occupied;
            uint16_t                   _flags;
#elif __LP64__   // 内联 mask，64 位，这里没有_mask，因为mask 被内联编码进 _bucketsAndMaybeMask 里面了。也就是说前面的_bucketsAndMaybeMask，不只是保存 buckets 指针，还顺便藏了 mask。所以第二个 word 就空出来一部分，可以放：
            uint32_t                   _disguisedPreoptCacheSignature;// 主要是区分是普通cache 还是伪装过的 preopt cache，这里放了一个用于 preopt cache 识别的 signature
            uint16_t                   _occupied;
            uint16_t                   _flags;
#else            // 内联 mask，32 位
            uint16_t                   _occupied;
            uint16_t                   _flags;
#endif
        };
        // 这是union的另一个成员
        explicit_atomic<preopt_cache_t *> _originalPreoptCache;   // dyld shared cache 里的预优化方法缓存，这个主要给系统类用，一些方法缓存可以在系统共享缓存里提前计算好，App运行不一定从空缓存开始
    };

    cache_t() : _bucketsAndMaybeMask(0) {}

    // —— 各 CACHE_MASK_STORAGE 方案下的 bucketsMask/maskShift 等 constexpr（337–453，存储方案内部常量，从略）——
	//是有工具函数
    bool isConstantEmptyCache() const;
    bool canBeFreed() const;
    mask_t mask() const;
    void incrementOccupied();
    void setBucketsAndMask(struct bucket_t *newBuckets, mask_t newMask);
    void reallocate(mask_t oldCapacity, mask_t newCapacity, bool freeOld);
    void collect_free(bucket_t *oldBuckets, mask_t oldCapacity);

    // 创建各种buckets
    static bucket_t *emptyBuckets();
    static bucket_t *mallocBuckets(mask_t newCapacity);
    static bucket_t *allocateBuckets(mask_t newCapacity);
    static bucket_t *emptyBucketsForCapacity(mask_t capacity, bool allocate = true);
    static struct bucket_t *endMarker(struct bucket_t *b, uint32_t cap);
    void bad_cache(id receiver, SEL sel) __attribute__((noreturn, cold));

public:
    unsigned capacity() const;
    struct bucket_t *buckets() const;
    Class cls() const;
    mask_t occupied() const;
    void initializeToEmpty();
	// 预优化缓存相关函数
    bool isConstantOptimizedCache(bool strict = false, uintptr_t empty_addr = 0) const;
    bool shouldFlush(SEL sel, IMP imp) const;
    bool isConstantOptimizedCacheWithInlinedSels() const;
    void initializeToEmptyOrPreoptimizedInDisguise();

    void insert(SEL sel, IMP imp, id receiver);     // 写缓存（Part2 主角）
    void copyCacheNolock(objc_imp_cache_entry *buffer, int len);
    void destroy();
    void eraseNolock(const char *func);
    static void init();
    static void collectNolock(bool collectALot);
    static size_t bytesForCapacity(uint32_t cap);

    // ===== FAST_CACHE_* 标志（存在 _flags 里）=====
#if CACHE_T_HAS_FLAGS  如果当前平台的 `cache_t` 支持 `_flags` 字段，就启用这些快速标志
#   if __arm64__
#       define FAST_CACHE_HAS_CXX_DTOR       (1<<0)   // 第 0 位，便于 bfi 进 isa_t::has_cxx_dtor
#       define FAST_CACHE_HAS_CXX_CTOR       (1<<1)
#       define FAST_CACHE_META               (1<<2)
#   else
#       define FAST_CACHE_META               (1<<0)
#       define FAST_CACHE_HAS_CXX_CTOR       (1<<1)
#       define FAST_CACHE_HAS_CXX_DTOR       (1<<2)
#   endif
#   if __LP64__
#       define FAST_CACHE_ALLOC_MASK         0x0ff8
#       define FAST_CACHE_ALLOC_MASK16       0x0ff0
#       define FAST_CACHE_ALLOC_DELTA16      0x0008
#       define FAST_CACHE_FLAGS_MASK         0xf000
#   endif
#   define FAST_CACHE_HAS_CUSTOM_DEALLOC_INITIATION (1<<12)
#   define FAST_CACHE_REQUIRES_RAW_ISA   (1<<13)   // 实例必须用 raw isa
#   define FAST_CACHE_HAS_DEFAULT_AWZ    (1<<14)   // 有默认 alloc/allocWithZone（存元类里）
#   define FAST_CACHE_HAS_DEFAULT_CORE   (1<<15)   // 有默认 new/self/class/...

    bool getBit(uint16_t flags) const { return _flags & flags; }
    void setBit(uint16_t set)   { __c11_atomic_fetch_or ((_Atomic(uint16_t)*)&_flags,  set, __ATOMIC_RELAXED); }
    void clearBit(uint16_t clr) { __c11_atomic_fetch_and((_Atomic(uint16_t)*)&_flags, ~clr, __ATOMIC_RELAXED); }
#endif

    // ===== 快速实例大小（§2 instanceSize 走的快路）=====
#if FAST_CACHE_ALLOC_MASK
    bool hasFastInstanceSize(size_t extra) const {
        if (__builtin_constant_p(extra) && extra == 0) return _flags & FAST_CACHE_ALLOC_MASK16;
        return _flags & FAST_CACHE_ALLOC_MASK;
    }
    size_t fastInstanceSize(size_t extra) const {
        ASSERT(hasFastInstanceSize(extra));
        if (__builtin_constant_p(extra) && extra == 0) return _flags & FAST_CACHE_ALLOC_MASK16;
        size_t size = _flags & FAST_CACHE_ALLOC_MASK;
        return align16(size + extra - FAST_CACHE_ALLOC_DELTA16);   // 去掉 setFastInstanceSize 加的 DELTA16
    }
    void setFastInstanceSize(size_t newSize) {
        uint16_t newBits = _flags & ~FAST_CACHE_ALLOC_MASK;
        uint16_t sizeBits = word_align(newSize) + FAST_CACHE_ALLOC_DELTA16;
        sizeBits &= FAST_CACHE_ALLOC_MASK;
        if (newSize <= sizeBits) newBits |= sizeBits;
        _flags = newBits;
    }
#endif
};
static_assert(sizeof(cache_t) == 2 * sizeof(void *), "cache_t must be two words");
```

缓存的桶 `bucket_t`（一个 `SEL → IMP` 槽）：

```objc
// objc-runtime-new.h:214
struct bucket_t {
private:
#if __arm64__                          // arm64：IMP 在前（利于 ptrauth）
    explicit_atomic<uintptr_t> _imp;
    explicit_atomic<SEL> _sel;
#else                                  // 其它：SEL 在前
    explicit_atomic<SEL> _sel;
    explicit_atomic<uintptr_t> _imp;
#endif
    uintptr_t modifierForSEL(bucket_t *base, SEL newSel, Class cls) const {
        return (uintptr_t)base ^ (uintptr_t)newSel ^ (uintptr_t)cls;   // ptrauth 签名修饰子
    }
    uintptr_t encodeImp(bucket_t *base, IMP newImp, SEL newSel, Class cls) const;  // IMP 按方案编码/签名
public:
    inline SEL sel() const { return _sel.load(memory_order_relaxed); }
    inline IMP rawImp(objc_class *cls) const;   // 解码取出 IMP
    // ... set()/imp() 等
};
```

一句话：`cache` 把「最近用过的 `SEL` → `IMP`」缓存起来，让重复的消息发送走极快的命中路径。还记得 §2 讲 `instanceSize` 时提到的 `FAST_CACHE_*` 吗？它存的就是这里的 `_flags`——苹果把一批高频标志和实例大小塞进了 `cache`，正是为了在发消息的热路径上少绕一层。

```objc
// 旧版（objc4-750 及之前）
struct cache_t {
    struct bucket_t *_buckets;   // 桶数组指针
    mask_t           _mask;      // 桶数 - 1
    mask_t           _occupied;  // 已占用桶数
};
```

![[cache_t_layout.html]]

## bits ：class_rw_t → class_ro_t

`bits` 本身只是个包装（`class_data_bits_t`），通过 `data()` 取出真正的数据结构 `class_rw_t`；`class_rw_t` 再通过 `ro()` 拿到 `class_ro_t`。也就是说，类的数据是**两层**结构：

```
objc_class.bits
     │ data()
     ▼
 class_rw_t   （运行期可写，read-write）
     │ ro()
     ▼
 class_ro_t   （编译期只读，read-only）
```

`bits` 的真身 `class_data_bits_t`（它用到的 `FAST_*` 掩码先列出）：

```objc
// objc-runtime-new.h:122（__LP64__）
#define FAST_IS_SWIFT_LEGACY    (1UL<<0)
#define FAST_IS_SWIFT_STABLE    (1UL<<1)
#define FAST_HAS_DEFAULT_RR     (1UL<<2)
#if TARGET_OS_IPHONE && !TARGET_OS_SIMULATOR
#define FAST_DATA_MASK          0x0f00007ffffffff8UL
#else
#define FAST_DATA_MASK          0x0f007ffffffffff8UL
#endif
#define FAST_FLAGS_MASK         0x0000000000000007UL
#define FAST_IS_RW_POINTER      0x8000000000000000UL   // 快速判断这是 rw 指针而非 ro
```
```objc
// objc-runtime-new.h:2364
struct class_data_bits_t {
    friend objc_class;
    explicit_atomic<uintptr_t> bits;   // 存的是 class_ro_t* 或 class_rw_t*（低位塞 FAST_ 标志）
private:
    bool getBit(uintptr_t bit) const { return bits.load(std::memory_order_relaxed) & bit; }
    void setAndClearBits(uintptr_t set, uintptr_t clear);   // 改标志位时带 ptrauth 验签+重签
    void setBits(uintptr_t set)   { setAndClearBits(set, 0); }
    void clearBits(uintptr_t clr) { setAndClearBits(0, clr); }
public:
    void copyRWFrom(const class_data_bits_t &other);
    void copyROFrom(const class_data_bits_t &other, bool authenticate);

    bool has_rw_pointer() const { return has_rw_pointer(bits.load(std::memory_order_relaxed)); }
    static bool has_rw_pointer(uintptr_t bits) {
#if FAST_IS_RW_POINTER
        return (bool)(bits & FAST_IS_RW_POINTER);
#else
        return bits != 0 && (flags(bits) & RW_REALIZED);
#endif
    }

    class_rw_t *data() const {       // 取 rw：arm64e 下验签 + FAST_DATA_MASK 抠地址
        ASSERT(has_rw_pointer());
        uintptr_t localBits = bits.load(std::memory_order_relaxed);
        return (class_rw_t *)((uintptr_t)ptrauth_auth_data((class_rw_t *)localBits,
            CLASS_DATA_BITS_RW_SIGNING_KEY,
            ptrauth_blend_discriminator(&bits, CLASS_DATA_BITS_RW_DISCRIMINATOR)) & FAST_DATA_MASK);
    }
    void setData(class_rw_t *newData);   // 存 rw：FAST_FLAGS_MASK | 新指针 | FAST_IS_RW_POINTER，再签名

    const class_ro_t *safe_ro() const {  // 并发 realize 期间也能安全取 ro
        uintptr_t bitsValue = bits.load(std::memory_order_relaxed);
        if (has_rw_pointer(bitsValue)) return data()->ro();
        return (const class_ro_t *)(/* 验签后 */ bitsValue & FAST_DATA_MASK);
    }

    static uint32_t flags(uintptr_t bits) {   // flags 在 ro/rw 起始处，直接 strip+mask 读
        return *(const uint32_t *)((uintptr_t)ptrauth_strip((const uint32_t *)bits,
                 CLASS_DATA_BITS_RO_SIGNING_KEY) & FAST_DATA_MASK);
    }
    uint32_t flags() const { return flags(bits.load(std::memory_order_relaxed)); }
};
```

先看底下那层 `class_ro_t`——它就是编译期写死、运行期不可变的部分：

```objc
// objc-runtime-new.h:1598 —— class_ro_t（编译期只读）
struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;       // §2 实例大小源头
#ifdef __LP64__
    uint32_t reserved;
#endif
    union {
        const uint8_t *ivarLayout;
        Class nonMetaclass;
    };
    explicit_atomic<const char *> name;
    // baseMethods/baseProtocols 用 PointerUnion 包了相对/绝对 + ptrauth 多种表示
    objc::PointerUnion<method_list_t,   relative_list_list_t<method_list_t>,   ...> baseMethods;
    objc::PointerUnion<protocol_list_t, relative_list_list_t<protocol_list_t>, ...> baseProtocols;
    const ivar_list_t *ivars;
    const uint8_t *weakIvarLayout;
    objc::PointerUnion<property_list_t, relative_list_list_t<property_list_t>, ...> baseProperties;

    // RO_HAS_SWIFT_INITIALIZER 时才存在
    _objc_swiftMetadataInitializer _swiftMetadataInitializer_NEVER_USE[0];
    _objc_swiftMetadataInitializer swiftMetadataInitializer() const;
    const char *getName() const { return name.load(std::memory_order_acquire); }
    class_ro_t *duplicate() const;
};
```

`§2` 里反复提到的 `instanceSize`、`instanceStart`，源头就在这里。再看上面那层 `class_rw_t`——它是运行期可写的部分：

```objc
// objc-runtime-new.h:2212 —— class_rw_t（运行期可写）
struct class_rw_t {
    uint32_t flags;
    uint16_t witness;
#if SUPPORT_INDEXED_ISA
    uint16_t index;
#endif
    explicit_atomic<uintptr_t> ro_or_rw_ext;   // 二选一：const class_ro_t* 或 class_rw_ext_t*
    Class firstSubclass;
    Class nextSiblingClass;

private:
    using ro_or_rw_ext_t = objc::PointerUnion<const class_ro_t, class_rw_ext_t,
                            PTRAUTH_STR("class_ro_t"), PTRAUTH_STR("class_rw_ext_t")>;
    const ro_or_rw_ext_t get_ro_or_rwe() const { return ro_or_rw_ext_t{ro_or_rw_ext}; }
    void set_ro_or_rwe(const class_ro_t *ro);
    void set_ro_or_rwe(class_rw_ext_t *rwe, const class_ro_t *ro);
    class_rw_ext_t *extAlloc(const class_ro_t *ro, bool deep = false);   // 真要改时才分配 rwe

public:
    void setFlags(uint32_t set);
    void clearFlags(uint32_t clear);
    void changeFlags(uint32_t set, uint32_t clear);

    class_rw_ext_t *ext() const;
    class_rw_ext_t *extAllocIfNeeded();
    class_rw_ext_t *deepCopy(const class_ro_t *ro);

    const class_ro_t *ro() const {           // rwe 在就从 rwe 取 ro，否则 ro_or_rw_ext 本身就是 ro
        auto v = get_ro_or_rwe();
        if (slowpath(v.is<class_rw_ext_t *>()))
            return v.get<class_rw_ext_t *>(&ro_or_rw_ext)->ro;
        return v.get<const class_ro_t *>(&ro_or_rw_ext);
    }
    void set_ro(const class_ro_t *ro);

    const method_array_t   methods() const;     // rwe 在取 rwe->methods；否则取 ro->baseMethods
    const property_array_t properties() const;
    const protocol_array_t protocols() const;
};
```

## ro 与 rw 的分离：clean memory vs dirty memory

为什么要分成 `ro` / `rw` 两层？这是新版 runtime 相对老博客**最大的变化**，也是 WWDC2020《Advancements in the Objective-C runtime》的重点。核心是一对内存概念：

- **`class_ro_t` = clean memory（干净内存）**：编译期就定死、运行期只读。它可以在进程间**共享**、能被系统按需换出/换入（page in/out），**不占用宝贵的 dirty memory**。`instanceSize`、`name`、原始方法列表都在这。
- **`class_rw_t` = dirty memory（脏内存）**：类一旦被 realize（运行期初始化），就需要一块可写的数据来合并 Category 方法、支持 `class_addMethod` 动态加方法、维护子类链。dirty memory 不能共享、不能换出，是实打实的内存开销。

苹果统计发现：**绝大多数类，运行期根本不会去改方法 / 属性 / 协议列表。** 给它们都分配一份内嵌了三个数组的完整 `rw`，太浪费。于是新版（818 起）做了关键优化——把那三个可变数组抽到一个**懒分配**的 `class_rw_ext_t` 里：

```objc
// objc-runtime-new.h:2202 —— 真要动态改时才分配的扩展
struct class_rw_ext_t {
    class_ro_t_authed_ptr<const class_ro_t> ro;   // 指回只读 ro（带 ptrauth）
    method_array_t   methods;      // 可写方法数组（base + category + 动态添加）
    property_array_t properties;
    protocol_array_t protocols;
    const char *demangledName;
    uint32_t version;
};
```

`class_rw_t` 里的 `ro_or_rw_ext` 就是个「二选一」的联合指针（`PointerUnion`）：

- **情况 A（绝大多数类）**：`ro_or_rw_ext` 直接指向 `class_ro_t`——**不额外分配 `rw_ext`**，省下一整块 dirty memory；
- **情况 B（真的要动态改方法时）**：才 `extAlloc` 一个 `class_rw_ext_t`，把可变的 `methods/properties/protocols` 装进去，`ro_or_rw_ext` 转而指向它。

### 新老对照

| | 老版（objc4-750 那代） | 新版 951.1（本文这版） |
|---|---|---|
| `class_rw_t` 怎么存方法/属性/协议 | **直接内嵌** `method_array_t methods; property_array_t properties; protocol_array_t protocols; const class_ro_t *ro;`——不管改不改，每个类都背着 | 抽到**懒分配**的 `class_rw_ext_t`，用 `ro_or_rw_ext` 一个联合指针「ro 或 rw_ext 二选一」 |
| 取只读数据 | `data()->ro`（`ro` 是字段，直接 `.`） | `data()->ro()`（`ro()` 是方法，从联合指针里取） |
| 内存代价 | 每个 realize 的类都摊一份完整 rw | 多数类只指向 ro，不分配 rw_ext，**省 dirty memory** |

一句话总结这一章：**类也是对象（有 isa）；它的四大件是 isa / superclass / cache / bits；真正的数据在 `bits` 身后的 `class_rw_t`（可写）→ `class_ro_t`（只读）两层里，而 ro/rw 的分离正是为了把编译期定死的部分放进可共享、可换出的 clean memory，省下宝贵的 dirty memory。**

那么——类的 `isa` 指向的「元类」，又是何方神圣？下一章见。


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
