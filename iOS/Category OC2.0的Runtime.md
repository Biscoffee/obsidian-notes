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
    const char *name;
    classref_t cls;
    struct method_list_t *instanceMethods;
    struct method_list_t *classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;
};
```

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
    const char *types;
    IMP imp;
};

```

最后，我们还有一个结构体 `category_list` 用来存储所有的 category，它的定义如下:

```
struct locstamped_category_list_t {
    uint32_t count;
    locstamped_category_t list[0];
};
struct locstamped_category_t {
    category_t *cat;
    struct header_info *hi;
};
typedef locstamped_category_list_t category_list;
```

除了标记存储的 category 的数量外，`locstamped_category_list_t` 结构体还声明了一个长度为零的数组，这其实是 C99 中的一种写法，允许我们在运行期动态的申请内存。

以上就是相关的数据结构，只要了解到这个程度就可以继续读源码了。
---

# category如何加载

```Objective-C
void _objc_init(void)
{
    static bool initialized = false;
    if (initialized) return;
    initialized = true;
   
    environ_init();
    tls_init();
    lock_init();
    exception_init();
       
    _dyld_register_func_for_remove_image(&unmap_image);
    dyld_register_image_state_change_handler(dyld_image_state_bound,
                                             1/*batch*/, &map_images);
    //  当一个mach-o镜像被加载并完成绑定时，调用map_images
    dyld_register_image_state_change_handler(dyld_image_state_dependents_initialized, 0/*not batch*/, &load_images);
    //当依赖镜像初始化完成后，调用load——images
}
```
这段代码是Objc的Runtime初始化入口，这段代码不直接把Category挂在累上，他做的事注册回调。
Category 是在 `map_images` 阶段附加到类上的，更准确地说，是 `map_images` 触发 `_read_images`，然后 `_read_images` 中发现并处理 Category。

read_images 里的 “Discover categories”
```Objective-C
// Discover categories. 
		for (EACH_HEADER) {//遍历每个mach-o镜像
    category_t **catlist =
	        _getObjc2CategoryList(hi, &count);//  从当前 Mach-O 镜像中取出 Category 列表。
    for (i = 0; i < count; i++) {
	    // 从所有category数组中取出来
        category_t *cat = catlist[i];
        // 找到目标lei
        class_t *cls = remapClass(cat->cls);

        if (!cls) {
            catlist[i] = NULL;
            ...
            continue;
        }

        BOOL classExists = NO;
        //  处理实例方法、协议、实例属性
        if (cat->instanceMethods ||  cat->protocols 
            ||  cat->instanceProperties)
        {
            addUnattachedCategoryForClass(cat, cls, hi);
            if (isRealized(cls)) {
                remethodizeClass(cls);
                classExists = YES;
            }
        }

        if (cat->classMethods  ||  cat->protocols)// 处理类方法和协议
        {
            addUnattachedCategoryForClass(cat, cls->isa, hi);
            if (isRealized(cls->isa)) {
                remethodizeClass(cls->isa);
            }
        }
    }
}
```
我们接着往里看，category的各种列表是怎么最终添加到类上的，就拿实例方法列表来说吧：

在上述的代码片段里，addUnattachedCategoryForClass只是把类和category做一个关联映射，而remethodizeClass才是真正去处理添加事宜的功臣。

```Objective-C
static void remethodizeClass(class_t *cls)
{
    category_list *cats;
    BOOL isMeta;

    rwlock_assert_writing(&runtimeLock);//这表示调用这个函数时必须持有 runtime 写锁, 因为他要修改类的数据结构，这些都是全局运行时结构，不加锁会出问题

    isMeta = isMetaClass(cls);
    // 如果是普通类对象，处理 Category 的实例方法、实例属性等。  
// 如果是元类对象，处理 Category 的类方法等。

    if ((cats = unattachedCategoriesForClass(cls))) {
        chained_property_list *newproperties;
        const protocol_list_t **newprotos;
       
        BOOL vtableAffected = NO;
        attachCategoryMethods(cls, cats, &vtableAffected);
       
        newproperties = buildPropertyList(NULL, cats, isMeta);
        if (newproperties) {
            newproperties->next = cls->data()->properties;
            cls->data()->properties = newproperties;
        }
       
        newprotos = buildProtocolList(cats, NULL, cls->data()->protocols);
        if (cls->data()->protocols  &&  cls->data()->protocols != newprotos) {
            _free_internal(cls->data()->protocols);
        }
        cls->data()->protocols = newprotos;
       
        _free_internal(cats);

        flushCaches(cls);
        if (vtableAffected) flushVtables(cls);
    }
}
```