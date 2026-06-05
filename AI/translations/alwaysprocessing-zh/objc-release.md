# Objective-C Internals: Release

> 原文：https://alwaysprocessing.blog/2023/10/01/objc-release
> 发布：2023-10-01　作者：Brian T. Kelley
> 译者：MiMo；代码块保留英文原样　|　仅供个人学习，未公开发布

---

尽管 release "仅仅"是 retain（保留）的逻辑逆操作，但它的实现要复杂得多，这主要是因为 ARM（精简指令集架构）同步模型。本文将探讨 release 实现相对于 retain 的独特之处，重点关注 ARM 上的内存顺序要求。

Objective-C 使用引用计数（reference counting）方法来管理内存。本文将关注 release 操作，该操作移除对象实例上的一个引用计数。上一篇文章介绍了 retain 的实现，该操作向对象实例添加一个引用计数。retain 和 release 的实现是相似的，因为它们是逆操作。我会参考之前的 retain 帧文章，其中的讨论是类似的，因此本文可以专注于 release 的独特方面。

**入口点**

引用计数操作有两个接口：长期存在的 NSObject API 和编译器内部供 ARC（自动引用计数）使用的私有 API，它们都调用一个核心实现。以下两个小节将分别考察每个接口的 release 实现，下一节将讨论核心实现。

**NSObject**

与 `-[NSObject retain]` 一样，`-[NSObject release]` 方法也十分简单——它直接调用 `_objc_rootRelease()` 来释放 `self`。

```
- (void)release {
  _objc_rootRelease(self);
}
```

术语 "root"，正如在 retain（保留）帖子中讨论的那样，表示操作是通过对象类层次结构中的根类接收到的 -release 消息发生的。

接下来，_objc_rootRelease 函数，这也非常简单，调用 `objc_object::rootRelease()`。

```
void _objc_rootRelease(id obj) {
  ASSERT(obj);
  obj->rootRelease();
}
```

最后，`objc_object::rootRelease()` 调用了 `rootRelease` 的一个重载版本。

```
bool objc_object::rootRelease() {
  return rootRelease(true, RRVariant::Fast);
}
```

此处调用的重载是核心实现，它有两个参数：

`performDealloc` 指定当引用计数降为零时，释放操作是否应释放对象实例。除非是由 `_objc_rootReleaseWasZero()` SPI[1]（译注：系统编程接口，供内部使用，区别于供第三方使用的应用程序编程接口）执行释放，否则运行时始终为此参数传递 `true`。

`variant` 提供有关调用路径的上下文信息，使核心实现能够省略不必要的工作。通过 `NSObject` 执行的释放使用 `RRVariant::Fast`，以跳过检查该类是否具有自定义引用计数实现，因为通过根类发生的操作，根据定义，不是自定义的。

**自动引用计数**

当启用 ARC（Automatic Reference Counting，自动引用计数）时，编译器通过一个为 ARC 优化的性能而添加的私有编译器 API（在保留操作一文中也有讨论）来执行引用计数操作。

```
void objc_release(id obj) {
  if (_objc_isTaggedPointerOrNil(obj)) return;
  return obj->release();
}
```

如果对象指针值引用堆上的对象，并通过与保留（retain）相同的机制派生得出，该函数会调用 `objc_object::release()` 来执行释放操作。

```
inline void objc_object::release() {
  ASSERT(!isTaggedPointer());
  rootRelease(true, RRVariant::FastOrMsgSend);
}
```

**rootRelease**

该函数调用核心实现（尽管 `rootRelease` 中的 `root` 在此处用词不准确），参数如下：

`performDealloc` 参数设为 `true`，原因与上文在 NSObject 入口点中讨论的相同。

`variant` 参数设为 `RRVariant::FastOrMsgSend`。此时尚未进行任何自省（introspection）（无论是直接的自省——见下文 `rootRelease` 说明，还是间接的——通过消息发送，见上文 NSObject 说明），因此尚不清楚该对象的类是否重写了任何引用计数方法（reference counting methods）（因此函数名中不包含 `root` 一词）。

