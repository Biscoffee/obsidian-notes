# 实验1 · Agent 评测体系（全库）

生成时间：2026-05-28 11:13　裁判模型：MiMo　评分尺度：Opus 设计

- 总题数：1574
- 有效评分：933
- agent 报错：638
- 裁判无效（空返回/解析失败）：3
- 裁判累计 token：1,728,624

## 总分

**综合平均 overall：4.26 / 5**

| 维度 | 平均分 |
|---|---|
| 准确性 | 4.25 |
| 完整性 | 3.93 |
| 相关性 | 4.34 |
| 清晰度 | 4.50 |

## 按来源分布

| source | 题数 | 平均分 |
|---|---|---|
| iosqa | 154 | 3.83 |
| jianshu:cc9d286a1a27 | 53 | 3.97 |
| jianshu:693ec962b3d3 | 51 | 4.33 |
| jianshu:90f9348b2ed6 | 48 | 4.51 |
| merged | 41 | 4.18 |
| jianshu:4f18226705ec | 38 | 4.53 |
| agentqa | 31 | 4.30 |
| jianshu:2953e86db051 | 31 | 4.73 |
| jianshu:25bcb6540045 | 26 | 4.45 |
| jianshu:873e530a0995 | 25 | 4.05 |
| jianshu:89ab04a91cbc | 25 | 4.20 |
| jianshu:2b660ec22fe0 | 24 | 4.40 |
| jianshu:496af9592d27 | 23 | 4.70 |
| jianshu:5ddd62fdaea9 | 21 | 4.70 |
| jianshu:494629e92692 | 20 | 4.10 |
| jianshu:2f7a1fb420d3 | 18 | 4.69 |
| jianshu:f306adf3480d | 16 | 4.50 |
| jianshu:358ea0945978 | 16 | 4.12 |
| jianshu:b72018e88a97 | 15 | 3.82 |
| jianshu:7fd6241a7124 | 15 | 4.38 |
| jianshu:2fae148f015f | 14 | 3.68 |
| jianshu:f7d9f6d86145 | 14 | 4.38 |
| jianshu:b838f04a9249 | 14 | 4.43 |
| jianshu:853ca8318a15 | 14 | 4.07 |
| jianshu:fe30ef8bd411 | 14 | 4.79 |
| jianshu:188b089d2617 | 14 | 4.52 |
| jianshu:a63fb211f7ac | 14 | 4.91 |
| jianshu:8bfd70a9d1ac | 13 | 4.27 |
| jianshu:5d35a384574d | 13 | 4.27 |
| jianshu:d4baff644ce5 | 13 | 3.33 |
| jianshu:94b6998c6038 | 12 | 4.50 |
| jianshu:e5a54813b93d | 11 | 4.95 |
| jianshu:bc16a644784d | 9 | 3.83 |
| jianshu:db765ff4e36a | 9 | 4.25 |
| jianshu:3a3e75af36a7 | 9 | 4.53 |
| jianshu:6b5ef101cf66 | 8 | 4.88 |
| jianshu:e0ececa61e8e | 7 | 4.68 |
| jianshu:3ad9166c02e5 | 7 | 4.64 |
| jianshu:ab8c754761bf | 7 | 3.68 |
| jianshu:4f90ffb873ab | 6 | 4.79 |
| jianshu:5c83da126b48 | 6 | 4.62 |
| jianshu:7ca16c92ca37 | 6 | 4.12 |
| jianshu:ee6a8ebc5bec | 4 | 3.00 |
| jianshu:1f10795c9468 | 4 | 4.88 |

## 分数档分布

| 档位 | 题数 |
|---|---|
| 5 分档 | 633 |
| 4 分档 | 150 |
| 3 分档 | 16 |
| 2 分档 | 19 |
| 1 分档 | 115 |

## 失败清单（overall < 3，共 139 条，列前 60）

