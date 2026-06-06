# 【iOS】Runtime - Part 2 && 消息发送：缓存、查找与转发

> 基于 objc4-951.1。延续 Part 1（对象/类/元类/isa 走位）。本篇打通一条完整消息链：快速路径（cache 命中）→ 慢速查找（`lookUpImpOrForward`）→ 动态解析 → 消息转发三部曲 → 崩溃。

## 目录

### 第一部分 · 快速路径（缓存命中）
- **1. 方法调用的本质**：`[obj foo]` → `objc_msgSend(obj, sel)`
- **2. 为什么 objc_msgSend 用汇编写**：性能 / 未知参数 / 尾调用
- **3. 快速路径汇编逐段**（`objc-msg-arm64.s`）
  - 3.1 取 isa → class（`GetClassFromIsa`，呼应 ISA_MASK）
  - 3.2 CacheLookup：按 SEL 哈希定位 bucket
  - 3.3 命中 `br IMP` / 未命中转入慢速查找
- **4. cache_t 数据结构**（`objc-runtime-new.h:337`）
  - 4.1 `_bucketsAndMaybeMask`：指针与 mask 的融合
  - 4.2 `bucket_t`：`{SEL, IMP}` 与 ptrauth
  - 4.3 哈希与开放寻址探测（`cache_hash` / `cache_next`）
- **5. 缓存的写入与扩容**（`objc-cache.mm`）
  - 5.1 `insert`：哈希落位（:873）
  - 5.2 `reallocate`：翻倍扩容、丢弃旧表不迁移（:809）
  - 5.3 装填因子与留空策略

### 第二部分 · 慢速查找（lookUpImpOrForward）
- **6. 入口**：加锁 + realize / +initialize（:7624）
- **7. 在当前类找方法**
  - 7.1 `getMethodNoSuper_nolock` 遍历方法列表（:7290）
  - 7.2 已排序表二分查找 `findMethodInSortedMethodList`（:7062）
- **8. 沿 superclass 链逐层上溯**（每层先查父类 cache）
- **9. 找到了**：`log_and_fill_cache` 回填缓存，闭环回快速路径（:7513）
- **10. 没找到**：动态方法解析（`resolveMethod_locked` :7480 → `resolveInstanceMethod` :7435）

### 第三部分 · 消息转发
- **11. 转发入口**：`_objc_msgForward_impcache`（:7626 / 汇编 :788）
- **12. 转发三部曲**
  - 12.1 快速转发 `forwardingTargetForSelector:`（换 receiver 重发）
  - 12.2 完整转发 `methodSignatureForSelector:` + `forwardInvocation:`
  - 12.3 兜底崩溃 `doesNotRecognizeSelector:`（`NSObject.mm:2578`）
- **13. 边界澄清**：objc4 只到 `_objc_msgForward`，三部曲调度在 CoreFoundation `___forwarding___`（闭源）

### 第四部分 · 实战与收尾
- **14. LLDB 实战**：cache 从空到有 / 命中对比 / 慢速断点 / 转发链逐站拦截
- **15. 收尾**：一条消息的命运 + 与 Part 1 的呼应 + 承接下一篇


![[objc_msgSend_full_pipeline.html]]

> 源码定位说明：`objc_msgSend` 汇编在 951.1 里已移到 `runtime/Messengers.subproj/objc-msg-arm64.s`（不在 `runtime/` 根下）。下文源码块均为 objc4-951.1 **逐字全文**（保留所有架构 `#if` 分支），中文注释为本文新增、英文原注释保留。LLDB 在 **系统 libobjc**（`/usr/lib/libobjc.A.dylib`）上下符号断点，`settings set target.disable-aslr true` 关 ASLR，环境 macOS 26 / arm64。
>
> 新旧对照说明：文中带 🔄 的「旧→新对照」框，旧版取自本地 **objc4-818.2**（可调试版）真实源码、新版为 **951.1**，均标注精确 `file:line`。注意：缓存早期那次著名的大改造（`_buckets`/`_mask`/`_occupied` 三个分离字段 → 融合成 `_bucketsAndMaybeMask`）发生在 **781 之前**，本地无该版本源码，故未做该处对照、也不凭记忆杜撰；818.2 与 951.1 同属「融合字段后」时代，下列对照都是这之后的增量演进。

---
# objc_msgSend简介

`objc_msgSend` 是 OC **所有方法调用的统一入口**——编译器把每一句 `[obj msg]` 都翻译成对它的调用，再由它在运行期找到方法实现（IMP）并跳过去。在正式拆「快速路径 → 慢速查找 → 转发」这三条路之前，这一节先把两件「会影响后文怎么读」的事讲清楚：一是它的**声明**为什么长得那么怪（`void objc_msgSend(void)`）、为什么调用前必须 cast；二是它其实**不是一个函数，而是一整个家族**。

## 0.1 声明：`void objc_msgSend(void)

`objc_msgSend` 在公开头文件里有**两种声明**，由 `OBJC_OLD_DISPATCH_PROTOTYPES` 切换：

```objc
// message.h:46  —— 头部说明
/* Basic Messaging Primitives
 *
 * On some architectures, use objc_msgSend_stret for some struct return types.
 * On some architectures, use objc_msgSend_fpret for some float return types.
 * On some architectures, use objc_msgSend_fp2ret for some float return types.
 *
 * These functions must be cast to an appropriate function pointer type
 * before being called.
 */
#if !OBJC_OLD_DISPATCH_PROTOTYPES
// 新（默认）：无原型，参数注释化 —— 逼你按真实方法签名 cast 后再调
OBJC_EXPORT void
objc_msgSend(void /* id self, SEL op, ... */ ) // 虽然表面时void，但是实际形式是（id self, SEL op, ...)，其中self是消息接收者，SEL op是方法编号，后面...是可变参数
    OBJC_AVAILABLE(10.0, 2.0, 9.0, 1.0, 2.0);
#else
// 旧：带具体原型 id(id, SEL, ...)
OBJC_EXPORT id _Nullable
objc_msgSend(id _Nullable self, SEL _Nonnull op, ...)
    OBJC_AVAILABLE(10.0, 2.0, 9.0, 1.0, 2.0);
#endif
```

默认（新）声明成 `void objc_msgSend(void)`，是苹果刻意为之——它要透传任意 OC 方法的参数，硬给一个固定原型反而会让编译器按错误的调用约定生成代码。理解这件事的钥匙，是先搞清楚 cast 是什么。

**Cast 是什么**
Cast 写法是在值前加括号里的目标类型：

```c
double x = 3.14;
int y = (int)x;   // 把 double 转成 int，y = 3
```

这种「值 cast」会真正转换数值。但**函数指针的 cast 不一样**——它不改变任何值，只改变编译器对这个指针的理解方式：

```c
void foo(void);   // 声明：无参数，无返回值
// foo 本质上就是一个内存地址，比如 0x1000
int (*fp)(int, int) = (int (*)(int, int))foo;
// fp 里存的还是 0x1000，一个字节都没变
// 变的是：编译器现在用 fp 的类型来决定怎么生成调用指令
```

**为什么「类型」那么重要**
arm64 调用约定规定参数按类型走不同的寄存器：
```
第1个整型参数 → x0
第2个整型参数 → x1
第1个浮点参数 → d0
第2个浮点参数 → d1
```
编译器生成调用代码时，**完全按声明的参数类型**决定往哪个寄存器放东西，跟函数体在哪里、实际长什么样无关。所以如果声明写错了：
```c
// 真实实现接收两个 double（浮点走 d0/d1）
double add(double a, double b) { return a + b; }

// 但声明说接收两个 int（整型走 x0/x1）
double add(int a, int b);

// 编译器按声明生成调用代码，把参数放进 x0/x1
// 函数体去 d0/d1 取浮点参数，取到的是空的或随机值 → 计算出错
```

这就是为什么 `objc_msgSend` 必须用 cast 调用——它声明成 `void(void)`，如果不 cast，编译器就不知道参数类型，寄存器就会放错，和上面那个 `add` 出错的原因一模一样。

**回到 objc_msgSend**
`objc_msgSend` 要能处理世界上所有 OC 方法，每个方法参数类型都不同，根本没法给一个固定原型。苹果把它声明成 `void objc_msgSend(void)`——逼迫每个调用点都显式 cast，从根源上防止「按错误签名生成调用代码」。

不cast：

```c
// 编译器看到「没有参数」的声明，不往任何寄存器放东西，直接跳过去
// objc_msgSend 去 x0 找 receiver，取到的是空值或残留
objc_msgSend(obj, sel, 1.0, 2.0);   //「错误」
```

cast 之后，该这么写：
```objc
// 假设 obj 有方法： - (double)addA:(CGFloat)a b:(CGFloat)b;
// CGFloat 在 64 位即 double，和上面一样走浮点寄存器（d0/d1）
double result =
    ((double (*)(id, SEL, CGFloat, CGFloat))objc_msgSend)(obj, sel, 1.0, 2.0);
```

以 `objc_msgSend)` 为界拆成两部分：

- **前半**：`(double (*)(id, SEL, CGFloat, CGFloat))objc_msgSend`——把 `objc_msgSend` 临时标记成「接收4个参数、返回 double」的函数指针。不产生任何指令，只改变「编译器这一次怎么看待它」；
- **后半**：`(obj, sel, 1.0, 2.0)` 才是真正调用。编译器按上面那张类型表生成汇编：

```asm
;; cast 后，编译器生成正确的参数赋值指令
mov  x0, obj      ;「中文」receiver → x0
mov  x1, sel      ;「中文」selector → x1
fmov d0, 1.0      ;「中文」第一个浮点参数 → d0
fmov d1, 2.0      ;「中文」第二个浮点参数 → d1
bl   objc_msgSend ;「中文」跳过去（地址没变，仍是同一个函数）
```

cast 改的是「跳进去之前」那几条 mov 怎么生成，不改 `objc_msgSend` 本身一个字节。

## 0.2 它不是一个函数，而是家族

上面一直盯着 `objc_msgSend` 一个讲，其实它只是这族 dispatch 函数里最常用的那个。Apple 头文件里有一段**较早、较通用**的描述（`message.h:77`）——读的时候注意：它把家族说成 `objc_msgSend / _stret / Super / Super_stret` 四个，但带 `_stret` 的那两个是 **x86 时代的产物，arm64 上已经没有**（本节末尾会用源码证明）。所以这段只当「历史背景 + 家族概念」看，**真正以 arm64 当下入口表为准**：

```objc
// message.h:77  —— 编译器如何选 messenger
// @note When it encounters a method call, the compiler generates a call to one of the
//  functions objc_msgSend, objc_msgSend_stret, objc_msgSendSuper, or objc_msgSendSuper_stret.
//  Messages sent to an object's superclass (using the super keyword) are sent using
//  objc_msgSendSuper; other messages are sent using objc_msgSend. Methods that have data
//  structures as return values are sent using objc_msgSendSuper_stret and objc_msgSend_stret.


// 这段意思就是说，编译器看到 Objective-C 方法调用时，不是直接调用方法实现，而是先选择一个“消息发送函数 messenger”。

/* messenger指的就是Runtime提供的消息发送函数，例如：
objc_msgSend
objc_msgSendSuper
objc_msgSend_stret
objc_msgSendSuper_stret */
```

`super` 调用走 `objc_msgSendSuper`，它吃的不是裸 receiver，而是一个 `objc_super`（`message.h:34`）——「从哪个类开始查」由 `super_class` 指定：

```objc
// message.h:34
struct objc_super {
    __unsafe_unretained _Nonnull id receiver;       // 真正的接收者（仍是 self）
    __unsafe_unretained _Nonnull Class super_class;  // 查找起点：从这个类开始（注释：first class to search）
};
```

arm64 汇编里这一整个家族的真实入口（`objc-msg-arm64.s`）：

```asm
;; objc-msg-arm64.s —— messenger 家族入口一览
MSG_ENTRY    _objc_msgSend              // :587  普通消息（本篇主线）
ENTRY        _objc_msgLookup            // :622  只查 IMP 不调用（返回 IMP）
ENTRY        _objc_msgSendSuper         // :673  super 调用（旧）
ENTRY        _objc_msgSendSuper2        // :683  super 调用（实际在用，查找起点=super）
ENTRY        _objc_msgLookupSuper2      // :702  super 版的只查不调
STATIC_ENTRY __objc_msgSend_uncached    // :737  缓存未命中 → 进慢速查找（第二部分）
STATIC_ENTRY __objc_msgForward_impcache // :788  转发占位 IMP（第三部分）
ENTRY        __objc_msgForward          // :796  转发入口
ENTRY        _objc_msgSend_noarg        // :806  无参快路径
```

回头印证开头那句「`_stret` 是历史包袱」——不用我空口断言，**头文件自己就把这些版本在 arm64 标成不可用**：

