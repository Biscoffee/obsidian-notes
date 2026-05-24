# Harness 对比实验：Context 管理策略

任务：长对话早期信息保持（3 组 needle，12 轮干扰，window=3）

| 策略 | 平均召回率 | 平均累计 prompt_tokens | 平均耗时 |
|---|---|---|---|
| full | 100% | 5,671 | 82s |
| window | 0% | 2,340 | 78s |
| compaction | 0% | 2,430 | 183s |
| rag | 100% | 2,512 | 115s |

**怎么读**：召回率高=没忘早期信息；prompt_tokens 低=省钱。
理想策略是「召回高 + token 低」——看哪种最接近左上角。
