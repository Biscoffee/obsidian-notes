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

> ivars 是 objc_ivar_list 成员变量列表的指针；methodLists 是指向 objc_method_list 指针的指针。*methodLists 是指向方法列表的指针。运行时可以把新的方法列表指针挂到 methodLists 管理的列表里，这就是早期 Runtime 中 Category 附加方法的核心思路之一。Category 不能添加 ivar，则和对象内存布局不能在运行时随意改变有关。

一级指针的话，`methodLists` 直接指向一个方法列表，想新增方法只能重新分配内存、拷贝旧列表、追加新方法，代价很高。

二级指针的话，`methodLists` 指向的是**一个指针数组**，每个元素指向一个独立的方法列表：
```
methodLists ──→ [ *list1, *list2, *list3, ... ]
                    ↓       ↓       ↓
                  原始方法  Category Category
                  列表      A的方法   B的方法
```
加载一个 Category 时，Runtime 不需要改写原来的方法列表内容，而是把 Category 自己的方法列表作为一个新的列表指针挂进去。可以把它理解成：**新增一组方法列表入口，而不是把方法逐个拷贝进原列表**。

`ivars` 是一级指针，且不能像方法列表一样动态扩展，根本原因是：**对象的内存布局不能在类已经投入运行后随意改变**。对象实例需要多大、每个 ivar 位于哪里，都会影响对象创建、成员访问和父子类布局。如果运行时强行插入一个新 ivar，后面 ivar 的偏移量、对象大小、已经创建的实例都会受到影响。

所以 `ivars` 根本没有设计成可动态扩展的，不是技术上做不到，而是**做了也没用，内存布局不允许**。

至于“Category 不能添加属性”这句话，更准确地说应该是“Category 不能添加带存储的属性”。
`@property` 可以拆成几类信息：

1. 一组属性元数据
2. 一个 getter 方法声明
3. 一个 setter 方法声明（只读属性没有 setter）
4. 在普通类扩展或类实现中，编译器还可能自动合成 ivar 和 getter/setter 实现

Category 里声明 `@property` 没问题，它会产生属性元数据，也会让编译器知道有 getter/setter 这两个访问入口。但 Category 不能自动合成 ivar，也不会自动生成 getter/setter 实现。所以准确的说法是：**Category 可以添加属性声明/属性元数据，但不能添加真正的 ivar 存储；如果要让这个属性可用，必须自己实现 getter/setter，通常再配合关联对象存值**。
这也是为什么用 Associated Objects 能绕过去——它根本没有修改对象的内存布局，而是在对象外部维护了一张独立的哈希表来存值，只是"看起来"像属性而已。


# OC 2.0
```Objective-C
Class
 └─ bits ─→ class_rw_t(运行时可写)
              ├─ ro ─→ class_ro_t(只读、编译期固定)
              │         ├─ baseMethods       ← 类自己写的方法
              │         ├─ ivars             ← 内存布局(不可改)
              │         ├─ baseProperties
              │         └─ baseProtocols
              ├─ methods       ← 这里!Category 方法附加到这里
              ├─ properties
              └─ protocols

```

![[Pasted image 20260504191505.png]]
- **`class_ro_t`** = read only。编译期定死的东西,放在 Mach-O 的只读段。包含 ivars、`instanceSize`、`baseMethods` 等等。
- **`class_rw_t`** = read write。运行时才创建,可写。Category 方法、`class_addMethod` 加的方法,全部进这里。

这里的图是理解模型。新版本 objc4 中还引入了 `class_rw_ext_t` 等扩展结构来承载部分可变数据，但从理解 Category 的角度看，仍然可以把它抽象成：**编译期固定信息在 ro，运行期可附加的方法、属性、协议在 rw/rw_ext 管理的列表里**。

在 2006 年，苹果在 WWDC 大会上就发布了 [Objective-C 2.0](https://en.wikipedia.org/wiki/Objective-C#Objective-C_2.0)，其中的改动包括 Mac OS X 平台上的垃圾回收机制(现已废弃)，runtime 性能优化等。

这意味着上述代码，以及任何带有 `OBJC2_UNAVAILABLE` 标记的内容，都已经在 2006 年就永远的告别了我们，只停留在历史的文档中。

虽然上述代码已经过时，但仍具备一定的参考意义，比如 `methodLists` 作为一个二级指针，可以帮助我们理解“多个方法列表挂到同一个类上”这件事：外层管理多个方法列表，内层每个方法列表里保存多个方法。


---

# category简介

- **Category 用来给一个已经存在的类添加额外方法，而且不需要继承。**
- 可以把类的实现分开在几个不同的文件里面。这样做有几个显而易见的好处：a)可以减少单个文件的体积， b)可以把不同的功能组织到不同的category里， c)可以由多个开发者共同完成一个类， d)可以通过动态库或 bundle 间接影响某些 Category 是否被加载。
- 声明私有方法