`variant` 的 `MsgSend` 部分指示核心实现执行必要的自省，以判断该对象的类是否重写了引用计数方法。如果重写了，核心实现将通过向该对象发送一条 `-release` 消息来执行释放操作（该消息可能会通过 `-[NSObject release]` 重新进入运行时）。

---

`objc_object::rootRelease(bool, RRVariant)` 函数规模较大，因此我们将分块进行分析。

```
if (slowpath(isTaggedPointer())) return (id)this;
```

尽管 ARC 入口点会检查标记指针（tagged pointer），但 NSObject 入口点并不执行此检查。我一时看不出 NSObject 实现为何省略这项检查，不过这项检查肯定会在某处执行，而在当前这个版本的运行时中，它就在此处。

接下来，运行时会加载对象的 isa 值[2]。

```
bool sideTableLocked = false;
isa_t newisa, oldisa;
oldisa = LoadExclusive(&isa().bits);
```

如果编译器私有 API 是释放操作的入口点，运行时则必须检查该类是否重写了任何引用计数方法[3]。

```
if (variant == RRVariant::FastOrMsgSend) {
  // These checks are only meaningful for objc_release()
  // They are here so that we avoid a re-load of the isa.
  if (slowpath(oldisa.getDecodedClass(false)->hasCustomRR())) {
    ClearExclusive(&isa().bits);
    if (oldisa.getDecodedClass(false)->canCallSwiftRR()) {
      swiftRelease.load(memory_order_relaxed)((id)this);
      return true;
    }
    ((void(*)(objc_object *, SEL))objc_msgSend)(this, @selector(release));
    return true;
  }
}
```

如果一个类有自定义的引用计数（reference counting）实现，运行时（runtime）会向该对象发送 `-release` 消息来完成 ARC 发起的释放操作。请注意，该对象随后可能调用 `-[NSObject release]`，但此代码块不会再次执行，因为变体（variant）会是 `RRVariant::Fast`。

消息发送变体中返回 `true` 是一种防御性设计。一个类实现自己的引用计数机制却调用 `_objc_rootRootReleaseWasZero()` 私有接口（SPI，Private Interface）——这是唯一使用返回值的入口点——这本身就不合理。如果一个类拥有自己的引用计数机制，那么根据定义，它必然知道引用计数何时归零。

假设一个类实现将其自身的引用计数机制与该私有接口结合使用。那么，在该类的实例首次被释放后不久，进程几乎必定会崩溃——因为当私有接口返回 `true` 时，该类实现会释放该实例，导致所有先前保留（retain）了该实例的其他对象持有一个悬垂指针（dangling pointer）。

继续看下一个代码块。

```
if (slowpath(!oldisa.nonpointer)) {
  // a Class is a Class forever, so we can perform this check once
  // outside of the CAS loop
  if (oldisa.getDecodedClass(false)->isMetaClass()) {
    ClearExclusive(&isa().bits);
    return false;
  }
}
```

类对象永远不会被释放，也不需要引用计数。因此，如果对象是类对象，该函数会直接返回它，不执行进一步操作。

**比较并交换循环**

比较并交换循环（compare-and-swap loop）是 release 实现的核心。它以一个可能出人意料的 `goto` 标签开始。下面的「完整变体」小节将讨论该函数中 `goto` 的使用。

```
retry:
do {
  newisa = oldisa;
```

循环首先将 `newisa` 设置为当前的 `isa` 值（即 `oldisa`），后续步骤将更新此值以反映递减后的保留计数（retain count）。

接着，循环检查该对象实例是否具有**非标记指针isa**（non-pointer isa）[4]。如果不是，则保留计数将被记录在辅助表（side table）中。此检查在循环中执行，是因为如果当前线程在比较并交换操作（compare-and-swap）中失败，可能是由于另一线程以某种方式修改了对象，使其不再使用非标记指针isa。

```
  if (slowpath(!newisa.nonpointer)) {
    ClearExclusive(&isa().bits);
    if (tryRetain) return sidetable_tryRetain() ? (id)this : nil;
    else return sidetable_retain(sideTableLocked);
  }
```

