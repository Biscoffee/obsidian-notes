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


---

# category简介

- **Category 用来给一个已经存在的类添加额外方法，而且不需要继承。**
- 可以把类的实现分开在几个不同的文件里面。这样做有几个显而易见的好处：a)可以减少单个文件的体积， b)可以把不同的功能组织到不同的category里， c)可以由多个开发者共同完成一个类， d)可以按需加载想要的category 等等。
- 声明私有方法
还有广大开发者衍生出的其他使用场景
- 模拟多继承
- 把framework私有方法公开

---

# category源码

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
category中没有iVars，也就是没有成员变量列表，这就解释了为什么他可以添加属性声明/属性元数据，但是不能直接添加成员变量因为对象的内存布局由类的 ivar list 决定。
一个对象创建时，它占多少内存、每个 ivar 在哪里，已经由类结构决定了。
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
所以 `method_t` 就是消息发送最终要找的核心记录.

这是Objective-C方法在Runtime的核心结构

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

除了标记存储的 category 的数量外，`locstamped_category_list_t` 结构体还声明了一个长度为零的数组，这其实是 C99 中的一种写法，允许我们在运行期动态的申请内存。

可以这样理解：
**`category_t`** **是真正的 Category 本体。**  
**`locstamped_category_t`** **是给 Category 加上“来源信息”的包装。**  
**`category_list`** **是多个** **`locstamped_category_t`** **组成的列表。**
```Objective-C
category_list
  ↓
里面有多个 locstamped_category_t

locstamped_category_t
  ↓
里面有 category_t *cat

category_t
  ↓
里面有 instanceMethods / classMethods

method_list_t
  ↓
里面有多个 method_t

method_t
  ↓
里面有 SEL name / const char *types / IMP imp
```
以上就是相关的数据结构，只要了解到这个程度就可以继续读源码了。


---
# category如何加载

```Objective-C
App 启动
    ↓
dyld 加载镜像
    ↓
runtime 调用 _objc_init
    ↓
map_images → 读取所有类和分类信息
    ↓
load_images → 调用 +load 方法
    ↓
attachCategories → 将分类方法合并到类的方法列表
```
```Objective-C
static void attachCategories(Class cls, category_list *cats, bool flush_caches) {
    if (!cats) return;
    bool isMeta = cls->isMetaClass();

    method_list_t **mlists = (method_list_t **)malloc(cats->count * sizeof(*mlists));
    // Count backwards through cats to get newest categories first
    int mcount = 0;
    int i = cats->count;
    while (i--) {
        auto& entry = cats->list[i];

        method_list_t *mlist = entry.cat->methodsForMeta(isMeta);
        if (mlist) {
            mlists[mcount++] = mlist;
        }
    }

    auto rw = cls->data();

    prepareMethodLists(cls, mlists, mcount, NO, fromBundle);
    rw->methods.attachLists(mlists, mcount);
    free(mlists);
    if (flush_caches  &&  mcount > 0) flushCaches(cls);
}
```
首先，通过 while 循环，我们遍历所有的 category，也就是参数 `cats` 中的 `list` 属性。对于每一个 category，得到它的方法列表 `mlist` 并存入 `mlists` 中。

换句话说，我们将所有 category 中的方法拼接到了一个大的二维数组中，数组的每一个元素都是装有一个 category 所有方法的容器。这句话比较绕，但你可以把 `mlists` 理解为文章开头所说，旧版本的 `objc_method_list **methodLists`。

在 while 循环外，我们得到了拼接成的方法，此时需要与类原来的方法合并:
```
auto rw = cls->data();
rw->methods.attachLists(mlists, mcount);
```
![](https://img.halfrost.com/Blog/ArticleImage/23_6.png)
![[Pasted image 20260504153045.png]]```

```
class_rw_t

bits 中包含了一个指向 class_rw_t 结构体的指针，它的定义如下:

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

`class_rw_t` 结构体中还有一个 `methods` 成员变量，它的类型是 `method_array_t`，继承自 `list_array_tt`。

`list_array_tt` 是一个泛型结构体，用于存储一些元数据，而它实际上是元数据的二维数组:
```
template <typename Element, typename List>{
    struct array_t {
        uint32_t count;
        List* lists[0];
    };
}
class method_array_t : public list_array_tt<method_t, method_list_t>

```

class_rw_t 结构体中还有一个 methods 成员变量，它的类型是 method_array_t，继承自 list_array_tt。
list_array_tt 是一个用于管理若干个 list 的容器。对 methods 来说，它管理的是多个 method_list_t，每个 method_list_t 里面再存多个 method_t。因此从逻辑上可以类比为二维数组。

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