还有广大开发者衍生出的其他使用场景
- 模拟多继承
- 把framework私有方法公开


Category（分类）看起来和 Extension（扩展）有点相似。Extension（扩展）有时候也被称为 **匿名分类**。但两者实质上是不同的东西。 Extension（扩展）是在编译阶段与该类同时编译的，是类的一部分。而且 Extension（扩展）中声明的方法只能在该类的 `@implementation` 中实现，这也就意味着，你无法对系统的类（例如 NSString 类）使用 Extension（扩展）。

而且和 Category（分类）不同的是，Extension（扩展）不但可以声明方法，还可以声明成员变量，这是 Category（分类）所做不到的。

---

# category 数据结构

所有的OC类和对象，在runtime层都是用struct表示的，category也不例外，在runtime层，category用结构体category_t（在objc-runtime-new.h中可以找到此定义），它包含了：

- 1)、类的名字（name）
- 2)、类（cls）
- 3)、category中所有给类添加的实例方法的列表（instanceMethods）
- 4)、category中所有添加的类方法的列表（classMethods）
- 5)、category实现的所有协议的列表（protocols）
- 6)、category中添加的所有属性（instanceProperties）

```Objective-C
struct category_t {
    const char *name;  // 名字
    classref_t cls;  // 要附加的目标类
    struct method_list_t *instanceMethods;
    struct method_list_t *classMethods;  //添加的类方法列表，被附加到元类上
    struct protocol_list_t *protocols;//协议列表
    struct property_list_t *instanceProperties;//属性列表，但它不代表真的生成了_name成员变量，普通 Category 不能直接添加 ivar，因此在 Category 里只会生成 property metadata，不会自动生成存储空间。如果你想让这个属性真的能存值，需要自己实现 getter / setter，通常用关联对象：
};
```
> category中没有ivars，也就是没有成员变量列表，这就解释了为什么它可以添加属性声明/属性元数据，但是不能直接添加成员变量。因为对象的内存布局由类的 ivar list 决定。
一个对象创建时，它占多少内存、每个 ivar 在哪里，已经由类结构在 realization 后确定下来，运行期间添加成员变量会破坏原有内存布局。

Category 是后期附加元数据，不会改变对象内存大小

```Objective-C
struct method_list_t : entsize_list_tt<method_t, method_list_t, 0x3> {
    // 成员变量和方法
};

template <typename Element, typename List, uint32_t FlagMask>
struct entsize_list_tt {
    uint32_t entsizeAndFlags;
    uint32_t count;
    Element first;
};
```

这里的 `entsize_list_tt` 可以理解为一个容器，拥有自己的迭代器用于遍历所有元素。 `Element` 表示元素类型，`List` 用于指定容器类型，最后一个参数为标记位。

虽然这段代码实现比较复杂，但仍可了解到 `method_list_t` 是一个存储 `method_t` 类型元素的容器。`method_t` 结构体的定义如下:

```
struct method_t {
    SEL name;
    const char *types;   //  方法类型编码
    IMP imp;
};

```
当消息发送时，
```Objective-C
1. 从 obj 的 isa 找到类对象 MyClass
2. 先查方法缓存 cache
3. cache 没有，就查方法列表 method_list_t
4. 遍历 method_t
5. 找到 method_t.name == @selector(printName)
6. 拿到 method_t.imp
7. 调用 imp(obj, @selector(printName))
8. 可能把 selector -> imp 缓存起来
```
所以 `method_t` 就是消息发送最终要找的核心记录, 是Objective-C方法在Runtime的核心结构

最后，我们还有一个结构体 `category_list` 用来存储所有的 category，它的定义如下:

```Objective-C
struct locstamped_category_list_t {
    uint32_t count;
    locstamped_category_t list[0];
};
struct locstamped_category_t {
    category_t *cat;   // category本身
    struct header_info *hi;  // category来自的mach-o镜像信息
};
typedef locstamped_category_list_t category_list;
```

> `locstamped_category_t`：Category + 来源镜像
>`locstamped_category_list_t`/`category_list`：多个带来源信息的 Category

