# 【iOS】Runtime - Part 2 && 消息发送与缓存

> 基于 objc4-951.1。延续 Part 1（对象/类/元类/isa 走位）。本篇只讲**快速路径**：`objc_msgSend` 怎么发消息、`cache_t` 怎么缓存命中的 `IMP`。命中不了走的慢速查找 `lookUpImpOrForward`，留给 Part 3。

---

## 0. 一句话定位

`[obj foo]` 在编译期就被翻成 `objc_msgSend(obj, @selector(foo))`。所谓「方法调用」本质是「发消息」：拿 `receiver` 的类、按 `SEL` 找 `IMP`、跳过去执行。这条链路一天被走几百万次，所以 Apple 把入口 `objc_msgSend` 用**汇编**写，并在类上挂了一层 `cache_t` 做记忆。

- [ ] 实证：`clang -rewrite-objc` 或反汇编看到 `bl _objc_msgSend`
- [ ] 源码锚点：`objc-msg-arm64.s`（`Messengers.subproj/`）

---

## 1. 为什么 objc_msgSend 是汇编写的

三个理由，写正文时各配一句源码/ABI 证据：

1. **性能** —— 它是所有方法调用的必经之路，热到不能有一点多余开销。
2. **未知参数** —— 入口不知道目标方法的参数类型/个数，没法用 C 函数签名声明；汇编里参数原样躺在寄存器/栈上，找到 `IMP` 后直接 `br` 过去，参数一个不动。
3. **尾调用（tail call）** —— 找到 `IMP` 是 `br`（跳转）不是 `bl`（调用），不压新栈帧，`objc_msgSend` 自己不出现在调用栈里。

---

## 2. 快速路径：objc_msgSend 汇编逐段

源码：`Messengers.subproj/objc-msg-arm64.s`，`ENTRY _objc_msgSend`。

- [ ] **2.1 取 isa → class**：`GetClassFromIsa_p16`，处理 nil receiver（直接返回）、tagged pointer、nonpointer isa 掩码（呼应 Part 1 的 `ISA_MASK`）。
- [ ] **2.2 CacheLookup**：宏 `CacheLookup NORMAL`，从 class 里取 `cache`，按 `SEL` 哈希定位 bucket。
- [ ] **2.3 命中 / 未命中**：
  - 命中 → `CacheHit` → `br` 跳到 `IMP`。
  - 未命中 → `__objc_msgSend_uncached` → `MethodTableLookup` → C 函数 `lookUpImpOrForward`（**Part 3 接力**）。

> 这一段是「概念图 → 汇编宏 → 真跳转」三步，正文配一张 Mermaid：isa → cache → (命中 br imp | 未命中 慢速查找)。

---

## 3. cache_t 数据结构

源码：`objc-runtime-new.h:337` `struct cache_t`。

- [ ] **3.1 字段**：`_bucketsAndMaybeMask`（`explicit_atomic<uintptr_t>`）—— 把 **buckets 指针**和 **mask** 塞进一个字，不同架构布局不同（arm64 高位放 mask / x86 分开存 `_mask`+`_occupied`）。还有 `_originalPreoptCache`（呼应 Part 1 preopt cache）。
- [ ] **3.2 bucket_t**：一对 `{ SEL, IMP }`，arm64 与 x86 字段**顺序相反**，且 arm64e 对 imp 做 ptrauth 签名。
- [ ] **3.3 哈希与探测**：`cache_hash(sel, mask) = sel & mask`（`objc-cache.mm:307`）；开放寻址，冲突时 `cache_next` 线性探测（arm64 向下、x86 向上，`:241/:245`）。

---

## 4. 缓存的写入与扩容

源码：`objc-cache.mm`。

- [ ] **4.1 insert**：`cache_t::insert`（`:873`）—— 容量不够先扩容，算哈希落位，写 bucket，`occupied++`。
- [ ] **4.2 扩容**：`reallocate`（`:809`）—— 容量翻倍（起步 `INIT_CACHE_SIZE`=4），**直接丢弃旧 buckets 不迁移**（旧的交给 GC 链表延迟释放，`bad_cache` `:829` 兜底）。写正文时讲清「为什么扩容即清空」：缓存丢了无所谓，大不了重走一次慢速查找再填回来，省去迁移成本。
- [ ] **4.3 装填因子**：扩容时机（occupied 占比阈值）、为什么留空位（开放寻址需要）。

---

## 5. 实战 LLDB

素材已就绪：`msgSend-demo/实战素材.md`（缓存命中/未命中证据）。

- [ ] **5.1 看 cache 从空到有**：对象第一次调某方法前后，打印 `cache_t` 的 `_occupied`，看从 0 变 1。
- [ ] **5.2 dump buckets**：`p (cache_t *)...` → 遍历 buckets 打出 `{sel, imp}`，验证哈希落位。
- [ ] **5.3 命中 vs 未命中**：同一方法连调两次，第二次走 `CacheHit`（可在 `__objc_msgSend_uncached` 下符号断点，看第二次不再触发）。

---

## 6. 收尾 & 承上启下

- 命中：`objc_msgSend` 全程汇编，cache 里抄出 `IMP` 直接 `br`，不进 C、不发真消息。
- 未命中：落到 `lookUpImpOrForward`，开始在方法列表里**慢速查找**、必要时沿 superclass 链上溯——这是 **Part 3** 的主线。
- 三次都没找到：触发**消息转发**——**Part 4**。

---

<!-- 待办：
1. 每个 [ ] 节点逐一下钻源码 + 补 LLDB 实测
2. Mermaid：快速路径流程图、cache 哈希落位示意
3. 与 Part 1 的呼应点：ISA_MASK、preopt cache、元类（类方法走元类 cache）
-->
