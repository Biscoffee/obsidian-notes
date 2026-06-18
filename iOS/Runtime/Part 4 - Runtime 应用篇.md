---
title: "【iOS】Runtime - Part 4 && Runtime 应用篇：动态能力如何落到业务代码"
published: 2026-06-16
description: "承接对象结构、消息发送与 Category 加载机制，梳理 Runtime 在业务代码中的典型应用：动态获取类信息、动态添加方法、Method Swizzling、关联对象与消息转发。"
tags: ["iOS", "Objective-C", "Runtime", "Method Swizzling", "objc4"]
category: "iOS"
series: "iOS Runtime 系列"
seriesSlug: "ios-runtime"
seriesOrder: 4
draft: true
---

# 这篇在系列中的位置

前三篇已经把 Runtime 的底层链路铺开了：

- Part 1：对象、类、元类、`isa`、`class_ro_t` / `class_rw_t`。
- Part 2：`objc_msgSend`、方法缓存、慢速查找、动态方法解析、消息转发。
- Part 3：Category 的编译产物、运行时加载、方法列表附加、覆盖假象和关联对象。

Part 4 不再继续只追源码细节，而是把这些机制拉回日常开发：**我们平时说的 Runtime 应用，到底是在使用哪几类动态能力？哪些用法是合理的，哪些用法容易把项目带进不可维护的状态？**

本文的主线是：

> Runtime 应用不是“黑魔法”的集合，而是围绕四件事展开：看见类结构、修改类行为、改变消息分发、给对象外挂数据。

# Runtime 应用到底在用什么能力

常见 Runtime 用法大致可以分成四类：

| 能力 | 典型 API | 背后的机制 |
| --- | --- | --- |
| 查看类结构 | `class_copyIvarList` / `class_copyPropertyList` / `class_copyMethodList` | 读取类元数据 |
| 动态加方法 | `class_addMethod` / `resolveInstanceMethod:` | 修改 `class_rw_t` / 方法解析 |
| 替换方法实现 | `method_exchangeImplementations` / `method_setImplementation` | 改变 `SEL -> IMP` 关系 |
| 外挂数据 | `objc_setAssociatedObject` / `objc_getAssociatedObject` | 对象外部关联表 |

这几类能力都能在前几篇里找到落点：

- “查看类结构”对应 Part 1 的类元数据。
- “动态加方法”对应 Part 2 的动态方法解析。
- “替换方法实现”对应 Part 2 的 `SEL -> IMP` 查找结果。
- “外挂数据”对应 Part 3 的关联对象。

# 1. 动态获取类信息

Runtime 最温和的一类应用，是在运行时读取类的 ivar、property、method、protocol 信息。它不改变程序行为，只是把编译进 Mach-O 的 Objective-C 元数据拿出来看。

常用 API：

```objc
unsigned int count = 0;

Ivar *ivars = class_copyIvarList([Person class], &count);
objc_property_t *properties = class_copyPropertyList([Person class], &count);
Method *methods = class_copyMethodList([Person class], &count);
Protocol *__unsafe_unretained *protocols = class_copyProtocolList([Person class], &count);
```

这些 `copy` 系列 API 返回的是 C 数组，用完要 `free`：

```objc
unsigned int count = 0;
objc_property_t *properties = class_copyPropertyList([Person class], &count);

for (unsigned int i = 0; i < count; i++) {
    const char *name = property_getName(properties[i]);
    const char *attrs = property_getAttributes(properties[i]);
    NSLog(@"%s: %s", name, attrs);
}

free(properties);
```

典型应用：

- 打印对象调试信息。
- 做轻量字段映射。
- 自动归档 / 解档。
- 字典转模型。
- 检查某个类是否实现了某些方法。

这类用法风险较低，但也要注意：**property 和 ivar 不是一回事**。`@property` 是属性元数据和访问器约定，ivar 才是真正的实例存储。Category 可以加 property 元数据，但不能加 ivar，这一点已经在 Part 3 讲过。

# 2. 动态添加方法

动态添加方法的核心 API 是：

```objc
BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types);
```

它做的事情可以粗略理解为：往类的运行时方法列表里追加一条 `SEL -> IMP` 记录。

示例：

```objc
void dynamicRun(id self, SEL _cmd) {
    NSLog(@"%@ %@", self, NSStringFromSelector(_cmd));
}

BOOL added = class_addMethod([Person class],
                             @selector(run),
                             (IMP)dynamicRun,
                             "v@:");
```

类型编码 `"v@:"` 的意思是：

- `v`：返回值是 `void`。
- `@`：第一个隐藏参数 `self`，类型是对象。
- `:`：第二个隐藏参数 `_cmd`，类型是 selector。

