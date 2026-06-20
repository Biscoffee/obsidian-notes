---
title: "【iOS】Runtime - Part 4 && Runtime 应用篇：动态能力如何落到业务代码"
published: 2026-06-16
description: "承接对象结构、消息发送与 Category 加载机制，梳理 Runtime 在业务代码中的典型应用：动态获取类信息、动态添加方法、Method Swizzling、KVO、关联对象、消息转发、自动归档与模型转换。"
tags: ["iOS", "Objective-C", "Runtime", "Method Swizzling", "objc4"]
category: "iOS"
series: "iOS Runtime 系列"
seriesSlug: "ios-runtime"
seriesOrder: 4
draft: true
---

# 前言

前三篇已经把 Runtime 的底层链路铺开了：

- Part 1：对象、类、元类、`isa`、`class_ro_t` / `class_rw_t`。
- Part 2：`objc_msgSend`、方法缓存、慢速查找、动态方法解析、消息转发。
- Part 3：Category 的编译产物、运行时加载、方法列表附加、覆盖假象和关联对象。

Part 4 不再继续只追源码细节，而是把这些机制拉回日常开发：**我们平时说的 Runtime 应用，到底是在使用哪几类动态能力？哪些用法是合理的，哪些用法容易把项目带进不可维护的状态？**

常见 Runtime 用法大致可以分成四类：

| 能力 | 典型 API | 背后的机制 |
| --- | --- | --- |
| 查看类结构 | `class_copyIvarList` / `class_copyPropertyList` / `class_copyMethodList` | 读取类元数据 |
| 动态加方法 | `class_addMethod` / `resolveInstanceMethod:` | 修改 `class_rw_t` / 方法解析 |
| 替换方法实现 | `method_exchangeImplementations` / `method_setImplementation` | 改变 `SEL -> IMP` 关系 |
| 外挂数据 | `objc_setAssociatedObject` / `objc_getAssociatedObject` | 对象外部关联表 |

如果把工程里的常见玩法继续拆开，会发现它们反复依赖的也就几点：

| 底层机制 | 典型应用 |
| --- | --- |
| 消息解析 / 消息转发 | 动态加方法、代理转发、组合对象能力透传、AOP |
| `SEL -> IMP` 映射修改 | Method Swizzling、埋点、异常保护 |
| `isa` 指向变化 | KVO / Isa Swizzling |
| 类元数据遍历 + 外部存储 | 关联对象、自动归档、字典模型互转 |

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

和方法相关的常用 API 还有一组：

```objc
SEL method_getName(Method m);
IMP method_getImplementation(Method m);
const char *method_getTypeEncoding(Method m);
unsigned int method_getNumberOfArguments(Method m);
char *method_copyReturnType(Method m);
char *method_copyArgumentType(Method m, unsigned int index);
void method_exchangeImplementations(Method m1, Method m2);
IMP method_setImplementation(Method m, IMP imp);
```

它们本质上都是在读写 `method_t` 表达的三件事：selector、类型编码、函数实现。动态加方法时最容易写错的是第三个参数和第四个参数：`IMP` 的真实 C 函数签名必须和 `types` 描述一致，否则消息发送时寄存器里的参数会被错误解释。

# 3. Method Swizzling

