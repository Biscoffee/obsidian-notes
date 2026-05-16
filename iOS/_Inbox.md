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
