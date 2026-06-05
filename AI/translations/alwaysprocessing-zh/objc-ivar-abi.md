# Objective-C Internals: Non-Fragile Instance Variables

> 原文：https://alwaysprocessing.blog/2023/03/12/objc-ivar-abi
> 发布：2023-03-12　作者：Brian T. Kelley
> 译者：MiMo；代码块保留英文原样　|　仅供个人学习，未公开发布

---

Objective-C 的实例变量可能会影响 ABI（应用二进制接口）的稳定性。在 Objective-C 2 中，Apple 引入了一种“非脆弱”布局，以在类实例变量的某些类型变更下保持 ABI 的稳定性。

Objective-C 运行时（runtime）的某些部分与 ABI 相关联，Objective-C 类中实例变量的演进就说明了这一点。

**脆弱实例变量**

在 32 位版本的 macOS 中，Objective-C 类的实例变量采用“脆弱”布局。这意味着，针对此部署目标，实例变量的访问方式是通过其相对于类起始位置的偏移量（offset）进行的，就如同整个类继承层次中的实例变量被拼接进了一个 C 语言结构体（struct）中。并且，如同 C 结构体中的字段一样，每个实例变量的偏移量都会被硬编码（hardcoded）到每条读写操作的机器码（machine code）中，一旦部署就无法更改。

因此，每个实例变量的大小、对齐方式（alignment）和偏移量都是其类 ABI 的一部分。在公开的 Objective-C 类中添加或移除实例变量可能会在运行时破坏子类，故称之为“脆弱”。（更改实例变量的类型可视为一次先删后增的操作。）

当链接到一个二进制库（例如，一个应用程序链接到AppKit框架）时，链接到该二进制库的实体依赖于其ABI（应用二进制接口）保持“稳定”。为了维持稳定的ABI，所有对二进制库的更改都不得使编译器和链接器与二进制库互操作所使用的契约失效。如果二进制库的新版本更改了契约，任何之前链接到它的内容都可能会失败。

为了探索在Objective-C 2之前Apple如何处理ABI稳定性的约束，让我们检查以下来自macOS 10.13 SDK的节选和注释的类定义。

```
// #import <objc/NSObject.h>
@interface NSObject <NSObject> {
  Class             isa;                // 0x00
}
@end

// #import <AppKit/NSResponder.h>
@interface NSResponder: NSObject {
  id                _nextResponder;     // 0x04
}
@end

// #import <AppKit/NSView.h>
typedef struct __VFlags {
  unsigned int flags;
} _VFlags;

@class _NSViewAuxiliary;

@interface NSView: NSResponder {
  /* All instance variables are private */
  NSRect            _frame;             // 0x08
  NSRect            _bounds;            // 0x18
  NSView           *_superview;         // 0x28
  NSArray          *_subviews;          // 0x2c
  NSWindow         *_window;            // 0x30
  id                _unused_was_gState; // 0x34
  id                _frameMatrix;       // 0x38
  CALayer          *_layer;             // 0x3c
  id                _dragTypes;         // 0x40
  _NSViewAuxiliary *_viewAuxiliary;     // 0x44
  _VFlags           _vFlags;            // 0x48
  struct __VFlags2 {
    unsigned int flags;
  }                 _vFlags2;           // 0x4c
}
@end
```

以上每个实例变量（instance variable）右侧的注释表明了编译器生成读写该变量代码时，相对于self指针的硬编码偏移量。

下文展示了`NSView`的实例变量应用二进制接口（ABI），它反映了对象的内存布局，以及编译器如何将实例变量视为C结构体中的字段来处理。

```
struct NSViewHeapLayout {
  Class             isa;                // 0x00
  id                _nextResponder;     // 0x04
  NSRect            _frame;             // 0x08
  NSRect            _bounds;            // 0x18
  NSView           *_superview;         // 0x28
  NSArray          *_subviews;          // 0x2c
  NSWindow         *_window;            // 0x30
  id                _unused_was_gState; // 0x34
  id                _frameMatrix;       // 0x38
  CALayer          *_layer;             // 0x3c
  id                _dragTypes;         // 0x40
  _NSViewAuxiliary *_viewAuxiliary;     // 0x44
  _VFlags           _vFlags;            // 0x48
  struct __VFlags2  _vFlags2;           // 0x4c
};
```

如果我们梳理自 Mac OS X 10.0 发布以来所有公开的 SDK，可以观察到 `NSObject` 和 `NSResponder` 的实例变量（ivar）并未改变（从而维持了 ABI 稳定性）。然而，`NSView` 中存在两个作为 ABI 稳定性维护遗留产物的、颇为奇特的实例变量：