动态添加方法最经典的落点，是 Part 2 讲过的动态方法解析：

```objc
+ (BOOL)resolveInstanceMethod:(SEL)sel {
    if (sel == @selector(run)) {
        class_addMethod(self, sel, (IMP)dynamicRun, "v@:");
        return YES;
    }
    return [super resolveInstanceMethod:sel];
}
```

这条链路是：

```text
objc_msgSend
  -> cache miss
  -> 方法列表没找到
  -> resolveInstanceMethod:
  -> class_addMethod
  -> 重新查找
  -> 命中新 IMP
```

也就是说，动态方法解析不是“直接处理消息”，而是**给类补一条真正的方法记录**。补上之后，后续同样的消息仍然回到正常的消息发送路径。

# 3. Method Swizzling

Method Swizzling 是 Runtime 应用里最常见、也最容易出问题的一类。

它的本质不是“交换两个方法名”，而是交换两个 selector 对应的 IMP：

```text
Before:
SEL A -> IMP A
SEL B -> IMP B

After:
SEL A -> IMP B
SEL B -> IMP A
```

常见 API：

```objc
void method_exchangeImplementations(Method m1, Method m2);
IMP method_setImplementation(Method m, IMP imp);
BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types);
void class_replaceMethod(Class cls, SEL name, IMP imp, const char *types);
```

最基础的交换写法：

```objc
Method original = class_getInstanceMethod([UIViewController class],
                                          @selector(viewDidAppear:));
Method swizzled = class_getInstanceMethod([UIViewController class],
                                          @selector(tw_viewDidAppear:));

method_exchangeImplementations(original, swizzled);
```

Category 里通常会这样写：

```objc
@implementation UIViewController (TWTracking)

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class cls = [self class];

        SEL originalSEL = @selector(viewDidAppear:);
        SEL swizzledSEL = @selector(tw_viewDidAppear:);

        Method originalMethod = class_getInstanceMethod(cls, originalSEL);
        Method swizzledMethod = class_getInstanceMethod(cls, swizzledSEL);

        BOOL didAdd = class_addMethod(cls,
                                      originalSEL,
                                      method_getImplementation(swizzledMethod),
                                      method_getTypeEncoding(swizzledMethod));

        if (didAdd) {
            class_replaceMethod(cls,
                                swizzledSEL,
                                method_getImplementation(originalMethod),
                                method_getTypeEncoding(originalMethod));
        } else {
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}

- (void)tw_viewDidAppear:(BOOL)animated {
    [self tw_viewDidAppear:animated];
    NSLog(@"track page: %@", NSStringFromClass([self class]));
}

@end
```

这里最容易绕住的一点是：交换后，`tw_viewDidAppear:` 这个 selector 指向了原来的 `viewDidAppear:` IMP，所以在 `tw_viewDidAppear:` 里调用 `[self tw_viewDidAppear:animated]`，并不是递归调用自己，而是在调用原实现。

## 为什么经常写在 `+load`

`+load` 在类和 Category 被加载进 Runtime 时直接调用，不依赖消息发送，因此很适合做“尽早生效”的方法替换。

但这也意味着它有明显代价：

- 调用时机很早，很多业务环境还没准备好。
- 所有 Category 的 `+load` 都会执行，启动期成本容易累积。
- 多个库同时 swizzle 同一个方法时，顺序不应该被业务依赖。

所以 `+load` 可以用，但不要把复杂业务逻辑放进去。它更适合只做一次非常小的交换动作。

## 实例方法和类方法

实例方法存在类对象的方法列表里：

```objc
Method m = class_getInstanceMethod([Person class], @selector(run));
```

类方法本质上是元类的实例方法，所以要去元类上找：

```objc
Class metaClass = object_getClass([Person class]);
Method m = class_getClassMethod([Person class], @selector(create));
```

或者理解成：

```text
实例方法：对象 -> 类 -> 方法列表
类方法：类对象 -> 元类 -> 方法列表
```

这正好呼应 Part 1 的元类结构。

## 继承链上的坑

如果当前类没有实现某个方法，而是继承自父类，直接 exchange 可能会影响父类的方法实现关系。更稳妥的模板通常是：

1. 先尝试 `class_addMethod`，把原 selector 加到当前类。
2. 如果添加成功，再用 `class_replaceMethod` 把 swizzled selector 替换成原实现。
3. 如果添加失败，说明当前类本身已有实现，再直接 exchange。

这就是上面模板里 `didAdd` 分支的意义。

# 4. 关联对象应用

Part 3 已经讲过关联对象的底层结构。放到应用层，最常见的用途是在 Category 里添加“看起来像属性”的存储。

