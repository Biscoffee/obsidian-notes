# Objective-C Internals: Retain

> 原文：https://alwaysprocessing.blog/2023/07/22/objc-retain
> 发布：2023-07-22　作者：Brian T. Kelley
> 译者：MiMo；代码块保留英文原样　|　仅供个人学习，未公开发布

---

Objective-C 的内存管理依赖于一种引用计数（reference counting）机制，它从一个相对简单的 API 演进为复杂且高度优化的实现，同时保持了源代码和 ABI 的兼容性。

**背景**

OS X 10.7 和 iOS 5 引入了自动引用计数（Automatic Reference Counting，简称 ARC），旨在通过消除样板代码并减少引用计数错误（泄漏和过度释放）的发生面，来提升 Objective-C 程序员的生产效率。

在 ARC 之前，`-[NSObject retain]`、`-[NSObject release]`[1] 和 `-[NSObject autorelease]`[2] 方法是管理对象引用计数的唯一接口。并且，直到 OS X 10.8 和 iOS 6，`NSObject` 的实现一直是 Foundation 框架的一部分，而非 Objective-C 运行时（runtime）本身。

ARC 的设计者们借鉴了苹果公司此前尝试为 Objective-C 添加垃圾回收（garbage collection）失败的经验教训，识别出一项关键需求以提高该功能成功的可能性：自动引用计数必须在同一进程中与手动引用计数透明地互操作，而无需重新编译现有代码（例如，仅提供二进制文件的第三方库）。

在 macOS 早期，某些对象覆写引用计数方法[3]以使用自定义实现（通常出于性能考虑）并不罕见。ARC 必须支持与这些自定义引用计数实现的透明互操作，才能满足上述要求。

**入口点**

引用计数操作有两个接口：历史悠久的 NSObject API 和 ARC 使用的编译器私有 API，两者都会调用核心实现。接下来两个小节将分别探讨每个接口的保留（retain）实现，下一节将讨论核心实现。

**NSObject**

`-[NSObject retain]` 的实现[4]非常简单——它仅调用 `_objc_rootRetain` 来保留 `self`。

```
- (id)retain {
  return _objc_rootRetain(self);
}
```

“root”这一术语表示对象所属类层次结构中的根类（root class）收到了 -retain 消息。因此，该类并未重写 -retain，或者其重写实现调用了父类方法，从而保证了保留操作（retain）必定使用运行时的实现。（我们将在下一节中看到，并非所有入口点都具备这种保证。）

接下来，同样是简单封装的 `_objc_rootRetain` 函数会调用 `objc_object::rootRetain()`。

```
id _objc_rootRetain(id obj) {
  ASSERT(obj);
  return obj->rootRetain();
}
```

这个函数的存在是一个历史遗留产物。在自动引用计数（Automatic Reference Counting, ARC）的最初实现中，此函数即为retain的实现，但它在后续版本的多次重构中得以保留，而这些重构已使其变得多余。该函数的唯一其他调用者是遗留的`Object`类（其以Objective-C++实现），因此可以直接调用`objc_object::rootRetain()`。

最后，`objc_object::rootRetain()`会调用一个重载版本的`rootRetain`。

```
id objc_object::rootRetain() {
  return rootRetain(false, RRVariant::Fast);
}
```

此处调用的重载是核心实现，具有两个参数：

`tryRetain` 启用加载弱引用的支持[5]。参数值为 `false`，因为弱引用（weak reference）无法执行此代码路径。（运行时必须首先从弱引用加载对象，之后对象才能接收消息；并且根据定义，通过加载操作获取的对象引用是强引用。）

`variant` 提供调用路径的上下文信息，使核心实现能够省略不必要的操作。通过 `NSObject` 执行的保留（retain）操作使用 `RRVariant::Fast`，以跳过对类是否具有自定义引用计数实现的检查，因为根据定义，通过根类执行的操作并非自定义实现。

**自动引用计数（Automatic Reference Counting）**

当启用 ARC 时，编译器通过一项为 ARC 新增的编译器私有 API 执行引用计数操作，以此作为性能优化。该 API 允许引用计数操作直接调用 Objective-C 运行时，从而跳过发送消息（send a message）的开销。

```
id objc_retain(id obj) {
  if (_objc_isTaggedPointerOrNil(obj)) return obj;
  return obj->retain();
}
```

该函数首先检查对象指针值，如果它不引用堆上的对象，则立即返回，这种情况可能发生在两种场景中：

