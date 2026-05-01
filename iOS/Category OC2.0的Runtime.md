# OC 1.0
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



# OC 2.0
在 2006 年，苹果在 WWDC 大会上就发布了 [Objective-C 2.0](https://en.wikipedia.org/wiki/Objective-C#Objective-C_2.0)，其中的改动包括 Mac OS X 平台上的垃圾回收机制(现已废弃)，runtime 性能优化等。

这意味着上述代码，以及任何带有 `OBJC2_UNAVAILABLE` 标记的内容，都已经在 2006 年就永远的告别了我们，只停留在历史的文档中。

虽然上述代码已经过时，但仍具备一定的参考意义，比如 `methodLists` 作为一个二级指针，其中每个元素都是一个数组，数组中的每个元素则是一个方法。