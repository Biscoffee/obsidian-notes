# 实验1 · Agent 评测体系（全库）

生成时间：2026-05-25 10:49　裁判模型：MiMo　评分尺度：Opus 设计

- 总题数：1574
- 有效评分：0
- agent 报错：1574
- 裁判无效（空返回/解析失败）：0
- 裁判累计 token：0

## 总分

**综合平均 overall：0.00 / 5**

| 维度 | 平均分 |
|---|---|
| 准确性 | 0.00 |
| 完整性 | 0.00 |
| 相关性 | 0.00 |
| 清晰度 | 0.00 |

## 按来源分布

| source | 题数 | 平均分 |
|---|---|---|

## 分数档分布

| 档位 | 题数 |
|---|---|
| 5 分档 | 0 |
| 4 分档 | 0 |
| 3 分档 | 0 |
| 2 分档 | 0 |
| 1 分档 | 0 |

## 失败清单（overall < 3，共 0 条，列前 60）


## agent 报错（1574 条）

- [iosqa] 编译链接流程：__AGENT_ERROR__ Boom: x
- [iosqa] 符号表和调试信息是什么？：__AGENT_ERROR__ Boom: x
- [iosqa] 讲一讲词法分析，语法分析，语义分析，目标代码生成和优化里面做了什么？：__AGENT_ERROR__ Boom: x
- [iosqa] 什么是dyld?：__AGENT_ERROR__ Boom: x
- [iosqa] iOS中如果两个app用了同一个动态库：__AGENT_ERROR__ Boom: x
- [iosqa] OC采用的编译器是什么？：__AGENT_ERROR__ Boom: x
- [iosqa] 启动流程：__AGENT_ERROR__ Boom: x
- [iosqa] 启动优化：__AGENT_ERROR__ Boom: x
- [iosqa] Mach-O加载动态库方法的流程：__AGENT_ERROR__ Boom: x
- [iosqa] 讲一讲iOS的内存管理（引用计数+MRC+ARC + AutoreleasePool + TagP）：__AGENT_ERROR__ Boom: x
- [iosqa] MRC管理内存的思想：__AGENT_ERROR__ Boom: x
- [iosqa] TaggedPointer：__AGENT_ERROR__ Boom: x
- [iosqa] iOS内存机制（内存压缩机制）：__AGENT_ERROR__ Boom: x
- [iosqa] 内存泄漏以及检测方法：__AGENT_ERROR__ Boom: x
- [iosqa] 三种循环引用：__AGENT_ERROR__ Boom: x
- [iosqa] class结构：__AGENT_ERROR__ Boom: x
- [iosqa] 字符串的三种类型：__AGENT_ERROR__ Boom: x
- [iosqa] 空指针和野指针的区别：__AGENT_ERROR__ Boom: x
- [iosqa] class、objc_getClass、object_getclass 方法有什么区别?：__AGENT_ERROR__ Boom: x
- [iosqa] 什么是AutoreleasePool？：__AGENT_ERROR__ Boom: x
- [iosqa] 多个线程可以共享一个AutoreleasePool吗：__AGENT_ERROR__ Boom: x
- [iosqa] 哨兵节点的作用：__AGENT_ERROR__ Boom: x
- [iosqa] 子线程默认不会开启 Runloop，那出现 Autorelease 对象如何处理？不手动处理会内存泄漏吗？：__AGENT_ERROR__ Boom: x
- [iosqa] 讲一讲Block是什么，Block的本质？：__AGENT_ERROR__ Boom: x
- [iosqa] 如何解决Block的循环引用：__AGENT_ERROR__ Boom: x
- [iosqa] 为什么要进行Block判空：__AGENT_ERROR__ Boom: x
- [iosqa] 深拷贝和浅拷贝：__AGENT_ERROR__ Boom: x
- [iosqa] 容器完全深拷贝的几种方式：__AGENT_ERROR__ Boom: x
- [iosqa] 如何实现一个支持拷贝的自定义类，还有深拷贝？：__AGENT_ERROR__ Boom: x
- [iosqa] mutableString 使用 copy 关键字修饰会发生什么？：__AGENT_ERROR__ Boom: x

## 怎么读这份报告

- **均分**反映 agent 在自己知识库题目上的综合表现。
- **按来源**看哪类知识 agent 答得差（检索没召回 or 综合能力弱）。
- **失败清单**是 Harness 反馈循环的输入：逐条看是检索没命中、答错、还是跑题，再针对性改 ask_kb 检索 / system prompt。
- 裁判尺度由 Opus 设计，跑完已用 `--export` 抽检校准（见同目录抽检报告）。