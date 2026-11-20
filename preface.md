## 序：不是把历史存下来，而是把注意力用对

大语言模型本身没有跨会话的持久状态。一次对话结束，除非把内容重新放回上下文，模型对过往一无所知。`context window` 虽然在变长，却有两条硬约束：它是有限的，且装入的 `token` 越多成本越高；把大量无关信息塞进去会触发 `lost in the middle`，也就是模型对中部信息的利用率明显低于首尾。于是真正的问题不是窗口够不够大，而是怎么在有限的 `attention budget` 里，把此刻最相关的历史放进去。

这本专题精读写的是这条链路的十二道关。先划清三层边界：当前 `working memory` 放什么、外部 `episodic memory` 与 `semantic memory` 存什么、`procedural memory` 与模型权重各自管什么。再跟着一条记忆走完 `memory write`、`memory retrieval`、`memory compression`、`memory conflict`、`memory forgetting`，看它怎么被评测，又怎么在 `context assembly` 里被装配成一次回答。后半本转向 `reflection` 与 `experience`、`knowledge graph`、`Mem0` / `Zep` / `MemGPT` 三类实现，以及 `privacy` 治理和「什么时候不该上记忆」。

**分工**：与两本已有的 Agent 书不同，那两本讲单 Agent 的循环、工具与协作；与《主流AI技术精读》也不同，那本讲方法地图。这本只讲一条记忆从写入到装配的链路，以及链路上真正卡人的判断。与《推理模型精读》分工：那本讲模型内部的推理，这本讲模型外部的记忆与上下文怎么组织。

## 一、一条记忆的链路速查

```text
+------------------------------------------------------------------------+
| AGENT MEMORY AND CONTEXT: FROM ONE WRITE TO ONE ASSEMBLY                |
|                                                                        |
| [1] boundary -> [2] write/conflict -> [3] retrieval/temporal            |
| [4] compress/forget -> [5] evaluation -> [6] context budget             |
| [7] assembly order -> [8] reflection -> [9] graph memory               |
| [10] systems -> [11] privacy -> [12] when-not-to-use                   |
|                                                                        |
| one write -> one retrieve -> one assemble -> one answer                  |
+------------------------------------------------------------------------+
```

这张图不是模块调用顺序，而是「一条记忆从产生到被用掉，代价依次落在哪里」：先问它该不该被记、记在哪一层，再问怎么取、取回的会不会过期或矛盾，再问压缩与遗忘丢掉了什么，最后问它在有限预算里以什么顺序进入提示。越往后，代价越隐性，也越需要在上线前想清楚。

## 二、十二章一句话速查

| 章 | 主题 | 一句话 |
|---|---|---|
| 1 | 三层边界 | `memory` 不是越多越好，先分清上下文、外部存储与权重三层 |
| 2 | 写入与冲突 | 写入精度比召回更重要，冲突要局部失效而非全局覆盖 |
| 3 | 检索与时间 | `hybrid retrieval` 加时间置信度，才答得了「什么时候」 |
| 4 | 压缩与遗忘 | `compression` 必然有损，关键是损失落在无关细节 |
| 5 | 长期评测 | `LongMemEval` 拆开提取、多跳、时序与更新四类能力 |
| 6 | 装配预算 | `prefix caching` 与 `KV cache` 决定反复装配的真实成本 |
| 7 | 装配顺序 | 首尾放关键指令，检索片段带 `source attribution` |
| 8 | 反思经验 | `Reflexion` 用语言反馈把失败轨迹变成下一次的上下文 |
| 9 | 图记忆 | `temporal knowledge graph` 让随时间变化的事实可回答 |
| 10 | 系统实现 | `Mem0` 偏写路径，`Zep` 偏时序，`MemGPT` 偏分层上下文 |
| 11 | 隐私治理 | 删除要传播到派生索引与摘要，`data provenance` 是前提 |
| 12 | 上线判断 | 绝大多数问题出在写入端与检索端，不是生成端 |

