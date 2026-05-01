# LLM 学习路径：从 LLM 到 Agent Harness

> **核心原则**：不要自下而上从理论学起，而是从中间切入动手做，遇到瓶颈再回头补底层知识。
> **本笔记定位**：我自己的学习成果笔记（按章节积累，写到哪记到哪）

---

## 一、三层学习模型

| 层次 | 目标 | 投入比例 | 包含概念 |
|---|---|---|---|
| **了解层** | 知道原理和边界，能在技术讨论中不露怯 | ~10% | Transformer 原理、Fine-tuning/RLHF/DPO、Scaling Laws |
| **理解层** | 能做技术选型和方案评审，知道什么场景用什么 | ~30% | LLM 能力边界、Prompt Engineering、RAG、Multi-Agent |
| **实践层** | 动手写代码、反复调试、积累工程经验 | ~60% | Tool Use、AI Agent、MCP、Context Engineering、Harness Engineering |

**我自己的判断口诀**：
- 原理类 → 1–2 小时建立直觉就走，不手推公式
- 选型/边界类 → 必须能讲清楚 trade-off
- 工程实践类 → 必须有可运行的 Demo

---

## 二、了解层

### 2.1 Transformer 架构原理

**我的收获**：LLM 本质就是个超大号"文字接龙"——每次只预测下一个 token。底层是 Transformer 的 Self-Attention，但作为应用层开发者，**不用懂数学**，只要建立直觉就行。

**记住这几点就够**：
- Token ≠ 字 ≠ 词，中文大约 1 个字 ≈ 1.5–2 个 Token（影响成本估算）
- 上下文窗口 = 模型一次能看到的最大 Token 数，**不是无限记忆**——这是后面 Context Engineering 的根因
- 现在主流是 Decoder-only（GPT/Claude/Llama），因为生成效率最高
- Self-Attention 直觉：每个 token 看一眼其他所有 token，决定该关注谁

> 类比：用 MySQL 不用懂 B+ 树细节。除非要从零训练或做底层推理优化，Transformer 数学不影响日常开发。

### 2.2 Fine-tuning / RLHF / DPO

**我的收获**：90% 的 AI 应用，**Prompt + RAG + Agent 已经够了**，微调是最后手段不是第一选择。

**我自己的决策树**：
```
需求是什么？
├── 需要最新知识 / 企业私有数据 → 用 RAG，不要微调
├── 需要特定回答风格 / 格式  → 先试 Prompt，不够再考虑微调
├── 需要教模型一个全新技能   → 才考虑 LoRA 微调
└── 需要模型行为更安全/对齐  → 这是模型厂商的事（RLHF/DPO）
```

**概念速记**：
- **Fine-tuning**：在预训练模型基础上用自己的数据继续训练
- **LoRA**：只训练一小部分"低秩"参数（~1%），主模型冻结，省显存省时间，是目前微调主流
- **RLHF**：用人类偏好排序训练 reward model，再用 RL 优化主模型
- **DPO**：RLHF 的简化版，省掉 reward model，直接用偏好对训练

### 2.3 Scaling Laws

**一句话**：模型性能 ~ 参数量、数据量、计算量的幂律关系，**但有边际递减**。

**实战含义**：选模型按任务复杂度匹配——简单任务 Haiku/Mini 就够，不要无脑上 Opus。

---

## 三、理解层

### 3.1 LLM 能力边界

**我的收获**：所有后续技术决策的基础。没建立边界感 → 容易出现"能 RAG 解决的非要微调""Agent 做不了的硬塞给 Agent"。

**判断矩阵**：

| 任务类型 | LLM 表现 | 是否需要辅助 |
|---|---|---|
| 创意写作、头脑风暴 | 很强 | 不需要 |
| 代码生成（常见语言）| 很强 | 不需要 |
| 实时信息查询 | 做不到 | Tool Use / RAG |
| 精确数学计算 | 不可靠 | Tool Use（代码执行）|
| 企业私有知识问答 | 不知道 | RAG |
| 多步复杂任务 | 单次做不好 | Agent |
| 长时间任务 | 会"走神" | Context Engineering |

