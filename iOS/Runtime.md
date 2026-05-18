https://halfrost.com/how_to_use_runtime/
# 简介
- runtime是实现“动态性”和“消息转发”的关键，是一套c语言编写的api库，程序在启动时会调用runtime来注册objc类结构，还有将分类整合进类中这些操作。
- Runtime 又叫运行时，是一套底层的 C 语言 API，是 iOS 系统的核心之一。开发者在编码过程中，可以给任意一个对象发送消息，在编译阶段只是确定了要向接收者发送这条消息，而接受者将要如何响应和处理这条消息，那就要看运行时来决定了。

C语言中，在编译期，函数的调用就会决定调用哪个函数。  
而OC的函数，属于动态调用过程，在编译期并不能决定真正调用哪个函数，只有在真正运行时才会根据函数的名称找到对应的函数来调用。

Objective-C 是一个动态语言，这意味着它不仅需要一个编译器，也需要一个运行时系统来动态得创建类和对象、进行消息传递和转发。

![](https://img.halfrost.com/Blog/ArticleImage/23_3.png)

Objective - C 在三个层面与 Runtime 系统交互：

1. Objc源代码
	一般情况开发者只需要编写 OC 代码即可，Runtime 系统自动在幕后把我们写的源代码在编译阶段转换成运行时代码，在运行时确定对应的数据结构和调用具体哪个方法。


2. 通过Foundation框架的NSObject 的方法
```Objective-C

// 类型检查
[obj isKindOfClass:[NSString class]];
[obj respondsToSelector:@selector(description)];

// 动态方法调用
[obj performSelector:@selector(run)];

// 获取类信息
[obj class];
[NSObject superclass];
```

3. 直接调用Runtime函数
```Objective-C
#import <objc/runtime.h>

// 动态添加方法
class_addMethod([MyClass class], @selector(newMethod), imp, "v@:");

// 方法交换（Method Swizzling）
Method m1 = class_getInstanceMethod(cls, sel1);
Method m2 = class_getInstanceMethod(cls, sel2);
method_exchangeImplementations(m1, m2);

// 关联对象
objc_setAssociatedObject(obj, key, value, policy);

// 获取所有属性
objc_property_t *props = class_copyPropertyList(cls, &count);
```


我们用人话来讲讲。
- 第一层告诉我们，消息发送不是函数调用，C++ 里 `obj.method()` 是编译期绑定的，地址在编译时就确定了。ObjC 的 `[obj method]` 是运行时查表，这个差别带来了巨大的灵活性。这意味着你可以在运行时替换"表里的地址"，从而改变方法的行为。这是后面所有黑魔法的根基。同时，为什么给nil发消息不崩溃呢？因为会先检查receiver是nil，则返回0/nil，不查表
- 第二层，我们通过他来写防御性代码或者完成工具性方法
- 第三层，我们可以使用method swizzling，关联对象，动态方法解析等来实现AOP


## NSObject源码

```Objective-c
typedef struct objc_class *Class;

@interface NSObject <NSObject> {
    Class isa  OBJC_ISA_AVAILABILITY;
}
```
```Objective-C
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;
    
#if !__OBJC2__
    Class super_class                                        OBJC2_UNAVAILABLE;
    const char *name                                         OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list *ivars  //一级指针                           OBJC2_UNAVAILABLE;
    struct objc_method_list **methodLists    //二级指针                OBJC2_UNAVAILABLE;
    struct objc_cache *cache                                 OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
#endif
    
} OBJC2_UNAVAILABLE;
```
在Objc2以前，class源码如上。

> ivars是objc_ivar_list成员变量列表的指针；methodLists是指向objc_method_list指针的指针。*methodLists是指向方法列表的指针。这里如果动态修改*methodLists的值来添加成员方法，这也是Category实现的原理，同样解释了Category不能添加属性的原因。

一级指针的话，`methodLists` 直接指向一个方法列表，想新增方法只能重新分配内存、拷贝旧列表、追加新方法，代价很高。

二级指针的话，`methodLists` 指向的是**一个指针数组**，每个元素指向一个独立的方法列表：
```
methodLists ──→ [ *list1, *list2, *list3, ... ]
                    ↓       ↓       ↓
                  原始方法  Category Category
                  列表      A的方法   B的方法
```
加载一个 Category 时，Runtime 只需要在这个指针数组里追加一个新指针，指向 Category 的方法列表就行了，**原有数据完全不动**。这就是"动态修改 `*methodLists` 的值"。

`ivars` 是一级指针，且不能动态修改，根本的原因是：**对象的内存布局在编译期就已经固定了**。偏移量是硬编码进二进制的。如果运行时强行插入一个新 ivar，后面所有 ivar 的偏移量都会错位，已经编译好的代码全部失效，直接崩溃。

所以 `ivars` 根本没有设计成可动态扩展的，不是技术上做不到，而是**做了也没用，内存布局不允许**。

至于Category不能添加属性的真实原因：
`@property` 本质上是三样东西的语法糖：

1. 一个 ivar（实例变量）
2. 一个 getter 方法
3. 一个 setter 方法
setter和getter方法加进去没问题，但是ivar会卡住，因为不能修改。所以准确的说法是：**Category 不能添加 ivar，导致编译器不允许你在 Category 里声明会自动合成 ivar 的 `@property`**。
这也是为什么用 Associated Objects 能绕过去——它根本没有修改对象的内存布局，而是在对象外部维护了一张独立的哈希表来存值，只是"看起来"像属性而已。


然后在2006年苹果发布Objc 2.0之后，objc_class的定义就变成下面这个样子了。
```Objective-C
typedef struct objc_class *Class;
typedef struct objc_object *id;

@interface Object { 
    Class isa; 
}

@interface NSObject <NSObject> {
    Class isa  OBJC_ISA_AVAILABILITY;
}

struct objc_object {
private:
    isa_t isa;
}

struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
}

union isa_t 
{
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }
    Class cls;
    uintptr_t bits;
}

```

![](https://img.halfrost.com/Blog/ArticleImage/23_6.png)

从上述源码中，我们可以看到，**Objective-C 对象都是 C 语言结构体实现的**，在objc2.0中，所有的对象都会包含一个isa_t类型的结构体。

objc_object被源码typedef成了id类型，这也就是我们平时遇到的id类型。这个结构体中就只包含了一个isa_t类型的结构体。这个结构体在下面会详细分析。

objc_class继承于objc_object。所以在objc_class中也会包含isa_t类型的结构体isa。至此，可以得出结论：**Objective-C 中类也是一个对象**。在objc_class中，除了isa之外，还有3个成员变量，一个是父类的指针，一个是方法缓存，最后一个这个类的实例方法链表。

object类和NSObject类里面分别都包含一个objc_class类型的isa。

## 2026-05-15 16:02 通过 Runtime 机制避免的常见 Crash 类型
**来源**: 自述　**confidence**: 0.90

- 通过 Runtime 机制检测和避免常见 Objective-C 崩溃
- 涉及 unrecognized selector 崩溃（对象和类方法）
- 包括 KVO、KVC、NSNotification、NSTimer 等机制崩溃
- 覆盖容器操作、字符串、野指针和线程问题崩溃

### 整理后内容

通过 Runtime 机制可以避免的常见 Crash ：
- unrecognized selector sent to instance（找不到对象方法的实现）
- unrecognized selector sent to class（找不到类方法实现）
- KVO Crash
- KVC Crash
- NSNotification Crash
- NSTimer Crash
- Container Crash（集合类操作造成的崩溃，例如数组越界，插入 nil 等）
- NSString Crash （字符串类操作造成的崩溃）
- Bad Access Crash （野指针）
- Threading Crash （非主线程刷 UI）
- NSNull Crash

<details><summary>原文</summary>

通过 Runtime 机制可以避免的常见 Crash ：

unrecognized selector sent to instance（找不到对象方法的实现）
unrecognized selector sent to class（找不到类方法实现）
KVO Crash
KVC Crash
NSNotification Crash
NSTimer Crash
Container Crash（集合类操作造成的崩溃，例如数组越界，插入 nil 等）
NSString Crash （字符串类操作造成的崩溃）
Bad Access Crash （野指针）
Threading Crash （非主线程刷 UI）
NSNull Crash

</details>

---

![[Pasted image 20260516134041.png]]



## 2026-05-16 14:51:46 Objective-C 1.0 与 2.0 类结构对比 ^dbec74
topic:: Runtime
date:: 2026-05-16 14:51:46
source:: 博客
confidence:: 0.90
tags:: Runtime, 博客
summary:: ObjC 1.0 中的 objc_class 结构体包含 isa、super_class、ivars、methodLists 等字段。；ObjC 2.0 中的 objc_object 包含 isa_t isa，objc_class 继承自 objc_object。；ObjC 2.0 类结构引入了 superclass、cache_t cache 和 class_data_bits_t bits。
**来源**: 博客　**confidence**: 0.90

- ObjC 1.0 中的 objc_class 结构体包含 isa、super_class、ivars、methodLists 等字段。
- ObjC 2.0 中的 objc_object 包含 isa_t isa，objc_class 继承自 objc_object。
- ObjC 2.0 类结构引入了 superclass、cache_t cache 和 class_data_bits_t bits。

### 整理后内容

## OC 1.0 vs 2.0 类结构对比

### Objc 1.0 objc_class 结构体

```objc
struct objc_class {
    Class isa;
    Class super_class;
    const char *name;
    long version;
    long info;
    long instance_size;
    objc_ivar_list *ivars;
    objc_method_list **methodLists;
    objc_cache *cache;
    objc_protocol_list *protocols;
};
```

### Objc 2.0 演进
- `objc_object` → 包含 `isa_t isa`
- `objc_class` → 继承自 `objc_object`，包含 `superclass`、`cache_t cache`、`class_data_bits_t bits`

> 图片原始文件已归档到 iOS-Inbox，这里提取核心知识点做结构化笔记。

<details><summary>原文</summary>

## OC 1.0 vs 2.0 类结构对比

### Objc 1.0 objc_class 结构体
```
struct objc_class {
    Class isa;
    Class super_class;
    const char *name;
    long version;
    long info;
    long instance_size;
    objc_ivar_list *ivars;
    objc_method_list **methodLists;
    objc_cache *cache;
    objc_protocol_list *protocols;
};
```

### Objc 2.0 演进
- `objc_object` → 包含 `isa_t isa`
- `objc_class` → 继承自 `objc_object`，包含 `superclass`、`cache_t cache`、`class_data_bits_t bits`

> 图片原始文件已归档到 iOS-Inbox，这里提取核心知识点做结构化笔记。

</details>

---


## 2026-05-16 16:27:54 Objective-C 运行时核心概念解析 ^35fa04
topic:: Runtime
date:: 2026-05-16 16:27:54
source:: 未知
confidence:: 0.80
tags:: Runtime
**来源**: 未知　**confidence**: 0.80


![[sec-507ca7f97db4.png]]

<details><summary>原文</summary>

低
高
Obj
self
super_class
_cmd
self

</details>

---
