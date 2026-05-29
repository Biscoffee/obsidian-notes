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