**建立直觉的方式**：
- 同一个任务用 Haiku / Sonnet / Opus 各跑一遍，对比质量和成本
- 故意问已知答案的专业问题测幻觉
- 长文档关键信息放在不同位置，看记忆衰减（参考后面 Lost in the Middle）

### 3.2 Prompt Engineering

**我的收获**：在 Agent 时代，单独的 Prompt 优化重要性在下降——它被吸收进了 **System Prompt 设计**和**工具描述设计**。底层功力要扎实，但不必当 Prompt 大师。

**我会用的核心技巧**：
- **Role Prompting**：给身份（"你是一位资深 SRE"）
- **Few-shot**：2–5 个高质量示例让模型模仿格式
- **CoT**：`Let's think step by step`，逼模型显式思考
- **结构化输入**：XML / Markdown / JSON 比纯文本鲁棒得多
- **明确约束**：说清"不能做什么"和边界条件

**System Prompt 五要素**（我打算照这个套路写）：
1. **角色**：身份与经验
2. **输入格式**：约定输入长什么样
3. **输出格式**：约定输出长什么样
4. **行为约束**：不确定时怎么办、能否擅自行动
5. **错误处理**：输入不符合预期时怎么办

> ⏳ 待练习：为"代码审查 Agent"写一份 System Prompt，跑 3 个真实 PR diff 验证。

### 3.3 RAG 检索增强生成

> **本节状态**：✅ 已完成实战闭环（项目：`~/rag_learning`）

#### 3.3.1 为什么不归入实践层

RAG 已经是非常成熟的技术，有大量现成框架/服务。除非核心工作就是搭 RAG，否则**理解原理 + 会调参**就够。Agent 的 RAG 能力通常由框架/平台提供。

#### 3.3.2 LangChain 官方架构（两阶段）

LangChain 把 RAG 分成两个核心阶段：

1. **Indexing（建立索引）**：把数据从来源加载进来并建立索引
2. **Retrieval and Generation（检索并生成回答）**：根据用户问题从索引中检索相关数据，再传给模型回答

![[CleanShot 2026-05-01 at 15.28.58@2x.png]]

搭 RAG 需要选三类组件：
1. **Chat Model**：OpenAI / Anthropic / Gemini / HuggingFace
2. **Embedding 模型**：把文本转成向量
3. **Vector Store**：保存向量并支持相似度搜索（InMemory / Chroma / FAISS / Milvus / PGVector / Pinecone / Qdrant）

#### 3.3.3 阶段一：Indexing 详解

**Loading documents（加载文件）**：可以用 `WebBaseLoader` 加载网页，或 `TextLoader` 加载本地 Markdown / PDF。重点是把资料加载进来。

**Splitting documents（拆分文档）**：
一篇文档可能很长，不能整个塞给模型。我们用 `RecursiveCharacterTextSplitter` 递归降级切分：

> 执行逻辑：
> - 优先用 `\n\n` 切段落
> - 检查每段长度
>   - 小于 `chunk_size` → 保留
>   - 超过 → 降级用 `\n` 继续切
>   - 还是太长 → 降级用空格 / 句号
> - 直到所有块满足长度要求

