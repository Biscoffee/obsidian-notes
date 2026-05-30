# 实验1 失败题 · 根因归类（Harness 反馈循环输入）

生成时间：2026-05-30 18:51　分类器：MiMo　分类尺度：Opus 设计　token：575,228

对实验1 中 overall<3 的 575 道失败题做根因归类（agent 直接报错的 1 题另计，不在此分类）。

## 失败类型分布

| 类型 | 题数 | 占比 |
|---|---|---|
| 拒答 | 513 | 89.2% |
| 跑题 | 34 | 5.9% |
| 事实错误 | 4 | 0.7% |
| 漏要点 | 21 | 3.7% |
| 其他 | 3 | 0.5% |

## 各类型的来源分布（哪类知识最容易栽在这）

- **拒答**（513）：obsidian:1122.md×76, iosqa×61, merged×56, obsidian:111.md×21, jianshu:cc9d286a1a27×19
- **跑题**（34）：iosqa×10, obsidian:1122.md×3, merged×2, jianshu:7ca16c92ca37×2, jianshu:d488b0bf3aaf×2
- **事实错误**（4）：jianshu:4f18226705ec×1, obsidian:1122.md×1, memory:user_learning_resources_llm_agent.md×1, jianshu:2b660ec22fe0×1
- **漏要点**（21）：obsidian:1122.md×6, iosqa×3, jianshu:e0ececa61e8e×2, merged×2, jianshu:693ec962b3d3×1
- **其他**（3）：obsidian:1122.md×2, obsidian:111.md×1

## 各类型样例（每类最多 8 条）

### 拒答
- [iosqa] 讲一讲iOS的内存管理（引用计数+MRC+ARC + AutoreleasePool + TagP）（1.0）：回答未提供实质内容，仅说明输出格式问题。
- [iosqa] TaggedPointer（1.0）：agent因工具调用上限停止，未提供实质内容。
- [iosqa] iOS内存机制（内存压缩机制）（1.0）：回答直接说知识库无结果，未提供任何实质内容。
- [iosqa] 子线程默认不会开启 Runloop，那出现 Autorelease 对象如何处理？不手动处理会内存泄漏吗？（1.0）：回答仅为错误提示，未提供任何实质内容。
- [iosqa] 如何解决Block的循环引用（1.0）：回答未提供实质内容，仅报告输出故障。
- [iosqa] mutableString 使用 copy 关键字修饰会发生什么？（1.0）：回答表示模型输出格式错误，未提供任何实质内容。
- [iosqa] const（常量）（1.2）：agent未回答问题，仅询问用户需求，无实质内容。
- [iosqa] #define（1.0）：agent未解释#define，反问用户意图，无实质内容。

### 跑题
- [iosqa] 哨兵节点的作用（2.5）：回答聚焦通用数据结构哨兵节点，未针对autoreleasepool场景解释分隔作用。
- [iosqa] extern（1.0）：agent未解释extern，反而询问用户意图，答非所问。
- [iosqa] 通知的一些特点总结（1.0）：agent未回答问题，反而要求澄清，答非所问。
- [iosqa] 多读单写（2.0）：agent猜测为Copy-on-Write并询问归档，未涉及参考要点。
- [iosqa] 优先级反转（1.0）：agent询问用户意图，未解释优先级反转概念
- [iosqa] 先调用主类的+load（1.0）：agent询问用户意图，未提供+load调用顺序的实质信息。
- [iosqa] 模型树、动画树、渲染树（节点是CALayer）（1.0）：回答讨论笔记归档，未涉及模型树等概念。
- [iosqa] delaysTouchesBegan（默认 NO）（1.0）：回答转向归档讨论，未解释属性含义。

### 事实错误
- [jianshu:4f18226705ec] 检查节点有效性与用户源代码（2.5）：提供的两项检查与参考要点不一致，如类型检查而非指针有效性。
- [obsidian:1122.md] 上下文管理失败的真实驱动因素（2.5）：驱动因素应为长工具结果塞栈，而非机制错位。
- [memory:user_learning_resources_llm_agent.md] Transformer原始论文的定位（2.0）：agent错误地将论文定位为'基础必读'，与参考要点'了解层可跳过'矛盾。
- [jianshu:2b660ec22fe0] size小于阈值时的处理（2.5）：agent说size被向上取整，与参考要点设置为NANO_REGIME_QUANTA_SIZE矛盾。

### 漏要点
- [iosqa] QOS_CLASS_USER_INTERACTIVE（用户交互级）最高优先级，直接影响用户界面响应（比如滑动列表、动画渲染），必须快速完成（通常毫秒级）。2. QOS_CLASS_USER_INITIATED（用户发起级）用户主动触发的任务（比如点击按钮后的网络请求），需要尽快完成（通常秒级内），用户会等待结果。3. QOS_CLASS_DEFAULT（默认级）没指定时的默认值，系统会根据上下文自动调整（建议显式指定其他等级，避免依赖默认值）。4. QOS_CLASS_UTILITY（工具级）耗时较长但用户不急需结果的任务（比如下载文件、数据解析），可以显示进度条，系统可能限制其 CPU 占用以节省电量。5. QOS_CLASS_BACKGROUND（后台级）完全不影响用户的任务（比如备份数据、同步日志），系统会在空闲时执行，优先级最低。（2.2）：未提供通过dispatch_set_target_queue绑定QoS等级的具体步骤
- [iosqa] CAlayer和UIView的区别（1.0）：仅提CALayer角色，未覆盖UIView区别及关键要点
- [iosqa] Id instancetype NSObject（1.0）：agent只询问意图，未提供任何概念解释或要点内容。
- [jianshu:e0ececa61e8e] NSObject的alloc为什么不走源码工程（2.8）：未提及编译器优化替换alloc为objc_alloc的关键点。
- [jianshu:e0ececa61e8e] objc_alloc的来源：LLVM编译器优化（2.5）：agent只描述运行时调用链，未涉及LLVM编译器优化这一关键点
- [jianshu:693ec962b3d3] 并发队列推入执行路径（2.5）：agent未提及关键函数_dispatch_lane_concurrent_push。
- [jianshu:cb137ba5a0e7] loadZombieProxyClass 函数（2.5）：未覆盖创建registeredClasses、调用objc_copyClassList等关键初始化步骤。
- [obsidian:1122.md] reasoning_content 计入 max_tokens 预算（2.0）：未提及 reasoning_content 计入 max_tokens 预算的关键点。

### 其他
- [obsidian:111.md] 核心竞争力在于Harness（1.0）：agent未给出实质内容，仅报告输出格式问题。
- [obsidian:1122.md] Hierarchical模式需设终止条件（1.0）：分类器空返回
- [obsidian:1122.md] RAG系统与Sub-Agent的天然契合（1.0）：分类器空返回

## 行动建议（喂回 Harness）

- 占比最高的是 **拒答**（513/575）：agent 在有知识时仍说「没有」——查检索阈值/话术约束，或 prompt 太保守，放开「基于检索内容作答」。
- 结合实验4：检索 rank1 占 96%，所以失败基本**不是检索没召回**，而是上面这些**生成端**问题——改 system prompt / 换作答模型比改检索更值。
- 这就是 Demo 11 反馈循环的闭环：评测→归因→定位到具体组件→改→再评。