除了标记存储的 category 的数量外，`locstamped_category_list_t` 结构体还声明了一个长度为零的数组。这类写法常用于表达“结构体尾部跟着一段可变长度数据”；标准 C99 的 flexible array member 写法通常是 `list[]`，而 `list[0]` 更像是编译器扩展/历史写法。


可以这样理解：

- category_t 是一个具体 Category 的 runtime 本体；
- method_list_t 是 category_t 中保存实例方法/类方法的列表；
- method_list_t 继承自 entsize_list_tt，内部存放多个 method_t；
- method_t 是一个具体方法，包含 SEL、types、IMP；
- locstamped_category_t 是 category_t 的包装，额外记录它来自哪个 Mach-O 镜像；
- locstamped_category_list_t 是多个 locstamped_category_t 的列表；
- category_list 是 locstamped_category_list_t 的别名，runtime 用它保存某个类待附加的 Category 集合。

```Objective-C
category_list / locstamped_category_list_t
    └── locstamped_category_t
            ├── category_t *cat
            │       ├── method_list_t *instanceMethods
            │       │       └── entsize_list_tt<method_t, method_list_t, ...>
            │       │               └── method_t[]
            │       │                       ├── SEL name
            │       │                       ├── const char *types
            │       │                       └── IMP imp
            │       │
            │       ├── method_list_t *classMethods
            │       │       └── entsize_list_tt<method_t, method_list_t, ...>
            │       │               └── method_t[]
            │       │
            │       ├── protocol_list_t *protocols
            │       └── property_list_t *instanceProperties
            │
            └── header_info *hi
```
以上就是相关的数据结构，只要了解到这个程度就可以继续读源码了。


---
# category如何加载

dyld 加载的流程大致是这样：

1. 配置环境变量；
2. 加载共享缓存；
3. 初始化主 APP；
4. 插入动态缓存库；
5. 链接主程序；
6. 链接插入的动态库；
7. 初始化主程序：OC, C++ 全局变量初始化；
8. 返回主程序入口函数。

> 初始化主程序中，Runtime 初始化的大致调用栈如下：

>`dyldbootstrap::start` ---> `dyld::_main` ---> `initializeMainExecutable` ---> `runInitializers` ---> `recursiveInitialization` ---> `doInitialization` ---> `doModInitFunctions` ---> `_objc_init`

在 `_objc_init` 这一步中，`Runtime` 会向 `dyld` 注册回调。之后当 `image` 加载到内存后，`dyld` 会通知 `Runtime` 进行处理：`map_images` 负责解析和注册 image 中的 ObjC 元数据；`_read_images` / `load_categories_nolock` 会读取 Category 列表，并把 Category 的内容附加到目标类或元类上；随后 `load_images` 中会调用 `call_load_methods`，按规则调用 `Class` 和 `Category` 的 `+load` 方法。

>加载Category（分类）的调用栈如下：

> `_objc_init` ---> 注册 dyld 回调 ---> `map_images` / `map_images_nolock` ---> `_read_images` 或 `load_categories_nolock`（读取并附加分类） ---> `load_images`（调用 `+load`）。

**一些说明：**

不同版本的 objc4 在函数拆分和命名上会有差异。旧版源码(可在早期开源版本中找到)用 `addUnattachedCategoryForClass` + `remethodizeClass` + `attachCategories` 三个函数清晰地划分了"登记 → 取出 → 合并"三个步骤。新版源码做了路径优化:对已经 realized 的类直接调用 `attachCategories` 立即合并,只有未 realized 的类才会先放入 unattached category 表,等到类 realization 时再统一附加。

但要意识到,**新旧版的差异是性能优化,不是机制变化**——核心思路都是"延迟合并":Category 加载时如果目标类还没准备好,就先存起来,等准备好了再合。本文下面保留旧版函数名来讲解,因为它把这个思路展示得最完整。理解了旧版,新版只是少了一次绕路。



