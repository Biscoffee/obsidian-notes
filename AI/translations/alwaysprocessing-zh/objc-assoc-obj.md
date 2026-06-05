# Objective-C Internals: Associated References

> 原文：https://alwaysprocessing.blog/2023/06/05/objc-assoc-obj
> 发布：2023-06-05　作者：Brian T. Kelley
> 译者：MiMo；代码块保留英文原样　|　仅供个人学习，未公开发布

---

本文对苹果公司的关联引用（Associated References）实现与我为历史背景编写的实现进行了比较，并附带了关于与标记指针（tagged pointer）对象配合使用以及赋值关联策略实际作用的说明。

我记得当时热切期盼着，我们将 Mac 版 Microsoft Office 2016（当时名称还未确定）的最低部署目标改为 Mac OS X 10.6[1] 的那一天到来。雪豹系统引入了许多新的 API，包括 Grand Central Dispatch 和块（blocks）。但我最兴奋的是能够开始使用 Objective-C 的关联引用功能，以替代一些糟糕的代码。

**旧有方式**

Objective-C 最大的优势（也是其劣势）在于其动态方法绑定（dynamic method binding）。几乎所有主要的第三方应用程序都（滥用）了这一特性，以填补功能缺口或缓解应用/系统架构之间的阻抗失配。

将一个对象的生命周期绑定到由第三方（即苹果）实例化并控制的对象的生命周期上，曾是此类功能缺口之一。在运行时提供此功能之前，应用程序可以通过部分预修补 `-[NSObject dealloc]` 方法实现来达成此目的。以下代码示例展示了第三方如何利用该方法实现关联引用（Associative References）。

```
#import <Foundation/Foundation.h>
#import <objc/runtime.h>

static OSSpinLock s_lock;               // for main side table
static NSMapTable *s_associatedObjects; // main side table
static IMP s_NSObject_dealloc;          // original implementation

void APAssociatedObjectSet(id object, id association) {
  id previousAssociation = nil;

  // retain does not require the lock, so do it outside of the
  // lock to minimize time spent holding the lock
  [association retain];

  OSSpinLockLock(&s_lock);
  previousAssociation = [s_associatedObjects objectForKey:object];
  if (association != nil) {
    [s_associatedObjects setObject:association forKey:object];
  } else {
    [s_associatedObjects removeObjectForKey:object];
  }
  OSSpinLockUnlock(&s_lock);

  // release outside of the lock in case this is the last
  // release, as the dealloc implementation acquires the lock
  [previousAssociation release];
}

id APAssociatedObjectGet(id object) {
  OSSpinLockLock(&s_lock);
  id association = [s_associatedObjects objectForKey:object];
  // retain the associated object to ensure it's not deallocated
  // while in use by the caller in case another thread changes the
  // associated object between now and then
  [association retain];
  OSSpinLockUnlock(&s_lock);

  return [association autorelease];
}

static void APAssociatedObject_dealloc(id self, SEL _cmd) {
  // release any associated object and remove the side table entry
  APAssociatedObjectSet(self, nil);
  (*s_NSObject_dealloc)(self, _cmd);
}

void APAssociatedObjectInitialize(void) {
  s_lock = OS_SPINLOCK_INIT;
  // The key is weak to prevent the object from becoming immortal.
  // The value is weak to explicitly control the retain count to
  // prevent dealloc reentrancy deadlocks.
  s_associatedObjects=[NSMapTable mapTableWithWeakToWeakObjects];

  // Pre-patch -[NSObject dealloc] to clean up s_associatedObjects
  Method m = class_getInstanceMethod([NSObject class],
                                     @selector(dealloc));
  s_NSObject_dealloc = method_getImplementation(m);
  method_setImplementation(m, (IMP)&APAssociatedObject_dealloc);
}
```

尽管实现仅包含59行代码（含空白和注释），但有几点需要说明：

此实现支持0或1个对象关联，但通过少量修改即可支持任意数量的关联，就像 `objc_setAssociatedObject()` 那样。或者，客户端也可以使用 `NSMutableDictionary` 来关联任意数量的对象。

每个 `-dealloc` 都需要获取锁以执行簿记（除了运行时（runtime）和分配器的锁获取外）。我们在前一篇文章中看到，对于没有关联引用（以及其他条件）的对象实例，运行时存在快速释放路径，这使得它在大多数情况下能避免锁开销。

移除关联可能导致关联对象被释放，而这又可能触发其关联对象的释放。因此，该实现必须避免在持有锁时发生递归，因为 `OSSpinLock` 不可重入。

