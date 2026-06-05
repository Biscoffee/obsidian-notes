# Objective-C Internals: Class Architecture

> 原文：https://alwaysprocessing.blog/2023/01/02/objc-class-arch
> 发布：2023-01-02　作者：Brian T. Kelley
> 译者：MiMo；代码块保留英文原样　|　仅供个人学习，未公开发布

---

Objective-C 拥有独特的类架构，其中类本身也是对象。这种优雅的设计通过编译器生成的元类（metaclass），实现了对所有对象类型的动态方法派发。

与许多主流语言类似，Objective-C 采用基于类的面向对象编程风格。为讨论 Objective-C 的类架构，我们将聚焦于类的行为（方法与属性），暂不讨论类的状态（即实例变量（ivar），相关内容将在本文后续探讨）。

虽然属性提供了对类状态的访问，但在 Objective-C 中，它们本质上只是方法的语法糖。（这也是一个适合未来文章的好主题！🤣）

探索 Objective-C 的类架构，我们只需要以下六行代码：

```
@interface MyObject: NSObject
+ (void)classMethod;
- (void)instanceMethod;
@end

MyObject *object = [[MyObject alloc] init];
```

**方法分派**

第 6 行实例化了 `MyObject` 类，并将新创建的对象实例赋值给变量 `object`。这个新的对象实例拥有自己的状态，并且能够响应消息（即方法调用），例如 `-instanceMethod` 和 `-init`。

Objective-C 对于每次消息发送（即每次方法调用）都使用动态分派（dynamic dispatch）[1]。运行时（runtime）通过查找实例 `isa` 变量所引用的类对象中的选择子（selector，即方法名）来找到方法的实现。（在所有 Objective-C 对象中，第一个实例变量（ivar）就是 `isa` 指针，它由编译器自动插入并由运行时初始化。）

上一段中使用“类对象（class object）”一词是故意的：在 Objective-C 中，类本身也是对象！这种巧妙的设计是类方法（class methods）语言特性的基础（例如调用 `[NSObject alloc]` 或 `[MyObject classMethod]`）：它使类方法能够完全多态（polymorphic）（即子类可以重写类方法），并且抹去了运行时中类方法与实例方法之间的任何区别（类方法和实例方法都通过 `objc_msgSend` 进行分派）。

由于类也是对象，它们同样拥有一个 `isa` 指针，该指针指向元类（metaclass）。元类为类对象提供了选择子（selector）到类方法实现的映射，其方式与类对象为类实例（即对象）提供选择子到实例方法实现的映射相同。

**继承**

任何类或元类仅提供其自身所实现方法的选择子到方法实现的映射。在上面的代码示例中，`MyObject` 的类对象拥有 `instanceMethod` 的映射，而 `MyObject` 的元类拥有 `classMethod` 的映射。

`MyObject` 也能响应 `+alloc` 和 `-init`，这些方法是在 `NSObject` 中实现的。每个类对象（包括每个元类）都持有一个指向其父类的引用。当解析一个选择子时，如果类/元类对象未定义相应的映射，运行时（runtime）会搜索下一个父类以寻找定义。（如果在父类链的末端仍未找到定义，运行时会抛出一个异常，不过运行时也提供了处理该场景的“逃生舱口”。）

**架构图**

下图说明了上述讨论的 Objective-C 类设计：

对象实例拥有一个 `isa` 变量，该变量指向 `MyClass` 类对象。

`MyClass` 类对象拥有：

- 一个 `isa` 变量，指向 `MyClass` 的元类（metaclass）。
- 一个 `super` 变量，指向 `NSObject` 类对象。

`MyClass` 的元类拥有：

- 一个 `isa` 变量，指向根对象（root object）`NSObject` 的元类。
- 一个 `super` 变量，指向 `NSObject` 的元类。

该图还展示了上述讨论未涵盖的几点：

每个元类的 `isa` 都指向根对象的元类，包括根对象元类自身。`isa` 不为 `nil` 是合理的（一个对象必须具有某种类型），但我不确定为什么所有元类都是根元类的类型，而不是，比如说，由运行时提供的某种类型。不过，我认为这实际上并不重要，因为元类本身从不接收消息。

根元类的父类是根类对象。在研究此图时得知这一点让我感到惊讶。我不太理解为何存在这个链接——我原以为它的父类会是 `nil`。

如果有人了解这些设计缘由的见解，请联系我告诉我！