忽略 `_read_images` 方法中其他与本文无关的代码，得到如下代码：
```Objective-C
// 从当前 Mach-O 镜像 hi 中获取 Category 列表。
// catlist 指向当前镜像中的 category_t 指针数组。
// count 表示当前镜像中 Category 的数量。
category_t **catlist = _getObjc2CategoryList(hi, &count);

// 判断当前镜像是否包含 Category 的 class property 元数据。
// 新版 runtime 中会用这个标志决定是否读取 cat->_classProperties。
bool hasClassProperties = hi->info()->hasCategoryClassProperties();


// 遍历当前镜像中的所有 Category。
for (i = 0; i < count; i++) {

    // 取出一个 Category。
    category_t *cat = catlist[i];

    // 根据 Category 中记录的目标类引用，找到 runtime 中真正的 Class。
    // remapClass 用于处理类重映射、weak-linked class 等情况。
    Class cls = remapClass(cat->cls);

    // 如果目标类不存在，说明这个 Category 无法附加。
    // 常见情况：目标类是 weak-linked，但当前系统中不存在。
    if (!cls) {
        catlist[i] = NULL;
        continue;
    }


    // ============================================================
    // 一、处理 Category 的“实例侧内容”
    //
    // 包括：
    // 1. 实例方法 instanceMethods
    // 2. 协议 protocols
    // 3. 实例属性元数据 instanceProperties
    //
    // 这些内容会附加到普通类对象 cls 上。
    // ============================================================

    bool classExists = NO;

    if (cat->instanceMethods ||
        cat->protocols ||
        cat->instanceProperties)
    {
    // 先把 Category 登记到目标类 cls 对应的待附加列表中。
        //
        // 注意：
        // addUnattachedCategoryForClass 并不是真正合并方法列表。
        // 它只是建立：
        //
        // cls -> category
        //
        // 的关联关系。
        addUnattachedCategoryForClass(cat, cls, hi);

        // 如果目标类已经 realized，
        // 说明它已经完成 runtime 初始化，可以立刻合并 Category 内容。
        if (cls->isRealized()) {

            // 真正把 Category 的实例方法、属性、协议等附加到 cls 上。
            remethodizeClass(cls);

            classExists = YES;
        }
    }


    // ============================================================
    // 二、处理 Category 的“类侧内容”
    //
    // 包括：
    // 1. 类方法 classMethods
    // 2. 协议 protocols
    // 3. 类属性元数据 _classProperties
    //
    // 这些内容会附加到目标类的元类 cls->ISA() 上。
    // ============================================================

    if (cat->classMethods ||
        cat->protocols ||
        (hasClassProperties && cat->_classProperties))
    {
        // 类方法本质上是元类的实例方法，
        // 所以 Category 的类方法要登记到 cls->ISA() 上，
        // 也就是目标类的元类。
        addUnattachedCategoryForClass(cat, cls->ISA(), hi);

        // 如果元类已经 realized，
        // 就立刻把 Category 的类方法、类属性元数据、协议等附加进去。
        if (cls->ISA()->isRealized()) {
            remethodizeClass(cls->ISA());
        }
    }
}
```
这段伪代码主要用到了两个方法：

- `addUnattachedCategoryForClass(cat, cls, hi);` 把分类先登记到目标类的待附加列表里
- `remethodizeClass(cls);` 触发待附加分类真正合并到类的方法、属性、协议列表中

通过这两个方法达到了两个目的：

1. 把 `Category（分类）` 的对象方法、协议、属性添加到类上；
2. 把 `Category（分类）` 的类方法、协议、类属性元数据添加到类的 `metaclass` 上。

下面来说说上边提到的这两个方法。