接下来，循环会检查是否再次竞争失败。

```
  if (slowpath(newisa.isDeallocating())) {
    ClearExclusive(&isa().bits);
    if (sideTableLocked) {
      ASSERT(variant == RRVariant::Full);
      sidetable_unlock();
    }
    return false;
  }
```

当线程试图释放一个对象时，该对象可能正在析构——这至少发生在以下两种场景中：

1.  另一个线程释放了该对象，导致它被析构，这通常是由于进程并发地读写一个**强引用、非原子属性**（strong, nonatomic property）时发生的竞争条件（race condition）所致。此场景中的所有行为都是**未定义行为**（undefined behavior）。

2.  `-dealloc` 方法中的逻辑触发了一次释放（例如，`-dealloc` 的实现将 `self` 传递给了一个清理例程，而 ARC 编译器在此例程中插入了一对 retain/release 操作）。此场景并非像上面那种情况那样的竞争条件，因为释放操作发生在执行析构的同一个线程上。

如果 `_objc_rootReleaseWasZero()` 这个 SPI 执行了释放操作，那么返回值为 `false` 表示调用方不应再启动析构过程，因为该对象已经在析构中了。除此之外，`NSObject` 和 ARC 的入口点并未使用该返回值。

最后，我们来进行实际的递减操作。

```
  // don't check newisa.fast_rr; we already called any RR overrides
  uintptr_t carry;
  newisa.bits = subc(newisa.bits, RC_ONE, 0, &carry);  // extra_rc--
  if (slowpath(carry)) {
    // don't ClearExclusive()
    goto underflow;
  }
```

回顾一下，non-pointer isa（非指针形式的 isa）是一个包含三种变体的位域。当将这个位域视为整数时，`RC_ONE` 的值是表示保留计数（retain count）为一的那个位。保留计数存储在 isa 的最高有效位中，因此如果所有保留计数位都为零，就会发生下溢或进位（在下面的小节中讨论）。否则，如果没有发生下溢，`newisa` 就包含递减后的保留计数，并准备好写回对象实例。

```
} while (slowpath(!StoreReleaseExclusive(&isa().bits, &oldisa.bits, newisa.bits)));
```

如果 `&isa()` 处的值与 `&oldisa` 中的值匹配，比较并交换（compare-and-swap）操作就会成功，并将 `newisa` 的值写入 `&isa()`，循环随之结束。

否则，说明此线程将值加载到 `oldisa` 后，`&isa()` 处的值已经发生了变化。此时比较并交换操作失败，并会将 `&isa()` 处的新值写入 `&oldisa`。循环将继续，直到该线程成功执行一次比较并交换操作，或者另一个线程更改了对象状态从而触发上述某个返回路径。

循环结束后，运行时（runtime）会检查保留计数（retain count）是否为零。如果是，则会释放（deallocate）该对象实例。

```
  if (slowpath(newisa.isDeallocating()))
    goto deallocate;
```

否则，该对象具有正的保留计数。如有必要，运行时将释放副作用表锁。函数最终返回 `false`，向 `_objc_rootReleaseWasZero()` SPI（系统编程接口）表明该对象不应被解除分配。

```
  if (variant == RRVariant::Full) {
    if (slowpath(sideTableLocked)) sidetable_unlock();
  } else {
    ASSERT(!sideTableLocked);
  }
  return false;
```

完整变体

如果保留计数（retain count）下溢至非指针 isa（non-pointer isa）的位数，运行时（runtime）会将对 newisa 的更改回滚。接着，它会检查是否有任何保留计数先前溢出至侧表（side table）。

```
underflow:
// newisa.extra_rc-- underflowed: borrow from side table or deallocate
newisa = oldisa; // abandon newisa to undo the decrement

if (slowpath(newisa.has_sidetable_rc)) {
```