```objc
// message.h:118 —— stret 版（结构体返回）
OBJC_EXPORT void
objc_msgSend_stret(void /* id self, SEL op, ... */ )
    OBJC_AVAILABLE(10.0, 2.0, 9.0, 1.0, 2.0)
    OBJC_ARM64_UNAVAILABLE;        // ←「中文」arm64 上根本没有这个符号

// message.h:158 —— fpret/fp2ret（浮点返回）的注释
// arm:    objc_msgSend_fpret not used
// arm:    objc_msgSend_fp2ret not used
```

所以 `objc_msgSend_stret`/`_fpret`/`fp2ret` 是 **x86/旧架构**专用：arm64 改用 `x8` 间接返回结构体、浮点直接走 `d0`，全部并回 `objc_msgSend`——这正是上面 arm64 入口清单里压根看不到 `_stret`/`_fpret` 的原因。

### 为什么这样设计？

普通返回值的返回位置由 ABI 决定：整型、指针通常放在 x0，浮点通常放在 d0；较大的结构体不会直接塞进普通返回寄存器，而是通过调用约定使用一块由调用方准备的返回内存。

这些差异不是 Runtime 在 objc_msgSend 内部临时判断出来的，而是编译器根据方法签名和目标架构的 ABI，提前生成正确的调用形式。历史上在一些架构中，结构体返回和浮点返回需要走专门入口，所以出现了 objc_msgSend_stret、objc_msgSend_fpret 这类变体。

在 arm64 上，结构体返回已经按 AArch64 ABI 统一到普通调用约定中，因此没有 objc_msgSend_stret 这个专门入口。普通对象消息一般走 objc_msgSend。

super 调用仍然需要单独入口，例如 objc_msgSendSuper2。原因不是接收者变成了父类对象，而是方法查找起点变了：[super foo] 的 receiver 仍然是 self，只是 Runtime 要从父类开始查找 IMP。因此传入的第一个参数不是普通 id，而是保存 receiver 和查找起点信息的 objc_super 结构体指针。

### 一个问题

输出什么？

```objc
@implementation Son : Father
- (id)init
{
    self = [super init];
    if (self)
    {
        NSLog(@"%@", NSStringFromClass([self class]));   // 输出什么？
        NSLog(@"%@", NSStringFromClass([super class])); // 输出什么？
    }
    return self;
}
@end
```

两行都输出 `Son`。

很多人第一反应是 `[super class]` 应该打印 `Father`——错误的根源就是把「查找起点」和「receiver」混为一谈。

拆开来看：

**`[self class]`** 没有悬念：receiver 是 Son 实例，查 `class` 方法，找到 `NSObject -class`，返回 `object_getClass(self)` = `Son`。

**`[super class]`** 编译器把它翻译成：

```objc
// 编译器生成
struct objc_super superInfo = {
    .receiver   = self,          // receiver 仍然是 Son 实例，没有变
    .super_class = Father.class, // 只是告诉 Runtime：查找从 Father 开始
};
objc_msgSendSuper2(&superInfo, @selector(class));
```

`objc_msgSendSuper2` 拿到这个结构体，从 `Father` 开始往上查 `class` 方法——`Father` 没实现，`NSObject` 有。进入 `NSObject -class`：

```objc
// NSObject -class 的实现本质上是：
- (Class)class {
    return object_getClass(self); // self 是谁？
}
```

这里的 `self` 是 `objc_super.receiver`——始终是那个 Son 实例。所以 `object_getClass(self)` 返回 `Son`。

结论：**`[super foo]` 改变的只是方法查找的起点，receiver 从始至终都是 `self`**。`[super class]` 和 `[self class]` 最终执行的是同一份 `NSObject -class` 实现，拿到的是同一个 receiver，自然输出同一个结果。


# 第一部分 · 快速路径（缓存命中）

## 1. 方法调用的本质：`[obj foo]` → `objc_msgSend(obj, sel)`

这和简介 0.1 讲的是同一套东西：`id receiver` 是指针（整型），按 arm64 调用约定放 `x0`；`SEL` 也是指针，放 `x1`；后续参数按类型依次填入 `x2…` 或 `d0…`。简介里说「声明写错了编译器就往错误的寄存器放参数」，这里就是那套规则的直接体现——`objc_msgSend` 之所以能从 `x0` 拿到 receiver、从 `x1` 拿到 selector，正是因为编译器在调用点按这套约定摆好了寄存器。下面这条 LLDB 实测的调用栈，直接显示了 `main` 里的一句 OC 调用最终是怎么落到 `objc_msgSend` → uncached → `lookUpImpOrForward` 的：

```text
(lldb) thread backtrace
* frame #0: libobjc.A.dylib`lookUpImpOrForward
  frame #1: libobjc.A.dylib`_objc_msgSend_uncached + 68
  frame #2: msgsend-demo`main at main.m:30:9     // [d bark]
  frame #3: dyld`start + 6992
(lldb) p (char *)sel_getName($x1)
(char *) "bark"
(lldb) po (Class)$x2
Dog
```

补充一个**编译器侧**的事实（objc4 源码里没有，靠 demo 反汇编看）：现代 clang（arm64）在调用点其实**不直接** `bl objc_msgSend`，而是 `bl objc_msgSend$bark`——一个**选择子 stub（selector stub）**，把「装 selector 到 x1」延后进 stub：

```text
;; demo `main` 调用点（lldb: disassemble --name main）
<+72>: bl  0x…a20    ; objc_msgSend$bark   ← [d bark] 第一次
<+84>: bl  0x…a20    ; objc_msgSend$bark   ← [d bark] 第二次