```Objective-C
// 将一个 Category 暂存到“目标类 cls 的待附加 Category 列表”中。
// 注意：这个函数并不会真正把 Category 的方法、属性、协议合并到类上。
// 真正合并发生在后面的 remethodizeClass(cls) 中。
static void addUnattachedCategoryForClass(category_t *cat,
                                          Class cls,
                                          header_info *catHeader)
{
    // 当前必须已经持有 runtimeLock。
    // 因为这里要修改 runtime 的全局 Category 映射表。
    runtimeLock.assertLocked();

    // 获取全局的“未附加 Category 映射表”。
    //
    // 这个表可以理解为：
    //
    //     Class -> category_list
    //
    // 也就是：
    //
    //     某个类 -> 这个类当前等待附加的 Category 列表
    //
    NXMapTable *cats = unattachedCategories();

    category_list *list;

    // 根据目标类 cls，从映射表中取出它已有的待附加 Category 列表。
    list = (category_list *)NXMapGet(cats, cls);

    if (!list) {
        // 如果这个类还没有待附加 Category 列表，
        // 就创建一个新的 category_list。
        //
        // sizeof(*list)：
        //     category_list 结构体头部大小，里面包含 count。
        //
        // sizeof(list->list[0])：
        //     一个 locstamped_category_t 的大小。
        //
        // 因为 list 里使用了 list[0] 这种零长度数组技巧，
        // 所以创建第一个元素时，需要额外申请一个元素的空间。
        list = (category_list *)calloc(
            sizeof(*list) + sizeof(list->list[0]),
            1
        );
    } else {
        // 如果这个类已经有待附加 Category 列表，
        // 就扩容，多申请一个 locstamped_category_t 的空间。
        //
        // 原来能存 list->count 个 Category，
        // 现在要存 list->count + 1 个。
        list = (category_list *)realloc(
            list,
            sizeof(*list) + sizeof(list->list[0]) * (list->count + 1)
        );
    }

    // 把当前 Category 和它所在的 Mach-O 镜像信息一起存入列表。
    //
    // locstamped_category_t 包含：
    //     cat       : Category 本体
    //     catHeader : Category 来自哪个 Mach-O 镜像
    //
    // list->count++：
    //     先把元素写入当前位置，再把 count 加 1。
    list->list[list->count++] = (locstamped_category_t){ cat, catHeader };

    // 把更新后的 list 重新插入全局映射表。
    //
    // 如果之前 cats 中已经有 cls 对应的旧 list，
    // 这里会用扩容后的新 list 覆盖旧记录。
    NXMapInsert(cats, cls, list);
}
```
`addUnattachedCategoryForClass(cat, cls, hi);` 的执行过程可以参考代码注释。执行完这个方法之后，系统会将当前分类 `cat` 放到该类 `cls` 对应的未依附分类 `unattached` 的列表 `list` 中。这句话有点拗口，简而言之，就是：**先把类和分类做一个关联映射，真正合并稍后发生**。

实际上真正起到添加加载作用的是下边的 `remethodizeClass(cls);` 方法。

```Objective-C
static void remethodizeClass(Class cls)
{
    category_list *cats;
    bool isMeta;

    // 当前必须已经持有 runtimeLock。
    // 因为这里会读取并修改 runtime 中类的全局结构。
    runtimeLock.assertLocked();

    // 判断当前传入的 cls 是普通类，还是元类。
    //
    // 如果是普通类：
    //     处理 Category 的实例方法、实例属性、协议等。
    //
    // 如果是元类：
    //     处理 Category 的类方法、类属性、协议等。
    isMeta = cls->isMetaClass();

    // 从全局 unattachedCategories 表中取出 cls 对应的待附加 Category 列表。
    //
    // false 表示当前不是在 realizing class 的流程中调用。
    //
    // 返回的 cats 可以理解为：
    //
    //     cls -> [CategoryA, CategoryB, CategoryC]
    //
    // 这些 Category 之前是通过 addUnattachedCategoryForClass(cat, cls, hi)
    // 暂存进去的。
    cats = unattachedCategoriesForClass(cls, false /* not realizing */);

    if (cats) {
        // 真正把 cats 中的 Category 附加到 cls 上。
        //
        // attachCategories 会处理：
        //     1. 方法列表 methods
        //     2. 属性列表 properties
        //     3. 协议列表 protocols
        //
        // 如果 cls 是普通类：
        //     Category 的实例方法会被附加到 cls。
        //
        // 如果 cls 是元类：
        //     Category 的类方法会被附加到 cls。
        //
        // true 表示附加完成后需要刷新方法缓存。
        attachCategories(cls, cats, true /* flush caches */);

        // cats 是从 unattachedCategoriesForClass 取出的临时列表。
        // attach 完成后释放。
        free(cats);
    }
}
```

`remethodizeClass(cls);` 方法主要就做了一件事：调用 `attachCategories(cls, cats, true);` 方法将未依附分类的列表 cats 附加到 cls 类上。所以，我们就再来看看 `attachCategories(cls, cats, true);` 方法。

