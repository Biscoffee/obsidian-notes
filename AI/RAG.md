3.3 RAG 检索增强生成

学到什么程度：理解完整架构，能做技术选型（嵌入模型、向量数据库、分块策略），知道 RAG 的典型失败模式和优化方向。

为什么不归入实践层：RAG 已经是非常成熟的技术，有大量现成的框架和服务。除非你的核心工作就是搭建 RAG 系统，否则理解原理 + 会调参就够了。Agent 的 RAG 能力通常由框架/平台提供。

学习方式：

|          |                               |      |
| -------- | ----------------------------- | ---- |
| 内容       | 方式                            | 时间   |
| RAG 完整流程 | 跟着 LangChain RAG Tutorial 走一遍 | 3 hr |
| 分块策略对比   | 对同一份文档用不同分块策略，对比检索效果          | 2 hr |
| 典型失败模式   | 故意测试边界：跨段落的问题、需要推理的问题、模糊查询    | 1 hr |

练习项目：

任务：搭建一个"公司技术文档问答系统"

步骤：

1. 准备 10-20 篇技术文档（Markdown/PDF）

2. 选择分块策略（推荐 RecursiveCharacterTextSplitter，1000 字符 + 200 重叠）

3. 选择嵌入模型（推荐 BGE-M3 或 OpenAI text-embedding-3-large）

4. 选择向量数据库（推荐 Chroma 用于学习）

5. 构建检索链，测试 10 个问题

6. 对比加 Reranker 前后的效果

验证指标：

- 检索准确率：Top-5 中包含正确答案的比例

- 回答质量：是否准确引用了原文

- 失败案例分析：哪些问题答错了，为什么

RAG（检索增强生成）

