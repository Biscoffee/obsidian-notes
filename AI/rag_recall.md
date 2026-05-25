# 实验4 · RAG 召回率诊断（纯检索，零 token）

生成时间：2026-05-25 16:18　检索器：retrieval.retrieve_knowledge（关键词加权，无向量）

对全库 1574 题：用每题 qa_q 当 query 检索 knowledge 表，看自身卡排第几名。

## 自身卡 rank 分布

| 档位 | 题数 | 占比 |
|---|---|---|
| rank1 | 1523 | 96.8% |
| rank2-3 | 37 | 2.4% |
| rank4-5 | 5 | 0.3% |
| rank6-10 | 3 | 0.2% |
| rank10+ | 6 | 0.4% |
| 未召回 | 0 | 0.0% |

## 累计 top_k 命中率（rank ≤ k）

| top_k | 命中率 |
|---|---|
| top1 | 96.8% |
| top3 | 99.1% |
| top5 | 99.4% |
| top10 | 99.6% |

## 按来源的 rank1 占比

| source | 题数 | rank1 占比 | 未召回 |
|---|---|---|---|
| iosqa | 204 | 98% | 0 |
| merged | 196 | 99% | 0 |
| obsidian:1122.md | 134 | 100% | 0 |
| jianshu:cc9d286a1a27 | 55 | 95% | 0 |
| jianshu:693ec962b3d3 | 51 | 88% | 0 |
| jianshu:90f9348b2ed6 | 48 | 100% | 0 |
| obsidian:111.md | 40 | 100% | 0 |
| jianshu:4f18226705ec | 38 | 100% | 0 |
| agentqa | 31 | 100% | 0 |
| jianshu:2953e86db051 | 31 | 100% | 0 |
| jianshu:1f10795c9468 | 30 | 100% | 0 |
| jianshu:25bcb6540045 | 26 | 46% | 0 |
| jianshu:5447a66d955f | 26 | 100% | 0 |
| jianshu:873e530a0995 | 25 | 100% | 0 |
| jianshu:89ab04a91cbc | 25 | 100% | 0 |
| jianshu:2b660ec22fe0 | 24 | 96% | 0 |
| jianshu:cb137ba5a0e7 | 24 | 96% | 0 |
| jianshu:496af9592d27 | 23 | 96% | 0 |
| jianshu:5ddd62fdaea9 | 21 | 100% | 0 |
| jianshu:494629e92692 | 20 | 90% | 0 |
| jianshu:7ca16c92ca37 | 20 | 100% | 0 |
| jianshu:a6af821a2806 | 20 | 95% | 0 |
| jianshu:d488b0bf3aaf | 19 | 95% | 0 |
| memory:user_learning_path_llm_to_agent.md | 19 | 74% | 0 |
| jianshu:2f7a1fb420d3 | 18 | 100% | 0 |
| jianshu:b72018e88a97 | 17 | 94% | 0 |
| jianshu:5c83da126b48 | 17 | 100% | 0 |
| jianshu:f306adf3480d | 16 | 100% | 0 |
| jianshu:358ea0945978 | 16 | 100% | 0 |
| jianshu:cb2b9e2b68d1 | 16 | 69% | 0 |
| memory:user_knowledge_tier_awareness.md | 16 | 100% | 0 |
| memory:user_learning_resources_llm_agent.md | 16 | 100% | 0 |
| jianshu:7fd6241a7124 | 15 | 87% | 0 |
| jianshu:853ca8318a15 | 15 | 100% | 0 |
| jianshu:d4baff644ce5 | 15 | 100% | 0 |
| jianshu:92a581bd9fdb | 15 | 100% | 0 |
| jianshu:2fae148f015f | 14 | 100% | 0 |
| jianshu:f7d9f6d86145 | 14 | 100% | 0 |
| jianshu:b838f04a9249 | 14 | 100% | 0 |
| jianshu:fe30ef8bd411 | 14 | 86% | 0 |
| jianshu:188b089d2617 | 14 | 100% | 0 |
| jianshu:a63fb211f7ac | 14 | 100% | 0 |
| memory:user_knowledge_tier_practice.md | 14 | 100% | 0 |
| jianshu:8bfd70a9d1ac | 13 | 100% | 0 |
| jianshu:5d35a384574d | 13 | 100% | 0 |
| jianshu:94b6998c6038 | 12 | 100% | 0 |
| memory:user_knowledge_tier_understanding.md | 12 | 100% | 0 |
| jianshu:e5a54813b93d | 11 | 100% | 0 |
| jianshu:db765ff4e36a | 10 | 100% | 0 |
| jianshu:bc16a644784d | 9 | 100% | 0 |
| jianshu:3a3e75af36a7 | 9 | 100% | 0 |
| jianshu:6b5ef101cf66 | 9 | 100% | 0 |
| jianshu:e0ececa61e8e | 7 | 100% | 0 |
| jianshu:3ad9166c02e5 | 7 | 100% | 0 |
| jianshu:ab8c754761bf | 7 | 100% | 0 |
| jianshu:4f90ffb873ab | 6 | 100% | 0 |
| jianshu:11b7d19b02b6 | 5 | 100% | 0 |
| jianshu:ee6a8ebc5bec | 4 | 100% | 0 |

## 与实验1交叉（检索质量 ↔ 答题质量）

- 实验1**低分题(overall<3)**的检索：155 题；rank1 占 96%；未召回 0（0%）
- 实验1**高分题(overall>=4)**的检索：154 题；rank1 占 99%；未召回 0（0%）

解读：若低分题的「未召回/非 rank1」明显高于高分题，说明 agent 答差**主要是检索没排对**（改检索）；若两者差不多，说明检索没问题、**症结在生成端**（改 system prompt / 模型）。

## 未召回清单（自身卡都没检索到，共 0 条，列前 60）


## 怎么读

- **rank1 占比**是检索健康度的核心指标：用原题检索都排不到第一，说明有更高分的干扰卡，agent 实际会优先读错卡。
- **未召回**通常是 query 无有效中文/英文关键词，或卡内容与提问用词不重叠——关键词检索的天然短板（无同义/语义匹配）。
- 这是关键词检索（无 embedding），天花板就在这；要再提升得上向量召回或查询改写/重排（对应学习路线 RAG 进阶）。