;; objc_msgSend$bark 这个 stub 内部（disassemble --start-address 0x…a20）
objc_msgSend$bark:
    adrp   x1, …            ; ldr x1, [x1, #0x120]   ← 把 selector "bark" 装进 x1
    adrp   x16, …           ; ldr x16, [x16]         ← 取 objc_msgSend 地址
    br     x16                                       ← 尾跳进 objc_msgSend
```

所以「receiver 在 x0、selector 在 x1」最终成立，只是 selector 的装载被挪进 stub；调用点本身只剩 `x0` + 一条 `bl …$bark`，更省指令、利于去重。下文回到 objc4 本体，从 `objc_msgSend` 入口讲起。

## 2. 为什么 objc_msgSend 用汇编写

三个原因：调用频率极高，手写汇编榨干每一周期；它要在**不知道目标方法参数个数/类型**的情况下原样透传所有寄存器；命中后用**尾调用**（`br`）直接跳进 IMP，自己不留栈帧。下面第 3 节的命中分支 `br x17`（实测解析成带 PAC 的 `brab`）正是「不返回、直接跳走」。

源码佐证「透传所有参数寄存器」：快速路径命中时直接尾跳、不存任何东西；而一旦 miss 要调 C 函数做慢速查找，`SAVE_REGS` 会先把**全部参数寄存器**存下来，查完恢复再尾跳 IMP——保证 IMP 拿到的参数和最初一模一样：

```asm
;; objc-msg-arm64.s:206 —— SAVE_REGS（关键部分）
.macro SAVE_REGS kind
    SignLR
    stp    fp, lr, [sp, #-16]!
    mov    fp, sp
    // save parameter registers: x0..x8, q0..q7
    sub    sp, sp,  #(10*8 + 8*16)
    stp    q0, q1,  [sp, #(0*16)]        //「中文」q0..q7：浮点/向量参数全保存
    stp    q2, q3,  [sp, #(2*16)]
    stp    q4, q5,  [sp, #(4*16)]
    stp    q6, q7,  [sp, #(6*16)]
    stp    x0, x1,  [sp, #(8*16+0*8)]    //「中文」x0..x7：整型参数（含 self/_cmd）
    stp    x2, x3,  [sp, #(8*16+2*8)]
    stp    x4, x5,  [sp, #(8*16+4*8)]
    stp    x6, x7,  [sp, #(8*16+6*8)]
.if \kind == MSGSEND
    stp    x8, x15, [sp, #(8*16+8*8)]    //「中文」x8：结构体返回的间接结果地址；x15：CacheLookup 暂存的 isa
    mov    x16, x15 // stashed by CacheLookup, restore to x16
.elseif \kind == METHOD_INVOKE
    str    x8,      [sp, #(8*16+8*8)]
.endif
.endmacro
```

`x0..x8 + q0..q7` 正是 arm64 调用约定里**全部整型/浮点参数 + 结构体返回辅助寄存器**——objc_msgSend 不知道目标方法签名，索性整体保下来透传，这就是「未知参数也能转发」的底气。

## 3. 快速路径汇编逐段（`objc-msg-arm64.s`）

入口 `_objc_msgSend`（:587），逐字全文：

```asm
;; objc-msg-arm64.s:587  —— _objc_msgSend 入口（全文）
	MSG_ENTRY _objc_msgSend
	UNWIND _objc_msgSend, NoFrame

	cmp	p0, #0			// nil check and tagged pointer check
					//「中文」receiver 是否为 0 / 是否 tagged pointer
#if SUPPORT_TAGGED_POINTERS
	b.le	LNilOrTagged		//  (MSB tagged pointer looks negative)
					//「中文」tagged 指针最高位为 1，看起来像负数 → 走特殊分支
#else
	b.eq	LReturnZero
#endif
	ldr	p14, [x0]		// p14 = raw isa
					//「中文」取对象首 8 字节 = 原始 isa（含标志位）
	GetClassFromIsa_p16 p14, 1, x0	// p16 = class
					//「中文」抹标志位 + ptrauth 解签 → 真正的 Class（见 3.1）
LGetIsaDone:
	// calls imp or objc_msgSend_uncached
	CacheLookup NORMAL, _objc_msgSend, __objc_msgSend_uncached
					//「中文」查缓存：命中尾跳 IMP，未命中跳 uncached（见 3.2/3.3）

#if SUPPORT_TAGGED_POINTERS
LNilOrTagged:
	b.eq	LReturnZero		// nil check
					//「中文」上面 cmp 若 ==0 即 nil，返回零值
	GetTaggedClass
	b	LGetIsaDone		//「中文」tagged pointer 从专表取 class，再回主流程
// SUPPORT_TAGGED_POINTERS
#endif

LReturnZero:
	// x0 is already zero
	mov	x1, #0			//「中文」给 nil 发消息：返回值寄存器全部清零
	movi	d0, #0
	movi	d1, #0
	movi	d2, #0
	movi	d3, #0
	ret

	END_ENTRY _objc_msgSend
```

### 3.0 先看类对象布局：`#CACHE` / `#SUPERCLASS` 偏移哪来的

上面入口里 `[x0]` 取 isa、后面 3.2 的 `[x16, #CACHE]`、第 8 节的 `[x16, #SUPERCLASS]`，这些偏移都来自 `objc_class` 的内存布局：

```objc
// objc-runtime-new.h:2635 —— 类对象本身就是一个 objc_class
struct objc_class : objc_object {
    // Class ISA;          //「中文」偏移 0：继承自 objc_object（3.1 节 ldr [x0] 取的就是它）
    Class superclass;      //「中文」偏移 8
    cache_t cache;         //「中文」偏移 16
    class_data_bits_t bits;//「中文」偏移 16+sizeof(cache_t)：class_rw_t* + rr/alloc 标志
};
```

汇编侧把这些字段偏移写死成宏（指针 8 字节）：

```asm
;; objc-msg-arm64.s:79 —— 类结构里被选用的字段偏移
#define SUPERCLASS       __SIZEOF_POINTER__        // = 8   → [x16, #SUPERCLASS]
#define CACHE            (2 * __SIZEOF_POINTER__)  // = 16  → [x16, #CACHE]
#define BUCKET_SIZE      (2 * __SIZEOF_POINTER__)  // = 16  → 3.2 的 ldp …, #-BUCKET_SIZE
```

对照 3.2 的 LLDB 反汇编 `ldr x10, [x16, #0x10]`：`0x10` = 16 = `#CACHE`，取的正是 `cache` 字段（即融合字 `_bucketsAndMaybeMask`）；`ldp …, #-0x10` 里的 `0x10` 就是 `BUCKET_SIZE`（每个 bucket 16 字节）。

### 3.1 取 isa → class（`GetClassFromIsa_p16`，:115，呼应 Part 1 的 ISA_MASK）

逐字全文（含全部架构分支）：

```asm
;; objc-msg-arm64.s:115  —— GetClassFromIsa_p16 宏（全文）
.macro GetClassFromIsa_p16 src, needs_auth, auth_address /* note: auth_address is not required if !needs_auth */

#if SUPPORT_INDEXED_ISA
	// Indexed isa
	//「中文」watchOS 等：isa 存的是类表下标，不是指针
	mov	p16, \src			// optimistically set dst = src
	tbz	p16, #ISA_INDEX_IS_NPI_BIT, 1f	// done if not non-pointer isa
	// isa in p16 is indexed
	adrp	x10, _objc_indexed_classes@PAGE
	add	x10, x10, _objc_indexed_classes@PAGEOFF
	ubfx	p16, p16, #ISA_INDEX_SHIFT, #ISA_INDEX_BITS  // extract index
	ldr	p16, [x10, p16, UXTP #PTRSHIFT]	// load class from array
1:

#elif __LP64__
	//「中文」arm64 真机走这里
.if \needs_auth == 0 // _cache_getImp takes an authed class already
	mov	p16, \src
.else
	// 64-bit packed isa
	ExtractISA p16, \src, \auth_address
	//「中文」ExtractISA = 用 ISA_MASK(0x7ffffffffffff8) 抹低位 + autda 解 ptrauth 签名
.endif
#else
	// 32-bit raw isa
	mov	p16, \src			//「中文」32 位：isa 就是裸指针，直接用

#endif

.endmacro
```

LLDB 实测：`__LP64__` 分支里 `ExtractISA` 落地为 `and x16, x14, #0x7ffffffffffff8`（**ISA_MASK**，与 Part 1 一致）+ `autda`：

```text
(lldb) disassemble --name objc_msgSend
objc_msgSend:
  <+0>:  cmp    x0, #0x0
  <+4>:  b.le   <+128>                       ; nil/tagged
  <+8>:  ldr    x14, [x0]                     ; raw isa
  <+12>: and    x16, x14, #0x7ffffffffffff8   ; ISA_MASK → class
  <+16>: mov    x10, x0
  <+20>: movk   x10, #0x6ae1, lsl #48
  <+24>: autda  x16, x10                      ; ptrauth 解签
```

### 3.2 CacheLookup：按 SEL 哈希定位 bucket（:336）

整段宏逐字全文（保留 `HIGH_16 / HIGH_16_BIG_ADDRS / LOW_4` 全部分支与 preopt 路径）：

```asm
;; objc-msg-arm64.s:336  —— CacheLookup 宏（全文）
.macro CacheLookup Mode, Function, MissLabelDynamic, MissLabelConstant
	//
	// Restart protocol:
	//
	//   As soon as we're past the LLookupStart\Function label we may have
	//   loaded an invalid cache pointer or mask.
	//
	//   When task_restartable_ranges_synchronize() is called,
	//   (or when a signal hits us) before we're past LLookupEnd\Function,
	//   then our PC will be reset to LLookupRecover\Function which forcefully
	//   jumps to the cache-miss codepath which have the following
	//   requirements:
	//
	//   GETIMP:
	//     The cache-miss is just returning NULL (setting x0 to 0)
	//
	//   NORMAL and LOOKUP:
	//   - x0 contains the receiver
	//   - x1 contains the selector
	//   - x16 contains the isa
	//   - other registers are set as per calling conventions
	//

	mov	x15, x16			// stash the original isa
					//「中文」备份 isa，转发指令里要用它判断是否回退到父类
LLookupStart\Function:
	// p1 = SEL, p16 = isa
#if CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16_BIG_ADDRS
	//「中文」macOS(M系列)/模拟器走这里（地址空间大，objc-config.h:218）：分两步取出 mask(高16)、buckets(低48)。本文 LLDB 实测就是这条
	ldr	p10, [x16, #CACHE]				// p10 = mask|buckets
	lsr	p11, p10, #48			// p11 = mask
	and	p10, p10, #0xffffffffffff	// p10 = buckets
#  if SEL_HASH_SHIFT_XOR
	eor	p12, p1, p1, LSR #7
	and	w12, w12, w11			// x12 = (_cmd ^ (_cmd >> 7)) & mask
#  else
	and	w12, w1, w11			// x12 = _cmd & mask
#  endif
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16
	//「中文」iOS 真机走这里（objc-config.h:218）：一条 ldr 取出 mask|buckets，后面边算哈希边移位，比 BIG_ADDRS 省一步
	ldr	p11, [x16, #CACHE]			// p11 = mask|buckets
#  if CONFIG_USE_PREOPT_CACHES
#    if __has_feature(ptrauth_calls)
	tbnz	p11, #0, LLookupPreopt\Function	//「中文」最低位=1 → 这是共享缓存预优化表，跳专门路径
	and	p10, p11, #0x0000ffffffffffff	// p10 = buckets
#    else
	and	p10, p11, #0x0000fffffffffffe	// p10 = buckets
	tbnz	p11, #0, LLookupPreopt\Function
#    endif
#  endif

#  if SEL_HASH_SHIFT_XOR
	eor	p12, p1, p1, LSR #7		//「中文」sel ^ (sel>>7)，对应 cache_hash
	and	p12, p12, p11, LSR #48		// x12 = (_cmd ^ (_cmd >> 7)) & mask
#  else
	and	p10, p11, #0x0000ffffffffffff	// p10 = buckets
	and	p12, p1, p11, LSR #48		// x12 = _cmd & mask
#  endif // CONFIG_USE_PREOPT_CACHES
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_LOW_4
#  if SEL_HASH_SHIFT_XOR
#    error SEL_HASH_SHIFT_XOR not supported for LOW_4
#  endif
	ldr	p11, [x16, #CACHE]				// p11 = mask|buckets
	and	p10, p11, #~0xf			// p10 = buckets
	and	p11, p11, #0xf			// p11 = maskShift
	mov	p12, #0xffff
	lsr	p11, p12, p11			// p11 = mask = 0xffff >> p11
	and	p12, p1, p11			// x12 = _cmd & mask
#else
#error Unsupported cache mask storage for ARM64.
#endif

	add	p13, p10, p12, LSL #(1+PTRSHIFT)
						// p13 = buckets + ((_cmd & mask) << (1+PTRSHIFT))
					//「中文」p13 指向哈希命中的那个 bucket（每个 bucket 16 字节）

						// do {
1:	ldp	p17, p9, [x13], #-BUCKET_SIZE	//     {imp, sel} = *bucket--
					//「中文」取出 {imp,sel}，并让指针向低地址移一格（探测方向）
	cmp	p9, p1				//     if (sel != _cmd) {
	b.ne	3f				//         scan more
						//     } else {
2:	CacheHit \Mode				// hit:    call or return imp
						//     }
3:	cbz	p9, \MissLabelDynamic		//     if (sel == 0) goto Miss;
					//「中文」撞到空槽(sel==0) → 缓存未命中，跳慢速查找
	cmp	p13, p10			// } while (bucket >= buckets)
	b.hs	1b

	// wrap-around:
	//   p10 = first bucket
	//   p11 = mask (and maybe other bits on LP64)
	//   p12 = _cmd & mask
	//
	// A full cache can happen with CACHE_ALLOW_FULL_UTILIZATION.
	// So stop when we circle back to the first probed bucket
	// rather than when hitting the first bucket again.
	//
	// Note that we might probe the initial bucket twice
	// when the first probed slot is the last entry.


#if CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16_BIG_ADDRS
	add	p13, p10, w11, UXTW #(1+PTRSHIFT)
						// p13 = buckets + (mask << 1+PTRSHIFT)
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16
	add	p13, p10, p11, LSR #(48 - (1+PTRSHIFT))
						// p13 = buckets + (mask << 1+PTRSHIFT)
						// see comment about maskZeroBits
					//「中文」回绕：跳到表尾（最后一个 bucket）从尾部继续探测
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_LOW_4
	add	p13, p10, p11, LSL #(1+PTRSHIFT)
						// p13 = buckets + (mask << 1+PTRSHIFT)
#else
#error Unsupported cache mask storage for ARM64.
#endif
	add	p12, p10, p12, LSL #(1+PTRSHIFT)
						// p12 = first probed bucket
					//「中文」记下最初探测的位置，绕一圈回到这里就停

						// do {
4:	ldp	p17, p9, [x13], #-BUCKET_SIZE	//     {imp, sel} = *bucket--
	cmp	p9, p1				//     if (sel == _cmd)
	b.eq	2b				//         goto hit
	cmp	p9, #0				// } while (sel != 0 &&
	ccmp	p13, p12, #0, ne		//     bucket > first_probed)
	b.hi	4b

LLookupEnd\Function:
LLookupRecover\Function:
	b	\MissLabelDynamic		//「中文」绕完整圈仍没有 → 未命中

#if CONFIG_USE_PREOPT_CACHES
#if CACHE_MASK_STORAGE != CACHE_MASK_STORAGE_HIGH_16
#error config unsupported
#endif
LLookupPreopt\Function:
	//「中文」以下整段是 dyld 共享缓存「预优化缓存」专用查找，普通动态类不会进
#if __has_feature(ptrauth_calls)
	and	p10, p11, #0x007ffffffffffffe	// p10 = buckets
	autdb	x10, x16			// auth as early as possible
#endif

	// x12 = (_cmd - first_shared_cache_sel)
	adrp	x9, _MagicSelRef@PAGE
	ldr	p9, [x9, _MagicSelRef@PAGEOFF]
	sub	p12, p1, p9

	// w9  = ((_cmd - first_shared_cache_sel) >> hash_shift & hash_mask)
#if __has_feature(ptrauth_calls)
	// bits 63..60 of x11 are the number of bits in hash_mask
	// bits 59..55 of x11 is hash_shift

	lsr	x17, x11, #55			// w17 = (hash_shift, ...)
	lsr	w9, w12, w17			// >>= shift

	lsr	x17, x11, #60			// w17 = mask_bits
	mov	x11, #0x7fff
	lsr	x11, x11, x17			// p11 = mask (0x7fff >> mask_bits)
	and	x9, x9, x11			// &= mask
#else
	// bits 63..53 of x11 is hash_mask
	// bits 52..48 of x11 is hash_shift
	lsr	x17, x11, #48			// w17 = (hash_shift, hash_mask)
	lsr	w9, w12, w17			// >>= shift
	and	x9, x9, x11, LSR #53		// &=  mask
#endif

	// sel_offs is 26 bits because it needs to address a 64 MB buffer (~ 20 MB as of writing)
	// keep the remaining 38 bits for the IMP offset, which may need to reach
	// across the shared cache. This offset needs to be shifted << 2. We did this
	// to give it even more reach, given the alignment of source (the class data)
	// and destination (the IMP)
	ldr	x17, [x10, x9, LSL #3]		// x17 == (sel_offs << 38) | imp_offs
	cmp	x12, x17, LSR #38

.if \Mode == GETIMP
	b.ne	\MissLabelConstant		// cache miss
	sbfiz x17, x17, #2, #38         // imp_offs = combined_imp_and_sel[0..37] << 2
	sub	x0, x16, x17        		// imp = isa - imp_offs
	SignAsImp x0, x17
	ret
.else
	b.ne	5f				        // cache miss
	sbfiz x17, x17, #2, #38         // imp_offs = combined_imp_and_sel[0..37] << 2
	sub x17, x16, x17               // imp = isa - imp_offs
.if \Mode == NORMAL
	br	x17
.elseif \Mode == LOOKUP
	orr x16, x16, #3 // for instrumentation, note that we hit a constant cache
	SignAsImp x17, x10
	ret
.else
.abort  unhandled mode \Mode
.endif

5:	ldur	x9, [x10, #-16]			// offset -16 is the fallback offset
	add	x16, x16, x9			// compute the fallback isa
	b	LLookupStart\Function		// lookup again with a new isa
.endif
#endif // CONFIG_USE_PREOPT_CACHES
```

**真机实测落在哪条分支？** 由 `objc-config.h:218` 决定：**iOS 真机 = `HIGH_16`，macOS（M 系列）/模拟器 = `HIGH_16_BIG_ADDRS`**（用户态地址空间更大，buckets 占满低 48 位）。两条逻辑几乎一样（mask 在高 16、buckets 在低 48），区别仅在 BIG_ADDRS 分两步取、且不含上面那段 preopt 共享缓存岔路（`CONFIG_USE_PREOPT_CACHES` 只对 `HIGH_16` 开）。下面这段是在 **macOS 上 LLDB 实测**，所以内联展开的正是 **BIG_ADDRS** 分支，且把命中（CacheHit）和回绕（wrap-around）也一并抓全：

```text
;; LLDB disassemble --frame：objc_msgSend 内联展开的 CacheLookup（macOS arm64 = HIGH_16_BIG_ADDRS）
;; 寄存器约定：x16=isa/cls  x1=_cmd(SEL)  x10=buckets基址  x11=mask  x13=当前bucket  x9=槽内SEL  x17=槽内IMP
  <+28>: mov  x15, x16                  ; 备份 isa（转发回退父类时要用，对应宏 `mov x15,x16`）
  <+32>: ldr  x10, [x16, #0x10]         ; x10 = cache 字段 = mask|buckets（#CACHE=0x10）
  <+36>: lsr  x11, x10, #48             ; x11 = mask（高 16 位）
  <+40>: and  x10, x10, #0xffffffffffff ; x10 = buckets（低 48 位）
  <+44>: eor  x12, x1, x1, lsr #7       ; sel ^ (sel>>7)  —— 对应 cache_hash 的 SEL_HASH_SHIFT_XOR
  <+48>: and  w12, w12, w11             ; x12 = hash & mask = 起始槽下标
  <+52>: add  x13, x10, x12, lsl #4     ; x13 = &buckets[idx]（每格 16B = BUCKET_SIZE）
;; ---------- do { 开放寻址探测 ----------
  <+56>: ldp  x17, x9, [x13], #-0x10    ; {imp,sel} = *bucket；指针 -16（向低地址挪一格）
  <+60>: cmp  x9, x1                    ; 槽里的 sel == _cmd ?
  <+64>: b.ne <+80>                     ;   不等 → 去判空 / 继续探测
;; ---------- CacheHit：命中 ----------
  <+68>: eor  x10, x10, x1              ; x10 = buckets ^ sel
  <+72>: eor  x10, x10, x16             ; x10 = buckets ^ sel ^ cls  ← modifierForSEL（3 项！呼应 4.2）
  <+76>: brab x17, x10                  ; 用该 modifier 认证解签 IMP 并尾跳 → 直接进入方法实现（不返回）
;; ---------- 未命中 / 是否继续 ----------
  <+80>: cbz  x9, ...b40                ; 撞到空槽(sel==0) → _objc_msgSend_uncached（慢速查找，第二部分）
  <+84>: cmp  x13, x10                  ; bucket 还没越过表头 buckets ?
  <+88>: b.hs <+56>                     ;   是 → 回 <+56> 继续  } while (bucket >= buckets)
;; ---------- wrap-around：绕到表尾再扫一圈 ----------
  <+92>:  add x13, x10, w11, uxtw #4    ; x13 = 表尾最后一个 bucket（mask 即末尾下标）
  <+96>:  add x12, x10, x12, lsl #4     ; x12 = 最初探测位置（绕回到这里就停）
  <+100>: ldp x17, x9, [x13], #-0x10    ; {imp,sel} = *bucket；指针 -16
  <+104>: cmp x9, x1                    ; sel == _cmd ?
  <+108>: b.eq <+68>                    ;   命中 → 回 <+68> 走 CacheHit
  <+112>: cmp x9, #0x0                  ; sel == 0 ?（空槽）
  <+116>: ccmp x13, x12, #0x0, ne       ; 且 bucket > 最初位置 ?（两条件与）
  <+120>: b.hi <+100>                   ;   都成立 → 继续绕
  <+124>: b   ...b40                    ; 绕完整圈仍没有 → _objc_msgSend_uncached（未命中）
```

> 把上面那串指令抽成「人话」流程（开放寻址 + 向低地址探测 + 末尾回绕）：

```text
idx    = (sel ^ (sel >> 7)) & mask        // cache_hash：算起始槽
bucket = &buckets[idx]
do {
    {imp, s} = *bucket                    // 读当前槽的 {IMP, SEL}
    if (s == sel) goto 命中               //   SEL 对上 → 命中
    if (s == 0)   goto 未命中             //   撞到空槽 → 缓存里压根没有
    bucket--                              // 向低地址挪一格（开放寻址线性探测）
} while (bucket >= buckets)               // 没越过表头就继续
// 越过表头 → wrap 到表尾，再扫到「最初起始槽」为止；仍没有 → 未命中

命中:  imp = auth(imp, buckets ^ sel ^ cls);  br imp   // 解签后尾跳，不留栈帧（§3.3 / §4.2）
未命中: b _objc_msgSend_uncached                        // 转入 lookUpImpOrForward（第二部分）
```

### 3.3 命中 `br IMP` / 未命中转入慢速查找（`CacheHit` :316）

逐字全文（含 NORMAL / GETIMP / LOOKUP 三种模式）：

```asm
;; objc-msg-arm64.s:315  —— CacheHit 宏（全文）
// CacheHit: x17 = cached IMP, x10 = address of buckets, x1 = SEL, x16 = isa
.macro CacheHit
.if $0 == NORMAL
	TailCallCachedImp x17, x10, x1, x16	// authenticate and call imp
					//「中文」objc_msgSend 走这条：解签 IMP 后 br 尾跳，不返回
.elseif $0 == GETIMP
	mov	p0, p17
	cbz	p0, 9f					// don't ptrauth a nil imp
	AuthAndResignAsIMP x0, x10, x1, x16, x17	// authenticate imp and re-sign as IMP
9:	ret						// return IMP
					//「中文」cache_getImp 走这条：把 IMP 当返回值给出，不调用
.elseif $0 == LOOKUP
	// No nil check for ptrauth: the caller would crash anyway when they
	// jump to a nil IMP. We don't care if that jump also fails ptrauth.
	AuthAndResignAsIMP x17, x10, x1, x16, x10	// authenticate imp and re-sign as IMP
	cmp	x16, x15
	cinc	x16, x16, ne			// x16 += 1 when x15 != x16 (for instrumentation ; fallback to the parent class)
	ret				// return imp via x17
					//「中文」objc_msgLookup 走这条：返回 IMP 但不跳（super 调用用）
.else
.abort oops
.endif
.endmacro
```

LLDB 实测：`NORMAL` 模式 `TailCallCachedImp` 落地为两条 `eor`（重算 ptrauth 修饰子）+ `brab x17`（带 PAC 校验尾跳）；未命中则 `cbz x9 → _objc_msgSend_uncached`：

```text
  <+64>: b.ne   <+80>
  <+68>: eor    x10, x10, x1                 ; 修饰子 ^= sel
  <+72>: eor    x10, x10, x16                ; 修饰子 ^= cls
  <+76>: brab   x17, x10                     ; 命中：带 PAC 尾跳 IMP
  <+80>: cbz    x9, 0x...b40                  ; _objc_msgSend_uncached（miss）
```

> ### 🔄 旧→新对照（objc4-818.2 → 951.1）：预优化缓存条目的位布局
>
> 共享缓存「预优化缓存」里每个条目把 sel 偏移与 imp 偏移打包进一个 64 位字，**两版位划分不同**。
>
> 旧（818.2，`objc-msg-arm64.s:474`）—— sel / imp 各 32 位：
> ```asm
> ldr  x17, [x10, x9, LSL #3]   // x17 == sel_offs | (imp_offs << 32)
> cmp  x12, w17, uxtw
> ...
> sub  x0, x16, x17, LSR #32    // imp = isa - imp_offs
> ...
> 5: ldursw x9, [x10, #-8]      // fallback offset 在 -8
> ```
> 新（951.1，`:500`）—— 改成 sel 26 位 / imp 38 位（imp 偏移再 `<<2`）：
> ```asm
> ldr   x17, [x10, x9, LSL #3]   // x17 == (sel_offs << 38) | imp_offs
> cmp   x12, x17, LSR #38
> ...
> sbfiz x17, x17, #2, #38        // imp_offs = bits[0..37] << 2
> sub   x0, x16, x17             // imp = isa - imp_offs
> ...
> 5: ldur  x9, [x10, #-16]       // fallback offset 移到 -16
> ```
> 动机（951 新增注释自陈）：共享缓存越来越大，32 位 imp 偏移不够「够到」远处的 IMP，于是把 imp 偏移扩到 38 位（再 `<<2` 进一步增大寻址范围），sel 偏移压到 26 位（足够寻址 ~64MB 选择子表）。

## 4. cache_t 数据结构（`objc-runtime-new.h:337`）

### 4.1 `_bucketsAndMaybeMask`：指针与 mask 的融合

`cache_t` 的**数据布局部分**逐字全文（含 `OUTLINED / HIGH_16 / HIGH_16_BIG_ADDRS / LOW_4` 全部分支常量；其后是访问器方法声明与 `FAST_CACHE_*` 快速分配标志，属另一主题，本块到 mask 存储常量为止）：

```objc
// objc-runtime-new.h:337  —— cache_t 字段与各架构 mask 存储常量（全文，越界处已标注）
struct cache_t {
private:
    explicit_atomic<uintptr_t> _bucketsAndMaybeMask;
    //「中文」核心字段：arm64 上高 16 位 = mask，低 48 位 = buckets 指针（一字双用）
    union {
        // Note: _flags on ARM64 needs to line up with the unused bits of
        // _originalPreoptCache because we access some flags (specifically
        // FAST_CACHE_HAS_DEFAULT_CORE and FAST_CACHE_HAS_DEFAULT_AWZ) on
        // unrealized classes with the assumption that they will start out
        // as 0.
        struct {
#if CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_OUTLINED && !__LP64__
            // Outlined cache mask storage, 32-bit, we have mask and occupied.
            explicit_atomic<mask_t>    _mask;
            uint16_t                   _occupied;
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_OUTLINED && __LP64__
            // Outlined cache mask storage, 64-bit, we have mask, occupied, flags.
            explicit_atomic<mask_t>    _mask;
            uint16_t                   _occupied;
            uint16_t                   _flags;
#   define CACHE_T_HAS_FLAGS 1
#elif __LP64__
            //「中文」arm64 真机走这里（HIGH_16）：注意没有独立 _mask！mask 在 _bucketsAndMaybeMask 高位
            // Inline cache mask storage, 64-bit, we have occupied, flags, and
            // empty space to line up flags with originalPreoptCache.
            //
            // Note: the assembly code for objc_release_xN knows about the
            // location of _flags and the
            // FAST_CACHE_HAS_CUSTOM_DEALLOC_INITIATION flag within. Any changes
            // must be applied there as well.
            uint32_t                   _disguisedPreoptCacheSignature;
            uint16_t                   _occupied;
            uint16_t                   _flags;
#   define CACHE_T_HAS_FLAGS 1
#else
            // Inline cache mask storage, 32-bit, we have occupied, flags.
            uint16_t                   _occupied;
            uint16_t                   _flags;
#   define CACHE_T_HAS_FLAGS 1
#endif

        };
        explicit_atomic<preopt_cache_t *, PTRAUTH_STR(originalPreoptCache, ptrauth_key_process_independent_data)> _originalPreoptCache;
        //「中文」与上面 struct 共用内存：预优化缓存时这里是 preopt_cache_t*
    };

    // Simple constructor for testing purposes only.
    cache_t() : _bucketsAndMaybeMask(0) {}

#if CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_OUTLINED
    // _bucketsAndMaybeMask is a buckets_t pointer

    static constexpr uintptr_t bucketsMask = ~0ul;
    static_assert(!CONFIG_USE_PREOPT_CACHES, "preoptimized caches not supported");
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16_BIG_ADDRS
    static constexpr uintptr_t maskShift = 48;
    static constexpr uintptr_t maxMask = ((uintptr_t)1 << (64 - maskShift)) - 1;
    static constexpr uintptr_t bucketsMask = ((uintptr_t)1 << maskShift) - 1;

    static_assert(bucketsMask >= OBJC_VM_MAX_ADDRESS, "Bucket field doesn't have enough bits for arbitrary pointers.");
#if CONFIG_USE_PREOPT_CACHES
    static constexpr uintptr_t preoptBucketsMarker = 1ul;
    static constexpr uintptr_t preoptBucketsMask = bucketsMask & ~preoptBucketsMarker;
#endif
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16
    //「中文」arm64 真机的 mask 存储常量定义
    // _bucketsAndMaybeMask is a buckets_t pointer in the low 48 bits

    // How much the mask is shifted by.
    static constexpr uintptr_t maskShift = 48;		//「中文」mask 放在第 48 位起的高 16 位

    // Additional bits after the mask which must be zero. msgSend
    // takes advantage of these additional bits to construct the value
    // `mask << 4` from `_maskAndBuckets` in a single instruction.
    static constexpr uintptr_t maskZeroBits = 4;	//「中文」紧邻 mask 的 4 位必须为 0，
							// 让 msgSend 一条指令算出 mask<<4（见 3.2 的 LSR #(48-(1+PTRSHIFT))）

    // The largest mask value we can store.
    static constexpr uintptr_t maxMask = ((uintptr_t)1 << (64 - maskShift)) - 1;

    // The mask applied to `_maskAndBuckets` to retrieve the buckets pointer.
    static constexpr uintptr_t bucketsMask = ((uintptr_t)1 << (maskShift - maskZeroBits)) - 1;

    // Ensure we have enough bits for the buckets pointer.
    static_assert(bucketsMask >= OBJC_VM_MAX_ADDRESS,
            "Bucket field doesn't have enough bits for arbitrary pointers.");

#if CONFIG_USE_PREOPT_CACHES
    static constexpr uintptr_t preoptBucketsMarker = 1ul;
#if __has_feature(ptrauth_calls)
    // 63..60: hash_mask_shift
    // 59..55: hash_shift
    // 54.. 1: buckets ptr + auth
    //      0: always 1
    static constexpr uintptr_t preoptBucketsMask = 0x007ffffffffffffe;
    static inline uintptr_t preoptBucketsHashParams(const preopt_cache_t *cache) {
        uintptr_t value = (uintptr_t)cache->shift << 55;
        // masks have 11 bits but can be 0, so we compute
        // the right shift for 0x7fff rather than 0xffff
        return value | ((objc::mask16ShiftBits(cache->mask) - 1) << 60);
    }
#else
    // 63..53: hash_mask
    // 52..48: hash_shift
    // 47.. 1: buckets ptr
    //      0: always 1
    static constexpr uintptr_t preoptBucketsMask = 0x0000fffffffffffe;
    static inline uintptr_t preoptBucketsHashParams(const preopt_cache_t *cache) {
        return (uintptr_t)cache->hash_params << 48;
    }
#endif
#endif // CONFIG_USE_PREOPT_CACHES
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_LOW_4
    // _bucketsAndMaybeMask is a buckets_t pointer in the top 28 bits

    static constexpr uintptr_t maskBits = 4;
    static constexpr uintptr_t maskMask = (1 << maskBits) - 1;
    static constexpr uintptr_t bucketsMask = ~maskMask;
    static_assert(!CONFIG_USE_PREOPT_CACHES, "preoptimized caches not supported");
#else
#error Unknown cache mask storage type.
#endif

    // ……（以下为 mask()/buckets()/insert()/reallocate() 等方法声明，
    //     以及 FAST_CACHE_* 快速分配标志位定义，属另一主题，本块略去）
};
```

要点：arm64（`CACHE_MASK_STORAGE_HIGH_16`）上**没有独立 `_mask` 字段**——mask 编码在 `_bucketsAndMaybeMask` 高 16 位，这正是 3.2 里「一条 `ldr` 取出 `mask|buckets`，再 `lsr #48` 拆 mask、`and` 低 48 位拆 buckets」的来源。`maskZeroBits = 4` 让 msgSend 单条指令就能从该字段构造 `mask << 4`。

而这套「融合字」的拆/装，C++ 侧有对应实现——和 3.2 的汇编一一镜像：

```objc
// objc-cache.mm（arm64 HIGH_16 分支）
void cache_t::setBucketsAndMask(struct bucket_t *newBuckets, mask_t newMask) {   // :625 编码（装）
    // mask 放高位、buckets 放低位，或成一个字
    _bucketsAndMaybeMask.store(((uintptr_t)newMask << maskShift) | (uintptr_t)newBuckets,
                               memory_order_release);
    _occupied = 0;
}

mask_t cache_t::mask() const {                                                   // :637 取 mask（拆高位）
    uintptr_t maskAndBuckets = _bucketsAndMaybeMask.load(memory_order_relaxed);
    return maskAndBuckets >> maskShift;             //「中文」>>48，对应汇编 lsr p11,#48
}

struct bucket_t *cache_t::buckets() const {                                      // :671 取 buckets（拆低位）
    uintptr_t addr = _bucketsAndMaybeMask.load(memory_order_relaxed);
    return (bucket_t *)(addr & bucketsMask);        //「中文」& 低位掩码，对应汇编 and p10,…
}
```

`setBucketsAndMask`（`(mask<<48)|buckets`）= 汇编看到的那个融合字是怎么写进去的；`mask()`（`>>48`）/`buckets()`（`& bucketsMask`）= 汇编 `lsr`/`and` 在 C++ 里的等价拆解。换言之第 3.2 节那几条指令，就是把这三个 C++ 函数手写进了汇编快速路径。

> ### 🔄 旧→新对照（objc4-756.2 → 951.1）：cache_t 三字段 → 融合字段【经典大改造】
>
> 缓存结构最有名的一次重构。旧版三个独立字段，新版融合成一个 `_bucketsAndMaybeMask`。
>
> 旧（756.2，`objc-runtime-new.h:82`）：
> ```objc
> struct cache_t {
>     struct bucket_t *_buckets;   // 桶数组指针（独立字段）
>     mask_t _mask;                // 容量掩码（独立字段）
>     mask_t _occupied;            // 已用槽数（独立字段）
>     ...
>     void expand();
>     struct bucket_t * find(SEL sel, id receiver);
> };
> ```
> 新（951.1，`:337`，arm64 走 HIGH_16）：
> ```objc
> struct cache_t {
>     explicit_atomic<uintptr_t> _bucketsAndMaybeMask;  // 高16位=mask，低48位=buckets，一字双用
>     union {
>         struct { uint32_t _disguisedPreoptCacheSignature; uint16_t _occupied; uint16_t _flags; };
>         explicit_atomic<preopt_cache_t *> _originalPreoptCache;
>     };
>     ...
> };
> ```
> 三字段 → 一字段：`_mask` 不再单独存，挤进 `_bucketsAndMaybeMask` 的高 16 位（每个类省 8 字节，且第 3.2 节那条「一条 `ldr` 同时取出 mask 和 buckets」正因此成立）；新增的 union 是给 dyld 共享缓存「预优化缓存」让位。`_occupied` 保留。

### 4.2 `bucket_t`：`{IMP, SEL}` 与 ptrauth（:214）

逐字全文（含编码/解码/签名全部成员）：

```objc
// objc-runtime-new.h:214  —— bucket_t（全文）
struct bucket_t {
private:
    // IMP-first is better for arm64e ptrauth and no worse for arm64.
    // SEL-first is better for armv7* and i386 and x86_64.
#if __arm64__
    explicit_atomic<uintptr_t> _imp;	//「中文」arm64：IMP 在前（利于 ptrauth）
    explicit_atomic<SEL> _sel;
#else
    explicit_atomic<SEL> _sel;
    explicit_atomic<uintptr_t> _imp;
#endif

    // Compute the ptrauth signing modifier from &_imp, newSel, and cls.
    uintptr_t modifierForSEL(bucket_t *base, SEL newSel, Class cls) const {
        return (uintptr_t)base ^ (uintptr_t)newSel ^ (uintptr_t)cls;
        //「中文」IMP 的签名修饰子 = &_imp ^ sel ^ cls（命中时汇编里两条 eor 就是在重算它）
    }

    // Sign newImp, with &_imp, newSel, and cls as modifiers.
    uintptr_t encodeImp(UNUSED_WITHOUT_PTRAUTH bucket_t *base, IMP newImp, UNUSED_WITHOUT_PTRAUTH SEL newSel, Class cls) const {
        if (!newImp) return 0;
#if CACHE_IMP_ENCODING == CACHE_IMP_ENCODING_PTRAUTH
        return (uintptr_t)
        bitcast_auth_and_resign(void *, newImp,
                                ptrauth_key_function_pointer,
                                ptrauth_function_pointer_type_discriminator(IMP),
                                ptrauth_key_process_dependent_code,
                                modifierForSEL(base, newSel, cls));
#elif CACHE_IMP_ENCODING == CACHE_IMP_ENCODING_ISA_XOR
        return (uintptr_t)newImp ^ (uintptr_t)cls;	//「中文」非 ptrauth 设备：IMP ^ cls 简单混淆
#elif CACHE_IMP_ENCODING == CACHE_IMP_ENCODING_NONE
        return (uintptr_t)newImp;
#else
#error Unknown method cache IMP encoding.
#endif
    }

public:
    static inline size_t offsetOfSel() { return offsetof(bucket_t, _sel); }
    inline SEL sel() const { return _sel.load(memory_order_relaxed); }

#if CACHE_IMP_ENCODING == CACHE_IMP_ENCODING_ISA_XOR
#define MAYBE_UNUSED_ISA
#else
#define MAYBE_UNUSED_ISA __attribute__((unused))
#endif
    inline IMP rawImp(MAYBE_UNUSED_ISA objc_class *cls) const {
        uintptr_t imp = _imp.load(memory_order_relaxed);
        if (!imp) return nil;
#if CACHE_IMP_ENCODING == CACHE_IMP_ENCODING_PTRAUTH
#elif CACHE_IMP_ENCODING == CACHE_IMP_ENCODING_ISA_XOR
        imp ^= (uintptr_t)cls;
#elif CACHE_IMP_ENCODING == CACHE_IMP_ENCODING_NONE
#else
#error Unknown method cache IMP encoding.
#endif
        return (IMP)imp;
    }

    inline IMP imp(UNUSED_WITHOUT_PTRAUTH bucket_t *base, Class cls) const {
        uintptr_t imp = _imp.load(memory_order_relaxed);
        if (!imp) return nil;
#if CACHE_IMP_ENCODING == CACHE_IMP_ENCODING_PTRAUTH
        SEL sel = _sel.load(memory_order_relaxed);
        return bitcast_auth_and_resign(IMP, imp,
                                       ptrauth_key_process_dependent_code,
                                       modifierForSEL(base, sel, cls),
                                       ptrauth_key_function_pointer,
                                       ptrauth_function_pointer_type_discriminator(IMP));
#elif CACHE_IMP_ENCODING == CACHE_IMP_ENCODING_ISA_XOR
        return (IMP)(imp ^ (uintptr_t)cls);
#elif CACHE_IMP_ENCODING == CACHE_IMP_ENCODING_NONE
        return (IMP)imp;
#else
#error Unknown method cache IMP encoding.
#endif
    }

    inline void scribbleIMP(uintptr_t value) {
        _imp.store(value, memory_order_relaxed);
    }

    template <Atomicity, IMPEncoding>
    void set(bucket_t *base, SEL newSel, IMP newImp, Class cls);
};
```

arm64 上 IMP 在前、SEL 在后，所以 3.2 的 `ldp p17, p9`（先 imp 后 sel）顺序与此一致；命中时用 `modifierForSEL`（`&_imp ^ sel ^ cls`）重算修饰子解签，正是 3.3 实测的两条 `eor`。

> ### 🔄 旧→新对照（756.2 → 951.1）：IMP 签名修饰子从 2 项到 3 项
>
> 旧（756.2，`objc-runtime-new.h:51`）—— 裸字段，修饰子只有 `&_imp ^ sel` 两项：
> ```objc
> #if __arm64__
>     uintptr_t _imp;          // 非 atomic 裸字段
>     SEL _sel;
> #endif
>     uintptr_t modifierForSEL(SEL newSel) const {
>         return (uintptr_t)&_imp ^ (uintptr_t)newSel;       // 仅 2 项
>     }
> ```
> 新（951.1，`:219` / `:227`）—— 原子字段，修饰子加入 `cls` 成 3 项：
> ```objc
> #if __arm64__
>     explicit_atomic<uintptr_t> _imp;     // 改为原子字段
>     explicit_atomic<SEL> _sel;
> #endif
>     uintptr_t modifierForSEL(bucket_t *base, SEL newSel, Class cls) const {
>         return (uintptr_t)base ^ (uintptr_t)newSel ^ (uintptr_t)cls;   // 加入 cls，3 项
>     }
> ```
> 把 `cls` 拌进 ptrauth 修饰子，让同一个 IMP 在不同类的缓存里签名不同，跨类伪造更难——这也是第 3.3 节命中时汇编要算**两条** `eor`（`^sel`、`^cls`）的原因；756.2 时只需异或 `sel` 一项。字段也从裸 `uintptr_t` 升级为 `explicit_atomic` 配合无锁读。下面这条（818.2 → 951.1）则是同一行更晚的一次细化：
>
> ### 🔄 旧→新对照（818.2 → 951.1）：IMP 签名加入「类型判别子」
>
> 旧（818.2，`objc-runtime-new.h:234`）—— 判别子恒为 `0`：
> ```objc
> ptrauth_auth_and_resign(newImp,
>                         ptrauth_key_function_pointer, 0,        // 判别子 = 0
>                         ptrauth_key_process_dependent_code,
>                         modifierForSEL(base, newSel, cls));
> ```
> 新（951.1，`:236`）—— 换成 IMP 类型判别子：
> ```objc
> bitcast_auth_and_resign(void *, newImp,
>                         ptrauth_key_function_pointer,
>                         ptrauth_function_pointer_type_discriminator(IMP),  // 绑定到 IMP 类型
>                         ptrauth_key_process_dependent_code,
>                         modifierForSEL(base, newSel, cls));
> ```
> 把固定判别子 `0` 换成 `ptrauth_function_pointer_type_discriminator(IMP)`，签名绑定到「函数指针类型」，arm64e 上更难被跨类型伪造；951 还新增了配套的 `rawImp()` 取值方法（见上方 `bucket_t` 全文）。

### 4.3 哈希与开放寻址探测（`cache_hash` :307 / `cache_next` :241）

逐字全文：

```objc
// objc-cache.mm:240  —— cache_next（全文，含两种配置）
#if CACHE_END_MARKER
static inline mask_t cache_next(mask_t i, mask_t mask) {
    return (i+1) & mask;			//「中文」带哨兵：向高地址探测，回绕到 0
}
#elif __arm64__
static inline mask_t cache_next(mask_t i, mask_t mask) {
    return i ? i-1 : mask;			//「中文」arm64：向低地址探测，到 0 则回绕到 mask（表尾）
}
#else
#error unexpected configuration
#endif

// objc-cache.mm:304
// Class points to cache. SEL is key. Cache buckets store SEL+IMP.
// Caches are never built in the dyld shared cache.

// objc-cache.mm:307  —— cache_hash（全文）
static inline mask_t cache_hash(SEL sel, mask_t mask)
{
    uintptr_t value = (uintptr_t)sel;
#if SEL_HASH_SHIFT_XOR
    value ^= value >> 7;			//「中文」与汇编 eor p12,p1,p1,LSR #7 一一对应
#endif
    return (mask_t)(value & mask);
}
```

## 5. 缓存的写入与扩容（`objc-cache.mm`）

### 5.1 `insert`：哈希落位（:873）

逐字全文：

```objc
// objc-cache.mm:873  —— cache_t::insert（全文）
void cache_t::insert(SEL sel, IMP imp, id receiver)
{
    lockdebug::assert_locked(&runtimeLock.get());

    // Never cache before +initialize is done
    if (slowpath(!cls()->isInitialized())) {
        return;					//「中文」+initialize 没跑完不缓存
    }

    if (isConstantOptimizedCache()) {
        _objc_fatal("cache_t::insert() called with a preoptimized cache for %s",
                    cls()->nameForLogging());
    }

#if DEBUG_TASK_THREADS
    return _collecting_in_critical();
#else
#if CONFIG_USE_CACHE_LOCK
    mutex_locker_t lock(cacheUpdateLock);
#endif

    ASSERT(sel != 0 && cls()->isInitialized());

    // Use the cache as-is if until we exceed our expected fill ratio.
    mask_t newOccupied = occupied() + 1;
    unsigned oldCapacity = capacity(), capacity = oldCapacity;
    if (slowpath(isConstantEmptyCache())) {
        // Cache is read-only. Replace it.
        if (!capacity) capacity = INIT_CACHE_SIZE;	//「中文」空表首次分配，INIT_CACHE_SIZE = 4
        reallocate(oldCapacity, capacity, /* freeOld */false);
    }
    else if (fastpath(newOccupied + CACHE_END_MARKER <= cache_fill_ratio(capacity))) {
        // Cache is less than 3/4 or 7/8 full. Use it as-is.
        //「中文」装填率没到上限：原表直接用
    }
#if CACHE_ALLOW_FULL_UTILIZATION
    else if (capacity <= FULL_UTILIZATION_CACHE_SIZE && newOccupied + CACHE_END_MARKER <= capacity) {
        // Allow 100% cache utilization for small buckets. Use it as-is.
    }
#endif
    else {
        capacity = capacity ? capacity * 2 : INIT_CACHE_SIZE;	//「中文」超阈值 → 容量翻倍
        if (capacity > MAX_CACHE_SIZE) {
            capacity = MAX_CACHE_SIZE;
        }
        reallocate(oldCapacity, capacity, true);		//「中文」扩容并释放旧表
    }

    bucket_t *b = buckets();
    mask_t m = capacity - 1;
    mask_t begin = cache_hash(sel, m);
    mask_t i = begin;

    // Scan for the first unused slot and insert there.
    // There is guaranteed to be an empty slot.
    do {
        if (fastpath(b[i].sel() == 0)) {
            incrementOccupied();
            b[i].set<Atomic, Encoded>(b, sel, imp, cls());	//「中文」空槽：落位 + 按 4.2 签名
            return;
        }
        if (b[i].sel() == sel) {
            // The entry was added to the cache by some other thread
            // before we grabbed the cacheUpdateLock.
            return;						//「中文」别的线程已填同一 sel
        }
    } while (fastpath((i = cache_next(i, m)) != begin));	//「中文」开放寻址，绕一圈回起点才停

    bad_cache(receiver, (SEL)sel);
#endif // !DEBUG_TASK_THREADS
}
```

### 5.2 `reallocate`：翻倍扩容、丢弃旧表不迁移（:809）

逐字全文：

```objc
// objc-cache.mm:808  —— cache_t::reallocate（全文）
ALWAYS_INLINE
void cache_t::reallocate(mask_t oldCapacity, mask_t newCapacity, bool freeOld)
{
    bucket_t *oldBuckets = buckets();
    bucket_t *newBuckets = allocateBuckets(newCapacity);

    // Cache's old contents are not propagated.
    // This is thought to save cache memory at the cost of extra cache fills.
    // fixme re-measure this
    //「中文」关键：扩容时旧缓存内容整张丢弃、不迁移，靠后续 miss 重新填
    //         —— 用「几次额外慢速查找」换「省内存 + 实现简单」

    ASSERT(newCapacity > 0);
    ASSERT((uintptr_t)(mask_t)(newCapacity-1) == newCapacity-1);

    setBucketsAndMask(newBuckets, newCapacity - 1);

    if (freeOld) {
        collect_free(oldBuckets, oldCapacity);		//「中文」延迟回收旧表（可能有线程在读）
    }
}
```

> ### 🔄 旧→新对照（756.2 → 951.1）：缓存写入从「三个自由函数」到「一个 insert 方法」
>
> 旧（756.2）把写入拆成多块：`cache_fill_nolock()`（自由函数，`objc-cache.mm:556`）判 3/4 装填 → 满了调 `cache_t::expand()`（`:539`，`oldCapacity*2` 翻倍）→ 再 `cache_t::find()`（`:519`，开放寻址找空槽 / 同 sel）落位。
> ```objc
> // 756.2：expand() —— 翻倍扩容
> uint32_t newCapacity = oldCapacity ? oldCapacity*2 : INIT_CACHE_SIZE;
> // 756.2：find() —— 独立的开放寻址探测
> do {
>     if (b[i].sel() == 0 || b[i].sel() == s) return &b[i];
> } while ((i = cache_next(i, m)) != begin);
> ```
> 新（951.1）把「判装填 + 扩容 + 探测落位」**合并进一个 `cache_t::insert()`**（见 5.1 全文），`reallocate` 还多了 `freeOld` 参数管旧表回收。
>
> 一处**没变**的设计（756.2 `:476` 与 951.1 `:814` 注释一字不差）：`// Cache's old contents are not propagated.`——扩容即丢弃旧表、不迁移，从那时起就是如此，并非新近退化。

### 5.3 装填因子与留空策略

`insert` 里 `cache_fill_ratio(capacity)`（默认 3/4）就是装填因子上限；开放寻址表必须留空槽，3.2 探测循环正是靠 `cbz p9`（撞到 sel==0 的空槽）判定 miss——所以缓存永不会被填满到没有空位。

---

# 第二部分 · 慢速查找（lookUpImpOrForward）

快速路径 miss 后，`__objc_msgSend_uncached`（:737）通过 `MethodTableLookup`（:720）调进 C++ 的 `lookUpImpOrForward`。三段逐字全文：

```asm
;; objc-msg-arm64.s:720  —— MethodTableLookup / uncached / msgForward（全文）
.macro MethodTableLookup

	SAVE_REGS MSGSEND			//「中文」保存参数寄存器（要调 C 函数，得守约定）

	// lookUpImpOrForward(obj, sel, cls, LOOKUP_INITIALIZE | LOOKUP_RESOLVER)
	// receiver and selector already in x0 and x1
	mov	x2, x16				//「中文」arg2 = cls
	mov	x3, #3				//「中文」arg3 = 3 = INITIALIZE|RESOLVER
	bl	_lookUpImpOrForward

	// IMP in x0
	mov	x17, x0				//「中文」慢速查找结果 IMP 回到 x17

	RESTORE_REGS MSGSEND

.endmacro

	STATIC_ENTRY __objc_msgSend_uncached
	UNWIND __objc_msgSend_uncached, FrameWithNoSaves

	// THIS IS NOT A CALLABLE C FUNCTION
	// Out-of-band p15 is the class to search

	MethodTableLookup
	TailCallFunctionPointer x17		//「中文」拿到 IMP 后尾跳过去（含转发占位 IMP）

	END_ENTRY __objc_msgSend_uncached
```

## 6. 入口：加锁 + realize / +initialize（:7624）

`lookUpImpOrForward` 是整条慢速链的中枢，逐字全文：

```objc
// objc-runtime-new.mm:7624  —— lookUpImpOrForward（全文）
IMP lookUpImpOrForward(id inst, SEL sel, Class cls, int behavior)
{
    const IMP forward_imp = (IMP)_objc_msgForward_impcache;	//「中文」转发占位 IMP（见第 11 节）
    IMP imp = nil;
    Class curClass;

    lockdebug::assert_unlocked(&runtimeLock.get());

    if (slowpath(!cls->isInitialized())) {
        // The first message sent to a class is often +new or +alloc, or +self
        // which goes through objc_opt_* or various optimized entry points.
        //
        // However, the class isn't realized/initialized yet at this point,
        // and the optimized entry points fall down through objc_msgSend,
        // which ends up here.
        //
        // We really want to avoid caching these, as it can cause IMP caches
        // to be made with a single entry forever.
        //
        // Note that this check is racy as several threads might try to
        // message a given class for the first time at the same time,
        // in which case we might cache anyway.
        behavior |= LOOKUP_NOCACHE;		//「中文」类还没初始化完，先别缓存
    }

    // runtimeLock is held during isRealized and isInitialized checking
    // to prevent races against concurrent realization.

    // runtimeLock is held during method search to make
    // method-lookup + cache-fill atomic with respect to method addition.
    // Otherwise, a category could be added but ignored indefinitely because
    // the cache was re-filled with the old value after the cache flush on
    // behalf of the category.

    runtimeLock.lock();				//「中文」全程持锁：查找+回填 对 方法增删 保持原子

    // We don't want people to be able to craft a binary blob that looks like
    // a class but really isn't one and do a CFI attack.
    //
    // To make these harder we want to make sure this is a class that was
    // either built into the binary or legitimately registered through
    // objc_duplicateClass, objc_initializeClassPair or objc_allocateClassPair.
    checkIsKnownClass(cls);			//「中文」防伪造 class 的 CFI 攻击

    cls = realizeAndInitializeIfNeeded_locked(inst, cls, behavior & LOOKUP_INITIALIZE);
    //「中文」必要时 realize（铺开 ro/rw、方法表排序）并触发 +initialize
    // runtimeLock may have been dropped but is now locked again
    lockdebug::assert_locked(&runtimeLock.get());
    curClass = cls;

    // The code used to lookup the class's cache again right after
    // we take the lock but for the vast majority of the cases
    // evidence shows this is a miss most of the time, hence a time loss.
    //
    // The only codepath calling into this without having performed some
    // kind of cache lookup is class_getInstanceMethod().

    // Has this class been disabled? Act like a message to nil.
    if (!cls || !cls->ISA()) {
#if __arm64__
        imp = _objc_returnNil;
        goto done;
#elif __x86_64
        if (behavior & LOOKUP_FPRET)
            imp = _objc_msgNil_fpret;
        else if (behavior & LOOKUP_FP2RET)
            imp = _objc_msgNil_fp2ret;
        else
            imp = _objc_msgNil;

        // We can't cache these on x86, in case some other caller tries sending
        // this selector with a different return type. If we con't cache then we
        // always come back here, and always choose the correct IMP for the
        // caller's expected return type.
        behavior |= LOOKUP_NOCACHE;

        goto done;
#else
#error "Don't know how to handle messages to disabled classes on this target."
#endif
    }

    for (unsigned attempts = unreasonableClassCount();;) {
        if (curClass->cache.isConstantOptimizedCache(/* strict */true)) {
#if CONFIG_USE_PREOPT_CACHES
            imp = cache_getImp(curClass, sel);		//「中文」预优化缓存分支
            if (imp) goto done_unlock;
            curClass = curClass->cache.preoptFallbackClass();
#endif
        } else {
            // curClass method list.
            method_t *meth = getMethodNoSuper_nolock(curClass, sel);	//「中文」7.1：查当前类方法表
            if (meth) {
                imp = meth->imp(false);
                goto done;				//「中文」找到 → 去回填缓存
            }

            if (slowpath((curClass = curClass->getSuperclass()) == nil)) {
                // No implementation found, and method resolver didn't help.
                // Use forwarding.
                imp = forward_imp;			//「中文」到顶(NSObject 之上=nil)还没有 → 转发
                break;
            }
        }

        // Halt if there is a cycle in the superclass chain.
        if (slowpath(--attempts == 0)) {
            _objc_fatal("Memory corruption in class list.");
        }

        // Superclass cache.
        imp = cache_getImp(curClass, sel);		//「中文」第 8 节：上溯一层先查父类缓存
        if (slowpath(imp == forward_imp)) {
            // Found a forward:: entry in a superclass.
            // Stop searching, but don't cache yet; call method
            // resolver for this class first.
            break;
        }
        if (fastpath(imp)) {
            // Found the method in a superclass. Cache it in this class.
            goto done;					//「中文」父类缓存命中 → 回填本类
        }
    }

    // No implementation found. Try method resolver once.

    if (slowpath(behavior & LOOKUP_RESOLVER)) {
        behavior ^= LOOKUP_RESOLVER;			//「中文」第 10 节：只给一次动态解析机会
        return resolveMethod_locked(inst, sel, cls, behavior);
    }

 done:
    if (fastpath((behavior & LOOKUP_NOCACHE) == 0)) {
#if CONFIG_USE_PREOPT_CACHES
        while (cls->cache.isConstantOptimizedCache(/* strict */true)) {
            cls = cls->cache.preoptFallbackClass();
        }
#endif
        log_and_fill_cache(cls, imp, sel, inst, curClass);	//「中文」第 9 节：回填缓存，闭环
    }
#if CONFIG_USE_PREOPT_CACHES
 done_unlock:
#endif
    runtimeLock.unlock();
    if (slowpath((behavior & LOOKUP_NIL) && imp == forward_imp)) {
        return nil;
    }
    return imp;
}
```

> ### 🔄 旧→新对照（756.2 → 951.1）：查找函数的签名与「乐观缓存查找」
>
> 旧（756.2，`objc-runtime-new.mm:5264`）—— Class 在前、三个 bool 参数；函数**开头自带一次乐观缓存查找**；对外入口是包装函数 `_class_lookupMethodAndLoadCache3`：
> ```objc
> IMP _class_lookupMethodAndLoadCache3(id obj, SEL sel, Class cls) {      // :5245 旧对外入口
>     return lookUpImpOrForward(cls, sel, obj, YES/*initialize*/, NO/*cache*/, YES/*resolver*/);
> }
> IMP lookUpImpOrForward(Class cls, SEL sel, id inst,
>                        bool initialize, bool cache, bool resolver)      // :5264 旧签名（3 个 bool）
> {
>     IMP imp = nil;
>     bool triedResolver = NO;
>     runtimeLock.assertUnlocked();
>     // Optimistic cache lookup
>     if (cache) {                          // ← 进锁前先查一次缓存
>         imp = cache_getImp(cls, sel);
>         if (imp) return imp;
>     }
>     runtimeLock.lock();
>     ...
> }
> ```
> 新（951.1，`:7624`）—— `inst` 在前、三个 bool 合并成一个 `int behavior` 位掩码；并**删掉了开头那次乐观缓存查找**：
> ```objc
> IMP lookUpImpOrForward(id inst, SEL sel, Class cls, int behavior)       // :7624 新签名（位掩码）
> // 注释（:7673）：进锁后这里曾经会再查一次缓存，但实测「绝大多数是 miss」反而费时，故移除
> ```
> 三个 bool → 一个 `behavior` 位掩码（`LOOKUP_INITIALIZE | LOOKUP_RESOLVER | LOOKUP_NIL | LOOKUP_NOCACHE`，更易扩展）；旧的 `triedResolver` 布尔也被 `behavior ^= LOOKUP_RESOLVER` 取代。下面这条（818.2 → 951.1）是更晚的一处增量：
>
> ### 🔄 旧→新对照（818.2 → 951.1）：`lookUpImpOrForward` 新增「禁用类」短路
>
> 951 在取锁、realize 之后、进入查找主循环之前，**多了一段**把「被禁用的类」按 nil 处理的短路（818.2 此处直接进 `for` 主循环）：
> ```objc
> // 951.1 新增（objc-runtime-new.mm:7681；旧版 818.2 在 curClass=cls 后直接进 for 循环）
> // Has this class been disabled? Act like a message to nil.
> if (!cls || !cls->ISA()) {
> #if __arm64__
>     imp = _objc_returnNil;
>     goto done;
> #elif __x86_64
>     ... 按返回类型选 _objc_msgNil / _objc_msgNil_fpret / _objc_msgNil_fp2ret ...
>     behavior |= LOOKUP_NOCACHE;
>     goto done;
> #endif
> }
> ```
> 另外两处小改：818.2 的 `Method meth = getMethodNoSuper_nolock(...)` 在 951 改为具体类型 `method_t *meth = ...`；818.2 紧跟其后的 `lookupMethodInClassAndLoadCache`（`.cxx_construct/.cxx_destruct` 专用）在 951 此处已被挪走。

## 7. 在当前类找方法

### 7.1 `getMethodNoSuper_nolock` 遍历方法列表（:7290）

逐字全文：

```objc
// objc-runtime-new.mm:7290  —— getMethodNoSuper_nolock（全文）
getMethodNoSuper_nolock(Class cls, SEL sel)
{
    lockdebug::assert_locked(&runtimeLock.get());

    ASSERT(cls->isRealized());
    // fixme nil cls?
    // fixme nil sel?

    auto alternates = cls->data()->methodAlternates();
    //「中文」取该类的方法列表形态（可能是单表 / 多表数组 / 相对偏移表）

    if (auto *relativeList = alternates.relativeList)
        return getMethodFromRelativeList(relativeList, sel);

    if (alternates.list)
        return getMethodFromListArray(&alternates.list, 1, sel);	//「中文」单个方法表

    if (auto *array = alternates.array) {
        auto listAlternates = array->listAlternates();
        if (listAlternates.oneList)
            return getMethodFromListArray(&listAlternates.oneList, 1, sel);
        if (auto innerArray = listAlternates.array)
            return getMethodFromListArray(innerArray, listAlternates.arrayCount, sel);
        if (auto *relativeList = listAlternates.listList)
            return getMethodFromRelativeList(relativeList, sel);
    }

    return nil;
}
```

### 7.2 已排序表二分查找 `findMethodInSortedMethodList`（:7062 → :7006）

逐字全文（含模板实现、`compare`、按 small/big 分发的入口）：

```objc
// objc-runtime-new.mm:7006  —— 二分查找实现（全文）
findMethodInSortedMethodList(SEL key, const method_list_t *list, const compareFunc &compare)
{
    ASSERT(list);

    auto first = list->begin();
    auto base = first;

    uint32_t count;

    // When to stop the binary search and move to a linear search.
    const uint32_t threshold = 4;			//「中文」区间缩到 ≤4 改线性扫更快

    for (count = list->count; count > threshold; count >>= 1) {
        auto probe = base + (count >> 1);

        int comparison = compare(probe);
        if (comparison == 0) {
            // `probe` is a match.
            // Rewind looking for the *first* occurrence of this value.
            // This is required for correct category overrides.
            while (probe > first && compare(probe - 1) == 0) {
                probe--;				//「中文」回退到首个相等项，保证分类覆盖取到后者
            }
            return &*probe;
        }

        if (comparison > 0) {
            base = probe + 1;
            count--;
        }
    }

    // Once we've shrunk the range enough, it's faster to do a linear search.
    while (count-- > 0) {
        auto comparison = compare(base);
        if (comparison == 0)
            return &*base;
        if (comparison < 0)
            return nil;
        base++;
    }

    return nil;
}

template<typename T>
ALWAYS_INLINE static int
compare(T lhs, T rhs) {
    if ((uintptr_t)lhs > (uintptr_t)rhs)
        return 1;
    if ((uintptr_t)lhs < (uintptr_t)rhs)
        return -1;
    return 0;
}

// objc-runtime-new.mm:7061  —— 按方法表种类分发的入口（全文）
ALWAYS_INLINE static method_t *
findMethodInSortedMethodList(SEL key, const method_list_t *list)
{
    switch (list->listKind()) {
        case method_t::Kind::small:
            //「中文」small method：名字以相对偏移存储（省内存、可放只读段）
            if (CONFIG_SHARED_CACHE_RELATIVE_DIRECT_SELECTORS && objc::inSharedCache((uintptr_t)list)) {
                if (!objc::inSharedCache((uintptr_t)key))
                    return nil;
                uintptr_t keyOffset = (uintptr_t)key - sharedCacheRelativeMethodBase();
                return findMethodInSortedMethodList(key, list, [=](method_t &m) { return compare(keyOffset, (uintptr_t)m.getSmallNameAsSELOffset()); });
            } else {
                return findMethodInSortedMethodList(key, list, [=](method_t &m) { return compare(key, m.getSmallNameAsSELRef()); });
            }
        case method_t::Kind::big:
            return findMethodInSortedMethodList(key, list, [=](method_t &m) { return compare(key, m.big().name); });
        case method_t::Kind::bigSigned:
            return findMethodInSortedMethodList(key, list, [=](method_t &m) { return compare(key, m.bigSigned().name); });
    }
}
```

方法列表按 SEL 地址排序故能二分；命中后**向前回退到第一个相等项**，保证分类覆盖原方法时取到分类版本。

> ### 🔄 旧→新对照（818.2 → 951.1）：二分查找入口从「二分类」到「三分类 + 相对偏移比较」
>
> 旧（818.2，`objc-runtime-new.mm:5962`）—— `isSmallList()` 两分支，lambda 直接给出 SEL：
> ```objc
> findMethodInSortedMethodList(SEL key, const method_list_t *list)
> {
>     if (list->isSmallList()) {
>         if (CONFIG_SHARED_CACHE_RELATIVE_DIRECT_SELECTORS && objc::inSharedCache((uintptr_t)list)) {
>             return findMethodInSortedMethodList(key, list, [](method_t &m) { return m.getSmallNameAsSEL(); });
>         } else {
>             return findMethodInSortedMethodList(key, list, [](method_t &m) { return m.getSmallNameAsSELRef(); });
>         }
>     } else {
>         return findMethodInSortedMethodList(key, list, [](method_t &m) { return m.big().name; });
>     }
> }
> ```
> 新（951.1，`:7061`）—— `switch (list->listKind())` 三种 Kind（**新增 `bigSigned`**），且共享缓存里改成按相对偏移 `compare(keyOffset, ...)` 直接比较，不再先解析出 SEL（全文见 7.2 上方）。
>
> 两点实质变化：① 方法名指针在 arm64e 上可被签名，于是多出 `bigSigned` 这一 Kind；② 共享缓存场景用 `key - sharedCacheRelativeMethodBase()` 得到的偏移直接比，省掉「解引用取 SEL」一步。

## 8. 沿 superclass 链逐层上溯（每层先查父类 cache）

逻辑全在第 6 节 `lookUpImpOrForward` 的 `for` 主循环里（已逐字给出）。提炼该循环的上溯部分：

- 当前类方法表没有 → `curClass = curClass->getSuperclass()`；
- 到 `nil`（NSObject 再往上）仍无 → `imp = forward_imp` 转发；
- 否则对**父类先查 cache**（`cache_getImp(curClass, sel)`）：命中转发标记则 break 去解析，命中真 IMP 则 `goto done` 回填到**本类**。

即每上溯一层，先走父类的快速缓存（汇编 `_cache_getImp`，CacheHit 的 GETIMP 模式），cache 没有再查父类方法表——快慢两条路在类链上交替。

## 9. 找到了：`log_and_fill_cache` 回填缓存，闭环回快速路径（:7513）

逐字全文：

```objc
// objc-runtime-new.mm:7513  —— log_and_fill_cache（全文）
log_and_fill_cache(Class cls, IMP imp, SEL sel, id receiver, Class implementer)
{
#if SUPPORT_MESSAGE_LOGGING
    if (slowpath(objcMsgLogEnabled && implementer)) {	//「中文」instrumentObjcMessageSends 开关打开才记日志
        bool cacheIt = logMessageSend(implementer->isMetaClass(),
                                      cls->nameForLogging(),
                                      implementer->nameForLogging(),
                                      sel);
        if (!cacheIt) return;
    }
#endif
    if (slowpath(msgSendCacheMissHook.isSet())) {
        auto hook = msgSendCacheMissHook.get();
        hook(cls, receiver, sel, imp);
    }

    cls->cache.insert(sel, imp, receiver);		//「中文」→ 第 5 节 insert，落进本类缓存
}
```

回填后下一次同样的消息就在第 3 节快速路径命中。**LLDB 实测**：`[d bark]` 第一次停在 `lookUpImpOrForward`，第二次**没有再停**（命中直接 `brab`），两行 `woof` 连续打印、进程干净退出：

```text
(lldb) continue          # bark 第二次
2026-... msgsend-demo[..] woof
2026-... msgsend-demo[..] woof
Process exited with status = 0 (0x00000000)
# 第二次未再触发 lookUpImpOrForward 断点 = cache 命中
```

## 10. 没找到：动态方法解析（`resolveMethod_locked` :7480 → `resolveInstanceMethod` :7435）

> 易混提示：**:7480 是 `resolveMethod_locked`**（解析入口），真正给元类发 `+resolveInstanceMethod:` 的 `resolveInstanceMethod` 函数在 **:7435**——两者别混（:7430 只是 `resolveInstanceMethod` 的注释行）。

两个函数逐字全文：

```objc
// objc-runtime-new.mm:7435  —— resolveInstanceMethod（全文）
static void resolveInstanceMethod(id inst, SEL sel, Class cls)
{
    lockdebug::assert_unlocked(&runtimeLock.get());
    ASSERT(cls->isRealized());
    SEL resolve_sel = @selector(resolveInstanceMethod:);

    if (!lookUpImpOrNilTryCache(cls, resolve_sel, cls->ISA(/*authenticated*/true))) {
        // Resolver not implemented.
        return;					//「中文」类没实现 +resolveInstanceMethod: 就直接跳过
    }

    BOOL (*msg)(Class, SEL, SEL) = (typeof(msg))objc_msgSend;
    bool resolved = msg(cls, resolve_sel, sel);	//「中文」给元类发 +resolveInstanceMethod:（一次机会）

    // Cache the result (good or bad) so the resolver doesn't fire next time.
    // +resolveInstanceMethod adds to self a.k.a. cls
    IMP imp = lookUpImpOrNilTryCache(inst, sel, cls);	//「中文」解析后看是否真的加上了 sel

    if (resolved  &&  PrintResolving) {
        if (imp) {
            _objc_inform("RESOLVE: method %c[%s %s] "
                         "dynamically resolved to %p",
                         cls->isMetaClass() ? '+' : '-',
                         cls->nameForLogging(), sel_getName(sel), imp);
        }
        else {
            // Method resolver didn't add anything?
            _objc_inform("RESOLVE: +[%s resolveInstanceMethod:%s] returned YES"
                         ", but no new implementation of %c[%s %s] was found",
                         cls->nameForLogging(), sel_getName(sel),
                         cls->isMetaClass() ? '+' : '-',
                         cls->nameForLogging(), sel_getName(sel));
        }
    }
}

// objc-runtime-new.mm:7479  —— resolveMethod_locked（全文）
static NEVER_INLINE IMP
resolveMethod_locked(id inst, SEL sel, Class cls, int behavior)
{
    lockdebug::assert_locked(&runtimeLock.get());
    ASSERT(cls->isRealized());

    runtimeLock.unlock();			//「中文」发 OC 消息前先解锁（避免重入死锁）

    if (! cls->isMetaClass()) {
        // try [cls resolveInstanceMethod:sel]
        resolveInstanceMethod(inst, sel, cls);	//「中文」实例方法走这里
    }
    else {
        // try [nonMetaClass resolveClassMethod:sel]
        // and [cls resolveInstanceMethod:sel]
        resolveClassMethod(inst, sel, cls);
        if (!lookUpImpOrNilTryCache(inst, sel, cls)) {
            resolveInstanceMethod(inst, sel, cls);
        }
    }

    // chances are that calling the resolver have populated the cache
    // so attempt using it
    return lookUpImpOrForwardTryCache(inst, sel, cls, behavior);
    //「中文」解析后再查一遍；这次 behavior 已去掉 RESOLVER 位 → 仍无则返回 forward_imp 转发
}
```

**LLDB 实测**（Cat 未实现解析，命中 `+[NSObject resolveInstanceMethod:]` 默认返回 NO，随后**再次**回到 `lookUpImpOrForward` 仍找不到 → `forward_imp`）：

```text
# resolveInstanceMethod: 命中 NSObject 默认实现
+[NSObject resolveInstanceMethod:]:
->  mov    w0, #0x0      ; 返回 NO
    ret
(lldb) bt
* frame #0: +[NSObject resolveInstanceMethod:]
  frame #1: resolveMethod_locked(...) + 396
  frame #2: _objc_msgSend_uncached + 68
  frame #3: main at main_fwd.m:14         // [c performSelector:@selector(noSuchMethod)]

# 解析失败 → resolveMethod_locked 再查一遍，又回到 lookUpImpOrForward
(lldb) bt
* frame #0: lookUpImpOrForward
  frame #1: resolveMethod_locked(...) + 420
  frame #2: _objc_msgSend_uncached + 68
  frame #3: main at main_fwd.m:14
```

---

# 第三部分 · 消息转发

## 11. 转发入口：`_objc_msgForward_impcache`（C++ 侧 :7626 / 汇编 :788）

> 易混提示：汇编 `:779` 只是注释块开头（`* id _objc_msgForward(...)`），真正的 `STATIC_ENTRY __objc_msgForward_impcache` 在 **:788**。

C++ 侧把 `_objc_msgForward_impcache` 当作「找不到」时返回的占位 IMP（第 6 节 :7626 定义、循环里 `imp = forward_imp` 返回）；汇编侧逐字全文：

```asm
;; objc-msg-arm64.s:777  —— _objc_msgForward 相关（全文）
/********************************************************************
*
* id _objc_msgForward(id self, SEL _cmd,...);
*
* _objc_msgForward is the externally-callable
*   function returned by things like method_getImplementation().
* _objc_msgForward_impcache is the function pointer actually stored in
*   method caches.
*
********************************************************************/

	STATIC_ENTRY __objc_msgForward_impcache
	//「中文」真正存进方法缓存的转发占位 IMP

	// No stret specialization.
	b	__objc_msgForward

	END_ENTRY __objc_msgForward_impcache


	ENTRY __objc_msgForward

	adrp	x17, __objc_forward_handler@PAGE
	add		x17, x17, __objc_forward_handler@PAGEOFF
	ldr		p16, [x17]			//「中文」取全局转发处理器指针
	TailCallSignedFunctionPointer x16, x17, 0x1c18
	//「中文」尾跳进 _objc_forward_handler；带 CF 的进程里它被换成 CF 的转发实现（见第 13 节）

	END_ENTRY __objc_msgForward
```

**LLDB 实测**：慢速查找返回后确实尾跳进 `_objc_msgForward`：

```text
_objc_msgForward:
->  adrp   x17, ...           ; _objc_forward_handler
(lldb) bt
* frame #0: _objc_msgForward
  frame #1: main at main_fwd.m:14
```

## 12. 转发三部曲

三步都由 CoreFoundation 的 `___forwarding___` 串起来调用（第 13 节）。下面给出每一步**实测命中的真实实现与调用栈**。

### 12.1 快速转发 `forwardingTargetForSelector:`（换 receiver 重发）

Cat 未实现，命中 `-[NSObject forwardingTargetForSelector:]` 默认返回 nil：

```text
-[NSObject forwardingTargetForSelector:]:
->  mov    x0, #0x0     ; 返回 nil → 进入完整转发
    ret
```

### 12.2 完整转发 `methodSignatureForSelector:` + `forwardInvocation:`

返回 nil 后，`___forwarding___` 调 `methodSignatureForSelector:` 要签名（实测命中 **CoreFoundation 版** `-[NSObject(NSObject) methodSignatureForSelector:]`，注意调用方是 `___forwarding___`）：

```text
(lldb) bt
* frame #0: CoreFoundation`-[NSObject(NSObject) methodSignatureForSelector:]
  frame #1: CoreFoundation`___forwarding___ + 420
  frame #2: CoreFoundation`_CF_forwarding_prep_0 + 96
  frame #3: msgsend-fwd`main at main_fwd.m:14
```

签名为 nil（没有 `forwardInvocation:` 可走）→ 直接进入兜底。

### 12.3 兜底崩溃 `doesNotRecognizeSelector:`（`NSObject.mm:2578`）

objc4 里的实现逐字全文（注意注释——真正跑的是 CF 版本）：

```objc
// NSObject.mm:2571  —— doesNotRecognizeSelector:（全文，含类方法/实例方法两版）
// Replaced by CF (throws an NSException)
+ (void)doesNotRecognizeSelector:(SEL)sel {
    _objc_fatal("+[%s %s]: unrecognized selector sent to instance %p",
                class_getName(self), sel_getName(sel), self);
}

// Replaced by CF (throws an NSException)
- (void)doesNotRecognizeSelector:(SEL)sel {
    _objc_fatal("-[%s %s]: unrecognized selector sent to instance %p",
                object_getClassName(self), sel_getName(sel), self);
}
```

**LLDB 实测**：命中的是 **CoreFoundation 版** `-[NSObject(NSObject) doesNotRecognizeSelector:]`，仍由 `___forwarding___` 调起；它 `throw` 一个 `NSInvalidArgumentException`（而非 objc4 这里的 `_objc_fatal`）：

```text
(lldb) bt
* frame #0: CoreFoundation`-[NSObject(NSObject) doesNotRecognizeSelector:]
  frame #1: CoreFoundation`___forwarding___ + 1480
  frame #2: CoreFoundation`_CF_forwarding_prep_0 + 96
  frame #3: msgsend-fwd`main at main_fwd.m:14

*** Terminating app due to uncaught exception 'NSInvalidArgumentException',
    reason: '-[Cat noSuchMethod]: unrecognized selector sent to instance 0x...'
    0   CoreFoundation        __exceptionPreprocess + 176
    1   libobjc.A.dylib       objc_exception_throw + 88
    3   CoreFoundation        ___forwarding___ + 1480
    4   CoreFoundation        _CF_forwarding_prep_0 + 96
    5   msgsend-fwd           main + 84
→ stop reason = signal SIGABRT
```

## 13. 边界澄清：objc4 只到 `_objc_msgForward`，三部曲调度在 CoreFoundation `___forwarding___`（闭源）

实测证据链把边界划得很清楚：

- libobjc 只负责到 `_objc_msgForward` → 取 `_objc_forward_handler` 尾跳走（汇编 :796），**不含**三部曲逻辑；
- `forwardingTargetForSelector:` / `methodSignatureForSelector:` / `doesNotRecognizeSelector:` 的调用方全是 `CoreFoundation`___forwarding___`（闭源），且后两者命中的是 **CF 重写版** `-[NSObject(NSObject) …]`，正好印证 `NSObject.mm` 里 `// Replaced by CF (throws an NSException)`；
- 所以「OC 抛 `NSInvalidArgumentException`」是 CF 干的，objc4 自己的 `doesNotRecognizeSelector:` 只会 `_objc_fatal`（纯 libobjc 进程才走得到）。

---

# 第四部分 · 实战与收尾

## 14. LLDB 实战

环境与下断点（系统 libobjc，关 ASLR）：

```lldb
settings set target.disable-aslr true
# 正常发送：慢速查找仅在 sel==bark 时停（过滤 new/alloc 噪声）
breakpoint set --name lookUpImpOrForward \
    --condition '(int)strcmp((char*)sel_getName($x1), "bark") == 0'
# 转发链各站
breakpoint set --selector resolveInstanceMethod:
breakpoint set --selector forwardingTargetForSelector:
breakpoint set --selector methodSignatureForSelector:
breakpoint set --selector doesNotRecognizeSelector:
breakpoint set --name _objc_msgForward
breakpoint set --name ___forwarding___
```

- **cache 从空到有 / 命中对比**：见第 9 节——`[d bark]` 第一次进 `lookUpImpOrForward`，第二次不再进。
- **慢速断点**：`p (char*)sel_getName($x1)`、`po (Class)$x2` 读出 receiver/SEL/cls。
- **转发链逐站拦截**：见第 10–12 节真实调用栈，完整顺序为：

```text
lookUpImpOrForward(miss)
  → resolveMethod_locked → +[NSObject resolveInstanceMethod:] = NO
  → lookUpImpOrForward(再 miss) → forward_imp
  → _objc_msgForward → CF ___forwarding___
      → -[NSObject forwardingTargetForSelector:] = nil
      → -[NSObject(NSObject) methodSignatureForSelector:] = nil
      → -[NSObject(NSObject) doesNotRecognizeSelector:] → throw → SIGABRT
```

## 15. 收尾

一条消息的命运：`objc_msgSend`（汇编 cache 命中→尾跳）→ miss 则 `lookUpImpOrForward`（当前类方法表→沿 superclass 链，命中即 `insert` 回填缓存）→ 仍无则动态解析一次 → 再无则 `_objc_msgForward` 交给 CF `___forwarding___` 跑转发三部曲 → 全不接则 `doesNotRecognizeSelector:` 抛异常崩溃。

与 Part 1 的呼应：第 3.1 节取 isa→class 用的 **ISA_MASK `0x7ffffffffffff8`**、bucket 里 IMP 的 ptrauth，都接着 Part 1 的 isa 走位与指针签名往下讲。

承接下一篇：缓存结构（`cache_t` 融合字、扩容丢弃策略）与方法列表（`method_t` small/big、相对偏移、分类覆盖）值得单开一篇细挖；以及 `super` 调用走的 `objc_msgSendSuper2`（本篇汇编 :715 已露面）如何改查找起点。