预先修补（pre-patching）对象所属类的 `-dealloc` 方法并非可行方案，原因有二：

一个类层次结构可能有多个补丁。例如，在 NSObject 上设置一个关联对象（associated object）并在 NSView 上设置另一个后，所有 NSView 实例，包括子类，在释放过程中会调用补丁两次。一种实现可以处理这种情况，但代价是增加额外的复杂性。

从补丁中调用正确的释放方法（-dealloc）变得更加具有挑战性。继续上面的例子，如果一个 NSTableView 正在释放，补丁如何知道它应该调用 -[NSView dealloc] 实现还是 -[NSObject dealloc] 实现？（self 的类标识始终是 NSTableView。）需要大量的簿记来跟踪对象在其释放链中的位置，并处理作为其释放一部分发生的额外释放。

Objective-C 自动引用计数（ARC）直到 OS X 10.7 Lion 才首次亮相。因此，我想强调两件对现代 Objective-C 程序员不再相关的事情：

`APAssociatedObjectGet()` 中的 `retain` 和 `autorelease` 调用确保返回的对象在当前自动释放作用域（autorelease scope）内存活。若不这样做，另一个线程可能在对象从映射表（map table）中被检索出来到返回给调用者之间导致该对象被释放。

在映射表的 `mapTableWithWeakToWeakObjects` 工厂方法中使用的 `weak` 并不具备 ARC 的置零弱引用（zeroing weak reference）语义。相反，它等同于 ARC 的 `unsafe_unretained`。

`APAssociatedObjectInitialize()` 本可以使用 `__attribute__((constructor))` 来在 `main()` 调用前初始化该特性。我之所以省略了这一点，是因为大型应用通常拥有复杂的初始化系统，它们会调用此函数。

接下来，让我们看看苹果的 Objective-C 运行时是如何实现这一特性的。

**苹果的方式**

上述第三方实现及评论与苹果的实现惊人地一致。（我说惊人是因为我在查阅苹果的实现之前就写了它。）

首先，我们来看 `objc_setAssociatedObject()`，它直接调用了 `_object_set_associative_reference()`。

```
DisguisedPtr<objc_object> disguised{(objc_object *)object};
ObjcAssociation association{policy, value};

// retain the new value (if any) outside the lock.
association.acquireValue();

bool isFirstAssociation = false;
{
  AssociationsManager manager;
  AssociationsHashMap &associations(manager.get());

  if (value) {
    auto refs_result = associations.try_emplace(disguised, ObjectAssociationMap{});
    if (refs_result.second) {
      /* it's the first association we make */
      isFirstAssociation = true;
    }

    /* establish or replace the association */
    auto &refs = refs_result.first->second;
    auto result = refs.try_emplace(key, std::move(association));
    if (!result.second) {
      association.swap(result.first->second);
    }
  } else {
    auto refs_it = associations.find(disguised);
    if (refs_it != associations.end()) {
      auto &refs = refs_it->second;
      auto it = refs.find(key);
      if (it != refs.end()) {
        association.swap(it->second);
        refs.erase(it);
        if (refs.size() == 0) {
          associations.erase(refs_it);
        }
      }
    }
  }
}

if (isFirstAssociation)
  object->setHasAssociatedObjects();

// release the old value (outside of the lock).
association.releaseHeldValue();
```

考虑到与前一节内容的关联性，我仅强调本实现中的关键异同点。

`DisguisedPtr` 用于抑制 `leaks` 等工具的堆跟踪。

`ObjcAssociation` 辅助对象实现了关联策略（包括 assign、retain 或 copy 语义的存储，以及读取操作的原子性/非原子性）。

`AssociationsManager` 是一个 RAII（资源获取即初始化）便利对象，用于锁定和解锁关联对象自旋锁（译注：现代系统中已改为不公平锁）。

对象关联使用哈希表（具体为 LLVM 的 `DenseMap`）存储。顶层哈希表将对象指针映射到另一个关联哈希表，该表将键映射到 `ObjcAssociation`（包含对象及其保留策略）。

将值设为 `nil` 会移除之前关联的对象。

当对象首次建立关联时，运行时会更新其状态以关闭快速释放路径。

任何之前关联的对象的释放操作都发生在锁之外。

与 setter 类似，`objc_getAssociatedObject()` 仅调用 `_object_get_associative_reference()`。获取路径相当直接，我没什么可补充的！🙊