|                                                                                                                                         |      |         |     |                  |
| --------------------------------------------------------------------------------------------------------------------------------------- | ---- | ------- | --- | ---------------- |
| 资源                                                                                                                                      | 类型   | 语言      | 时长  | 说明               |
| [LangChain RAG Tutorial](https://python.langchain.com/docs/tutorials/rag/)                                                              | 官方文档 | English | ~2h | 手把手搭建 RAG 系统     |
| [Building RAG Agents with LLMs - NVIDIA](https://www.nvidia.com/en-us/training/instructor-led-workshops/building-rag-agents-with-llms/) | 课程   | English | ~3h | 工业级 RAG 实现方案     |
| [RAG from Scratch - LangChain YouTube](https://youtube.com/playlist?list=PLfaIDFEXuae2LXbO1_PKyVJiQ23ZztA0x)                            | 视频系列 | English | ~5h | 从零构建 RAG 系统的完整系列 |

学习建议：属于"理解层"，建议跟着 LangChain 教程实现一个简单 RAG demo。

---
LangChain 文档里把 RAG 分成两个核心阶段：

第一阶段叫 Indexing，也就是“建立索引”。
第二阶段叫 Retrieval and Generation，也就是“检索并生成回答”。文档说明，Indexing 是把数据从来源加载进来并建立索引；真正运行时，RAG 会根据用户问题从索引中检索相关数据，再传给模型回答

![[CleanShot 2026-05-01 at 15.28.58@2x.png]]

LangChain 文档里说，搭 RAG 需要选三类组件：
第一，聊天模型 Chat Model，比如 OpenAI、Anthropic、Gemini、HuggingFace 等。
第二，Embedding 模型，也就是把文本转成向量的模型。
第三，Vector Store，也就是保存向量并支持相似度搜索的数据库，比如 InMemory、Chroma、FAISS、Milvus、PGVector、Pinecone、Qdrant 等。



# Loading documents 加载文件

文档里用的是一个网页作为知识来源，通过 `WebBaseLoader` 加载网页内容。它会把网页 HTML 解析成文本，并返回 LangChain 的 `Document` 对象。文档示例还用 BeautifulSoup 只保留网页中真正有用的部分，比如标题、正文和 header。

当然，你可以使用其他数据来源，这个不是核心，重点是把你的资料加载进来


# Splitting documents  拆分文档
一篇文档可能很长，不能整个塞给模型，容易把上下文撑爆。而且检索的时候 如果文档过大，搜索也不会精准。
为了解决这个问题，我们将 [`Document`](https://reference.langchain.com/python/langchain-core/documents/base/Document) 分割成多个块，分别用于嵌入和矢量存储。这样可以帮助我们在运行时仅检索博客文章中最相关的部分。

我们使用 `RecursiveCharacterTextSplitter` 分割器，它会使用换行符等常用分隔符递归分割文档，直到每个文本块达到合适的大小。这是推荐用于通用文本用例的文本分割器。

> `RecursiveCharacterTextSplitter` 它是 LangChain 框架内部自带的一个核心组件。
>  执行逻辑如下：
>  - 他优先用 `"\n\n"` 把文章切成一个个段落。
>  - 检查切出来的段落
>  - - 如果某个段落的长度小于chunk_size，那么就保留这个段落
>  - - 某个段落超过了 `chunk_size`，它就会对**这一个特别长的段落**降级，使用下一个分隔符 `"\n"` 继续切。
>  - - - 如果按 `"\n"` 切完还是太长，就继续降级用空格 `" "` 切，直到所有切出来的块都满足你的长度要求。
>   

这种“递归降级”的策略能确保语义最紧密的部分（段落、句子）尽量被打包在一起。

```Python
from langchain_text_splitters import RecursiveCharacterTextSplitter

text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,  # chunk size (characters)
    chunk_overlap=200,  # chunk overlap (characters)
    add_start_index=True,  # track index in original document
)
all_splits = text_splitter.split_documents(docs)

print(f"Split blog post into {len(all_splits)} sub-documents.")
```

overloop指的是相邻文本块之间的重叠部分

# Storing documents 存储文档
切完之后，需要把每个 chunk 转成向量，然后放进向量数据库。
```Python
document_ids = vector_store.add_documents(documents=all_splits)

print(document_ids[:3])
```
在这个过程中，我们遍历每个chunk，调用Embedding模型，将文本转成高维向量，保存文本+向量+metadata。然后放进向量数据库。

metadata

> Embedding模型  
> 传统的计算机搜索是基于**“字面匹配”**的。比如你搜“苹果手机”，传统搜索引擎必须在文章里找到“苹”、“果”、“手”、“机”这几个字。如果文章里写的是“iPhone”，它可能就搜不到。  
而大模型需要理解的是**“语义”**。Embedding ，它可以把一段文字变成一串长长的数字（向量），意思越相近的文字，它们变成数字后的空间距离就越近。（Transformer）

> 向量数据库
> 他专门用来存储这些高维度坐标，MySQL更像一个表哥，我们通过一条条记录进行精准匹配，但是向量是一串浮点数，我们在查找时是寻找距离最近而非一模一样的数字，而这就真需要向量数据库的索引算法。这种技术叫 近似最近搜索ANN

![[CleanShot 2026-05-01 at 17.17.54@2x.png]]
# 检索和生成

1. 检索：给定一个用户输入，使用检索器从存储中检索相关的拆分
2. 生成：模型实验包含问题和检索数据的提示生成答案

![retrieval_diagram](https://mintcdn.com/langchain-5e9cc07a/I6RpA28iE233vhYX/images/rag_retrieval_generation.png?fit=max&auto=format&n=I6RpA28iE233vhYX&q=85&s=994c3585cece93c80873d369960afd44)

# RAG agent 与 RAG chain

RAG agent也就是把“检索”包装成一个工具，让模型自己决定什么时候调用工具。
文档中说，RAG agent 是一个带有检索工具的简单 agent，可以通过工具包装 vector store

RAG Chain 更简单粗暴：
每次用户提问，都先检索，然后把检索结果塞进 prompt，再让模型回答。
文档里说，RAG Chain 是一种 two-step chain，每次都运行搜索，并把结果作为上下文放进一次 LLM 调用里；它的延迟更低，但灵活性不如 Agent。


参考资料：
- https://docs.langchain.com/oss/python/langchain/rag#1-indexing

---

# 我的实践：rag_learning 项目实战闭环

> 项目路径：`~/rag_learning`
> 任务原型：笔记里的"公司技术文档问答系统"
> 最终交付：RAG Chain 完整流程 + 5 组分块实验 + 6 类失败分析 + Reranker 对比 + 安全工程闭环

## 1. 项目最终结构

```
rag_learning/
├── src/
│   ├── rag_system.py            ← 主系统（含 15 题黄金集 + 13 条查询扩展）
│   ├── chunk_compare.py         ← 分块策略对比（5 组）
│   └── rerank_compare.py        ← Reranker 对比
├── docs/                        ← 16 篇技术文档
│   ├── 01-10                    ← 后端/运维（7 篇保留 + 4 篇通用八股已删）
│   ├── 11-16                    ← 深度文（排程/音频/iOS/AI/WCDB/上手指南）
│   ├── 17_RecyclerView刷新与DiffUtil.md
│   ├── 18_Handler消息机制全解析.md
│   ├── 19_ViewPager2与TabLayout标签页.md
│   ├── 20_布局优化_include_merge_ViewStub.md
│   └── _experiments/
│       ├── chunk_compare.md     ← 实验报告
│       ├── failure_modes.md     ← 失败模式归类
│       └── rerank_compare.md    ← Reranker 对比报告
├── data/chroma_db/              ← 5.1MB 向量库（baseline 配置）
├── .gitignore                   ← 加固到 18 行（含 *.key/*.pem/.env.local）
├── .env / .env.example          ← 真 Key 已撤销重发
└── requirements.txt
```

## 2. 笔记概念 ↔ 我的代码（一对一映射）

### 阶段一：Indexing

| 笔记里的概念 | 代码位置 | 实际选择 + 关键细节 |
|---|---|---|
| Loader（不限于 WebBaseLoader）| `rag_system.py:90 load_documents()` | `TextLoader` 加载本地 Markdown，UTF-8 编码 |
| RecursiveCharacterTextSplitter，1000+200 | `rag_system.py:99-102` | **照抄笔记推荐**，但分隔符做了**中文化**：`["\n## ", "\n### ", "\n\n", "\n", "。", "，", ""]`——递归降级链的中文版 |
| Embedding 模型 | `rag_system.py:113-117` | BGE-M3（笔记推荐选项之一）+ `device="mps"` 加速 + L2 归一化 |
| Vector Store | `rag_system.py:118-122` | Chroma（笔记推荐）+ 本地持久化到 `data/chroma_db/` |
| ANN 近似最近邻 | `data/chroma_db/data_level0.bin` | Chroma 底层用 **HNSW** 索引（416KB 实物文件）|

### 阶段二：Retrieval and Generation

| 笔记里的概念 | 代码位置 | 备注 |
|---|---|---|
| 检索 = 从存储中找相关 chunk | `rag_system.py:147 retrieve()` | `similarity_search_with_score(k=5)` |
| **超出笔记的优化：查询扩展** | `rag_system.py:138-143 expand_query()` | 13 条同义词词典（K8s→Kubernetes、Handler→消息 Looper），**实测让"那个发消息的东西怎么用？" 5/5 全过** |
| Chat Model（笔记三组件之一）| `rag_system.py:171-214 generate_answer()` | 小米 MiMo（OpenAI 兼容），temperature=0.2 |
| Prompt 工程（笔记没强调）| `rag_system.py:178-186` | 系统 prompt 强约束：必须标注来源文件名，禁止 `[1]` 编号 |

### 我的实现属于"RAG Chain"（不是 RAG Agent）

笔记结尾区分 Chain vs Agent：
- **RAG Chain**（我做的）：每次提问 → 强制检索 → 塞 prompt → 生成。延迟低、确定性高
- RAG Agent：把检索包装成工具，模型自己决定是否调用。灵活但慢

`main()` 里的 `for question in questions: results = retrieve(...) → generate_answer(...)` 是教科书 Chain 实现。

## 3. 实验一：分块策略对比（笔记 2 小时任务）

5 组实验跑同一份 16 篇文档 + 同一份 14 题黄金集（剔除文档外问题）：

| 组          | chunk_size | overlap |  chunks |   Top-1   |   Top-5   |
| ---------- | ---------: | ------: | ------: | :-------: | :-------: |
| tiny       |        300 |      50 |    1002 |   12/14   |   13/14   |
| small      |        500 |     100 |     602 |   11/14   |   13/14   |
| baseline   |       1000 |     200 |     293 |   12/14   | **14/14** |
| **large**  |   **1500** | **300** | **191** | **13/14** | **14/14** |
| no_overlap |       1000 |       0 |     285 |   12/14   |   14/14   |

**核心结论**：
- 颗粒越大越好——Markdown 段落自带语义完整性，1500/300 比 1000/200 Top-1 还高 1
- 小块（300）反而伤精准——chunks 多 5 倍，把上下文打散
- overlap 在本数据集**性价比低**——`no_overlap` 和 baseline 同分
- **xlarge (2000)** 在 MPS 上嵌入**卡死 11 分钟**，已剔除（生产风险信号）

> 这就是笔记里"递归降级保持语义紧密"原理的实测验证——chunk 大让段落级语义不被切碎。

## 4. 实验二：典型失败模式（笔记 1 小时任务，6 类）

| 类型 | 描述 | 代表问题 | 缓解方向 |
|---|---|---|---|
| A. 评测集歧义 | 多文档都对 | "Android 列表卡顿可以从哪些方向优化？"（17/20 都对）| 改多答案集合评测 |
| B. 关键词冲突 | 错误文档里语义近 | baseline："K8s 自动扩缩容"误中 Handler | 加 Reranker / 增大 chunk |
| C. 推理需求 | 要从描述里推抽象概念 | "RecyclerView 为什么要用 ViewHolder 模式？"——小 chunk 都错 | 大 chunk / Reranker |
| D. 跨文档 | 答案分散 | "K8s 节点 OOM 应该如何排查？"（04+10）| Reranker / multi-query |
| E. 模糊查询 | 用代词、上下文缺失 | "那个发消息的东西怎么用？"——查询扩展救场 ✅ 5/5 | 查询扩展词典（已做）|
| F. 文档外 | 答案根本不在库 | "如何用 Compose 重写 RecyclerView？" | 相似度阈值 + Prompt 约束 |

> **这 6 类正是笔记说的"跨段落问题、需要推理的问题、模糊查询"边界——全部实测覆盖**。

## 5. 实验三：Reranker 对比（笔记练习项目第 6 步）

用 `BAAI/bge-reranker-v2-m3` 做交叉编码重排，召回 top_n=20 → 重排 top_5：

| 指标 | 单阶段（仅 Chroma）| 加 Reranker | Δ |
|---|:---:|:---:|:---:|
| Top-1 | 12/14 | 12/14 | **+0** |
| Top-5 | 14/14 | **13/14** | **-1** ⬇️ |

**反直觉的真结果——Reranker 没帮上忙**。逐题 diff：
- ✅ 修复了 1 题：K8s 自动扩缩容 Top-1 从 Handler 纠正到 Kubernetes
- ❌ 退步了 1 题：RecyclerView ViewHolder 模式 Top-1 被推给 ViewPager2

**根因分析**：
1. **天花板效应**：BGE-M3 单阶段已 Top-5 100%，没空间让 Reranker 立功
2. **语言适配**：BGE-Reranker-V2-M3 训练分布偏英文，对中文短问句判别不一定优于 BGE-M3 单阶段
3. **召回宽度反作用**：top_n 拉到 20 引入更多噪音 chunk
4. **chunk 颗粒度**：Reranker 看到的是 1000 字符片段不是完整文档，判别难度上升

> **学习收获**：Reranker 不是银弹。**知道何时不该用 Reranker** 比"无脑加 Reranker"值钱。这个负结果对应笔记里"理解原理 + 会调参"的"会调参"——知道哪个旋钮在哪种数据上没效果。

## 7. 笔记验证指标 vs 实际产出

| 笔记验证指标 | 实际产出 |
|---|---|
| 检索准确率：Top-5 包含正确答案的比例 | **Top-5 100%（14/14），Top-1 86%（12/14）** |
| 回答质量：是否准确引用原文 | Prompt 强约束 `（来源：xx.md）`，目测 100%；自动评估未做 |
| 失败案例分析 | 6 类失败 + 实测对应表（见 `failure_modes.md`）|

## 8. 笔记练习项目 6 步 vs 实际完成

| 笔记步骤                                        | 实际                                       | 状态      |
| ------------------------------------------- | ---------------------------------------- | ------- |
| 1. 准备 10-20 篇技术文档                           | 16 篇（删 4 通用八股，加 4 篇 Android 深度文）         | ✅       |
| 2. RecursiveCharacterTextSplitter, 1000+200 | 完全照抄                                     | ✅       |
| 3. BGE-M3 嵌入                                | 本地推理 + MPS 加速                            | ✅       |
| 4. Chroma 向量库                               | 持久化到 `data/chroma_db/`                   | ✅       |
| 5. 检索链 + 测试 10 个问题                          | 升级到 **15 题黄金集**（标准 6 + Android 4 + 难题 5） | ✅ 加强版   |
| 6. 加 Reranker 前后对比                          | 实测发现 Reranker 在本数据集**反而退步**              | ✅ + 负结果 |

## 9. 学到的"会调参"——技术选型决策树

| 场景 | 选什么 | 为什么 |
|---|---|---|
| 中文为主、隐私敏感、本地推理 | **BGE-M3** | 中英双优化，本地无 API 成本 |
| 英文为主、追求最高质量 | OpenAI text-embedding-3-large | 工业级质量，但有成本和隐私问题 |
| 学习/原型 | **Chroma**（本地持久化）| 零运维，HNSW 内置 |
| 生产/规模 | Milvus / Qdrant / PGVector | 分布式、高并发 |
| 文档段落语义完整（如 Markdown）| **chunk_size 1500、overlap 300** | 段落不被切碎，Top-1 更准 |
| 文档松散（PDF 扫描件、长篇小说）| chunk_size 500-1000、overlap 200 | 防止单段塞太多无关信息 |
| 已有 Top-5 100% 的强基线 | **不用** Reranker | 天花板效应，反而可能退步 |
| 召回不行（Top-5 < 80%）| **该上** Reranker | 用交叉编码精排，但要做 A/B 对照 |
| 中文短问句判别 | 找中文专项 Reranker，慎用 v2-m3（偏英文）| 训练分布要匹配 |
| 模糊查询（"那个东西"）| **查询扩展词典** | 同义词补全，比换模型见效快 |

## 10. 一句话带走

**笔记说"理解原理 + 会调参就够了"——通过这次实战，我现在知道：**
- **原理**：RAG = 索引（Loader→Splitter→Embedding→VectorStore）+ 检索生成（相似度搜索→塞 Prompt→LLM）
- **调参**：chunk_size 选大、查询扩展见效快、Reranker 看场景、相似度阈值能让"文档外"问题正确拒答
- **失败模式**：6 类（歧义/关键词冲突/推理/跨文档/模糊/文档外）每一类都有数据归因
- **工程**：API Key 必须从一开始就 .gitignore + gitleaks，事后清历史代价高

3.3 RAG ✅ 闭环。