**这种"递归降级"策略能确保语义最紧密的部分（段落、句子）被打包在一起。**

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200,  # 相邻块之间的重叠
    add_start_index=True,
)
all_splits = text_splitter.split_documents(docs)
```

**Storing documents（存储文档）**：每个 chunk 转成向量后放进向量数据库。

> **Embedding 模型**：传统搜索基于"字面匹配"（搜"苹果手机"找不到"iPhone"）；Embedding 把文字变成向量，**意思越相近，空间距离越近**——理解的是语义。
>
> **向量数据库**：专门存储高维坐标。MySQL 是精准匹配，向量库是**近似最近邻搜索（ANN）**——找距离最近而非一模一样。Chroma 底层用 HNSW 算法。

![[CleanShot 2026-05-01 at 17.17.54@2x.png]]

#### 3.3.4 阶段二：检索和生成

1. **检索**：给定用户输入，使用检索器从存储中找相关 chunk
2. **生成**：把问题 + 检索结果塞进 prompt，模型生成答案

#### 3.3.5 RAG Agent vs RAG Chain

| 维度 | RAG Chain | RAG Agent |
|---|---|---|
| 触发方式 | 每次提问都强制检索 | 模型自己决定何时检索（包装成工具）|
| 延迟 | 低 | 高 |
| 灵活性 | 低（每次必检索）| 高（可多轮检索/不检索）|
| 适用场景 | 知识问答、固定检索流程 | 复杂研究类任务 |

#### 3.3.6 我的实战闭环：`~/rag_learning` 项目

**项目结构**：

```
rag_learning/
├── src/
│   ├── rag_system.py            # 主系统（15 题黄金集 + 13 条查询扩展）
│   ├── chunk_compare.py         # 分块策略对比（5 组）
│   └── rerank_compare.py        # Reranker 对比
├── docs/                        # 16 篇技术文档（后端 + Android 深度文）
│   └── _experiments/            # 三份实验报告
├── data/chroma_db/              # 5.1MB 向量库
├── .gitignore                   # 加固到 18 行
└── .env.example
```

**笔记概念 ↔ 我的代码**：

| 概念 | 代码位置 | 实际选择 |
|---|---|---|
| Loader | `rag_system.py:90 load_documents()` | `TextLoader`（本地 Markdown）|
| RecursiveCharacterTextSplitter 1000+200 | `rag_system.py:99-102` | 照抄笔记，分隔符**中文化**：`["\n## ", "\n### ", "\n\n", "\n", "。", "，", ""]` |
| Embedding | `rag_system.py:113-117` | BGE-M3 + MPS 加速 + L2 归一化 |
| Vector Store | `rag_system.py:118-122` | Chroma + 本地持久化 |
| ANN 索引 | `data_level0.bin` | HNSW（Chroma 底层）|
| 检索 | `rag_system.py:147` | `similarity_search_with_score(k=5)` |
| **超出笔记的优化：查询扩展** | `rag_system.py:138-143` | 13 条同义词词典（K8s→Kubernetes、Handler→消息 Looper），让"那个发消息的东西怎么用？"5/5 全过 |
| Chat Model | `rag_system.py:171-214` | 小米 MiMo（OpenAI 兼容），temperature=0.2 |

**实验一：分块策略对比**（5 组 × 14 题黄金集）

| 组 | chunk_size | overlap | chunks | Top-1 | Top-5 |
|---|---:|---:|---:|:---:|:---:|
| tiny | 300 | 50 | 1002 | 12/14 | 13/14 |
| small | 500 | 100 | 602 | 11/14 | 13/14 |
| baseline | 1000 | 200 | 293 | 12/14 | **14/14** |
| **large** | **1500** | **300** | **191** | **13/14** | **14/14** |
| no_overlap | 1000 | 0 | 285 | 12/14 | 14/14 |

**核心结论**：
- 颗粒越大越好——Markdown 段落自带语义完整性
- 小块（300）反而伤精准——chunks 多 5 倍，上下文打散
- overlap 在本数据集**性价比低**——`no_overlap` 和 baseline 同分
- xlarge (2000) 在 MPS 上嵌入**卡死 11 分钟**，已剔除（生产风险信号）

**实验二：6 类典型失败模式**

| 类型 | 描述 | 代表问题 | 缓解方向 |
|---|---|---|---|
| A. 评测集歧义 | 多文档都对 | "Android 列表卡顿如何优化"（17/20 都对）| 改多答案集合评测 |
| B. 关键词冲突 | 错误文档语义近 | "K8s 自动扩缩容"误中 Handler | Reranker / 增大 chunk |
| C. 推理需求 | 要从描述推抽象 | "RecyclerView 为什么用 ViewHolder" | 大 chunk / Reranker |
| D. 跨文档 | 答案分散 | "K8s 节点 OOM 排查"（04+10）| Reranker / multi-query |
| E. 模糊查询 | 用代词、缺上下文 | "那个发消息的东西"——查询扩展救场 ✅ 5/5 | 查询扩展词典 |
| F. 文档外 | 答案不在库 | "Compose 重写 RecyclerView" | 相似度阈值 + Prompt 约束 |

**实验三：Reranker 对比（反直觉负结果）**

用 `BAAI/bge-reranker-v2-m3` 做交叉编码重排，召回 top_n=20 → 重排 top_5：

| 指标 | 单阶段 | 加 Reranker | Δ |
|---|:---:|:---:|:---:|
| Top-1 | 12/14 | 12/14 | +0 |
| Top-5 | 14/14 | **13/14** | **-1** ⬇️ |

**根因分析**：
1. **天花板效应**：BGE-M3 单阶段已 Top-5 100%，没空间立功
2. **语言适配**：BGE-Reranker-V2-M3 训练分布偏英文
3. **召回宽度反作用**：top_n=20 引入更多噪音
4. **chunk 颗粒度**：Reranker 看到的是 1000 字符片段不是完整文档

> **学习收获**：Reranker 不是银弹。**知道何时不该用 Reranker** 比"无脑加 Reranker"值钱。

#### 3.3.7 技术选型决策树（可调参的"调参"）

| 场景 | 选什么 | 为什么 |
|---|---|---|
| 中文为主、隐私敏感、本地推理 | **BGE-M3** | 中英双优化，本地无 API 成本 |
| 英文为主、追求最高质量 | OpenAI text-embedding-3-large | 工业级质量，但有成本和隐私问题 |
| 学习/原型 | **Chroma** | 零运维，HNSW 内置 |
| 生产/规模 | Milvus / Qdrant / PGVector | 分布式、高并发 |
| Markdown 等结构化文档 | **chunk_size 1500、overlap 300** | 段落不被切碎 |
| 松散长文（PDF 扫描）| chunk_size 500–1000 | 防止单段塞太多无关信息 |
| 已有 Top-5 100% 强基线 | **不用** Reranker | 天花板效应 |
| 召回不行（Top-5 < 80%）| **该上** Reranker | 但要做 A/B 对照 |
| 中文短问句 | 找中文专项 Reranker，慎用 v2-m3 | 训练分布要匹配 |
| 模糊查询 | **查询扩展词典** | 比换模型见效快 |

#### 3.3.8 一句话带走

笔记说"理解原理 + 会调参就够了"——通过实战我现在知道：

- **原理**：RAG = 索引（Loader→Splitter→Embedding→VectorStore）+ 检索生成
- **调参**：chunk_size 选大、查询扩展见效快、Reranker 看场景
- **失败模式**：6 类（歧义/关键词冲突/推理/跨文档/模糊/文档外）每一类都有数据归因
- **工程**：API Key 必须从一开始就 .gitignore + gitleaks，事后清历史代价高

参考资料：https://docs.langchain.com/oss/python/langchain/rag

---

### 3.4 Multi-Agent

> **本节状态**：✅ 概念闭环（实践推迟到 4.x 自然遇到时）

#### 3.4.1 学习目标

**学到什么程度**：理解 5 种协作模式，重点掌握 **Sub-Agent**，能判断**是否真的需要** Multi-Agent。

#### 3.4.2 底层逻辑：为什么是反模式重灾区

Multi-Agent 听起来很酷，但**绝大多数场景下，一个设计良好的单 Agent 完胜一群 Agent 互相聊天**：

- **上下文耦合**：A 的输出是 B 的输入，A 一抖动 B 全错；调试要同时盯两条上下文链
- **通信开销**：Agent 间传话靠自然语言或 JSON，每跳一次都是"翻译损耗"，错误累积放大
- **调试黑盒**：单 Agent 看一条 trace；多 Agent 要拼链路、对时序、判断谁的锅

**先把单 Agent 做到极致**——合理工具、清晰 prompt、合适上下文——证明它真的不够了，**再**上 Multi-Agent。

#### 3.4.3 判断框架

```
我需要 Multi-Agent 吗？
├── 任务能拆成完全独立的子任务吗？
│   ├── 能 → 考虑并行 Multi-Agent
│   └── 不能 → 先用单 Agent
├── 单 Agent 上下文会溢出吗？
│   ├── 会 → 用 Sub-Agent 模式隔离上下文
│   └── 不会 → 不需要 Multi-Agent
└── 需要不同"专业角色"吗？
    ├── 需要（如写代码+审代码） → 考虑串行 Multi-Agent
    └── 不需要 → 不需要 Multi-Agent
