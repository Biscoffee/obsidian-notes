# 【iOS】Runtime - Part 2 && 消息发送：缓存、查找与转发

> 基于 objc4-951.1。延续 Part 1（对象/类/元类/isa 走位）。本篇打通一条完整消息链：快速路径（cache 命中）→ 慢速查找（`lookUpImpOrForward`）→ 动态解析 → 消息转发三部曲 → 崩溃。

## 目录

### 第一部分 · 快速路径（缓存命中）
- **1. 方法调用的本质**：`[obj foo]` → `objc_msgSend(obj, sel)`
- **2. 为什么 objc_msgSend 用汇编写**：性能 / 未知参数 / 尾调用
- **3. 快速路径汇编逐段**（`objc-msg-arm64.s`）
  - 3.1 取 isa → class（`GetClassFromIsa`，呼应 ISA_MASK）
  - 3.2 CacheLookup：按 SEL 哈希定位 bucket
  - 3.3 命中 `br IMP` / 未命中转入慢速查找
- **4. cache_t 数据结构**（`objc-runtime-new.h:337`）
  - 4.1 `_bucketsAndMaybeMask`：指针与 mask 的融合
  - 4.2 `bucket_t`：`{SEL, IMP}` 与 ptrauth
  - 4.3 哈希与开放寻址探测（`cache_hash` / `cache_next`）
- **5. 缓存的写入与扩容**（`objc-cache.mm`）
  - 5.1 `insert`：哈希落位（:873）
  - 5.2 `reallocate`：翻倍扩容、丢弃旧表不迁移（:809）
  - 5.3 装填因子与留空策略

### 第二部分 · 慢速查找（lookUpImpOrForward）
- **6. 入口**：加锁 + realize / +initialize（:7624）
- **7. 在当前类找方法**
  - 7.1 `getMethodNoSuper_nolock` 遍历方法列表（:7290）
  - 7.2 已排序表二分查找 `findMethodInSortedMethodList`（:7062）
- **8. 沿 superclass 链逐层上溯**（每层先查父类 cache）
- **9. 找到了**：`log_and_fill_cache` 回填缓存，闭环回快速路径（:7513）
- **10. 没找到**：动态方法解析（`resolveInstanceMethod`，:7480 / :7430）

### 第三部分 · 消息转发
- **11. 转发入口**：`_objc_msgForward_impcache`（:7626 / 汇编 :779）
- **12. 转发三部曲**
  - 12.1 快速转发 `forwardingTargetForSelector:`（换 receiver 重发）
  - 12.2 完整转发 `methodSignatureForSelector:` + `forwardInvocation:`
  - 12.3 兜底崩溃 `doesNotRecognizeSelector:`（`NSObject.mm:2578`）
- **13. 边界澄清**：objc4 只到 `_objc_msgForward`，三部曲调度在 CoreFoundation `___forwarding___`（闭源）

### 第四部分 · 实战与收尾
- **14. LLDB 实战**：cache 从空到有 / 命中对比 / 慢速断点 / 转发链逐站拦截
- **15. 收尾**：一条消息的命运 + 与 Part 1 的呼应 + 承接下一篇


```html-embed
objc_msgSend_full_pipeline.html
```
