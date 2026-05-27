# mikeash.com: Friday Q&A 2009-03-13: Intro to the Objective-C Runtime

> 原文：https://www.mikeash.com/pyblog/friday-qa-2009-03-13-intro-to-the-objective-c-runtime.html
> 发布：2009-03-13　作者：Mike Ash
> 译者：MiMo（mimo-v2.5-pro）；代码块保留英文原样

---

下一篇文章：Friday Q&A 2009-03-20：Objective-C 消息发送机制  
上一篇文章：Friday Q&A 2009-03-06：使用 Clang 静态分析器  

标签：fridayqna objectivec  

欢迎在又一个“黑色星期五”回到周五问答专栏。本周我将采纳奥利弗·穆尼的建议，探讨 Objective-C 运行时（runtime）——它的工作原理以及它能为你做些什么。  

许多 Cocoa 程序员对 Objective-C 运行时只有模糊的认知。他们知道它的存在（有些人甚至不知道这一点！），知道它很重要，没有它就无法运行 Objective-C，但大概也就止步于此了。  

今天我想详细阐述 Objective-C 在运行时层面究竟是如何工作的，以及你可以用它实现哪些功能。  

（注：我仅讨论 10.5 及之后版本上苹果的运行时。10.4 及更早版本的运行时缺失许多 API，转而强制使用直接结构体访问；而 GNU 和 Cocotron 的运行时则是完全不同的实现体系。）  

**对象**  

在 Objective-C 中我们时刻都在操作对象，但对象究竟*是*什么？让我们通过构造一些能揭示其本质的代码来一探究竟。  

首先，我们知道对象是通过指针引用的，例如 `NSObject *`

我们知道使用 `+alloc` 方法来创建它们。相关文档只说明该方法会调用 `+allocWithZone:`。再沿着文档追溯，我们会发现 `NSDefaultMallocZone`，并了解到它们仅仅是通过 `malloc` 来分配内存的。很简单！

但分配后的对象在内存中是什么样的呢？让我们来看看：

```
#import <Foundation/Foundation.h>
@interface A : NSObject { @public int a; } @end
@implementation A @end
@interface B : A { @public int b; } @end
@implementation B @end
@interface C : B { @public int c; } @end
@implementation C @end
int main(int argc, char **argv)
{
[NSAutoreleasePool new];
C *obj = [[C alloc] init];
obj->a = 0xaaaaaaaa;
obj->b = 0xbbbbbbbb;
obj->c = 0xcccccccc;
NSData *objData = [NSData dataWithBytes:obj length:malloc_size(obj)];
NSLog(@"Object contains %@", objData);
return 0;
}
```

为了获取正确的长度，我们使用 `malloc_size` 函数，并利用 `NSData` 来打印出格式整齐的十六进制表示。以下是运行结果：

```
2009-01-27 15:58:04.904 a.out[22090:10b] Object contains <20300000 aaaaaaaa bbbbbbbb cccccccc>
```

那么，这个 `20300000` 到底是什么？它出现在 A 的实例变量之前，所以它一定是 NSObject 的实例变量。我们来看一下 NSObject 的定义：

```objc
/*********** 基类 ***********/
@interface NSObject
{
    Class isa;
}
```

果然，这里还有另一个实例变量。但是这个 `Class` 是什么？如果我们让 Xcode 跳转到它的定义，就会进入 `/usr/include/objc/objc.h`，其中包含了：

```
typedef struct objc_class *Class;
```

`/usr/include/objc/runtime.h`  
该文件包含：

```
struct objc_class {
Class isa;
#if !__OBJC2__
Class super_class OBJC2_UNAVAILABLE;
const char *name OBJC2_UNAVAILABLE;
long version OBJC2_UNAVAILABLE;
long info OBJC2_UNAVAILABLE;
long instance_size OBJC2_UNAVAILABLE;
struct objc_ivar_list *ivars OBJC2_UNAVAILABLE;
struct objc_method_list **methodLists OBJC2_UNAVAILABLE;
struct objc_cache *cache OBJC2_UNAVAILABLE;
struct objc_protocol_list *protocols OBJC2_UNAVAILABLE;
#endif
} OBJC2_UNAVAILABLE;
```

`Class` 是一个指向结构的指针，该结构以另一个 `Class` 开头。
我们来看看另一个根类 `NSProxy`：

@interface NSProxy
// ...其他声明...
{
    Class isa;
}
// ...
@end

它同样存在。我们再看看 `id` 的定义——`id` 是 Objective-C 中表示「任意对象」的类型：
```c
typedef struct objc_object {
    Class isa;
} *id;
```

```
typedef struct objc_object {
Class isa;
} *id;
```

`Class isa`

，即便是类对象也是如此。但它究竟是什么？
正如其名及类型所示，`isa`（实例变量）表明了特定对象所属的类。每个 Objective-C 对象都必须以 isa 指针开头，否则运行时（runtime）将无从知晓如何操作它。特定对象类型的所有信息都凝聚在这个小小的指针中。对象的其余部分本质上只是一大块内存，对运行时而言无关紧要。赋予这块内存意义的责任在于各个类自身。

