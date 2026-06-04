# Rust API Bindings: Core Foundation Memory Management and Mutability

> 原文：https://alwaysprocessing.blog/2023/12/28/rust-ffi-cf-rc
> 发布：2023-12-28　作者：Brian T. Kelley
> 译者：MiMo；代码块保留英文原样　|　仅供个人学习，未公开发布

---

**Rust API 绑定：Core Foundation 的内存管理与可变性**

Core Foundation 在内存管理和可变性方面所使用的设计模式，与 Rust 的惯用风格竟有着惊人的契合。本文将分享我如何通过艰难的方式得出这一结论的概要。

在我为 Core Foundation 设计 Rust API 绑定时，我希望面向用户的 API 能尽可能贴近 Rust 标准库。内存管理是设计中的关键领域，其设计会极大地影响 API 的表面。Core Foundation 与 Rust 标准库在内存管理方法上存在（至少）两个关键差异：

1.  所有 Core Foundation 对象都分配在堆上，并且是引用计数（reference counted）的。通常，Rust 类型可以是栈分配的，也可以是堆分配且具有独占所有权（uniquely owned）的，或者是堆分配并具有共享所有权（shared ownership）的。
2.  Core Foundation 使用不同的类型来表示不可变（immutable）和可变（mutable）对象，而 Rust 则通过类型系统来表达可变性（mutability）。

下文总结了我在这一领域的探索、我的内存管理设计目标，以及我是如何实现这些目标的。

**临时方案**

对于许多 C API（应用程序编程接口）而言，将指针包装在元组结构体（tuple struct）中并实现 Drop 特征（trait），就足以提供一个针对外部接口的地道的 Rust API。以下是一个使用此方法包装 CFString 的示例：

```
struct String(*const __CFString);

impl String {
  fn len(&self) -> CFIndex {
    unsafe { CFStringGetLength(self.0) }
  }
}

impl Drop for String {
  fn drop(&mut self) {
      unsafe { CFRelease(self.0.cast()) }
  }
}
```

这种方法直截了当，但并非零成本抽象（zero-cost abstraction）。每当需要获取 Core Foundation 对象指针时（例如在 `len` 方法中），编译器都必须生成对元组结构体 `&self` 的解引用操作以加载 Core Foundation 指针值。这种间接寻址无法避免，因为我们必须定义一个类型来实现 `Drop`（译注：现代 Rust 可能已优化部分场景），尽管在实践中其开销可忽略不计。

虽然 Core Foundation 是基于 C 的 API，但许多类型具有逻辑上的子类关系。若以此方式为 `CFMutableString` 添加 Rust API 绑定，就需要定义一个全新的独立类型。而添加 `Deref` trait（特征）实现则能通过解引用强制转换（deref coercion）使逻辑子类获得其逻辑父类的所有方法，由此产生的 Rust API 仍将保持合理的惯用性。

尽管这是条已有人探索的路径[1]，我希望找到一种能满足以下条件的设计：

- 是真正的零成本抽象；
- 能通过类型系统体现 Core Foundation 对象的堆分配特性（例如使用 `Box`）；
- 将 Rust 的可变引用与 Core Foundation 的可变类型相结合。

我首先着手探索设计空间，通过构建标准库中 `Box` 和 `Arc` 类型的等价实现。我意识到仿照 `Box` 的独占所有权（unique ownership）将构成可变性（mutability）方案的一部分，而类似 `Arc` 的共享指针（shared pointers）则是 Apple 平台编程的基础组成部分。

通过这项实践，我立刻达成了前两个设计目标。请参见以下以 `CFString` 为例的示例代码：

```
struct Box<T>(NonNull<T>);

impl<T> Deref for Box<T> {
  type Target = T;

  #[inline]
  fn deref(&self) -> &Self::Target {
    unsafe { self.0.as_ref() }
  }
}

struct String;

impl String {
  fn new() -> Box<Self> { /* ... */ }

  fn len(&self) -> CFIndex {
    let cf: *const _ = self;
    unsafe { CFStringGetLength(cf.cast()) }
  }
}
```

与上一节的方法类似，元组结构体包装了原始的 Core Foundation 对象实例指针。但这个包装器存在三个本质区别。