![image.png](https://cdn.jsdelivr.net/gh/Biscoffee/piccbes@master/img/20260620212834907.png)


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

常见落点有三类：

- **埋点统计**：例如统一 hook `viewDidAppear:` 做页面曝光。
- **AOP**：在原方法前后插入日志、鉴权、缓存刷新等横切逻辑。
- **异常保护**：例如数组越界、字典插入 nil 的兜底处理。

最后一类尤其要克制。系统类经常是 class cluster，表面上写的是 `NSArray` / `NSMutableArray`，真实运行类可能是 `__NSArrayI`、`__NSArrayM`、`__NSDictionaryI`、`__NSDictionaryM` 等。hook 错类，代码根本不生效；hook 太多私有真实类，又会增加系统版本升级风险。

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

这也是很多 Swizzling 模板选择 `+load` 而不是 `+initialize` 的原因：`+initialize` 是懒触发，某个类如果一直没收到消息，它就不会执行；而 Swizzling 改的是全局行为，通常希望类一加载就完成。

配套还有两个规矩：

- 交换动作放进 `dispatch_once`。Swizzling 执行两次就可能把实现换回去，尤其是父类、子类或多个 Category 同时操作同一个 selector 时，很容易出现“看似执行了，实际失效”的结果。
- `+load` 里不要主动调 `[super load]`。Runtime 会按加载流程调用各自的 `+load`，手动调 super 可能让父类交换逻辑重复执行。

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

还有一个不直观的坑：Swizzling 会影响方法内部看到的 `_cmd`。交换后，如果原实现里依赖 `_cmd` 判断当前 selector，它可能看到的是 swizzled selector，而不是原始 selector。更稳妥的做法是保存原 IMP，用函数指针显式调用旧实现：

```objc
static void (*original_setFrame)(id self, SEL _cmd, CGRect frame);

static void tw_setFrame(id self, SEL _cmd, CGRect frame) {
    NSLog(@"%@", NSStringFromCGRect(frame));
    original_setFrame(self, @selector(setFrame:), frame);
}
```

这种写法比在 swizzled 方法里再发一条 `[self tw_setFrame:]` 更啰嗦，但它把“调用哪个 IMP、传什么 `_cmd`”写得非常明确，适合基础库或 SDK 场景。

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

还有两种常见写法：

```objc
static char TWPageNameKey;
objc_setAssociatedObject(self, &TWPageNameKey, value, policy);

objc_setAssociatedObject(self, @selector(tw_pageName), value, policy);
```

`@selector(getter)` 这种写法也很常见，优点是不用额外定义 key，并且天然和属性 getter 对应。

内存策略可以按属性语义选：

| 属性语义 | 关联策略 |
| --- | --- |
| `assign` | `OBJC_ASSOCIATION_ASSIGN` |
| `strong, nonatomic` | `OBJC_ASSOCIATION_RETAIN_NONATOMIC` |
| `copy, nonatomic` | `OBJC_ASSOCIATION_COPY_NONATOMIC` |
| `strong, atomic` | `OBJC_ASSOCIATION_RETAIN` |
| `copy, atomic` | `OBJC_ASSOCIATION_COPY` |

注意：关联对象不是 ivar。它不会改变对象大小，也不会改变 ivar 偏移。它只是把值放进 Runtime 维护的外部表里。

这里有两个容易误用的点：

1. `OBJC_ASSOCIATION_ASSIGN` 不是 `weak`，更接近 `unsafe_unretained`。对象释放后，关联表里不会自动置 nil，再读就可能访问野指针。
2. `objc_removeAssociatedObjects(obj)` 会移除这个对象上的**全部**关联对象，不适合只删除自己那一个 key。删除单个关联值，应该对同一个 key 设置 `nil`。

从底层看，关联对象本质上是 Runtime 维护的外部映射。老资料常把它讲成两层哈希：先用对象地址找到这个对象的关联表，再用 key 找到具体值；新版实现细节有调整，但模型仍然可以这么理解：

```text
object pointer
  -> association map
      key -> { policy, value }
```

第一次给对象设置关联值时，Runtime 还会标记对象“有过关联对象”。这能和 Part 1 的 `isa_t.has_assoc` 对上：对象本身的大小没有变，但对象头里会留下一个快速判断标志，方便释放对象时决定是否需要清理外部关联表。

# 5. 消息转发应用

消息转发可以用于代理转发、容错、多播等场景。它承接 Part 2 的转发链。严格说，完整兜底管线前面还有一次**动态方法解析**：如果 `resolveInstanceMethod:` / `resolveClassMethod:` 成功补上方法，就不会进入转发；只有动态解析也放弃，才会继续走下面三步。

```text
objc_msgSend 查缓存/方法列表
  -> resolveInstanceMethod: / resolveClassMethod:
  -> forwardingTargetForSelector:
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

这里还有一个反面边界：`forwardingTargetForSelector:` 可以返回另一个对象，也可以返回 `nil` / `[super forwardingTargetForSelector:]` 让 Runtime 继续走完整转发，但**不要返回 `self`**。返回 `self` 等于把同一条消息重新发给自己，下一轮又进 `forwardingTargetForSelector:`，最终无限递归。

## 组合对象能力透传（老文所谓“模拟多继承”）

Objective-C 在语言层面只支持单继承。这里讨论的不是让一个类真的继承多个父类，而是借助**消息转发（Message Forwarding）​**做组合对象能力透传：当宿主对象自己没有某个方法实现时，把这条消息在运行时"透传"给它持有的其他对象去执行。把多个能力提供方组合在一起、再通过转发让宿主对象统一对外暴露它们的方法，对外看像一个聚合了多方能力的对象，实际执行者仍是各自的真实对象。

### forwardingTargetForSelector 做"快速转发"​

```objc
// Father 类有方法 work；Mother 类有方法 cook
@interface Son : Father
@property (nonatomic, strong) Mother *mother;
@end

@implementation Son
- (instancetype)init {
    if (self = [super init]) {
        _mother = [Mother new];
    }
    return self;
}

// 自己处理不了的消息，问问 mother 能不能处理
- (id)forwardingTargetForSelector:(SEL)aSelector {
    if ([_mother respondsToSelector:aSelector]) {
        return _mother;   // 把 receiver 替换成 mother
    }
    return nil;           // 返回 nil 则继续走完整转发
}
@end

// 使用
Son *son = [[Son alloc] init];
[son work];                              // 继承自 Father → I'm working.
[son performSelector:@selector(cook)];   // 透传给 Mother → I'm cooking.

```

从调用者视角看， `Son` 同时"拥有"了 `Father`（真继承）和 `Mother`（透传借来）的能力。它的局限也很明确：**只能把消息整体转给一个对象，无法对参数和返回值做任何加工，也无法同时分发给多个对象**。 这不是真正的多继承，只是消息被转手了。`NSObject` 的很多查询默认也不理解这种“能力透传”：

```objc
[son respondsToSelector:@selector(cook)] // 默认可能仍是 NO
[son isKindOfClass:[Mother class]]       // 仍是 NO
```

如果你希望这个透传行为更像“这个对象真的支持该能力”，就要同步覆写 `respondsToSelector:`、`methodSignatureForSelector:`、`conformsToProtocol:`，甚至类方法侧的 `instancesRespondToSelector:`。这也是为什么消息转发不应该被当作继承的常规替代品：它适合做组合和代理，不适合伪造类型体系。

### forwardInvocation 做"完整转发"​

```objc
@interface Son : Father
@property (nonatomic, strong) Mother *mother;
@property (nonatomic, strong) NSMutableArray<id> *forwardTargets;
@end

@implementation Son

- (instancetype)init {
    if (self = [super init]) {
        _mother = [Mother new];
        _forwardTargets = [NSMutableArray arrayWithObject:_mother];
    }
    return self;
}

// 第一步：给找不到的方法补一份方法签名。
// 返回 nil，Runtime 就会进入 doesNotRecognizeSelector: 崩溃。
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    NSMethodSignature *signature = [super methodSignatureForSelector:aSelector];
    if (signature) {
        return signature;
    }

    for (id target in self.forwardTargets) {
        signature = [target methodSignatureForSelector:aSelector];
        if (signature) {
            return signature;
        }
    }

    return nil;
}