`_unused_was_gState`

在早期版本的 Mac OS X 中，`NSView` 拥有一个 `_gState` 实例变量以支持其与图形栈的集成。该状态在后续版本中变得过时，因此 AppKit 的维护者们将其重命名以表明该变量已被故意弃用。

AppKit 的维护者无法移除该实例变量，因为移除操作会使其后所有实例变量（包括子类中的）的偏移量发生移动。

我推测维护者未重新利用该实例变量，是因为某些应用程序很可能曾读取（甚至写入）过该变量。通常，这类应用程序无法正确处理指向一个完全不同的不透明类型的指针，因此重新利用该变量可能会破坏这些应用。然而，任何受影响的应用程序很可能在使用该变量的默认/占位值时仍能正常运行。

在 Objective-C 语言中，`@private` 访问修饰符是与 Objective-C 2.0 的发布同时引入的。在引入访问控制之前，**惯例**是防止子类直接访问超类状态的唯一工具（这也是 NSView 实例变量区块开头注释的由来）。

`_viewAuxiliary`：当一个类需要 ABI（应用程序二进制接口）稳定性时，使用一个私有辅助类是一种典型的模式，以便为每个版本添加或移除实例变量保留能力。因此，NSView 的每个实例都会分配一个 `_NSViewAuxiliary` 实例来存储实例变量和状态，从而不影响 NSView 的 ABI。

当一个类需要 ABI 稳定性但无法使用私有辅助类时（例如 `NSObject` 或 `NSResponder`），另一种用于添加或移除实例变量的典型模式是使用一个**侧表（side table）**。（在 Mac OS X 10.6 和 iPhoneOS 3.1 中，Objective-C 运行时通过其**关联引用（associated references）** 特性为侧表存储添加了通用支持。）（译注：现代系统可能已变化）

**非脆弱实例变量（Non-Fragile Instance Variables）**

在 Objective-C 2 中，苹果更改了 Objective-C 运行时与 ABI（应用程序二进制接口）以支持“非脆弱”（non-fragile）布局，该特性在所有版本的 iOS、tvOS、watchOS 以及 macOS 的 64 位版本中均可用。此特性能在向类中添加实例变量（instance variable）以及删除非公开实例变量时保持 ABI 稳定性。因此，为维护 ABI 稳定性，不再需要使用上述模式（废弃实例变量、私有辅助类与侧表存储）。

非脆弱实例变量布局有两项主要要求来保持 ABI 稳定性：

向类中添加实例变量需要更新用于访问其后所有实例变量的偏移量，包括所有子类中的实例变量。

从类中移除实例变量时，必须确保该实例变量未在类的二进制映像外部被访问。

编译 Objective-C 2 代码时，编译器会为每个实例变量生成一个偏移量符号，其使用满足了 ABI 稳定性要求：

当Objective-C运行时检测到某个类的父类大小增长时，会更新该类的实例变量（ivar）偏移符号以适应更大的父类大小。

对于使用`@package`或`@private`访问控制声明的实例变量，其生成的符号在对象文件中具有私有外部可见性，因此不会从二进制镜像（binary image）中导出。移除这些非公开实例变量是一项ABI（Application Binary Interface，应用程序二进制接口）稳定变更，因为任何先前试图访问它们的代码在链接时都会失败。

Objective-C运行时目前不会在某个类的父类大小缩小时减少该类实例变量的偏移量。这种方式倾向于最小化Objective-C运行时的堆使用和应用启动时间，代价是受影响的类的实例会增加堆使用。

作为一种优化，每个偏移符号的初始值是构建时实例变量的偏移值，这使得如果基类没有增长，Objective-C运行时可以在每次应用启动时跳过偏移量的计算。

**脆弱 vs. 非脆弱 示例**

为了说明Objective-C 1和Objective-C 2之间编译器生成代码的差异，让我们看一些加载`_superview`实例变量的简单代码。

```
NSView *superview = aView->_superview;
```

无论是在编译 `NSView` 自身，还是在构建第三方应用程序（假设该实例变量（instance variable）仍隐式地为 `@public`），编译器所发出的代码都将是相同的。

在采用脆弱布局（fragile layout）的 Objective-C 1 中，编译器只需简单地将该实例变量在编译时观察到的偏移量，加到对象实例指针上，即可计算出加载该实例变量的地址。

