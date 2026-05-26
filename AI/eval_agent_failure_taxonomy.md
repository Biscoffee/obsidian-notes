# 实验1 失败题 · 根因归类（Harness 反馈循环输入）

生成时间：2026-05-26 10:41　分类器：MiMo　分类尺度：Opus 设计　token：728,940

对实验1 中 overall<3 的 804 道失败题做根因归类（agent 直接报错的 90 题另计，不在此分类）。

## 失败类型分布

| 类型 | 题数 | 占比 |
|---|---|---|
| 拒答 | 770 | 95.8% |
| 跑题 | 23 | 2.9% |
| 事实错误 | 3 | 0.4% |
| 漏要点 | 8 | 1.0% |
| 其他 | 0 | 0.0% |

## 各类型的来源分布（哪类知识最容易栽在这）

- **拒答**（770）：merged×102, obsidian:1122.md×97, iosqa×68, jianshu:cc9d286a1a27×39, jianshu:90f9348b2ed6×25
- **跑题**（23）：iosqa×4, obsidian:1122.md×4, agentqa×1, jianshu:2b660ec22fe0×1, jianshu:7fd6241a7124×1
- **事实错误**（3）：jianshu:90f9348b2ed6×1, jianshu:a6af821a2806×1, obsidian:1122.md×1
- **漏要点**（8）：iosqa×2, obsidian:1122.md×2, jianshu:2b660ec22fe0×1, jianshu:4f18226705ec×1, memory:user_knowledge_tier_awareness.md×1

## 各类型样例（每类最多 8 条）

### 拒答
- [iosqa] 编译链接流程（1.0）：agent因工具调用次数上限停止，未提供实质内容。
- [iosqa] 符号表和调试信息是什么？（1.0）：回答为空，无实质内容提供。
- [iosqa] OC采用的编译器是什么？（1.0）：空回复未提供任何实质内容，类似拒绝回答。
- [iosqa] 启动流程（1.0）：Agent空回复，未提供任何实质内容。
- [iosqa] Mach-O加载动态库方法的流程（1.0）：回复为空，无实质内容提供。
- [iosqa] 讲一讲iOS的内存管理（引用计数+MRC+ARC + AutoreleasePool + TagP）（1.0）：未提供任何实质内容，只说明输出故障。
- [iosqa] 内存泄漏以及检测方法（2.0）：Agent未回答问题，转而询问用户意图，无实质内容。
- [iosqa] 字符串的三种类型（1.0）：agent回复为空，未提供任何实质内容。

### 跑题
- [iosqa] 哨兵节点的作用（2.0）：解释数据结构哨兵，未涉及autoreleasepool。
- [iosqa] const（常量）（1.0）：回答内容与const问题无关，只提及归档状态。
- [iosqa] 十大排序（1.8）：agent回答询问用户需求，未提供排序算法信息。
- [iosqa] 存储方式（2.0）：回答转向iOS存储方式，未涉及结构体与类的存储对比。
- [agentqa] 多智能体（multi-agent）适用与代价（1.0）：回答聚焦于用户需求而非多智能体适用与代价，答非所问。
- [jianshu:2b660ec22fe0] 计算所需量子数k的算法（2.5）：回答转向量子力学原理，未涉及参考要点的编程公式。
- [jianshu:7fd6241a7124] 伪继承机制（2.5）：回答讨论Category和isa-swizzling，未覆盖结构体包含的内存布局实现。
- [jianshu:f7d9f6d86145] 排除分类重名方法原因（2.0）：回答针对二分查找算法，而非参考要点中的iOS方法冲突问题。

### 事实错误
- [jianshu:90f9348b2ed6] attachToClass循环逻辑（2.2）：外部循环应为遍历分类，agent错误描述为遍历类层级。
- [jianshu:a6af821a2806] Fastlane私有lane定义（2.8）：Agent错误声称Fastlane没有private_lane关键字，与参考要点相反。
- [obsidian:1122.md] fetch_url_raw 工具功能（1.0）：错误否认fetch_url_raw工具存在

### 漏要点
- [iosqa] 启动优化（2.0）：只列出优化方向名称，未覆盖参考要点中的具体措施。
- [iosqa] 用NSUserDefaults存储用户偏好（1.8）：回答仅确认状态，未提供任何技术要点或细节。
- [jianshu:2b660ec22fe0] iOS 默认对齐系数（2.8）：回答未明确给出默认对齐系数为8字节，转而解释自然对齐。
- [jianshu:4f18226705ec] 检查节点有效性与用户源代码（2.0）：未覆盖指针有效性和用户源代码文件检查
- [obsidian:1122.md] 上下文溢出复现条件（2.8）：Agent未提供复现条件，仅讨论用户场景未复现。
- [obsidian:1122.md] 摘要 prompt 措辞对压缩的影响（2.5）：回答泛泛而谈，未覆盖参考要点中的关键实验和具体现象。
- [memory:user_knowledge_tier_awareness.md] 按需选择模型规模（2.8）：回答聚焦训练最优，未涉及任务复杂度选模型要点
- [merged] Objective-C类注册函数（2.0）：未覆盖参考要点中的内部函数addClassTableEntry和addNamedClass。

## 行动建议（喂回 Harness）

- 占比最高的是 **拒答**（770/804）：agent 在有知识时仍说「没有」——查检索阈值/话术约束，或 prompt 太保守，放开「基于检索内容作答」。
- 结合实验4：检索 rank1 占 96%，所以失败基本**不是检索没召回**，而是上面这些**生成端**问题——改 system prompt / 换作答模型比改检索更值。
- 这就是 Demo 11 反馈循环的闭环：评测→归因→定位到具体组件→改→再评。