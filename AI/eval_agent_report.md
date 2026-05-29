# 实验1 · Agent 评测体系（全库）

生成时间：2026-05-29 23:04　裁判模型：MiMo　评分尺度：Opus 设计

- 总题数：1792
- 有效评分：1367
- agent 报错：411
- 裁判无效（空返回/解析失败）：14
- 裁判累计 token：6,342,812

## 总分

**综合平均 overall：4.56 / 5**

| 维度 | 平均分 |
|---|---|
| 准确性 | 4.55 |
| 完整性 | 4.26 |
| 相关性 | 4.68 |
| 清晰度 | 4.75 |

## 按来源分布

| source | 题数 | 平均分 |
|---|---|---|
| iosqa | 203 | 4.64 |
| merged | 97 | 4.58 |
| jianshu:693ec962b3d3 | 50 | 4.75 |
| jianshu:90f9348b2ed6 | 48 | 4.56 |
| jianshu:cc9d286a1a27 | 48 | 4.44 |
| obsidian:1122.md | 44 | 3.25 |
| jianshu:4f18226705ec | 38 | 4.71 |
| agentqa | 31 | 4.98 |
| jianshu:2953e86db051 | 31 | 4.73 |
| jianshu:25bcb6540045 | 26 | 4.58 |
| jianshu:89ab04a91cbc | 25 | 4.77 |
| jianshu:2b660ec22fe0 | 24 | 4.65 |
| jianshu:873e530a0995 | 24 | 4.56 |
| jianshu:496af9592d27 | 22 | 4.82 |
| jianshu:5ddd62fdaea9 | 21 | 4.64 |
| jianshu:494629e92692 | 20 | 4.11 |
| jianshu:a6af821a2806 | 20 | 4.75 |
| jianshu:5447a66d955f | 19 | 4.83 |
| jianshu:2f7a1fb420d3 | 18 | 4.99 |
| jianshu:b72018e88a97 | 17 | 3.85 |
| jianshu:5c83da126b48 | 17 | 4.87 |
| jianshu:f306adf3480d | 16 | 4.81 |
| jianshu:358ea0945978 | 16 | 4.89 |
| jianshu:1f10795c9468 | 16 | 4.62 |
| jianshu:7fd6241a7124 | 15 | 4.37 |
| jianshu:d4baff644ce5 | 15 | 4.45 |
| jianshu:2fae148f015f | 14 | 4.23 |
| jianshu:f7d9f6d86145 | 14 | 4.39 |
| jianshu:b838f04a9249 | 14 | 4.98 |
| jianshu:853ca8318a15 | 14 | 4.32 |
| jianshu:fe30ef8bd411 | 14 | 4.57 |
| jianshu:188b089d2617 | 14 | 5.00 |
| jianshu:a63fb211f7ac | 14 | 4.98 |
| jianshu:cb2b9e2b68d1 | 14 | 4.75 |
| memory:user_learning_resources_llm_agent.md | 14 | 3.57 |
| jianshu:8bfd70a9d1ac | 13 | 4.65 |
| jianshu:5d35a384574d | 13 | 4.52 |
| jianshu:92a581bd9fdb | 13 | 4.12 |
| obsidian:111.md | 13 | 4.31 |
| jianshu:94b6998c6038 | 12 | 4.88 |
| jianshu:7ca16c92ca37 | 12 | 4.40 |
| jianshu:e5a54813b93d | 11 | 4.95 |
| jianshu:d488b0bf3aaf | 11 | 4.70 |
| memory:user_knowledge_tier_practice.md | 11 | 4.00 |
| jianshu:db765ff4e36a | 10 | 4.78 |
| memory:user_knowledge_tier_awareness.md | 10 | 4.47 |
| memory:user_learning_path_llm_to_agent.md | 10 | 2.88 |
| jianshu:bc16a644784d | 9 | 4.11 |
| jianshu:3a3e75af36a7 | 9 | 4.81 |
| jianshu:6b5ef101cf66 | 9 | 4.92 |
| jianshu:e0ececa61e8e | 7 | 4.86 |
| jianshu:3ad9166c02e5 | 7 | 4.82 |
| jianshu:ab8c754761bf | 7 | 4.43 |
| jianshu:cb137ba5a0e7 | 7 | 4.21 |
| jianshu:4f90ffb873ab | 6 | 5.00 |
| jianshu:11b7d19b02b6 | 5 | 4.15 |
| memory:user_knowledge_tier_understanding.md | 5 | 4.50 |
| jianshu:ee6a8ebc5bec | 4 | 4.00 |
| mikeash/2009-03-13-intro-to-the-objective-c-runtime | 4 | 4.94 |
| mikeash/2017-06-30-dissecting-objc_msgsend-on-arm64 | 3 | 4.92 |
| mikeash/2014-05-23-a-heartbleed-inspired-paranoid-memory-allocator | 3 | 5.00 |
| mikeash/2012-01-20-fork-safety | 3 | 4.42 |
| mikeash/2011-01-04-practical-floating-point | 3 | 4.67 |
| mikeash/2011-02-18-compound-literals | 3 | 5.00 |
| mikeash/2011-12-23-disassembling-the-assembly-part-2 | 3 | 5.00 |
| mikeash/2011-12-30-disassembling-the-assembly-part-3-arm-edition | 3 | 3.50 |
| mikeash/2012-08-24-things-you-never-wanted-to-know-about-c | 3 | 4.83 |
| mikeash/2012-12-28-what-happens-when-you-load-a-byte-of-memory | 3 | 4.92 |
| mikeash/2013-05-31-c-quiz | 3 | 4.92 |
| mikeash/2013-10-11-why-registers-are-fast-and-ram-is-slow | 3 | 5.00 |
| mikeash/2009-03-20-objective-c-messaging | 2 | 4.88 |
| mikeash/2009-03-27-objective-c-message-forwarding | 2 | 5.00 |
| mikeash/2009-05-22-objective-c-class-loading-and-initialization | 2 | 5.00 |
| mikeash/2010-04-09-comparison-of-objective-c-enumeration-techniques | 2 | 4.75 |
| mikeash/2012-07-06-lets-build-nsnumber | 2 | 4.25 |
| mikeash/2012-11-16-lets-build-objc_msgsend | 2 | 4.88 |
| mikeash/2013-03-08-lets-build-nsinvocation-part-i | 2 | 4.88 |
| mikeash/2013-10-25-nsobject-the-class-and-the-protocol | 2 | 5.00 |
| mikeash/2009-08-28-intro-to-grand-central-dispatch-part-i-basics-and-dispatch-queues | 2 | 5.00 |
| mikeash/2011-03-04-a-tour-of-osatomic | 2 | 5.00 |
| mikeash/2015-02-06-locks-thread-safety-and-swift | 2 | 4.88 |
| mikeash/2014-06-06-secrets-of-dispatch_once | 2 | 5.00 |
| mikeash/2015-02-20-lets-build-synchronized | 2 | 4.62 |
| mikeash/2015-09-04-lets-build-dispatch_queue | 2 | 5.00 |
| mikeash/2017-10-27-locks-thread-safety-and-swift-2017-edition | 2 | 5.00 |
| mikeash/2011-10-28-generic-block-proxying | 2 | 3.75 |
| mikeash/2011-12-16-disassembling-the-assembly-part-1 | 2 | 5.00 |
| mikeash/2012-02-03-ring-buffers-and-mirrored-memory-part-i | 2 | 4.38 |
| mikeash/2012-02-17-ring-buffers-and-mirrored-memory-part-ii | 2 | 4.25 |
| mikeash/2012-11-30-lets-build-a-mach-o-executable | 2 | 4.50 |
| mikeash/2012-11-09-dyld-dynamic-linking-on-os-x | 2 | 3.00 |
| mikeash/2013-09-27-arm64-and-you | 2 | 4.62 |
| mikeash/2010-12-17-custom-object-allocators-in-objective-c | 2 | 5.00 |
| mikeash/2012-03-02-key-value-observing-done-right-take-2 | 2 | 4.88 |
| mikeash/2013-01-25-lets-build-nsobject | 1 | 3.50 |
| mikeash/2013-02-08-lets-build-key-value-coding | 1 | 5.00 |
| mikeash/2010-04-30-dealing-with-retain-cycles | 1 | 5.00 |
| mikeash/2015-07-31-tagged-pointer-strings | 1 | 3.75 |
| mikeash/2010-07-16-zeroing-weak-references-in-objective-c | 1 | 5.00 |
| mikeash/2012-06-01-a-tour-of-plweakcompatibility-part-ii | 1 | 5.00 |
| mikeash/2009-09-18-intro-to-grand-central-dispatch-part-iv-odds-and-ends | 1 | 5.00 |
| mikeash/2015-05-29-concurrent-memory-deallocation-in-the-objective-c-runtime | 1 | 1.00 |
| mikeash/2009-09-11-intro-to-grand-central-dispatch-part-iii-dispatch-sources | 1 | 4.75 |
| mikeash/2011-04-01-signal-handling | 1 | 5.00 |
| mikeash/2013-01-11-mach-exception-handlers | 1 | 5.00 |
| mikeash/2010-01-29-method-replacement-for-fun-and-profit | 1 | 5.00 |
| mikeash/2012-07-27-lets-build-tagged-pointers | 1 | 4.75 |
| mikeash/2011-05-20-the-inner-life-of-zombies | 1 | 5.00 |
| mikeash/2010-07-30-zeroing-weak-references-to-corefoundation-objects | 1 | 5.00 |
| mikeash/2011-09-30-automatic-reference-counting | 1 | 5.00 |
| mikeash/2014-11-07-lets-build-nszombie | 1 | 5.00 |
| mikeash/2009-09-04-intro-to-grand-central-dispatch-part-ii-multi-core-performance | 1 | 5.00 |
| mikeash/2009-09-25-gcd-practicum | 1 | 5.00 |
| mikeash/2011-10-14-whats-new-in-gcd | 1 | 5.00 |
| mikeash/2011-06-03-objective-c-blocks-vs-c0x-lambdas-fight | 1 | 5.00 |

