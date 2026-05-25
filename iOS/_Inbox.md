# _Inbox


## 2026-05-15 12:07 ARC 下 Block 的内存类型与 __block 作用
**来源**: 未知　**confidence**: 0.90

- Block 在 ARC 下默认是 __NSStackBlock__。
- 捕获外部变量后，Block 自动 copy 到堆区变为 __NSMallocBlock__。
- __block 修饰的变量被包装成堆上结构体，实现 Block 内外内存共享。

### 整理后内容

Block 在 ARC 下默认是 __NSStackBlock__，捕获外部变量后会自动 copy 到堆区变成 __NSMallocBlock__。__block 修饰的变量会被包装成结构体放到堆上，让 block 内外共享同一份内存。

<details><summary>原文</summary>

Block 在 ARC 下默认是 __NSStackBlock__，捕获外部变量后会自动 copy 到堆区变成 __NSMallocBlock__。__block 修饰的变量会被包装成结构体放到堆上，让 block 内外共享同一份内存。

</details>

---



## 2026-05-16 14:51:30 图片 ^aa4e18
topic:: iOS-Inbox
date:: 2026-05-16 14:51:30
source:: 未知
confidence:: 0.00
tags:: iOS-Inbox
**来源**: 未知　**confidence**: 0.00


![[sec-01756e1b1aa5.png]]

<details><summary>原文</summary>

(图片，无 OCR 文本)

</details>

---


## 2026-05-25 11:32:51 const 关键字基础介绍 ^49af5c
topic:: iOS-Inbox
date:: 2026-05-25 11:32:51
source:: 未知
confidence:: 0.70
tags:: iOS-Inbox
summary:: const 用于声明常量，表示变量的值不可被修改。；在 C/Objective-C 中，const 可以修饰变量、指针等。；const 有助于提高代码的安全性和可读性。
**来源**: 未知　**confidence**: 0.70

- const 用于声明常量，表示变量的值不可被修改。
- 在 C/Objective-C 中，const 可以修饰变量、指针等。
- const 有助于提高代码的安全性和可读性。

### 整理后内容

**const（常量）**

`const` 是编程语言中的一个关键字，用于声明一个常量，即其值在初始化后不可被修改。

在 C 和 Objective-C 语言中，`const` 可以修饰基本类型的变量、指针等，例如：
- `const int MAX = 100;` 声明一个整型常量 `MAX`。
- `NSString * const kMyConstant = @"value";` 声明一个指向 `NSString` 的常量指针（指针本身不可变）。

使用 `const` 有助于防止意外修改，增强代码的安全性和可读性。

<details><summary>原文</summary>

const（常量）

</details>

---


## 2026-05-25 11:46:29 NSNotificationCenter与NSNotificationQueue的区别 ^e0a747
topic:: iOS-Inbox
date:: 2026-05-25 11:46:29
source:: 自述
confidence:: 0.80
tags:: iOS-Inbox, 自述
summary:: 主题是 NSNotificationCenter 与 NSNotificationQueue 的区别；被标记为‘重要’，可能是一个学习重点或面试题；内容属于 iOS 开发中的通知机制，但不属于 Runtime 等已有主题
**来源**: 自述　**confidence**: 0.80

- 主题是 NSNotificationCenter 与 NSNotificationQueue 的区别
- 被标记为‘重要’，可能是一个学习重点或面试题
- 内容属于 iOS 开发中的通知机制，但不属于 Runtime 等已有主题

### 整理后内容

NSNotificationCenter 与 NSNotificationQueue 的区别（重要）

<details><summary>原文</summary>

NSNotificationCenter与NSNotificationQueue的区别（重要）

</details>

---


## 2026-05-25 12:22:29 iOS 中的模型树、动画树与渲染树 ^3e8339
topic:: iOS-Inbox
date:: 2026-05-25 12:22:29
source:: 未知
confidence:: 0.70
tags:: iOS-Inbox
summary:: CALayer 作为渲染树的节点；模型树管理属性状态；动画树处理动画过渡；渲染树负责最终像素输出
**来源**: 未知　**confidence**: 0.70

- CALayer 作为渲染树的节点
- 模型树管理属性状态
- 动画树处理动画过渡
- 渲染树负责最终像素输出

### 整理后内容

模型树、动画树、渲染树（节点是 CALayer）。

<details><summary>原文</summary>

模型树、动画树、渲染树（节点是CALayer）

</details>

---