```objc
@interface UIViewController (TWTracking)
@property (nonatomic, copy) NSString *tw_pageName;
@end

@implementation UIViewController (TWTracking)

static const void *TWPageNameKey = &TWPageNameKey;

- (void)setTw_pageName:(NSString *)tw_pageName {
    objc_setAssociatedObject(self,
                             TWPageNameKey,
                             tw_pageName,
                             OBJC_ASSOCIATION_COPY_NONATOMIC);
}

- (NSString *)tw_pageName {
    return objc_getAssociatedObject(self, TWPageNameKey);
}

@end
```

key 的常见写法：

```objc
static const void *Key = &Key;
```

它的优点是地址稳定、唯一性强、不依赖字符串内容。

内存策略可以按属性语义选：

| 属性语义 | 关联策略 |
| --- | --- |
| `assign` | `OBJC_ASSOCIATION_ASSIGN` |
| `strong, nonatomic` | `OBJC_ASSOCIATION_RETAIN_NONATOMIC` |
| `copy, nonatomic` | `OBJC_ASSOCIATION_COPY_NONATOMIC` |
| `strong, atomic` | `OBJC_ASSOCIATION_RETAIN` |
| `copy, atomic` | `OBJC_ASSOCIATION_COPY` |

注意：关联对象不是 ivar。它不会改变对象大小，也不会改变 ivar 偏移。它只是把值放进 Runtime 维护的外部表里。

# 5. 消息转发应用

消息转发可以用于代理转发、容错、多播等场景。它承接 Part 2 的转发链：

```text
forwardingTargetForSelector:
  -> methodSignatureForSelector:
  -> forwardInvocation:
  -> doesNotRecognizeSelector:
```

如果只是把消息交给另一个对象，优先用快速转发：

```objc
- (id)forwardingTargetForSelector:(SEL)aSelector {
    if ([self.helper respondsToSelector:aSelector]) {
        return self.helper;
    }
    return [super forwardingTargetForSelector:aSelector];
}
```

如果要记录参数、修改参数、延迟执行、转发给多个对象，才进入完整转发：

```objc
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    NSMethodSignature *signature = [super methodSignatureForSelector:aSelector];
    if (!signature) {
        signature = [self.target methodSignatureForSelector:aSelector];
    }
    return signature;
}

- (void)forwardInvocation:(NSInvocation *)invocation {
    if ([self.target respondsToSelector:invocation.selector]) {
        [invocation invokeWithTarget:self.target];
        return;
    }
    [super forwardInvocation:invocation];
}
```

消息转发很灵活，但不适合做普通业务分支。它会让调用关系变得隐蔽，调试成本也更高。能用明确协议、组合、代理解决的问题，不要优先用消息转发。

# 6. NSProxy + forwardInvocation: 实现 AOP

`forwardInvocation:` 更完整的应用，是用 `NSProxy` 做一层动态代理，把一次消息调用包起来，在目标方法前后插入额外逻辑。这就是 AOP（Aspect Oriented Programming，面向切面编程）在 Objective-C 里的经典做法之一。

`NSProxy` 和 `NSObject` 一样，都是 Objective-C 根层级里的抽象基类。不同的是，`NSProxy` 天生就是为了“转发消息”设计的：它本身不像 `NSObject` 那样带大量默认行为，而是要求子类重点实现：

```objc
- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel;
- (void)forwardInvocation:(NSInvocation *)invocation;
```

系统里常见的 `NSDistantObject`、`NSProtocolChecker` 都是基于 `NSProxy` 思路做消息代理或协议检查。

下面搭一个最小 AOP 代理：调用目标方法前后，分别执行 `preInvoke:` 和 `postInvoke:`。

```objc
@protocol Invoker <NSObject>
- (void)preInvoke:(NSInvocation *)invocation;
- (void)postInvoke:(NSInvocation *)invocation;
@end

@interface AuditingInvoker : NSObject <Invoker>
@end

@implementation AuditingInvoker

- (void)preInvoke:(NSInvocation *)invocation {
    NSLog(@"before %@", NSStringFromSelector(invocation.selector));
}

- (void)postInvoke:(NSInvocation *)invocation {
    NSLog(@"after %@", NSStringFromSelector(invocation.selector));
}

@end
```

代理对象保存三个东西：

- `proxyTarget`：真正接收消息的对象。
- `invoker`：切面逻辑执行者。
- `selectors`：哪些 selector 需要切面增强。