## 分数档分布

| 档位 | 题数 |
|---|---|
| 5 分档 | 1070 |
| 4 分档 | 178 |
| 3 分档 | 20 |
| 2 分档 | 16 |
| 1 分档 | 83 |

## 失败清单（overall < 3，共 103 条，列前 60）

- [iosqa] **const（常量）**（1.00）　待评回答是错误消息，未提供任何相关内容。
- [iosqa] **调度组：用 dispatch_group_async () 把要先做的任务塞进这个组里，让它们自己跑，最后用 dispatch_group_notify () 告诉系统：&quot;等组里所有任务都跑完了，就执行后面这个新任务&quot;。2. 信号量：先调 create (0) 弄个信号量（初始状态是没钥匙），让前一个任务先运行，运行完的时候调用signal ()，后一个任务开始前，先调 wait ()。**（1.00）　待评回答未回答问题，仅表示内容已存储。
- [iosqa] **QOS_CLASS_USER_INTERACTIVE（用户交互级）最高优先级，直接影响用户界面响应（比如滑动列表、动画渲染），必须快速完成（通常毫秒级）。2. QOS_CLASS_USER_INITIATED（用户发起级）用户主动触发的任务（比如点击按钮后的网络请求），需要尽快完成（通常秒级内），用户会等待结果。3. QOS_CLASS_DEFAULT（默认级）没指定时的默认值，系统会根据上下文自动调整（建议显式指定其他等级，避免依赖默认值）。4. QOS_CLASS_UTILITY（工具级）耗时较长但用户不急需结果的任务（比如下载文件、数据解析），可以显示进度条，系统可能限制其 CPU 占用以节省电量。5. QOS_CLASS_BACKGROUND（后台级）完全不影响用户的任务（比如备份数据、同步日志），系统会在空闲时执行，优先级最低。**（1.00）　回答是拒答，未提供任何相关信息。
- [iosqa] **调用方式不同**（1.00）　待评回答未提供实质答案，仅请求澄清问题
- [iosqa] **UICollecitonVie的调用顺序**（1.00）　待评回答拒答，未回答问题。
- [iosqa] **UICollectionView的Layout和FlowLayout**（1.00）　待评回答是系统提示，未提供任何答案内容。
- [iosqa] **讲NSURLSession**（1.00）　待评回答是拒答，没有提供任何相关信息，完全未回答问题。
- [iosqa] **数据库语法**（1.00）　待评回答未提供任何语法信息，仅询问澄清，未回答问题。
- [iosqa] **存储方式**（1.00）　待评回答未提供参考要点内容，要求澄清而非回答，类似拒答。
- [iosqa] **功能特性**（1.00）　待评回答未提供任何答案，而是请求具体化问题，类似拒答。
- [merged] **Objective-C中isKindOfClass与isMemberOfClass原理与对比**（1.00）　待评回答为拒答，未提供任何相关信息。
- [jianshu:11b7d19b02b6] **Apple开源源码下载地址**（1.00）　待评回答为空，未提供任何有效信息。
- [jianshu:b72018e88a97] **callAlloc中的编译器优化判断**（1.00）　待评回答拒答，未提供任何实质内容。
- [jianshu:b72018e88a97] **为什么需要16字节对齐**（1.00）　待评回答是拒答消息，未提供任何问题相关内容。
- [jianshu:b72018e88a97] **实例方法init的实现**（1.00）　待评回答没有提供答案，属于拒答。
- [jianshu:2fae148f015f] **Base SDK错误解决**（1.00）　待评回答完全没有回答问题，而是输出无关的工具说明
- [jianshu:2b660ec22fe0] **输出键值pKey的计算**（1.00）　回答未提供计算信息，仅要求补充上下文，未覆盖参考要点。
- [jianshu:7fd6241a7124] **isa位域功能详解**（1.00）　待评回答为拒答，未回答任何问题。
- [jianshu:7fd6241a7124] **magic位域用途**（1.00）　回答是系统提示，非实质回答。
- [jianshu:873e530a0995] **CJLPerson与CJLTeacher的定义示例**（1.00）　待评回答拒答，未提供任何相关信息。
- [jianshu:bc16a644784d] **objc_setProperty函数的优化选择逻辑**（1.00）　待评回答为系统拒答提示，未回答问题，四维均不达标。
- [jianshu:bc16a644784d] **weak属性在底层的调用函数**（1.00）　待评回答未回答问题，要求更多信息，视为拒答。
- [jianshu:f7d9f6d86145] **父类缓存查找逻辑**（1.00）　回答为系统拒答，未提供任何相关信息。
- [jianshu:853ca8318a15] **结构体LGStruct2大小计算**（1.00）　待评回答拒答，未提供任何具体答案，类似知识库无内容回应。
- [jianshu:853ca8318a15] **结构体LGStruct1大小计算**（1.00）　回答拒答具体问题，只提供通用规则，未覆盖参考要点。
- [jianshu:ee6a8ebc5bec] **Launch Closure 的生成与缓存策略**（1.00）　待评回答为拒答，未提供任何相关信息。
- [jianshu:25bcb6540045] **dyld与objc的关联目的**（1.00）　待评回答表示未看到文章内容，属于拒答，未回答问题。
- [jianshu:25bcb6540045] **三个回调参数与ObjC函数的等价关系**（1.00）　待评回答是拒答，没有提供任何问题相关信息。
- [jianshu:90f9348b2ed6] **修复消息引用（objc_msgSend_fixup）**（1.00）　待评回答是系统错误消息，未回答问题。
- [jianshu:90f9348b2ed6] **headerIsPreoptimized与类加载判断**（1.00）　待评回答未提供任何答案，属于拒答。
- [jianshu:90f9348b2ed6] **set_ro源码实现与ro获取**（1.00）　待评回答为工具调用上限错误，未提供任何答案。
- [jianshu:90f9348b2ed6] **attachLists情况3：一对多**（1.00）　待评回答为拒答消息，未提供任何操作步骤，答非所问。
- [jianshu:ab8c754761bf] **属性值验证API**（1.00）　待评回答是拒答消息，没有回答问题。
- [jianshu:2953e86db051] **NSThread定义与特点**（1.00）　待评回答是拒答，未提供任何实质性内容。
- [jianshu:2953e86db051] **dispatch_group_t的应用场景**（1.00）　回答是拒答，没有提供任何信息。
- [jianshu:693ec962b3d3] **dispatch_group_wake 中的通知执行**（1.00）　回答是拒答，未提供任何实质性答案。
- [jianshu:494629e92692] **Demo 中线程 1 的行为**（1.00）　待评回答拒答，未提供任何关于线程1执行流程的信息。
- [jianshu:494629e92692] **Demo 中线程 3 的行为**（1.00）　回答是拒答，没有提供线程3执行流程的信息。
- [jianshu:494629e92692] **Demo 中线程 2 的行为**（1.00）　回答未提供任何执行流程信息，而是请求更多细节，相当于拒答。
- [jianshu:494629e92692] **Demo 的最终打印顺序及原因**（1.00）　待评回答拒答，未提供任何答案。
- [jianshu:fe30ef8bd411] **全局Block的条件**（1.00）　待评回答为空，未提供任何内容。
- [jianshu:4f18226705ec] **配置MatchFinder绑定节点标识**（1.00）　待评回答未提供任何实质性信息，属于拒答。
- [jianshu:5d35a384574d] **OC类数量对启动的影响**（1.00）　待评回答是系统提示，未回答问题。
- [jianshu:d4baff644ce5] **生产环境 Page Fault 额外开销**（1.00）　待评回答是工具调用限制的提示，未回答问题。
- [jianshu:d4baff644ce5] **SanitizerCoverage 避免死循环配置**（1.00）　待评回答是拒答，没有提供任何相关信息。
- [jianshu:5ddd62fdaea9] **dealloc 流程完整性**（1.00）　待评回答为系统拒答，未提供任何答案，相当于未回答问题。
- [jianshu:cc9d286a1a27] **hotPage函数作用**（1.00）　待评回答拒答，未提供任何功能信息。
- [jianshu:cc9d286a1a27] **popPage模板函数的作用**（1.00）　拒答，未提供任何答案。
- [jianshu:cc9d286a1a27] **popPage中的调试分支**（1.00）　待评回答表示无法依据笔记回答，未提供任何答案内容。
- [jianshu:cc9d286a1a27] **popPage中删除空子页的逻辑**（1.00）　待评回答是拒答，要求更多上下文而非提供答案。
- [jianshu:cb137ba5a0e7] **测试代码的作用**（1.00）　回答承认未找到具体方法内容，类似拒答，未提供问题答案。
- [obsidian:1122.md] **fetch_url_raw 工具功能**（1.00）　回答拒答 fetch_url_raw 问题，没有提供相关信息。
- [obsidian:1122.md] **Harness 组件装配的开关机制**（1.00）　待评回答拒答，未提供任何问题答案。
- [obsidian:1122.md] **Demo 11 反馈循环实验的核心发现**（1.00）　拒答，未提供任何答案内容。
- [obsidian:1122.md] **companion 真实痛点分析**（1.00）　待评回答未回答问题，仅要求澄清，未覆盖任何参考要点。
- [obsidian:1122.md] **实验代码落地的核心经验**（1.00）　待评回答未回答问题，而是询问用户意图，类似拒答。
- [memory:user_knowledge_tier_practice.md] **上下文溢出问题**（1.00）　待评回答拒答，未提供任何关于实验验证的信息。
- [memory:user_learning_path_llm_to_agent.md] **W2检查清单**（1.00）　回答拒答，未提供任何相关信息。
- [memory:user_learning_path_llm_to_agent.md] **W8检查清单**（1.00）　回答拒答，未提供任何第8周学习检查清单的实质内容。
- [memory:user_learning_resources_llm_agent.md] **Transformer原始论文的定位**（1.00）　回答拒答，未提供论文定位信息。