若无保留计数溢出至旁侧表（side table），则无剩余保留计数，此时释放操作将解除分配对象实例——尽管实践中我认为这种情况不会发生，因为其出现意味着存在过度释放。要么存在旁侧表中的保留计数，要么保留计数已归零并通过上述代码路径解除了对象实例的分配。在我看来，若运行时在此情形下触发陷阱处理会更合理，因为当 `-dealloc` 被第二次调用时进程很可能崩溃。

若保留计数先前确实溢出至旁侧表，运行时会检查本次函数调用是否携带 `Fast` 或 `FastOrMsgSend` 变体标识。若是，则终止本次释放操作尝试，将任务转交至 `rootRelease_underflow()` 处理。

```
  if (variant != RRVariant::Full) {
    ClearExclusive(&isa().bits);
    return rootRelease_underflow(performDealloc);
  }
```

该函数立即以完整变体（Full variant）回调 `objc_object::rootRelease(bool, RRVariant)`。

```
NEVER_INLINE uintptr_t objc_object::rootRelease_underflow(bool performDealloc) {
  return rootRelease(performDealloc, RRVariant::Full);
}
```

我在保留帖子中推测，该函数的目的可能是为了在堆栈跟踪中提供一个帧，以帮助Apple工程师排查运行时中的释放崩溃问题，因为侧表锁定（使用不可重入自旋锁（non-reentrant spin lock））的相互作用在推理上颇具挑战性。

当使用 Full 变体（译注：此处可能指某种配置模式或枚举值）且释放计数递减发生下溢时，运行时会获取侧表锁。

```
  // Transfer retain count from side table to inline storage.
  if (!sideTableLocked) {
    ClearExclusive(&isa().bits);
    sidetable_lock();
    sideTableLocked = true;
    // Need to start over to avoid a race against the nonpointer -> raw pointer transition.
    oldisa = LoadExclusive(&isa().bits);
    goto retry;
  }
```

获取侧表锁（side table lock）可能导致线程挂起，因此运行时首先会移除其对 isa 地址的独占监视器（exclusive monitor），这是正确使用独占监视器所必需的。引自《ARM 架构参考手册》（着重号为原文所加）：

> 独占机制支持每个执行的处理器线程有一个未完成的独占访问。……如果 STREX（存储独占）指令的目标地址与同一线程中先前的 LDREX（加载独占）指令的目标地址不同，行为可能不可预测。因此，只有当 LDREX/STREX 指令对以相同地址执行时，才能最终依赖其成功。在可能发生上下文切换……从而改变执行线程的情况下，必须执行 CLREX 指令……以避免非预期的效果……

在获得侧表锁后，运行时会重新加载 isa 值，并再次启动比较-交换循环（compare-and-swap loop）来执行递减操作。必须重新加载 isa，因为在当前线程等待获取侧表锁期间，其他线程可能已经修改了该 isa 值。

最后，如果这次递减操作再次导致下溢（underflow），那么运行时从侧表中加载任何额外的保留计数（retain counts）就是安全的。

```
  // Try to remove some retain counts from the side table.
  auto borrow = sidetable_subExtraRC_nolock(RC_HALF);
```

`sidetable_subExtraRC_nolock()` 返回一个 `SidetableBorrow` 结构体（这里的 borrow 指的是在减法运算中取高位数值的含义，而不是从侧表租赁值），该结构体有两个字段：

`borrowed`：从侧表中取出的 retain counts（引用计数）的数量。

`remaining`：侧表中剩余的 retain counts 的数量。

运行时首先检查是否所有 retain counts 都已从侧表中移除，以便后续执行额外的簿记。然后，它检查侧表是否返回了任何 retain counts。如果侧表为空，没有剩余的 retain counts，因此释放操作将解分配对象实例。

```
  bool emptySideTable = borrow.remaining == 0; // we'll clear the side table if no refcounts remain there

  if (borrow.borrowed > 0) {
```

如果侧边表（side table）返回了对象实例的保留计数（retain count），运行时（runtime）会尝试用侧边表中的保留计数更新非指针型 isa（non-pointer isa）。