```Objective-C
static void attachCategories(Class cls,
                             category_list *cats,
                             bool flush_caches)
{
    if (!cats) return;

    if (PrintReplacedMethods) {
        printReplacements(cls, cats);
    }

    // 判断当前 cls 是普通类还是元类。
    // 普通类：处理 Category 的实例方法、实例属性、协议。
    // 元类：处理 Category 的类方法、类属性、协议。
    bool isMeta = cls->isMetaClass();

    // 创建三个临时数组，用来收集所有 Category 中的方法列表、属性列表、协议列表。
    method_list_t **mlists = (method_list_t **)
        malloc(cats->count * sizeof(*mlists));

    property_list_t **proplists = (property_list_t **)
        malloc(cats->count * sizeof(*proplists));

    protocol_list_t **protolists = (protocol_list_t **)
        malloc(cats->count * sizeof(*protolists));

    // 记录实际收集到的方法列表、属性列表、协议列表数量。
    int mcount = 0;
    int propcount = 0;
    int protocount = 0;

    // 从后往前遍历 Category。
    // 这样可以保证“最新加载的 Category”排在前面。
    int i = cats->count;

    // 标记这些 Category 是否来自 bundle。
    bool fromBundle = NO;

    while (i--) {
        // 取出当前 Category 以及它所在的 Mach-O 镜像信息。
        auto& entry = cats->list[i];

        // 取出当前 Category 的方法列表。
        //
        // 如果 cls 是普通类：
        //     methodsForMeta(false) 取 instanceMethods。
        //
        // 如果 cls 是元类：
        //     methodsForMeta(true) 取 classMethods。
        method_list_t *mlist = entry.cat->methodsForMeta(isMeta);

        if (mlist) {
            // 将当前 Category 的方法列表保存到临时方法列表数组中。
            mlists[mcount++] = mlist;

            // 记录是否有来自 bundle 的 Category。
            fromBundle |= entry.hi->isBundle();
        }

        // 取出当前 Category 的属性列表。
        //
        // 如果 cls 是普通类：
        //     通常取 instanceProperties。
        //
        // 如果 cls 是元类：
        //     可能取 classProperties；
        //     如果当前镜像不支持 classProperties，则可能返回 nil。
        property_list_t *proplist =
            entry.cat->propertiesForMeta(isMeta, entry.hi);

        if (proplist) {
            proplists[propcount++] = proplist;
        }

        // 取出当前 Category 声明遵守的协议列表。
        protocol_list_t *protolist = entry.cat->protocols;

        if (protolist) {
            protolists[protocount++] = protolist;
        }
    }

    // 取出当前类的运行时可变数据 class_rw_t。
    auto rw = cls->data();

    // 对即将附加的方法列表做准备工作。
    //
    // 例如：
    //     排序
    //     修正方法列表
    //     检查特殊方法
    //     标记是否来自 bundle
    prepareMethodLists(cls, mlists, mcount, NO, fromBundle);

    // 将 Category 的方法列表附加到 class_rw_t 的 methods 中。
    //
    // 注意：
    //     attachLists 会把新列表放到旧列表前面。
    //     所以 Category 的同名方法会优先于原类方法被查找到。
    rw->methods.attachLists(mlists, mcount);

    // 临时数组已经被 attachLists 使用完，可以释放。
    free(mlists);

    // 如果需要刷新缓存，并且确实附加了方法列表，就清理方法缓存。
    //
    // 因为 Category 可能添加或“覆盖”同名方法，
    // 如果不清缓存，objc_msgSend 可能继续命中旧 IMP。
    if (flush_caches && mcount > 0) {
        flushCaches(cls);
    }

    // 将 Category 的属性列表附加到 class_rw_t 的 properties 中。
    rw->properties.attachLists(proplists, propcount);

    // 释放属性临时数组。
    free(proplists);

    // 将 Category 的协议列表附加到 class_rw_t 的 protocols 中。
    rw->protocols.attachLists(protolists, protocount);

    // 释放协议临时数组。
    free(protolists);
}
```
首先，通过 while 循环，我们遍历所有的 category，也就是参数 `cats` 中的 `list` 属性。对于每一个 category，得到它的方法列表 `mlist` 并存入 `mlists` 中。

换句话说，我们将所有 category 中的方法拼接到了一个大的二维数组中，数组的每一个元素都是装有一个 category 所有方法的容器。这句话比较绕，但你可以把 `mlists` 理解为文章开头所说，旧版本的 `objc_method_list **methodLists`。

在 while 循环外，我们得到了拼接成的方法，此时需要与类原来的方法合并:
```
auto rw = cls->data();
rw->methods.attachLists(mlists, mcount);
```

## class_rw_t

bits 中包含了一个指向 class_rw_t 结构体的指针，它的定义如下:

```Objective-C
struct class_rw_t {
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;

    method_array_t methods;
    property_array_t properties;
    protocol_array_t protocols;
}
```
在 Objective-C 2.0 的 non-fragile ABI 中，类的实例布局信息保存在 class_ro_t 中，例如 instanceStart 和 instanceSize 分别描述本类实例变量的起始位置和对象实例总大小。更重要的是，ivar 的偏移量不再被编译期完全写死，而是通过 ivar_list_t 中的 offset 指针等运行时元数据进行间接访问。这样当父类新增实例变量导致实例大小变化时，runtime 可以在类 realization 阶段重新调整子类 ivar 的偏移量，从而避免子类必须重新编译。这就是 Objective-C 2.0 ABI 稳定性的一部分。