- [iosqa] **启动优化**（1.00）　待评回答未回答问题，而是询问用户需求，未覆盖任何参考要点。
- [iosqa] **Mach-O加载动态库方法的流程**（1.00）　回答为工具调用失败消息，未提供任何相关信息，属拒答。
- [iosqa] **TaggedPointer**（1.00）　回答未提供实质内容，类似拒答，未解答问题。
- [iosqa] **内存泄漏以及检测方法**（1.00）　待评回答是系统拒答，未提供任何实质内容。
- [iosqa] **const（常量）**（1.00）　回答未提供const的相关信息，而是讨论笔记归档，完全不切题。
- [iosqa] **extern**（1.00）　回答未提供关于extern的任何信息，而是询问意图，答非所问。
- [iosqa] **@private / @protected / @public / @package**（1.00）　待评回答为拒答，未回答问题。
- [iosqa] **NSNotificationQueue和RunLoop的关系**（1.00）　待评回答无内容，相当于拒答，未回答问题。
- [iosqa] **通知的一些特点总结**（1.00）　待评回答未回答问题，仅询问类型，类似拒答。
- [iosqa] **多读单写**（1.00）　回答未提供任何多读单写并发模式的信息，仅陈述归档动作，类似拒答。
- [iosqa] **线程间通信（背）**（1.00）　回答未提供线程间通信信息，而是反问需求，类似拒答。
- [iosqa] **调度组：用 dispatch_group_async () 把要先做的任务塞进这个组里，让它们自己跑，最后用 dispatch_group_notify () 告诉系统：&quot;等组里所有任务都跑完了，就执行后面这个新任务&quot;。2. 信号量：先调 create (0) 弄个信号量（初始状态是没钥匙），让前一个任务先运行，运行完的时候调用signal ()，后一个任务开始前，先调 wait ()。**（1.00）　待评回答是拒答，没有提供任何回答内容。
- [iosqa] **调用方式不同**（1.00）　待评回答未提供答案，而是询问用户意图，属拒答。
- [iosqa] **线程保活和线程常驻**（1.00）　待评回答未提供实质内容，而是询问用户偏好，类似拒答。
- [iosqa] **模型树、动画树、渲染树（节点是CALayer）**（1.00）　待评回答为拒答，未回答问题，未覆盖参考要点。
- [iosqa] **layoutSubviews计算的时机**（1.00）　待评回答是拒答，未提供任何答案，类似知识库查询失败。
- [iosqa] **UICollectionView的Layout和FlowLayout**（1.00）　待评回答是拒答形式，未提供任何答案内容。
- [iosqa] **响应优先级**（1.00）　回答未提供内容，仅请求澄清，未覆盖参考要点。
- [iosqa] **cancelsTouchesInView（默认对大多数手势为 YES）**（1.00）　回答未提供问题相关信息，类似拒答。
- [iosqa] **delaysTouchesBegan（默认 NO）**（1.00）　回答是拒答，没有提供问题相关信息。
- [iosqa] **delaysTouchesEnded（默认 NO）**（1.00）　待评回答为拒答，未提供任何相关信息。
- [iosqa] **计算机渲染原理**（1.00）　待评回答拒答，未提供实质性答案。
- [iosqa] **xml和JSON和pb协议**（1.00）　回答未提供实质信息，仅澄清用户意图，类似拒答。
- [iosqa] **用NSUserDefaults存储用户偏好**（1.00）　答非所问，未涉及NSUserDefaults存储用户偏好的任何内容。
- [iosqa] **文件存储**（1.00）　回答未提供任何文件存储知识，仅反问用户，类似拒答。
- [iosqa] **讲一下FMDB的使用**（1.00）　待评回答是拒答消息，未提供任何关于FMDB使用的内容。
- [iosqa] **SDWebImage**（1.00）　回答未提供SDWebImage相关信息，仅询问用户需求，类似拒答。
- [iosqa] **AFNetworking**（1.00）　回答未提供任何AFNetworking信息，而是反问意图，完全答非所问。
- [iosqa] **YYModel**（1.00）　回答未涉及YYModel内容，仅询问用户意图，属于拒答。
- [iosqa] **JSONModel**（1.00）　待评回答未提供任何JSONModel信息，反问用户意图，完全不相关。
- [iosqa] **Masonry**（1.00）　回答未提供Masonry相关信息，仅询问用户意图，类似拒答。
- [iosqa] **内存管理**（1.00）　回答未涉及内存管理内容，仅询问用户意图，类似拒答。
- [iosqa] **可变性**（1.00）　回答未提供任何关于可变性的信息，仅询问用户意图，类似拒答。
- [iosqa] **功能特性**（1.00）　回答未提供信息，仅询问细节，类似拒答。
- [iosqa] **git指令**（1.00）　待评回答未提供任何git指令信息，是答非所问的反问。
- [agentqa] **多智能体（multi-agent）适用与代价**（1.00）　回答未提供实质内容，类似拒答。
- [agentqa] **RAG 召回不准怎么优化**（1.00）　回答为系统拒答消息，未提供任何优化内容。
- [agentqa] **温度为 0 就完全可复现吗**（1.00）　待评回答是错误消息，未提供实质性内容，类似拒答。
- [agentqa] **微调（fine-tune）适合解决什么**（1.00）　待评回答为空，未提供任何回答内容。
- [agentqa] **Agent 可观测性要记什么**（1.00）　待评回答以拒答开头，声称知识库里没有相关内容。
- [merged] **Objective-C对象内存对齐与初始化**（1.00）　待评回答是拒答消息，未回答问题，因此所有维度均不合格。
- [merged] **iOS底层初始化调用链**（1.00）　待评回答是工具调用提示，没有回答问题，属于拒答。
- [merged] **Objective-C中isKindOfClass与isMemberOfClass原理与对比**（1.00）　待评回答是工具错误信息，未实际回答问题。
- [merged] **iOS RunLoop核心机制与应用**（1.00）　回答是拒答，未提供任何RunLoop相关信息。
- [merged] **Weak引用底层原理与流程**（1.00）　待评回答为拒答，未提供任何实质内容。
- [merged] **iOS锁与队列类型对比**（1.00）　待评回答未提供实质内容，类似拒答。
- [merged] **自旋锁与互斥锁区别与应用**（1.00）　待评回答为系统错误提示，未提供任何答案。
- [jianshu:b72018e88a97] **fastpath与slowpath宏定义与作用**（1.00）　待评回答表示需要查知识库，未提供任何实质答案，属于拒答。
- [jianshu:b72018e88a97] **fastInstanceSize中的16字节对齐计算**（1.00）　待评回答是工具调用失败拒答，没有提供任何答案内容。
- [jianshu:b72018e88a97] **为什么需要16字节对齐**（1.00）　待评回答是工具调用错误，未回答问题。
- [jianshu:2fae148f015f] **解决_simple.h找不到**（1.00）　待评回答未提供解决方案，只询问上下文，类似拒答。
- [jianshu:2b660ec22fe0] **计算所需量子数k的算法**（1.00）　回答未提供计算量子数k的方法，而是要求澄清上下文，属于答非所问。
- [jianshu:8bfd70a9d1ac] **segregated_next_block 的 band 开辟**（1.00）　回答为拒答，未提供任何与问题相关的信息。
- [jianshu:7fd6241a7124] **isa类型为Class的原因**（1.00）　待评回答是拒答，没有回答问题。
- [jianshu:7fd6241a7124] **适配器设计模式应用**（1.00）　回答为系统拒答消息，未回答问题。
- [jianshu:873e530a0995] **objc_object与对象的关系**（1.00）　待评回答为系统错误消息，未回答问题，属拒答。
- [jianshu:873e530a0995] **CJLPerson与CJLTeacher的定义示例**（1.00）　待评回答表示没有相关内容，是拒答，未提供任何信息。
- [jianshu:873e530a0995] **验证NSObject唯一性的两种方式**（1.00）　回答是拒答，没有提供任何问题相关内容。
- [jianshu:873e530a0995] **class_rw_t 的 properties 方法**（1.00）　待评回答未回答问题，仅表示需查阅资料，类似拒答。
- [jianshu:873e530a0995] **class_ro_t 结构体中 ivars 的偏移**（1.00）　待评回答为拒答，未提供任何相关信息。