- 指针为 `nil`。向 `nil` 发送消息是合法的，因此 `-[NSObject retain]` 的这一优化也必须支持 `nil` 指针。
- 指针是一个标记指针（tagged pointer）。标记指针是 Objective-C 运行时的一种实现细节，对编译器不可见，因此编译器无法消除保留操作。标记指针不参与引用计数（无需跟踪堆分配），因此无需继续处理。

如果对象指针值引用了堆上的对象，该函数将调用 `objc_object::retain()` 来执行保留操作。

```
inline id objc_object::retain() {
  ASSERT(!isTaggedPointer());
  return rootRetain(false, RRVariant::FastOrMsgSend);
}
```

此函数以以下参数调用核心实现（尽管此时 `rootRetain` 中的 `root` 命名已属误称）：

`tryRetain` 参数为 `false`，与上述 `NSObject` 入口点中讨论的原因相同。

`variant` 参数为 `RRVariant::FastOrMsgSend`。此前尚未进行任何内省（introspection）——无论是直接（见下文 `rootRetain` 说明）还是间接（通过消息发送，见上文 `NSObject` 说明）——因此尚不清楚该对象的类是否重写了任何引用计数方法（这也正是函数名称中不含 `root` 术语的原因）。

变体中的 `MsgSend` 部分指示核心实现执行必要的内省，以确定该对象的类是否重写了引用计数方法。若确实如此，核心实现将通过向对象发送 `-retain` 消息（可能通过 `-[NSObject retain]` 重新进入运行时）来执行保留（retain）操作。

**rootRetain**

`objc_object::rootRetain(bool, RRVariant)` 函数篇幅较长，因此我们将分段进行分析。

```
if (slowpath(isTaggedPointer())) return (id)this;
```

尽管ARC的入口点会检查标记指针（tagged pointer），但NSObject的入口点并未进行此检查。我不太清楚为何NSObject的实现不执行这项检查，但标记指针的检查必然会在某处进行，在这个版本的运行时中，它就在这里。

接下来，运行时会加载对象的isa值（实例变量）。

```
bool sideTableLocked = false;
bool transcribeToSideTable = false;
isa_t oldisa = LoadExclusive(&isa().bits);
isa_t newisa;
```

在所有现代苹果平台上，isa 存储着对象的保留计数（retain count）。在 arm64 架构上，Objective-C 运行时使用 ARM 的独占监视器同步原语（exclusive monitor synchronization primitive）来管理并发，这也正是 `LoadExclusive` 函数名称的由来。在包括 arm64e 在内的所有其他架构上，Objective-C 运行时则使用 C11 原子操作（C11 atomics）。（我不确定是 arm64 还是 arm64e 属于特例，也不知道原因。）

如果编译器私有 API 是 retain 操作的入口点，那么运行时必须检查该类是否重写了任何引用计数方法（reference counting methods）。

```
if (variant == RRVariant::FastOrMsgSend) {
  // These checks are only meaningful for objc_retain()
  // They are here so that we avoid a re-load of the isa.
  if (slowpath(oldisa.getDecodedClass(false)->hasCustomRR())) {
    ClearExclusive(&isa().bits);
    if (oldisa.getDecodedClass(false)->canCallSwiftRR()) {
      return swiftRetain.load(memory_order_relaxed)((id)this);
    }
    return ((id(*)(objc_object *, SEL))objc_msgSend)(this, @selector(retain));
  }
}
```

自定义引用计数实现较为罕见，因此运行时通过其 `slowpath()` 宏提示 CPU 分支预测单元该路径不太可能执行。`getDecodedClass()` 返回对象的 Class 对象，该对象有一个标志指示该类是否重写了任何引用计数方法。这一快速检查为 ARC 入口点提供了必要的类内省，以最小开销支持自定义引用计数实现。

`getDecodedClass()` 中的 "decoded" 一词可能指从非指针 isa 中提取类对象指针。该函数的具体实现取决于目标架构：

arm64_32：Apple Watch 的 ABI 使用 32 位指针，因此其非指针 isa 存储的是一个指向存储类对象的表的索引（因为位数不足以在指针值中存储额外数据）。如果 isa 不是指针，函数会调用 `classForIndex()` 从表中获取类对象。

否则，如果 isa 是指针，它就是类对象的指针，因此函数直接返回 isa 位。

在所有其他目标架构上，该函数是 `getClass()` 的别名，它通过从 `isa` 值中屏蔽非指针位来返回类对象。