首先，类型名 `Box<T>` 向读者明确表示 `T` 是堆分配的，并且实例 `T` 是唯一的。

其次，它实现了 `Deref` 指向 `T`（即实现 API 绑定的 Rust 类型），这对实现零成本抽象至关重要。当编译器对 Box 进行解引用时——例如调用 `len` 方法时——Box 会将 Core Foundation 指针值作为对 `T` 的引用返回。该引用值（即 `&self`）与 Core Foundation 指针值在位级上完全一致，因此可以直接传递给 C API。

在两种方法中编译器都会解引用元组结构体，那么为何其中一种能实现零成本抽象而另一种不能？

在上一节中，`&self` 是指向元组结构体的引用。编译器必须通过该引用从元组结构体的字段中加载 Core Foundation 指针值。用 C 语言示例可能有助于阐明这一点：

```
struct String { const struct __CFString *s; };

CFIndex String_len(const struct String *self) {
  return CFStringGetLength(self->s);
  // dereference+member access ^^ is an abstraction cost
}

struct String s = /* ... */;
CFIndex length = String_len(&s);
```

在这一部分中，`&self` 是指针值。解引用操作是一个逻辑过程，而非物理过程，本质上等同于一次成员访问。用 C 语言来演示如下：

```
struct Box__CFString { const struct __CFString *v; };

CFIndex String_len(const struct String *self) {
  return CFStringGetLength((const struct __CFString *)self);
}

struct Box__CFString s = /* ... */;
CFIndex length = String_len(s.v);
//        member access here ^ is zero cost
```

最后，将类型绑定（例如 `String`）与内存管理机制（例如 `Box<T>`）分离，使得对 Core Foundation 类型绑定的引用能够以惯用的、零成本的方式使用。考虑一下 `CFArrayGetValueAtIndex` 的潜在绑定方式。采用本节的方法，该函数的绑定可以简单地将指针强制转换为一个具有数组生命周期的引用。

而采用上一节的方法，此函数的绑定可能：

- 为返回的值创建一个新的绑定实例，并对该对象执行保留（retain）和释放（release）。由于新的绑定实例没有与数组关联的生命周期，因此保留操作是必要的，以确保对象至少与绑定实例存活一样长。然而在许多情况下，保留/释放是不必要的开销。
- 使用一个中间类型将生命周期关联到绑定实例，并规避其保留/释放，这实际上等同于本节方法中的函数绑定，但需要更多的仪式来正确消除保留/释放。

**用于 Core Foundation 的 Arc**

经过更多探索和尝试，我终于找到了一种方法，以实现我的第三个设计目标：将 Rust 的可变引用（mutable references）与 Core Foundation 的可变类型相结合。

在某个时刻，我自问：“为什么不可变对象需要独占所有权（exclusive ownership）？” 我最终说服了自己：“它们并不需要！” 回头看，我不明白为什么这个道理当初不是更显而易见。Rust 对其 `Arc` 类型的文档清楚地说明：

> 你通常无法获得对 `Arc` 内部内容的可变引用。

有了这个见解，制定出识别合适智能指针类型的指导原则就相当直接了：这个 Core Foundation 对象实例是否是一个由原始指针独占拥有的可变类型（即，一个 `Create` 或 `Copy` 函数返回了该指针）？如果是，则使用 `Box<T>`；否则使用 `Arc<T>`。

我为 Core Foundation 实现的 `Box<T>` 和 `Arc<T>` 几乎完全相同，主要区别在于 `Box<T>` 还实现了 `DerefMut`、`AsMut` 和 `BorrowMut`。

智能指针类型中引用计数（reference counting）与可变性的结合[2]，满足了我的设计目标，并最终产生了令人惊讶地符合 Rust 惯用风格的代码。

```
impl String {
  fn append(&mut self, s: &String) {
    let cf: *mut _ = self;
    let s: *const _ = s;
    unsafe { CFStringAppend(cf.cast(), s.cast()) };
  }
}

fn main() {
  let mut s: Box<String> = String::new();
  s.append(cfstr!("Hello, World!"));
  println!("{s}");
}
```