## agent 报错（638 条）

- [iosqa] 讲一讲Block是什么，Block的本质？：__AGENT_ERROR__ APITimeoutError: Request timed out.
- [iosqa] 如何解决Block的循环引用：__AGENT_ERROR__ APITimeoutError: Request timed out.
- [iosqa] 链表在iOS中的应用：__AGENT_ERROR__ APIConnectionError: Connection error.
- [iosqa] 十大排序：__AGENT_ERROR__ APIConnectionError: Connection error.
- [iosqa] 栈在iOS中的应用：__AGENT_ERROR__ APIConnectionError: Connection error.
- [iosqa] 解决哈希冲突的两种方法：__AGENT_ERROR__ APIConnectionError: Connection error.
- [iosqa] 开散列和闭散列的不同？何时使用开散列？：__AGENT_ERROR__ APIConnectionError: Connection error.
- [iosqa] 设计一个百万级用户信息的哈希表，选择合适的冲突解决策略？：__AGENT_ERROR__ APIConnectionError: Connection error.
- [iosqa] NSDictionary查值流程和key需要遵循那些协议？：__AGENT_ERROR__ APIConnectionError: Connection error.
- [iosqa] NSDictionary实现原理：__AGENT_ERROR__ APIConnectionError: Connection error.
- [iosqa] 对比NSMutableDictionary和NSHashTable：__AGENT_ERROR__ APIConnectionError: Connection error.
- [iosqa] 对比NSMutableDictionary和NSMapTable：__AGENT_ERROR__ APIConnectionError: Connection error.
- [iosqa] 如果我现在有一个百万级别的数据，使用开散列，现在需要扩展，但是又不能一次性处理，怎么办？：__AGENT_ERROR__ APIConnectionError: Connection error.
- [iosqa] 哈希表的扩容：__AGENT_ERROR__ APIConnectionError: Connection error.
- [iosqa] 哈希因子为什么采用0.75,哈希为什么偶数幂扩容：__AGENT_ERROR__ APIConnectionError: Connection error.
- [iosqa] 讲讲MVC和MVVM：__AGENT_ERROR__ APIConnectionError: Connection error.
- [iosqa] MVVM的优势和缺点：__AGENT_ERROR__ APIConnectionError: Connection error.
- [iosqa] MVC和MVVM的buuton的点击事件写在哪里：__AGENT_ERROR__ APIConnectionError: Connection error.
- [iosqa] 讲讲你知道的设计模式（八股）：__AGENT_ERROR__ APIConnectionError: Connection error.
- [iosqa] 六大设计原则：__AGENT_ERROR__ APIConnectionError: Connection error.
- [iosqa] 设计模式的三大类型：__AGENT_ERROR__ APIConnectionError: Connection error.
- [iosqa] 建造者模式：__AGENT_ERROR__ APIConnectionError: Connection error.
- [iosqa] 工厂模式：__AGENT_ERROR__ APIConnectionError: Connection error.
- [iosqa] 代理模式：__AGENT_ERROR__ APIConnectionError: Connection error.
- [iosqa] 责任链模式：__AGENT_ERROR__ APIConnectionError: Connection error.
- [iosqa] 组合模式：__AGENT_ERROR__ APIConnectionError: Connection error.
- [iosqa] 什么是依赖倒置：__AGENT_ERROR__ APIConnectionError: Connection error.
- [iosqa] 数据库三大范式：__AGENT_ERROR__ APIConnectionError: Connection error.
- [iosqa] 数据库索引的底层实现：__AGENT_ERROR__ APIConnectionError: Connection error.
- [iosqa] 对于WCDB的认识：__AGENT_ERROR__ APIConnectionError: Connection error.

## 怎么读这份报告

- **均分**反映 agent 在自己知识库题目上的综合表现。
- **按来源**看哪类知识 agent 答得差（检索没召回 or 综合能力弱）。
- **失败清单**是 Harness 反馈循环的输入：逐条看是检索没命中、答错、还是跑题，再针对性改 ask_kb 检索 / system prompt。
- 裁判尺度由 Opus 设计，跑完已用 `--export` 抽检校准（见同目录抽检报告）。