```
NSView *superview = *(NSView **)((intptr_t)aView + 0x28);
```

如果NSView的布局在此编译之后发生变化，从硬编码偏移地址加载的结果可能变得未定义。

在Objective-C 2中，采用非脆弱布局（non-fragile layout）时，编译器会将实例变量偏移量符号的值与对象实例指针相加，从而计算出应加载实例变量的地址。

```
extern uint32_t OBJC_IVAR_$_NSView._superview;
NSView *superview = *(NSView **)((intptr_t)aView + OBJC_IVAR_$_NSView._superview);
```

NSView 的布局对此次编译过程而言是不透明的，因此只要实例变量（ivar）存在，加载结果就仍是明确定义的。但如果该实例变量被移除，dyld 将无法加载二进制镜像，因为实例变量偏移量符号将无法被解析。（我想这总比出现未定义的运行时行为要好！）

**Objective-C 运行时实现**

类实例变量偏移量的更新发生于 `realizeClassWithoutSwift()` 中该类的首次初始化过程中。

```
static Class realizeClassWithoutSwift(Class cls, Class previously) {
  // ...
  // Reconcile instance variable offsets / layout.
  // This may reallocate class_ro_t, updating our ro variable.
  if (supercls && !isMeta) reconcileInstanceVariables(cls, supercls, ro);
  // ...
}
```

只有在超类“增长到”子类内部（相对于编译子类时计算的布局）时，更新才是必要的。超类可能增加了大小，因为它添加了实例变量（instance variables）、将实例变量更改为更大尺寸的类型、向作为实例变量存储的结构体添加了字段，或者它的超类增长了。（请记住，如果超类缩小，运行时（runtime）会进行无操作。）

`reconcileInstanceVariables()` 函数首先确保，如果有必要更新类的实例变量偏移量，那么类的 `class_ro_t`[1] 数据结构已被复制到堆（heap）上，因为初始数据结构值是从可执行文件的只读部分（read-only section）映射的。需要一个可写副本，以便运行时可以存储更新的类布局元数据。

```
static void reconcileInstanceVariables(Class cls, Class supercls, const class_ro_t*& ro) {
  // ...
  if (ro->instanceStart >= super_ro->instanceSize) {
    // Superclass has not overgrown its space. We're done here.
    return;
  }

  if (ro->instanceStart < super_ro->instanceSize) {
    // Superclass has changed size. This class's ivars must move.
    // Also slide layout bits in parallel.
    // This code is incapable of compacting the subclass to
    //   compensate for a superclass that shrunk, so don't do that.
    class_ro_t *ro_w = make_ro_writeable(rw);
    ro = rw->ro();
    moveIvars(ro_w, super_ro->instanceSize);
  }
}
```

`moveIvars()` 函数会对类的实例变量偏移量符号进行必要的位移调整，以适应父类大小的增长。

```
static void moveIvars(class_ro_t *ro, uint32_t superSize) {
  uint32_t diff = superSize - ro->instanceStart;

  if (ro->ivars) {
    // Find maximum alignment in this class's ivars
    uint32_t maxAlignment = 1;
    for (const auto& ivar : *ro->ivars) {
      if (!ivar.offset) continue;  // anonymous bitfield

      uint32_t alignment = ivar.alignment();
      if (alignment > maxAlignment) maxAlignment = alignment;
    }

    // Compute a slide value that preserves that alignment
    uint32_t alignMask = maxAlignment - 1;
    diff = (diff + alignMask) & ~alignMask;

    // Slide all of this class's ivars en masse
    for (const auto& ivar : *ro->ivars) {
      if (!ivar.offset) continue;  // anonymous bitfield

      uint32_t oldOffset = (uint32_t)*ivar.offset;
      uint32_t newOffset = oldOffset + diff;
      *ivar.offset = newOffset;
    }
  }

  *(uint32_t *)&ro->instanceStart += diff;
  *(uint32_t *)&ro->instanceSize += diff;
}
```
```

上述 for 循环“滑动”了实例变量（ivar）的偏移量——从类起始位置向后移动，以容纳更大的基类，同时保持对齐。它所写入的 ivar 变量正是实例变量偏移量符号，例如 `OBJC_IVAR_$_NSView._superview`（如前文所述）。

因此，通过增加一步间接访问来读写实例变量值，并承受微小的启动开销，Objective-C 运行时（runtime）得以优雅地消除一项重大的 ABI（应用程序二进制接口）兼容性问题，且仅带来极小的开销与复杂度。