// 第二步：Runtime 把原始调用包装成 NSInvocation 交给这里。
- (void)forwardInvocation:(NSInvocation *)invocation {
    SEL selector = invocation.selector;
    BOOL handled = NO;
    BOOL rewroteObjectArgument = NO;

    // 示例：读取并改写参数。注意 self 和 _cmd 占 index 0、1，
    // 业务参数从 index 2 开始。
    if (selector == @selector(cookWithFood:)) {
        __unsafe_unretained NSString *food = nil;
        [invocation getArgument:&food atIndex:2];

        if (food.length == 0) {
            NSString *defaultFood = [NSString stringWithFormat:@"%@", @"rice"];
            [invocation setArgument:&defaultFood atIndex:2];
            rewroteObjectArgument = YES;
        }
    }

    // NSInvocation 默认不持有参数。只要改写了对象参数，或者 invocation
    // 可能被缓存、延迟、跨线程执行，就应该让它持有当前参数。
    if (rewroteObjectArgument) {
        [invocation retainArguments];
    }

    for (id target in self.forwardTargets) {
        if (![target respondsToSelector:selector]) {
            continue;
        }

        [invocation invokeWithTarget:target];
        handled = YES;

        // 有返回值的方法只能选择一个目标的返回值；
        // void 方法才适合继续分发给多个对象。
        const char *returnType = invocation.methodSignature.methodReturnType;
        if (strcmp(returnType, @encode(void)) != 0) {
            if (returnType[0] == '@') {
                __unsafe_unretained id returnValue = nil;
                [invocation getReturnValue:&returnValue];
                NSLog(@"forward %@ return: %@",
                      NSStringFromSelector(selector),
                      returnValue);
            }
            return;
        }
    }

    if (handled) {
        return;
    }

    [super forwardInvocation:invocation];
}