```objc
@interface AspectProxy : NSProxy

@property (nonatomic, strong, readonly) id proxyTarget;
@property (nonatomic, strong, readonly) id<Invoker> invoker;
@property (nonatomic, strong, readonly) NSMutableSet<NSString *> *selectors;

+ (instancetype)proxyWithTarget:(id)target invoker:(id<Invoker>)invoker;
- (void)registerSelector:(SEL)selector;

@end

@implementation AspectProxy

+ (instancetype)proxyWithTarget:(id)target invoker:(id<Invoker>)invoker {
    AspectProxy *proxy = [AspectProxy alloc];
    proxy->_proxyTarget = target;
    proxy->_invoker = invoker;
    proxy->_selectors = [NSMutableSet set];
    return proxy;
}

- (void)registerSelector:(SEL)selector {
    [self.selectors addObject:NSStringFromSelector(selector)];
}

- (BOOL)shouldInterceptSelector:(SEL)selector {
    return [self.selectors containsObject:NSStringFromSelector(selector)];
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)selector {
    return [self.proxyTarget methodSignatureForSelector:selector];
}

- (void)forwardInvocation:(NSInvocation *)invocation {
    SEL selector = invocation.selector;

    if ([self shouldInterceptSelector:selector]) {
        [self.invoker preInvoke:invocation];
        [invocation invokeWithTarget:self.proxyTarget];
        [self.invoker postInvoke:invocation];
        return;
    }

    [invocation invokeWithTarget:self.proxyTarget];
}

- (BOOL)respondsToSelector:(SEL)aSelector {
    return [self.proxyTarget respondsToSelector:aSelector];
}

@end
```

测试对象：

```objc
@interface Student : NSObject
- (void)study:(NSString *)subject andRead:(NSString *)book;
- (void)sleep;
@end

@implementation Student

- (void)study:(NSString *)subject andRead:(NSString *)book {
    NSLog(@"study %@, read %@", subject, book);
}

- (void)sleep {
    NSLog(@"sleep");
}

@end
```

使用方式：

```objc
Student *student = [Student new];
AuditingInvoker *invoker = [AuditingInvoker new];
AspectProxy *proxy = [AspectProxy proxyWithTarget:student invoker:invoker];

[proxy registerSelector:@selector(study:andRead:)];

[(id)proxy study:@"Runtime" andRead:@"objc4"];
[(id)proxy sleep];
```

输出逻辑可以理解成：

```text
before study:andRead:
study Runtime, read objc4
after study:andRead:
sleep
```

`study:andRead:` 被注册进 `selectors`，所以一次调用会触发 `pre -> target -> post` 三步；`sleep` 没注册，就只透传给目标对象。

这和 Method Swizzling 的差别很重要：

- Swizzling 是直接修改类的方法实现映射，影响这个类的所有对象。
- `NSProxy` 是给某个对象包一层代理，只影响经过这个 proxy 的调用。

所以如果你要做全局埋点、系统方法替换，Swizzling 更直接；如果你要做对象级别的代理增强、权限检查、日志审计，`NSProxy + forwardInvocation:` 更可控。

# 7. Runtime 应用边界

Runtime 能力强，但边界也要清楚。

**第一，签名必须一致。**

Swizzling 或动态添加方法时，IMP 的真实函数签名必须和 selector 对应的方法签名一致。签名错了，参数寄存器和返回值解释都会错。

**第二，不要依赖多个 swizzle 的顺序。**

多个库都替换同一个方法时，最终调用链取决于加载和交换顺序。这个顺序可以观察，但不应该成为业务前提。

**第三，少碰系统私有 API。**

Runtime 可以看见很多东西，但能看见不代表应该调用。私有 API 有审核风险，也有系统升级后的兼容风险。

**第四，优先保留原实现。**

无论是 `method_setImplementation` 还是 `class_replaceMethod`，如果后续还需要调用旧逻辑，应当提前保存原 IMP。

**第五，应用层不要把 Runtime 当作架构手段。**

Runtime 更适合框架层、基础设施层、调试工具、兼容补丁。业务代码如果大量依赖 Runtime 隐式改行为，长期维护成本会非常高。

# 小结

Part 4 真正想说明的是：Runtime 应用并不是独立于底层机制之外的一套“技巧”，它们都能落回前三篇讲过的结构和流程。

- 动态获取类信息：读类元数据。
- 动态添加方法：改运行期方法列表。
- Method Swizzling：改 `SEL -> IMP` 映射。
- 关联对象：在对象外部挂值。
- 消息转发：改变找不到方法之后的处理路径。
- NSProxy/AOP：把 `forwardInvocation:` 封装成对象级代理增强。

理解这些应用，关键不是背 API，而是知道每个 API 最终动的是哪一层结构、会影响哪一段消息发送路径。

# 参考方向

- objc4 源码：`objc-runtime-new.mm`、`objc-cache.mm`、`message.h`
- Apple Runtime Reference
- Effective Objective-C 2.0：消息发送、关联对象、Method Swizzling 相关章节