**类（Classes）**

那么，类究竟包含哪些内容呢？那些“不可用”的结构体成员提供了很好的线索。（它们是为了兼容 Leopard 之前的运行时而存在的，如果以 Leopard 或更高版本为目标平台则不应使用，但它们仍然能告诉我们那里存放着什么类型的信息。）首先是 `isa`，它使得类也能像对象一样运作。接着是指向父类的指针，以此确立正确的类层次结构。随后是关于该类的其他一些基本信息。末尾才是真正有趣的部分：那里有实例变量（instance variables）列表、方法（methods）列表，以及协议（protocols）列表。所有这些内容在运行时都是可访问的，并且可以在运行时进行修改。

`cache`（缓存）成员在运行时操作中并非真正有用，但它揭示了实现细节的有趣一面。每次发送消息（`[foo bar]`）时，运行时都需要在目标对象所属类的方法列表中搜寻以查找要调用的实际代码。然而，方法默认以大型线性列表存储，因此速度很慢。缓存就是一个将选择子（selector）映射到代码的哈希表（hash table）。首次发送消息时会进行缓慢耗时的查找，但结果会被存入哈希表。后续调用将在哈希表中找到条目，使过程快得多。

浏览 `runtime.h` 的其余部分，你会看到许多用于访问和操作这些属性的函数。每个函数的前缀表明其操作对象。通用运行时函数以 `objc_` 开头，操作类的函数以 `class_` 开头，依此类推。例如，你可以调用 `class_getInstanceMethod` 获取特定方法的信息，如参数/返回类型。或者你可以调用 `class_addMethod`（译注：此处原文中断，现代系统中此函数用于动态添加方法实现）。

在运行时向现有类添加*新*方法。你甚至可以通过使用 `objc_allocateClassPair` 在运行时创建一个全新的类。

**实际应用**

利用这类运行时元信息可以实现大量实用功能，以下是一些思路。

**自动查找实例变量/方法。** Apple的键值编码(KVC)已实现此类功能：你提供一个名称，它会基于该名称查找方法或实例变量并进行操作。你可以自行实现类似机制，例如需要根据名称查找实例变量等情况。

**自动注册/调用子类。** 使用 `objc_getClassList`

你可以获取运行时（runtime）当前已知的所有类列表，并通过追踪类层次结构，识别出哪些类是指定类的子类。这使得你可以编写子类来处理特定数据格式或其他类似情况，并让父类在不需手动注册每个子类的情况下查找它们。**自动对每个类调用某个方法。**这对自定义单元测试框架等场景很有用。与第2点类似，但这里是寻找一个方法的实现，而非特定的类层次结构。**运行时重写方法。**运行时提供了一整套工具，可将方法重新指向自定义实现，从而在不修改源代码的情况下改变类的行为。**自动释放合成属性。**`@synthesize` 关键字虽然便于让编译器生成 setter/getter，但仍强制你在 `-dealloc` 中编写清理代码。（译注：现代 Objective-C 中 `@synthesize` 通常由编译器自动处理，且 ARC 机制会自动生成对应的释放代码）

通过读取类属性的元信息，你可以编写代码来自动遍历并清理所有合成属性，而无需为每种情况单独编写代码。**桥接功能** 通过在运行时动态生成类，并按需查找必要的属性，你可以在Objective-C与另一种（足够动态的）语言之间建立桥梁。**更多可能性** 不要局限于以上用途，尽情发挥你的创意吧！

**总结**

Objective-C是一门强大的语言，其全面的运行时API（Application Programming Interface，应用程序编程接口）是其中极其有用的部分。尽管在那些C代码中摸索可能有些繁琐，但实际操作起来并不困难，而且它所提供的强大能力绝对值得投入时间。

本周的Friday Q&A到此结束。请通过下方评论或电子邮件发送你的建议（若不愿公开姓名请告知）。Friday Q&A依赖读者的建议运行，欢迎来信分享！

对Objective-C运行时有何钟爱的使用方式？或有什么不满之处？有实用技巧想分享？请在下方畅所欲言。

评论：

http://vgable.com/blog/2008/12/20/automatically-freeing-every-property/

回过头来看，我对指针的处理可能过于保守了，但我确实还没用过 pointer-properties（指针属性）。

**事实上**，编译器只是不允许你再通过名称访问它了。公共头文件中的声明与 runtime（运行时）在实现中创建和使用的实际结构体并无关联。

这难道不意味着 `runtime.h` 中可用的 struct 与 runtime 内部实际使用的 struct 大小不一致吗？这难道不会破坏针对 **struct objc_class**（Objective-C 类的结构体）类型进行的指针算术运算吗？是否只是因为这属于 runtime 内部事务，外界根本无需进行此类操作？

特别是关于 bridging（桥接）的方面！！