Apple的实现提供了一个奇特的函数，`objc_removeAssociatedObjects()`。老实说，我不确定为什么这是一个公共API——`runtime.h`中的注释建议不要使用它（并且有充分的理由）：

此函数的主要目的是方便地将对象恢复到“原始状态”。你不应该使用此函数来一般性地移除对象的关联（associated objects），因为它也会移除其他客户端可能添加到对象的关联。通常，你应该使用`objc_setAssociatedObject`并传入`nil`值来清除关联。

与getter和setter函数类似，`objc_removeAssociatedObjects()`调用`_object_remove_associations()`。但是，这个内部函数接受一个额外的参数：`bool deallocating`，当由`objc_removeAssociatedObjects()`调用时，其值为`false`。这个内部函数只有一个其他调用者，`objc_destructInstance()`，它毫不意外地传递`true`作为`deallocating`。

那么，`deallocating`标志做什么呢？函数中的一条注释解释了其目的：

如果我们没有正在销毁（deallocating），那么`SYSTEM_OBJECT associations`（系统对象关联）将被保留。

Apple 有一个内部策略标志 `OBJC_ASSOCIATION_SYSTEM_OBJECT`，它会阻止其关联对象被 `objc_removeAssociatedObjects()` 移除。虽然你可以用这个函数给自己制造麻烦（“自找麻烦”），但 Apple 会阻止你违反他们的假设。

我猜这就是为什么关联对象键的类型是 `void *`：指针类型的键在 Apple 的框架中很难识别，进而在第三方应用程序中难以（被滥用）。例如，与字符串类型的键（如 `NSNotifiationName`）相比，字符串键相对容易被发现和使用。

**Tagged Pointer 对象**

在一个 tagged pointer（标记指针）对象上设置关联对象会发生什么？其效果与将该对象赋值给一个具有相同存储策略的全局变量相同：该对象会一直存在，直到被赋予新值。因此，在一个 tagged pointer 对象上设置关联对象，实际上会导致关联对象泄漏。

关联对象的实现中没有处理 tagged pointer 的代码路径（甚至没有在控制台输出警告的代码）。因此，运行时（runtime）会将 tagged pointer 存储在关联哈希表（associations hash map）中，并且由于 tagged pointer 对象永远不会被释放，它们会在其中无限期地存在。

标记指针对象的另一个副作用是它们实质上会驻留（intern）所有值。虽然像 `NSNumber` 这样的类型已知实现了某种形式的驻留，但 `NSString` 此前没有这种行为。然而，`NSString` 的标记指针代码路径（code path）非常激进，以至于从磁盘加载的本地化字符串（localized strings）也可能产生标记指针对象！因此，任何在 `NSString` 类型上设置关联对象的代码，如果字符串实例是标记指针对象而非独立实例，可能会发现关联对象相互覆盖（stomping on each other）。

虽然使用标记指针对象被认为是一种内部实现细节，但可以看看哪些类使用了标记指针，并避免对这些类型的对象使用关联对象。

**深入探讨 `assign` 存储策略**

在撰写这篇文章时，我意识到自己错误地使用这个 API 已经超过十年了 🤦‍♂️。`OBJC_ASSOCIATION_ASSIGN` 策略旁边的注释说明：

指定一个对关联对象的弱引用（weak reference）。（译注：在现代 Objective-C Runtime 中，`assign` 策略通常对应 `OBJC_ASSOCIATION_ASSIGN`，它模拟了一个弱引用行为，但并非真正的零弱引用（zeroing weak reference），需注意其使用限制。）

如前所述，在自动引用计数（ARC）之前，术语“弱引用”（weak）等同于 ARC 中的 `unsafe_unretained`；此标志不使用 ARC 的零化弱引用（zeroing weak reference）语义。请检查实现中是否有任何对 `weak` 的使用。实际上一个都没有！

本周我需要进行大量的代码检索与审查工作……

**总结**

第三方实现的关联对象（Associated References）在功能上几乎能媲美苹果的官方实现，而官方实现的主要优势在于：对于没有关联对象的对象，提供了一条快速释放路径（fast deallocation path）。新的运行时优化（例如，标记指针对象（tagged pointer objects））可能会导致代码中将对象与那些唯一性和生命周期可能随操作系统版本变化的对象进行关联时，出现未预期的行为。此外，历史背景至关重要——文档背后的基础假设可能会随时间变化，从而扭曲其原意。（译注：此节讨论基于较旧的 Objective-C 运行时实现，现代系统可能已变化）