```
    // Side table retain count decreased.
    // Try to add them to the inline count.
    bool didTransitionToDeallocating = false;
    newisa.extra_rc = borrow.borrowed - 1;  // redo the original decrement too
    newisa.has_sidetable_rc = !emptySideTable;

    bool stored = StoreReleaseExclusive(&isa().bits, &oldisa.bits, newisa.bits);
```

`borrow.borrowed` 字段包含从旁表（side table）获取的引用计数（retain count）。运行时从此计数中减一（请记住，这是下溢的代码路径，因此释放记账尚未发生），并将该值存储在非指针型 isa 的 `extra_rc` 字段中。随后，它更新 `has_sidetable_rc` 标志位，以反映旁表是否仍持有该对象实例的溢出引用计数。

接着，它尝试存储新的 isa 值。此存储操作可能会失败，后续代码块将处理这种情况。

```
    if (!stored && oldisa.nonpointer) {
      // Inline update failed.
      // Try it again right now. This prevents livelock on LL/SC architectures
      // where the side table access itself may have dropped the reservation.
      uintptr_t overflow;
      newisa.bits = addc(oldisa.bits, RC_ONE * (borrow.borrowed-1), 0, &overflow);
      newisa.has_sidetable_rc = !emptySideTable;
      if (!overflow) {
        stored = StoreReleaseExclusive(&isa().bits, &oldisa.bits, newisa.bits);
        if (stored) {
          didTransitionToDeallocating = newisa.isDeallocating();
        }
      }
    }
  }
```

如果将从side table（侧表）中取出的retain counts（保留计数）放入non-pointer isa（非指针isa）的操作失败，运行时会立即重试。运行时的`StoreReleaseExclusive()`函数会在排他存储失败时执行排他加载操作，因此`oldisa`是最新的值。在为此次release（释放）操作减去一个计数后，它加上从side table中取得的retain counts，更新跟踪对象实例是否在side table中拥有retain counts的位标志，然后再次尝试存储更新后的isa。这个重试过程很可能少于32条指令（符合ARM的建议；参见下方的旁注），并且比通用路径更容易成功。

如果存储成功，当retain count降为零时，运行时会将`didTransitionToDeallocating`设置为`true`。但这在实际中永远不会发生，因为刚刚成功地向non-pointer isa添加了`RC_HALF - 1`个retain counts。

注释中的"LL/SC"指 Load-Linked 和 Store-Conditional 指令，这是 AArch64 架构中 `lxdr`（加载独占）和 `stxr`（存储独占）指令的通用名称。"独占"一词源自 ARM 的独占监视器（exclusive monitor）同步原语，并不暗示任何关于执行行为的约束（即其他处理器或内核对同一地址的读写操作不受任何限制）。

在以下情况下，存储指令可能失败：

* 另一个处理器或内核向最近加载独占操作关联的地址范围进行了写入；
* 在加载独占与存储独占操作之间发生了上下文切换（例如中断、线程抢占），且处理例程清除了独占监视器；
* 子例程执行了另一次加载独占/存储独占操作，导致先前的加载独占操作失效。

当注释提到"辅助表（side table）访问可能已释放保留标记"时，其失败原因可能源于上述任意情形。

ARM 建议在独占加载（load exclusive）和独占存储（store exclusive）指令之间保持不超过 128 字节的间隔，以最大限度地减少在操作期间发生上下文切换（context switch）导致监视器（monitor）被清除的可能性。我还没有测量过这里包含的机器指令数量，但我敢打赌，在主循环中，加载和存储之间至少有超过 32 条指令。

我只能设想出一种可能发生活锁（live lock）的场景：一个线程连续地对对象进行保留（retain）和释放（release）操作，导致对 `isa` 进行大量写入，从而阻碍了正在处理下溢（underflow）的线程成功执行独占存储操作。一次保留/释放循环在执行侧表查找所需的时间内可以进行多次迭代，因此如果没有快速路径（fast path），释放操作可能永远无法完成。

如果另一个线程执行了一次或多次保留操作，则不会发生活锁，因为当前线程应该能够从该计数开始递减。或者，如果另一个线程执行了一次或多次释放操作，也不会发生活锁，因为会有一个线程在竞争中获胜并获取侧表锁（side table lock）。