arm64e：如果启用了 `isa` 指针认证，该函数将使用编译器计算的掩码提取类对象指针值。然而，由于调用者传递了 `false` 给 `authenticate` 参数，该函数会跳过认证。文档中说明不进行认证的理由是：认证作为 `objc_msgSend` 的一部分已经发生，因此不需要额外的认证。

否则，该函数将使用静态掩码定义来提取类对象指针值。

如果一个类有自定义的引用计数实现，运行时会向对象发送一条 `-retain` 消息以满足由 ARC 发起的保留操作。注意，该对象随后可能会调用 `-[NSObject retain]`，但此代码块将不会再次执行，因为变体将是 `RRVariant::Fast`。

纯 Swift 类（即不继承自 `NSObject` 的类）通过继承 `SwiftObject` 类（仅限于 Apple 平台）实现与 Objective-C 的兼容性。Swift 使用自身的引用计数系统（reference counting system），因此 `SwiftObject` 实现了相关引用计数方法，以支持将纯 Swift 对象桥接（bridging）至 Objective-C。针对此场景的优化处理中，Objective-C 运行时（runtime）会直接调用 Swift 运行时的 `swift_retain()` 函数[6]（而非通过消息发送来保留对象）。

```
if (slowpath(!oldisa.nonpointer)) {
  // a Class is a Class forever, so we can perform this check once
  // outside of the CAS loop
  if (oldisa.getDecodedClass(false)->isMetaClass()) {
    ClearExclusive(&isa().bits);
    return (id)this;
  }
}
```

Class objects（类对象）永远不会被释放，也不需要引用计数。因此，如果对象是类对象，函数会直接返回它而不执行任何进一步的工作。

**比较并交换循环**

比较并交换循环是 retain（保留）实现的核心。它从（重新）初始化循环的开始状态开始。

```
do {
  transcribeToSideTable = false;
  newisa = oldisa;
```

它将 `newisa` 设置为当前的 `isa` 值（即 `oldisa`），该值将在循环中更新以反映递增后的引用计数（retain count）。接下来的小节将探讨 `transcribeToSideTable` 的使用。

```
  if (slowpath(!newisa.nonpointer)) {
    ClearExclusive(&isa().bits);
    if (tryRetain) return sidetable_tryRetain() ? (id)this : nil;
    else return sidetable_retain(sideTableLocked);
  }
```

首先，循环检查对象实例是否具有非指针型isa（non-pointer isa）。如果不符合，则将保留计数记录在旁路表（side table）中。该检查在循环内执行，是因为如果当前线程在比较并交换（compare-and-swap）操作中失败，可能是由于另一个线程正在以某种方式修改对象，使其不再使用非指针型isa。

接下来，循环检查是否又输掉了一次竞态。

```
  // don't check newisa.fast_rr; we already called any RR overrides
  if (slowpath(newisa.isDeallocating())) {
    ClearExclusive(&isa().bits);
    if (sideTableLocked) {
      ASSERT(variant == RRVariant::Full);
      sidetable_unlock();
    }
    if (slowpath(tryRetain)) {
      return nil;
    } else {
      return (id)this;
    }
  }
```

一个对象可能正在被释放，而此时一个线程正试图保留（retain）它，这种情况（至少）出现在以下三种场景中：

当 `tryRetain` 为 true，且此线程在弱引用对象开始释放之前，未能成功加载该弱引用对象。函数返回 nil，表示它无法获得一个强引用。在此场景中，调用者 `objc_loadWeakRetained()` 持有指向弱引用侧表（weak reference side table）的锁，这防止了对象被释放，因此从对象指针读取 isa 是定义明确的行为。

另一个线程释放了该对象，导致其被释放，这通常是因为进程并发读写一个强（strong）、非原子（nonatomic）属性时发生竞争条件（race condition）所致。此场景中的所有情况都是未定义行为（undefined behavior）。函数返回 self 以履行 `-retain` 协议，但它将成为一个悬垂指针（dangling pointer），几乎肯定会导致该线程在不久的将来崩溃。在竞争条件下命中此代码路径是“幸运的”。实际上，通过悬垂指针读取的 isa 位（isa bits）可能会将此函数导向任意多种路径，从而导致不可预测的后果。（译注：上述描述基于特定运行时版本，现代系统可能已变化）

