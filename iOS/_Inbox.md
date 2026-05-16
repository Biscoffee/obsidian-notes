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



## 2026-05-16 13:42:29 列出所有主题分类 ^2f5608
topic:: iOS-Inbox
date:: 2026-05-16 13:42:29
source:: 未知
confidence:: 0.80
tags:: iOS-Inbox
summary:: 用户请求列出所有预定义的主题（topic）分类；分类涵盖 iOS 技术栈（Mach-O、Runtime、dyld 等）和 AI 相关主题；也包括面经、项目经验和完全不属于 iOS/AI 的内容类别
**来源**: 未知　**confidence**: 0.80

- 用户请求列出所有预定义的主题（topic）分类
- 分类涵盖 iOS 技术栈（Mach-O、Runtime、dyld 等）和 AI 相关主题
- 也包括面经、项目经验和完全不属于 iOS/AI 的内容类别

### 整理后内容

列出所有 topic

<details><summary>原文</summary>

列出所有 topic

</details>

---