但这并不意味着 Category 可以在运行时给一个已经存在的类插入 ivar。non-fragile ABI 解决的是“父类布局变化时，子类不必重新编译”的问题；Category 加载发生在类的运行时元数据附加阶段，它只能附加方法、属性元数据和协议，不能改变对象实例大小。

`class_rw_t` 结构体中还有一个 `methods` 成员变量，它的类型是 `method_array_t`，继承自 `list_array_tt`。

`list_array_tt` 是一个泛型结构体，用于存储一些元数据，而它实际上是元数据的二维数组:
```Objective-C
template <typename Element, typename List>{
    struct array_t {
        uint32_t count;
        List* lists[0];
    };
}
class method_array_t : public list_array_tt<method_t, method_list_t>

```

更准确地说，list_array_tt 是一个用于管理若干个 list 的容器。对 methods 来说，它管理的是多个 method_list_t，每个 method_list_t 里面再存多个 method_t。因此从逻辑上可以类比为二维数组。

```Objective-C
template <typename Element, typename List>
class list_array_tt {
    struct array_t {
        uint32_t count;
        List* lists[0];
    };

    // 内部可能用一个指针/位标记来表示 empty / single list / array
};
```
Element 表示元数据的类型，比如 method_t，而 List 则表示用于存储元数据的一维数组，比如 method_list_t。

list_array_tt有三种状态：
1. 空：没有任何 method_list_t，可以类比为 []
2. 单列表：直接指向一个 method_list_t，可以类比为 [[m1, m2]]
3. 多列表：指向一个 array_t，array_t 里保存多个 method_list_t*，可以类比为 [[m1, m2], [m3, m4]]

类刚 realize 后，通常会把 `class_ro_t` 里的 `baseMethods` 放进 `class_rw_t.methods`。如果类本身没有方法，可能为空；如果有方法，可能是单列表状态。
当 runtime 后续通过 Category、class_addMethod 等方式追加新的方法列表时，methods 通常会从空/单列表形态扩展为多列表形态。

```
void attachLists(List* const * addedLists, uint32_t addedCount) {
    if (addedCount == 0) return;
    uint32_t oldCount = array()->count;
    uint32_t newCount = oldCount + addedCount;
    setArray((array_t *)realloc(array(), array_t::byteSize(newCount)));
    array()->count = newCount;
    memmove(array()->lists + addedCount, array()->lists, oldCount * sizeof(array()->lists[0]));
    memcpy(array()->lists, addedLists, addedCount * sizeof(array()->lists[0]));
}
```
这段代码很简单，其实就是先调用 `realloc()` 函数将原来的空间拓展，然后把原来的数组复制到后面，最后再把新数组复制到前面。

在实际代码中，比上面略复杂一些。因为为了提高性能，苹果做了一些优化，比如当 List 处于第二种状态(只有一个指针，指向一个元数据的集合)时，其实并不需要在原地扩容空间，而是只要重新申请一块内存，并将最后一个位置留给原来的集合即可。

这样只多花费了很少的内存空间，也就是原来二维数组占用的内存空间，但是 `malloc()`的性能优势会更加明显，这其实是一个空间换时间的权衡问题。

需要注意的是，无论执行哪种逻辑，参数列表中的方法都会被添加到二维数组的前面。而我们简单的看一下 runtime 在查找方法时的逻辑:
```Objective-C
static method_t *getMethodNoSuper_nolock(Class cls, SEL sel){
    for (auto mlists = cls->data()->methods.beginLists(), 
              end = cls->data()->methods.endLists(); 
         mlists != end;
         ++mlists) {
        method_t *m = search_method_list(*mlists, sel);
        if (m) return m;
    }

    return nil;
}

static method_t *search_method_list(const method_list_t *mlist, SEL sel) {
    for (auto& meth : *mlist) {
        if (meth.name == sel) return &meth;
    }
}
```
可见搜索的过程是按照从前向后的顺序进行的，一旦找到了就会停止循环。因此 category 中定义的同名方法不会替换类中原有的方法，但是对原方法的调用实际上会调用 category 中的方法。

## Category 加载流程总览

把上面的源码细节收起来，Category 的加载流程可以理解成下面这条链路：