## 三、症状指向哪一章

| 症状 | 先读 | 为什么 |
|---|---|---|
| 机器人记不住上周说过的话 | 第 1、2 章 | 先看边界划分，再看写入判定是否漏写 |
| 越聊越慢、提示越来越长 | 第 6、7 章 | 预算与装配顺序决定了重复装入多少历史 |
| 取回了过期或自相矛盾的事实 | 第 3、2 章 | 检索的时间置信度与冲突的局部失效要一起看 |
| 长对话一到后段就答偏 | 第 6 章 | `lost in the middle` 与 `attention budget` 的直接后果 |
| 压缩之后关键约束没了 | 第 4 章 | 抽取式与摘要式的损失落点不同 |
| 说不清记忆系统到底有没有变好 | 第 5 章 | 答案准确率、操作指标与成本要同时报告 |
| 失败教训下次又犯 | 第 8 章 | 缺的是把 `trajectory` 写成可复用 `experience` |
| 「以前住北京现在住上海」答错 | 第 9 章 | 需要 `bi-temporal` 与有效期，而非简单覆盖 |
| 用户要求删除却删不干净 | 第 11 章 | 删除要联动缓存、索引与摘要 |
| 上了记忆反而更贵更差 | 第 12 章 | 先看这个场景该不该上，再看上线回归 |

## 四、二十个自测问题

1. `context window` 和 `memory` 的分工是什么，为什么窗口变大不等于记忆解决？
2. `working memory`、`episodic memory`、`semantic memory`、`procedural memory` 各回答什么问题？
3. 什么信息值得写进 `long-term memory`，谁来判断「值得」？
4. `fast write path` 与 `slow channel` 为什么在多个系统里反复出现？
5. 两条 `memory` 矛盾时，按什么信号裁决，为什么不能一律「最新覆盖最旧」？
6. 单纯 `vector search` 在「谁告诉我的」这类问题上为什么会输？
7. `hybrid retrieval` 叠加 `metadata filtering` 与 `reranking` 各补了什么？
8. 时间衰减与 `temporal reasoning` 是不是一回事？
9. `prompt compression` 为什么必然有损，损失该落在哪里？
10. `rate-distortion` 框架对压缩给出了什么结论？
11. `memory forgetting` 是缺陷还是资源治理，依据是什么？
12. `LongMemEval` 把长程记忆拆成哪几类能力？
13. 为什么评测要同时报告准确率、`recall` 与成本？
14. `context assembly` 里哪些内容必须稳定不变，哪些要按查询动态挑？
15. 为什么关键指令放首尾，中部放什么？
16. `prefix caching` 与 `KV cache` 分别省的是哪一段开销？
17. `Reflexion` 的 `verbal reinforcement` 与 `fine-tuning` 有什么本质区别？
18. `Agent Workflow Memory` 和 `Memory-R1` 把什么固化成了可复用 `skill`？
19. `temporal knowledge graph` 里 `valid time` 与事实有效期怎么配合？
20. 什么情况下根本不该上 `memory`，上线前该跑哪些回归？

## 五、一个提醒：语料边界与口径

一手论文与官方文档逐字留存，用于机制与术语的精确写法；代码、配置项与系统名都以此为准。综合报告只补工程判断，凡涉及比例、成本、延迟、榜单分数与趋势，一律标注「综合报告口径」，这些数字只用于建立量级直觉，引用前必须回一手来源核对。

十二关的顺序、上面的场景、速查表和自测题属于「延伸，非语料原文」，是编排者组织的阅读路径，不是任何一篇论文或官方文档的原文；这些行不加行内代码标记，行内代码只用语料中确实出现的标识符。本书明确不写厂商价格、版本时间线、具体部署命令与法律结论；`privacy` 与 `compliance` 内容只作工程提示，不构成法律意见。资料年代口径不一致处按各自来源标注，不替来源调和。