## agent 报错（411 条）

- [jianshu:853ca8318a15] 8字节对齐算法：__AGENT_ERROR__ RuntimeError: Could not determine home directory.
- [jianshu:cc9d286a1a27] 添加 Timer 到 RunLoop 的逻辑：__AGENT_ERROR__ RateLimitError: Error code: 429 - {'error': {'code': '429', 'message': 'Too many requests', 'type': 'lim
- [jianshu:cc9d286a1a27] Timer 的执行总结：__AGENT_ERROR__ RateLimitError: Error code: 429 - {'error': {'code': '429', 'message': 'Too many requests', 'type': 'lim
- [jianshu:cc9d286a1a27] RunLoop 底层原理调用栈：__AGENT_ERROR__ RateLimitError: Error code: 429 - {'error': {'code': '429', 'message': 'Too many requests', 'type': 'lim
- [jianshu:cc9d286a1a27] CFRunLoopRunSpecific 主流程：__AGENT_ERROR__ RateLimitError: Error code: 429 - {'error': {'code': '429', 'message': 'Too many requests', 'type': 'lim
- [jianshu:cc9d286a1a27] RunLoop Observer 通知时机：__AGENT_ERROR__ RateLimitError: Error code: 429 - {'error': {'code': '429', 'message': 'Too many requests', 'type': 'lim
- [jianshu:cc9d286a1a27] RunLoop 终止条件：__AGENT_ERROR__ RateLimitError: Error code: 429 - {'error': {'code': '429', 'message': 'Too many requests', 'type': 'lim
- [jianshu:cc9d286a1a27] RunLoop Mode 的作用：__AGENT_ERROR__ RateLimitError: Error code: 429 - {'error': {'code': '429', 'message': 'Too many requests', 'type': 'lim
- [jianshu:7ca16c92ca37] 组件化的定义：__AGENT_ERROR__ RateLimitError: Error code: 429 - {'error': {'code': '429', 'message': 'Too many requests', 'type': 'lim
- [jianshu:7ca16c92ca37] 组件化原则中的分层：__AGENT_ERROR__ RateLimitError: Error code: 429 - {'error': {'code': '429', 'message': 'Too many requests', 'type': 'lim
- [jianshu:7ca16c92ca37] 必须考虑组件化的项目特征：__AGENT_ERROR__ RateLimitError: Error code: 429 - {'error': {'code': '429', 'message': 'Too many requests', 'type': 'lim
- [jianshu:1f10795c9468] CTMediator safePerformAction实现：__AGENT_ERROR__ RateLimitError: Error code: 429 - {'error': {'code': '429', 'message': 'Too many requests', 'type': 'lim
- [jianshu:1f10795c9468] target-action方案的优缺点：__AGENT_ERROR__ RateLimitError: Error code: 429 - {'error': {'code': '429', 'message': 'Too many requests', 'type': 'lim
- [jianshu:1f10795c9468] BeeHive从Plist加载模块：__AGENT_ERROR__ RateLimitError: Error code: 429 - {'error': {'code': '429', 'message': 'Too many requests', 'type': 'lim
- [jianshu:1f10795c9468] createService方法调用链：__AGENT_ERROR__ RateLimitError: Error code: 429 - {'error': {'code': '429', 'message': 'Too many requests', 'type': 'lim
- [jianshu:1f10795c9468] 通过load方法注册Protocol：__AGENT_ERROR__ RateLimitError: Error code: 429 - {'error': {'code': '429', 'message': 'Too many requests', 'type': 'lim
- [jianshu:1f10795c9468] serviceImplClass方法：Protocol到类的映射：__AGENT_ERROR__ RateLimitError: Error code: 429 - {'error': {'code': '429', 'message': 'Too many requests', 'type': 'lim
- [jianshu:1f10795c9468] BHTimeProfiler类的用途：__AGENT_ERROR__ RateLimitError: Error code: 429 - {'error': {'code': '429', 'message': 'Too many requests', 'type': 'lim
- [jianshu:1f10795c9468] BHContext类的作用与属性：__AGENT_ERROR__ RateLimitError: Error code: 429 - {'error': {'code': '429', 'message': 'Too many requests', 'type': 'lim
- [jianshu:cb137ba5a0e7] 异常处理的两大类型：__AGENT_ERROR__ RateLimitError: Error code: 429 - {'error': {'code': '429', 'message': 'Too many requests', 'type': 'lim
- [jianshu:cb137ba5a0e7] Mach异常EXC_BAD_INSTRUCTION的说明：__AGENT_ERROR__ RateLimitError: Error code: 429 - {'error': {'code': '429', 'message': 'Too many requests', 'type': 'lim
- [jianshu:cb137ba5a0e7] 硬件异常的流转流程：__AGENT_ERROR__ RateLimitError: Error code: 429 - {'error': {'code': '429', 'message': 'Too many requests', 'type': 'lim
- [jianshu:cb137ba5a0e7] 监控策略的分类：__AGENT_ERROR__ RateLimitError: Error code: 429 - {'error': {'code': '429', 'message': 'Too many requests', 'type': 'lim
- [jianshu:cb137ba5a0e7] 自定义 dealloc 中的延迟释放：__AGENT_ERROR__ RateLimitError: Error code: 429 - {'error': {'code': '429', 'message': 'Too many requests', 'type': 'lim
- [jianshu:cb137ba5a0e7] __sync_fetch_and_add 原子操作：__AGENT_ERROR__ RateLimitError: Error code: 429 - {'error': {'code': '429', 'message': 'Too many requests', 'type': 'lim
- [jianshu:cb137ba5a0e7] init_safe_free 函数的作用：__AGENT_ERROR__ RateLimitError: Error code: 429 - {'error': {'code': '429', 'message': 'Too many requests', 'type': 'lim
- [jianshu:cb137ba5a0e7] 白名单的实现：__AGENT_ERROR__ RateLimitError: Error code: 429 - {'error': {'code': '429', 'message': 'Too many requests', 'type': 'lim
- [jianshu:cb137ba5a0e7] hook dealloc 的核心流程：__AGENT_ERROR__ RateLimitError: Error code: 429 - {'error': {'code': '429', 'message': 'Too many requests', 'type': 'lim
- [jianshu:cb137ba5a0e7] 测试用例中的属性声明：__AGENT_ERROR__ RateLimitError: Error code: 429 - {'error': {'code': '429', 'message': 'Too many requests', 'type': 'lim
- [jianshu:5447a66d955f] dSYMs文件夹的作用：__AGENT_ERROR__ RateLimitError: Error code: 429 - {'error': {'code': '429', 'message': 'Too many requests', 'type': 'lim

## 怎么读这份报告

- **均分**反映 agent 在自己知识库题目上的综合表现。
- **按来源**看哪类知识 agent 答得差（检索没召回 or 综合能力弱）。
- **失败清单**是 Harness 反馈循环的输入：逐条看是检索没命中、答错、还是跑题，再针对性改 ask_kb 检索 / system prompt。
- 裁判尺度由 Opus 设计，跑完已用 `--export` 抽检校准（见同目录抽检报告）。