如果重试未成功（例如，添加 `RC_HALF - 1` 个引用计数导致溢出），运行时会通过以下操作中止此事务：清除独占监视器，将引用计数放回辅助表，并在跳回比较并交换循环的起点前重新加载非指针 isa。然而，它仍然持有辅助表的锁。

```
  if (!stored) {
      // Inline update failed. Put the retains back in the side table.
      ClearExclusive(&isa().bits);
      sidetable_addExtraRC_nolock(borrow.borrowed);
      oldisa = LoadExclusive(&isa().bits);
      goto retry;
  }
```

如果任一存储尝试成功，且辅助表中不再包含该对象实例的额外引用计数，运行时将从辅助表中移除该对象实例对应的条目。

```
  // Decrement successful after borrowing from side table.
  if (emptySideTable)
      sidetable_clearExtraRC_nolock();
```

最后，如果必要，运行时会释放其侧表锁，并返回 false 以指示保留计数未归零（此情况仅被 `_objc_rootReleaseWasZero()` 这个 SPI 接口使用）。在该路径上保留计数无法归零（见上文），因此在成功更新侧表后，释放操作即告结束。

```
  if (!didTransitionToDeallocating) {
    if (slowpath(sideTableLocked)) sidetable_unlock();
    return false;
  }
}
```

否则，执行流程将继续进入释放逻辑。

**释放**

当引用计数（retain count）降为零（或发生下溢且对象实例未在辅助表（side table）中存储引用计数）时，运行时（runtime）会释放[5]该对象。

```
deallocate:
// Really deallocate.
ASSERT(newisa.isDeallocating());
ASSERT(isa().isDeallocating());

if (slowpath(sideTableLocked)) sidetable_unlock();

__c11_atomic_thread_fence(__ATOMIC_ACQUIRE);

if (performDealloc) {
  ((void(*)(objc_object *, SEL))objc_msgSend)(this, @selector(dealloc));
}
return true;
```

首先，运行时会释放侧面表锁（side table lock），如果有必要的话。接下来，它有一个获取栅栏（acquire fence）。我推测这是一种原子栅栏同步（atomic-fence synchronization），但我不清楚这个栅栏与哪个释放操作同步。栅栏之后唯一可能有争用的读取是几行后消息发送中的isa（isa指针），但这个线程刚刚设置了isa。我预计从另一个线程写入isa是未定义行为（undefined behavior），因为保留计数（retain count）为零。然而，这样的写入可能会改变类，由于栅栏的存在，这对当前线程是可见的。所以，据我所知，栅栏唯一可能的影响是在非常罕见和奇怪的情况下对消息发送的影响。这个栅栏可能是一个先前实现的遗留产物，已经不再相关，是发布前最后一分钟为"修复"内存排序问题（memory ordering problem）而做的更改，或者是一个不必要的补充，或者是我误解了这个行为。最后，如果释放不是通过_objc_rootReleaseWasZero() SPI发生的，那么会向对象发送一条-dealloc消息。

release 操作最终返回 `true`，表示保留计数已归零，这一返回值会被除 `_objc_rootReleaseWasZero()` SPI 之外的所有调用者忽略。

**结语**

当我决定分别介绍 retain 和 release 时，本以为 release 的文章会比 retain 短很多，结果反而长了 20%！

当保留计数溢出（overflow）时，运行时会将计数的一半保存在 non-pointer isa（非指针 isa）中，另一半加入 side table（侧表）。它可以先快速完成对 non-pointer isa 的关键写入，再处理对 side table 的昂贵写入。相反，当保留计数下溢（underflow）时，运行时必须先从 side table 获取保留计数，以获得执行关键 non-pointer isa 写入所需的信息。在独占加载（exclusive load）和存储（store）之间插入昂贵的读取操作，会增加存储失败的概率，这便形成了 release 操作必须处理的独特角落案例（corner case）。写作是绝佳的学习方式。

带着这个教训，我祈祷下一篇关于 autorelease 的文章能更直白一些！