@end
```

`forwardInvocation:` 的强大之处在于 `NSInvocation` 把"消息调用的全部细节"都封装好了，你可以在调用前后插入逻辑、改写参数、读取返回值。这也是 AOP（面向切面编程）和很多 Hook 框架的实现基础——业务方法被替换成 `_objc_msgForward`，最终都汇聚到 `forwardInvocation:` 里统一加工。

上面 `retainArguments` 是一个很重要的边界：`NSInvocation` 默认只保存参数字节，不 retain 对象参数，也不 copy C 字符串。这个 demo 是同步调用，即使不加也大概率能跑；但只要你改写的是动态生成的对象，或者把 invocation 存起来稍后执行，就可能留下悬垂指针。实际写框架时，只要有“改写对象参数 / 延迟调用 / 跨线程调用”的可能，就应该显式调用 `retainArguments`。

#### **​能力透传的边界：内省要诚实**

透传出来的能力是"借来"的，`NSObject` 的内省方法只认真正的继承体系，**不认转发链**。也就是说，即便 `son` 能执行 `cook`，`[son respondsToSelector:@selector(cook)]` 默认仍可能返回 `NO`。如果你的透传对象要对外表现成"能响应这些消息"，可以把转发算法补进部分内省方法里：

```objc
- (BOOL)respondsToSelector:(SEL)aSelector {
    if ([super respondsToSelector:aSelector]) {
        return YES;
    }

    for (id target in self.forwardTargets) {
        if ([target respondsToSelector:aSelector]) {
            return YES;
        }
    }

    return NO;
}

- (BOOL)conformsToProtocol:(Protocol *)aProtocol {
    if ([super conformsToProtocol:aProtocol]) {
        return YES;
    }

    for (id target in self.forwardTargets) {
        if ([target conformsToProtocol:aProtocol]) {
            return YES;
        }
    }

    return NO;
}

// 类方法只能回答静态能力。如果转发目标是每个实例动态配置的，
// 这里就不能完整表达，只能列出固定会透传的能力提供方。
+ (BOOL)instancesRespondToSelector:(SEL)aSelector {
    return [super instancesRespondToSelector:aSelector] ||
           [Mother instancesRespondToSelector:aSelector];
}
```

但这里要有“诚实度”边界：

| 查询方法 | 是否建议伪造 | 原因 |
| --- | --- | --- |
| `respondsToSelector:` | 可以谨慎伪造 | 它回答的是“能不能响应这条消息”，把转发目标算进去是自洽的。 |
| `conformsToProtocol:` | 可以谨慎伪造 | 只对运行时查询生效，适合表达“我可以透传这个协议能力”。 |
| `instancesRespondToSelector:` | 只适合静态透传 | 类方法不知道每个实例运行时持有哪些 target，只能回答固定能力。 |
| `isKindOfClass:` | 不建议伪造 | `son` 的 `isa` 仍然指向 `Son`，它真的不是 `Mother`。强行返回 YES 会误导依赖真实类型和内存布局的代码。 |

这类覆写只影响运行时内省，比如 `[obj respondsToSelector:]`。它不会改变编译期类型检查：形参写成 `id<SomeProtocol>`、变量静态类型、编译器警告，都不会因为你覆写了 `conformsToProtocol:` 而改变。

另外，伪造 `respondsToSelector:` 也不是零成本。一旦调用方相信它返回 YES 并真的发消息，这条消息在宿主类里仍然找不到实现，每次都要走转发路径。快速转发只是多一跳，完整转发则要创建 `NSInvocation`、匹配签名、包装参数，开销明显高于直接调用。高频热路径上，优先用显式组合、协议方法或快速转发，不要把所有能力都压到 `forwardInvocation:`。

所以，"组合对象能力透传"不是让 Objective-C 真的拥有多继承，而是用 Runtime 把组合对象的能力在消息层面暴露出去。




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

还有一种更“侵入”的 AOP 做法，是把目标 selector 的 IMP 替换成 `_objc_msgForward`，强行让这条消息进入完整转发，再在 `forwardInvocation:` 里拿到 `NSInvocation` 做 before / instead / after 逻辑。经典 Aspects 类库就是这个方向：注册阶段记录切面信息并替换 IMP，执行阶段借完整转发拿到参数、返回值和原始 IMP，再决定何时调用原实现。

这类方案能力更强，但风险也更高：它同时碰了 Swizzling、消息转发和 `NSInvocation`，对方法签名、返回值、结构体参数、多个 hook 的顺序都很敏感。理解它的底层路径即可，日常业务里优先选更显式的组合、代理或上面的 `NSProxy`。

这和 Method Swizzling 的差别很重要：

- Swizzling 是直接修改类的方法实现映射，影响这个类的所有对象。
- `NSProxy` 是给某个对象包一层代理，只影响经过这个 proxy 的调用。

所以如果你要做全局埋点、系统方法替换，Swizzling 更直接；如果你要做对象级别的代理增强、权限检查、日志审计，`NSProxy + forwardInvocation:` 更可控。

# 7. Isa Swizzling 与 KVO

Method Swizzling 改的是 `SEL -> IMP` 映射；Isa Swizzling 改的是对象的 `isa` 指向。最典型的系统级应用就是 KVO。

当你对一个对象添加观察者：

```objc
Student *stu = [Student new];
[stu addObserver:self
      forKeyPath:@"name"
         options:NSKeyValueObservingOptionNew
         context:nil];
