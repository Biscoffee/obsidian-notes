# v4 失败 179 题规则核查(2026-05-30)

总失败 179 道 = 5.0 类比为底色,**真实可改进的质量缺口仅 15 道(8%)**

| 类别 | 占比 | 性质 |
|---|---|---|
| 工具超限 | 53 (30%) | ⚠️ 工程问题 - 提高 AGENT_MAX_TOOL_ITERS 或加死循环检测 |
| 上下文依赖 | 32 (18%) | ✓ agent 诚实拒答(题目脱离上下文,非 agent bug) |
| 其他拒答 | 0 (0%) | ✓ 笔记/库没相关内容时不编造 |
| 跑题 | 10 (6%) | 部分需复查 |
| 漏要点 | 12 (7%) | ⚠️ 真质量缺口 |
| 事实错误 | 3 (2%) | ⚠️ 真质量缺口 |

## 真质量缺口 15 道(漏要点 + 事实错误)

### 漏要点（12 道）
- [iosqa] **QOS_CLASS_USER_INTERACTIVE（用户交互级）最高优先级，直接影响用户界面响应（比如滑动列表、动画渲染），必须快速完成（通常毫秒级）。2. QOS_CLASS_USER_INITIATED（用户发起级）用户主动触发的任务（比如点击按钮后的网络请求），需要尽快完成（通常秒级内），用户会等待结果。3. QOS_CLASS_DEFAULT（默认级）没指定时的默认值，系统会根据上下文自动调整（建议显式指定其他等级，避免依赖默认值）。4. QOS_CLASS_UTILITY（工具级）耗时较长但用户不急需结果的任务（比如下载文件、数据解析），可以显示进度条，系统可能限制其 CPU 占用以节省电量。5. QOS_CLASS_BACKGROUND（后台级）完全不影响用户的任务（比如备份数据、同步日志），系统会在空闲时执行，优先级最低。** (overall 1.0)
  Q: QOS_CLASS_USER_INTERACTIVE（用户交互级）最高优先级，直接影响用户界面响应（比如滑动列表、动画渲染），必须快速完成（通常毫秒级）。2. 
  原因: 未提供通过dispatch_set_target_queue绑定QoS等级的具体步骤
- [jianshu:853ca8318a15] **结构体LGStruct1大小计算** (overall 1.0)
  Q: LGStruct1结构体的大小是多少？如何计算？
  原因: 未提供LGStruct1的具体大小和计算，只讲通用规则。
- [memory:user_knowledge_tier_awareness.md] **需求决策：先 Prompt 后微调** (overall 2.5)
  Q: 需要特定回答风格或格式时应如何决策？
  原因: 回答涉及决策逻辑，但未覆盖参考要点中的Prompt Engineering和Fine-tuning。
- [merged] **iOS缓存buckets查找机制** (overall 2.8)
  Q: iOS缓存中buckets数组的定位与向前遍历机制是怎样的？
  原因: 未覆盖具体地址计算公式和向前遍历的字节操作细节
- [obsidian:1122.md] **Harness层的四个核心机制** (overall 2.5)
  Q: 针对四种Agent失败模式，Harness层需要补充哪四个核心机制？
  原因: 回答的机制与参考要点不符，未覆盖关键内容。
- [obsidian:1122.md] **fetch_url 调用完整流程** (overall 2.8)
  Q: 从用户请求到 LLM 拿到结果，fetch_url 的完整调用流程包含哪些步骤？
  原因: 只覆盖部分步骤，遗漏关键流程如缓存、解码、序列化等。
- [mikeash/2011-10-28-generic-block-proxying] **通用 Block 代理** (overall 2.5)
  Q: MAFakeBlock类的内存布局是如何设计以模拟Block结构的？其初始化过程涉及哪些关键步骤？
  原因: 未提及MAFakeBlock具体字段和初始化中的转发机制步骤。
- [obsidian:1122.md] **规则约束导致死循环的风险** (overall 2.2)
  Q: 规则约束导致死循环的风险是什么？
  原因: agent回答未覆盖核心风险，如耗尽步数预算和对外部无警示。
- [obsidian:1122.md] **reasoning_content 计入 max_tokens 预算** (overall 1.0)
  Q: MiMo 的 max_tokens 预算如何分配？
  原因: 未提及 reasoning_content 计入 max_tokens 预算的关键点。
- [obsidian:1122.md] **11 个 Demo 的演化路径** (overall 1.0)
  Q: 11 个 Demo（4.1-4.10）之间是什么关系？
  原因: 未覆盖Demo间演化路径和递进关系，只做不完整分组。

### 事实错误（3 道）
- [jianshu:2b660ec22fe0] **size小于阈值时的处理** (overall 2.5)
  Q: 当size小于NANO_REGIME_QUANTA_SIZE时，size会被设置为什么值？
  原因: agent说size被向上取整，与参考要点设置为NANO_REGIME_QUANTA_SIZE矛盾。
- [memory:user_learning_resources_llm_agent.md] **Transformer原始论文的定位** (overall 1.0)
  Q: "Attention Is All You Need"原始论文在学习资源中的定位是什么？
  原因: agent错误地将论文定位为'基础必读'，与参考要点'了解层可跳过'矛盾。
- [obsidian:1122.md] **上下文管理失败的真实驱动因素** (overall 2.0)
  Q: 上下文管理失败的真实驱动因素是什么？
  原因: 驱动因素应为长工具结果塞栈，而非机制错位。

## 工具超限 53 道(下一步该攻的工程问题)

agent 在多轮 ask_kb/web_search 间循环,超过 AGENT_MAX_TOOL_ITERS=8 被强停。可改进:
- 加循环检测(同工具+同参数两次=有问题)
- 在 5/8 轮时插入「现有信息够吗?够就 final 答」的自检 prompt
- 提高 cap 到 12 但加超时