```

#### 3.4.4 5 种协作模式

| 模式 | 形态 | 典型场景 | 何时不要用 |
|---|---|---|---|
| **Sequential** | A → B → C 流水线 | 写代码 → 审代码 → 写测试 | 上下游强耦合、信息反复来回 |
| **Parallel** | 分发 → N 并行 → 聚合 | 3 种策略同时跑后投票 | 子任务有依赖、聚合比子任务还复杂 |
| **Sub-Agent ⭐** | 父 Agent 调子 Agent，独立上下文 | 代码库探索、长链路调试 | 一两个工具调用就能解决 |
| **Router** | 分类 → 路由到专业 Agent | 客服意图分发 | 意图边界模糊、跨域协作多 |
| **Hierarchical** | 管理 Agent 派活给执行 Agent | 大型多步项目 | **没有终止条件**——会陷入"再优化一轮"死循环 |

#### 3.4.5 Sub-Agent 深入（重点）

**核心价值**：保护父 Agent 的上下文窗口。

子 Agent 跑完只把"结论"返回给父：内部消耗 30k token 中间过程，父只收到 200 字结论，主上下文不被污染。

**与 Function Call 的边界**：

| 维度 | Function Call | Sub-Agent |
|---|---|---|
| 调用对象 | 确定性函数 | 另一个 LLM 实例 |
| 输入输出 | 结构化（schema 严格）| 自然语言任务描述 |
| 决策能力 | 无（只执行）| 有（自主规划+多步）|
| 适用场景 | 单步、可枚举 | 多步、需要探索 |

**判断口诀**：能用 Function Call 解决就别起 Sub-Agent；起了 Sub-Agent，**任务描述要像写给一个新来的同事**。

**在 Claude Code 里的对应**：`Agent` tool 就是 Sub-Agent 模式的工程化实现。`subagent_type` 指定人设（`Explore` / `Plan` / `general-purpose`），子 Agent 拥有独立上下文窗口和工具集，父 Agent 只能看到最终返回。

```python
# 父 Agent 主流程
result = call_subagent(
    type="Explore",
    prompt="在 src/ 下找出所有调用 deprecated_api() 的位置，"
           "返回文件路径+行号清单，不要返回代码内容。",
)
# result ~ 几百 token；子 Agent 内部可能消耗了几万 token
decide_next_step(result)
```

#### 3.4.6 反模式清单

- ❌ **"会议室"型设计**：3 个 Agent 围桌讨论 → 同一个模型自己跟自己车轱辘话
- ❌ **过度拆分**：把单 Agent 能做的拆成 3 个 → 通信成本 > 收益
- ❌ **Hierarchical 没终止条件** → 烧光预算
- ❌ **用 Multi-Agent 掩盖 Prompt 写得烂**：根因是 prompt 没写清楚，不是数量不够
- ❌ **没做可观测性就上 Multi-Agent**：链路看不见 = 出问题没法闭环

#### 3.4.7 与 3.3 RAG 的衔接

RAG 系统天然适合做成 Sub-Agent：

- 父 Agent 收到问题 → 调用"检索 Sub-Agent"
- 检索 Sub-Agent 内部完成：query expansion → 向量检索 → reranker → 返回 top-k 片段
- 父 Agent 拿到精炼上下文，做最终生成

当 RAG 流程从"一次检索"长到"多轮检索 + multi-query + 自校验"时，**Sub-Agent 模式是自然过渡**。

#### 3.4.8 一句话带走

> **Multi-Agent 不是"用得多就高级"**：唯一合法用途是 ① 上下文隔离、② 真正独立的并行、③ 不可替代的角色分工。除此之外都是用复杂度换"看起来很厉害"。**先把单 Agent 做到 100 分，再谈 Multi-Agent。**