```

系统不会直接修改 `Student` 这个类，而是动态创建一个中间子类，常见名字类似：

```text
NSKVONotifying_Student
```

然后把这个具体对象的 `isa` 从 `Student` 指向 `NSKVONotifying_Student`。所以 KVO 改变的是“这个对象运行时属于哪个类”，而不是“Student 类本身长什么样”。

这个中间子类通常会覆写几类方法：

- `setter`：在赋值前后插入 `willChangeValueForKey:` / `didChangeValueForKey:`，后者再触发观察回调。
- `class`：对外仍返回原类 `Student`，隐藏中间子类的存在。
- `dealloc`：做 KVO 相关清理。
- `_isKVOA`：私有标记，表明这是 KVO 生成的类。

这也解释了一个常见差异：

```objc
[stu class]              // Student
object_getClass(stu)     // NSKVONotifying_Student
```

`class` 是方法，可以被 KVO 子类覆写；`object_getClass` 是 Runtime API，会直接沿对象的 isa 取真实类。也正因为 isa 可能被系统换掉，工程里不要用“直接读 isa”判断类型关系，应该用 `class`、`isKindOfClass:`、协议或明确的业务字段。

KVO 这个例子很好地把 Part 1 和 Part 4 接起来：Part 1 讲 isa 是对象找到类的入口；这里看到的就是系统真的在运行时改了这个入口。

# 8. 基于遍历的自动归档与模型转换

`class_copyIvarList` / `class_copyPropertyList` 还有两个很常见的工程用途：自动归档和字典模型互转。它们的共同点是：先遍历类元数据，再通过 KVC、getter/setter 或 `objc_msgSend` 读写值。

## NSCoding 自动归档 / 解档

手写 `encodeWithCoder:` 和 `initWithCoder:` 时，属性一多就会出现大量重复代码：

```objc
- (void)encodeWithCoder:(NSCoder *)coder {
    [coder encodeObject:self.name forKey:@"name"];
    [coder encodeInteger:self.age forKey:@"age"];
}
```

可以用 ivar 遍历减少样板代码：

```objc
- (void)encodeWithCoder:(NSCoder *)coder {
    unsigned int count = 0;
    Ivar *ivars = class_copyIvarList([self class], &count);

    for (unsigned int i = 0; i < count; i++) {
        const char *name = ivar_getName(ivars[i]);
        NSString *key = [NSString stringWithUTF8String:name];
        id value = [self valueForKey:key];
        [coder encodeObject:value forKey:key];
    }

    free(ivars);
}