在 `-dealloc` 中的逻辑会导致执行一个 retain（保留）（例如，`-dealloc` 的实现将 self 传递给一个清理例程，其中 ARC（自动引用计数）编译器会发出一个 retain/release 对）。这种情况不像上面两种情况是竞态条件（race condition），因为 retain 发生在执行销毁操作的同一线程上。然而，如果执行 retain 的函数要求对象实例在其调用范围后继续存在（例如，将 self 存储在另一个对象的强属性中），这种情况可能导致未定义行为（undefined behavior），因为当销毁完成时，指针（pointer）将变为悬垂指针（dangling）。最后，我们进入实际的递增部分。

```
  uintptr_t carry;
  newisa.bits = addc(newisa.bits, RC_ONE, 0, &carry);  // extra_rc++
```

回想一下，non-pointer isa（非指针 isa）是一个具有三种变体的位域。RC_ONE 的值是当将位域视为整数时表示保留计数（retain count）为一的位。保留计数存储在 isa 的最高有效位中，因此如果所有保留计数位都被使用，将发生溢出或进位（将在下一小节讨论）。如果没有发生溢出，newisa 包含递增的保留计数，并准备写回对象实例。

```
} while (slowpath(!StoreExclusive(&isa().bits, &oldisa.bits, newisa.bits)));
```

如果 `&isa()` 处的值与 `&oldisa` 处的值匹配，那么比较并交换（compare-and-swap）操作就会成功，并将 `newisa` 的值写入 `&isa()`，循环随之结束。

否则，说明自该线程将 `&isa()` 的值加载到 `oldisa` 以来，该值已经发生了变化。比较并交换操作失败，并将 `&isa()` 处的新值写入 `&oldisa`。循环会继续执行，直到该线程成功执行一次比较并交换操作，或者另一个线程更改了对象状态，从而触发上述某个返回路径。

**完整变体**

如果引用计数（retain count）溢出了非指针形式的 isa（non-pointer isa）所能容纳的位数，运行时会使用一个旁路表（side table）来存储部分引用计数。

```
  if (slowpath(carry)) {
    // newisa.extra_rc++ overflowed
    if (variant != RRVariant::Full) {
      ClearExclusive(&isa().bits);
      return rootRetain_overflow(tryRetain);
    }
    // Leave half of the retain counts inline and
    // prepare to copy the other half to the side table.
    if (!tryRetain && !sideTableLocked) sidetable_lock();
    sideTableLocked = true;
    transcribeToSideTable = true;
    newisa.extra_rc = RC_HALF;
    newisa.has_sidetable_rc = true;
  }
```

如果此函数调用使用的是 `Fast` 或 `FastOrMsgSend` 变体，它将中止保留操作的尝试，并将任务转交给 `rootRetain_overflow()`。

```
NEVER_INLINE id objc_object::rootRetain_overflow(bool tryRetain) {
  return rootRetain(tryRetain, RRVariant::Full);
}
```

我推测此函数的目的在于为堆栈跟踪提供一个帧，以协助苹果工程师诊断运行时中的保留计数崩溃（retain crash）——由于涉及旁表锁（side table locking）（其中使用了非可重入自旋锁（non-reentrant spin lock））的交互可能难以推导分析。

当根保留（rootRetain）操作中使用 Full 变体导致保留计数溢出时，其实现会将保留计数值的一半发送到旁表，并将另一半保留在非指针 isa（non-pointer isa）中。将保留计数减半是为了最小化旁表访问次数（从而减少所需的 CPU 指令数和锁获取次数）。如果实现仅将溢出位发送到旁表，那么在溢出边界值附近的引用计数操作可能会成为系统的性能瓶颈。

```
  if (variant == RRVariant::Full) {
    if (slowpath(transcribeToSideTable)) {
      // Copy the other half of the retain counts to the side table.
      sidetable_addExtraRC_nolock(RC_HALF);
    }
    if (slowpath(!tryRetain && sideTableLocked)) sidetable_unlock();
  }
```

在比较并交换（compare-and-swap）成功后，侧表（side table）会被更新，因为另一个线程可能赢得将溢出保留计数（overflow retain count）移动到侧表的竞态。比较并交换循环会获取侧表的锁，这防止了如果另一个线程也尝试读取或写入侧表时发生的竞争条件（race condition）。

**返回 self**

在比较并交换成功并且（如有必要）侧表更新后，保留操作（retain operation）就完成了。

```
return (id)this;
```

最后一步是返回 `self[8]` 以履行 `-[NSObject retain]` 约定：

作为便利，retain 会返回 self，因为它可能用于嵌套表达式中。

自从 20 多年前 Mac OS X 首次发布以来，retain 已经演进为一个高度优化的操作。release、autorelease 和 dealloc（我们稍后会看到）也同样如此。