```
1. 编译器把 Category 编译成 category_t
2. category_t 被放进 Mach-O 的 __objc_catlist
3. runtime 在 _read_images 中读取 __objc_catlist
4. 得到 category_t **catlist
5. 遍历每个 category_t
6. 找到它的目标类 cls
7. 调用 addUnattachedCategoryForClass(cat, cls, hi)
8. 把 category_t 和 header_info 包装成 locstamped_category_t
9. 放入 cls 对应的 category_list
10. 如果 cls 已经 realized，调用 remethodizeClass(cls)
11. remethodizeClass 取出 category_list
12. attachCategories 遍历 category_list
13. 从每个 locstamped_category_t 里取出 category_t
14. 从 category_t 里取出 method_list_t
15. 把 method_list_t attach 到 class_rw_t.methods
```

这里最关键的点有三个：

1. Category 不是把方法“拷贝进类原来的方法列表”，而是把自己的 `method_list_t` 挂到类的 `method_array_t` 前面。
2. Category 的实例方法附加到类对象；Category 的类方法附加到元类，因为类方法本质上是元类的实例方法。
3. Category 中的同名方法并没有真正删除或替换原方法，只是在普通消息查找顺序上排在更前面，所以表现为“覆盖”。


# Category 和 Class 的 +load 方法

Category（分类）中的方法、属性元数据、协议附加到类上的操作，是在 `+ load` 方法执行之前进行的。也就是说，在 `+ load` 方法执行之前，类中就已经能查到 Category（分类）中的方法、属性元数据、协议。

而 Category（分类）和 Class（类）的 `+ load` 方法的调用顺序规则如下所示：

1. 先调用主类，按照继承关系由父类向子类调用；同一层级中再受编译/链接顺序影响；
2. 调用完主类，再调用分类；同一个 Mach-O 内通常受编译/链接顺序影响，跨动态库、bundle 或不同 image 时不应该依赖这个顺序；
3. `+ load` 方法除非主动调用，否则只会调用一次。

`+load` 方法不会因为 Category 中也实现了 `+load` 就把主类的 `+load` 覆盖掉，因为 runtime 调用 `+load` 时是直接拿到对应方法的 IMP 调用，而不是经过普通的 `objc_msgSend` 查找流程。普通方法“看起来被覆盖”的底层原因，是 `objc_msgSend` 会通过 isa 指针到方法列表中按顺序查找，先找到谁就调用谁。

通过这样的调用规则，我们可以知道：主类的 `+ load` 方法调用一定在分类 `+ load` 方法调用之前。但是分类 `+ load` 方法调用顺序并不是按照继承关系调用的，而是受 image 加载顺序、编译/链接顺序等因素影响。因此，不要把多个 Category 的 `+load` 顺序设计成业务依赖。一个可能的顺序是：`父类 -> 子类 -> 父类类别 -> 子类类别`，也可能是 `父类 -> 子类 -> 子类类别 -> 父类类别`。


# Category与关联对象

之前我们提到过，在 Category 中虽然可以声明属性，但是不会自动生成对应的成员变量，也不会自动生成 `getter`、`setter` 实现。因此，如果只声明属性但不自己实现访问器，在调用 Category 中声明的属性时就会因为找不到方法而报错。

那么就没有办法使用 Category 中的属性了吗？

答案当然是否定的。

我们可以自己来实现 `getter`、`setter` 方法，并借助关联对象（Objective-C Associated Objects）来实现 `getter`、`setter` 方法。关联对象能够帮助我们在运行时阶段将任意的属性关联到一个对象上。具体需要用到以下几个方法：

```Objective-C
// 1. 通过 key : value 的形式给对象 object 设置关联属性
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy);

// 2. 通过 key 获取关联的属性 object`
id objc_getAssociatedObject(id object, const void *key);

// 3. 使用前两个后，使用第三个移除对象所关联的属性
void objc_removeAssociatedObjects(id object);
```


# 其他

- 在多个分类中实现同名方法时，普通消息发送会命中方法列表中最靠前的那个实现。通常可以理解为“后附加的分类优先级更高”，但跨 image 的加载顺序不应依赖。正常的 `[obj method]` 调用无法指定调用被遮蔽的主类实现或某个特定分类实现；如果业务确实需要保留旧实现，应该在附加/交换前主动保存 IMP，或者避免在多个 Category 中定义同名方法。



https://itcharge.cn/tech/ios-dev/ios-runtime-03/?utm_source=chatgpt.com
https://tech.meituan.com/2015/03/03/diveintocategory.html?utm_source=chatgpt.com