- (instancetype)initWithCoder:(NSCoder *)coder {
    self = [super init];
    if (!self) return nil;

    unsigned int count = 0;
    Ivar *ivars = class_copyIvarList([self class], &count);

    for (unsigned int i = 0; i < count; i++) {
        const char *name = ivar_getName(ivars[i]);
        NSString *key = [NSString stringWithUTF8String:name];
        id value = [coder decodeObjectForKey:key];
        [self setValue:value forKey:key];
    }

    free(ivars);
    return self;
}
```

实际项目里还要处理父类 ivar、基本类型、忽略字段、字段重命名和安全归档等问题；但核心思路就是“遍历元数据 + 按 key 读写”。

## 字典与模型互转

字典转模型也是同一套路：

1. 用 `class_copyPropertyList` 拿到属性列表。
2. 用 `property_getName` 得到属性名。
3. 从字典里取同名 value。
4. 通过 KVC 或 setter 赋值。
5. 如果属性本身还是模型，再递归转换。

模型转字典则反过来：遍历属性，生成 getter，取出值后写进字典。

MJExtension、JSONModel 这类库都建立在这条思路上，只是它们额外处理了容器泛型、字段映射、类型转换、嵌套模型、黑白名单等大量工程细节。

# 9. Runtime 应用边界

Runtime 能力强，但边界也要清楚。

**第一，签名必须一致。**

Swizzling 或动态添加方法时，IMP 的真实函数签名必须和 selector 对应的方法签名一致。签名错了，参数寄存器和返回值解释都会错。

**第二，不要依赖多个 swizzle 的顺序。**

多个库都替换同一个方法时，最终调用链取决于加载和交换顺序。这个顺序可以观察，但不应该成为业务前提。

**第三，Swizzling 改的是全局行为。**

你交换的不是某个对象的方法，而是类的方法映射。一个 Category 里的交换，可能影响整个 App 中所有这个类及其子类的实例。改动非自己拥有的类时，必须默认它会和系统、第三方库、未来版本发生交叉影响。

**第四，命名冲突要提前规避。**

Category 方法没有命名空间。如果两个库都加了 `xxx_viewDidAppear:`，后加载的一方可能覆盖前一方。基础库里更稳的做法，是使用足够明确的前缀，或者直接保存原 IMP，用 C 函数作为新实现，减少 selector 命名冲突。

**第五，`_cmd` 可能改变。**

这一点前面已经提过：交换后，方法体里看到的 `_cmd` 不一定还是原始 selector。如果原方法依赖 `_cmd`，用“互相调用 swizzled selector”的模板就可能埋坑；保存原 IMP 并显式传入原 selector 更可控。

**第六，少碰系统私有 API。**

Runtime 可以看见很多东西，但能看见不代表应该调用。私有 API 有审核风险，也有系统升级后的兼容风险。

**第七，优先保留原实现。**

无论是 `method_setImplementation` 还是 `class_replaceMethod`，如果后续还需要调用旧逻辑，应当提前保存原 IMP。

**第八，调试成本要算进去。**

Swizzling 和消息转发都会让堆栈不再直观：你看到的 selector、实际执行的 IMP、调用栈上的方法名可能不是同一件事。做 SDK 或团队基础设施时，凡是改过 Runtime 行为的地方，都应该写清楚注释、文档和开关。

**第九，应用层不要把 Runtime 当作架构手段。**

Runtime 更适合框架层、基础设施层、调试工具、兼容补丁。业务代码如果大量依赖 Runtime 隐式改行为，长期维护成本会非常高。

# 小结

Part 4 真正想说明的是：Runtime 应用并不是独立于底层机制之外的一套“技巧”，它们都能落回前三篇讲过的结构和流程。

- 动态获取类信息：读类元数据。
- 动态添加方法：改运行期方法列表。
- Method Swizzling：改 `SEL -> IMP` 映射。
- 关联对象：在对象外部挂值。
- 消息转发：改变找不到方法之后的处理路径。
- NSProxy/AOP：把 `forwardInvocation:` 封装成对象级代理增强。
- KVO / Isa Swizzling：改变单个对象的 `isa` 指向。
- 自动归档 / 模型转换：遍历 ivar / property 元数据并读写值。

理解这些应用，关键不是背 API，而是知道每个 API 最终动的是哪一层结构、会影响哪一段消息发送路径。

# 参考方向

- objc4 源码：`objc-runtime-new.mm`、`objc-cache.mm`、`message.h`
- Apple Runtime Reference
- Effective Objective-C 2.0：消息发送、关联对象、Method Swizzling 相关章节
- sunnyxx / halfrost Runtime 实战类文章：组合转发、KVO、Swizzling 风险点
