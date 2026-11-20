# 第1章 三层边界：先分清记在哪，再谈记多少

**一句话**：`memory` 不是存得越多越好，先划清 `working memory`、外部 `long-term memory` 与模型权重三层，才知道什么该留。

**为什么需要它**：你接手一个刚加上记忆的助手，发现它把上周的闲聊和关键约束一起塞进 `context window`，回答越来越慢、越来越偏。第一反应往往是「换个更大的向量库，存更多」。但这不是存储量的问题——`attention budget` 有限，`lost in the middle` 让中部信息被忽略，存得越多反而取回越难。真正该先做的是划边界：哪些内容此刻就该在台面上，哪些该落到外部存储按需取回，哪些其实该沉淀进 `procedural memory` 或模型权重，而不是每次都靠 `in-context learning` 重新带进来。

**核心结构**：第一层是 `sensory memory` 与 `working memory`，对应当前窗口里的内容，是唯一无需检索即可被注意力直接访问的部分。第二层是外部 `episodic memory` 与 `semantic memory`：前者记「发生过什么」，带时间戳与情境；后者记「是什么」，去情境化的事实与关系。第三层是 `procedural memory` 与模型权重，管「怎么做」和被训练固化的能力。`short-term memory` 与 `long-term memory` 是另一条正交视角；`persistent memory` 与 `stateful agents` 强调的是跨会话存活，`memory scope` 则把存储切成 `user-level memory`、`session-level memory` 与 `agent-level memory`。

**关键设计**：真正要盯的不是总量，而是「访问频率、时效敏感度、结构化程度」三个维度。综合报告口径下的判断是，四类记忆在真实系统里往往共存，边界是模糊的：一条 `episodic memory` 被反复提炼可上升为 `semantic memory`，一段 `procedural memory` 可由经历归纳而来。所以不必纠结纯分类，而要给每层定清「谁能写、谁能读、活多久、删了会怎样」，这正是 `memory governance` 的起点。

**拆解**：
### 1. 窗口是台面，不是仓库
`working memory` 的组织方式直接决定推理质量，因为它已经在台面上。把无关历史持续堆进窗口，不会让模型记得更牢，只会挤占真正有用的 `attention budget`。

### 2. 外部记忆是按需取回的档案
`episodic memory` 与 `semantic memory` 放在外部存储，靠 `memory retrieval` 取回。它们的价值不在存了多少，而在被问到时能不能被准确找到。这是 `RAG` 思路在 Agent 里的延伸。

### 3. 权重与记忆的边界
有些东西该写进权重，有些该留在外部。边界由更新成本与可解释性决定：频繁变化、需要逐条删除的事实适合放外部；稳定且难以显式描述的模式才交给训练。`in-context learning` 是两者之间的临时通道。

### 4. 三层按三个维度分
访问频率高的留近，时效敏感的记录时间戳，结构化程度高的进 `knowledge graph`。这套分层比「一个库存所有」更能控制成本与噪声。

**今天怎么做**：拿你在维护的一个助手，把当前保存的全部内容导出成一张表，逐条标注「属于哪一层、谁写的、上次被读到是什么时候」。你会发现大量 `session-level memory` 从不跨会话复用，却被当作 `user-level memory` 永久保存。以下落地建议属「延伸，非语料原文」，请按你实际使用的框架版本核对。

<div class="chapend" markdown="1">

**容易误解的地方**：以为窗口变长就能替代 `memory`。窗口再长也是有限的台面，而 `memory` 解决的是「把什么放上台面」。

**记住这一句**：先划三层边界，再谈存多少；`memory` 的目标是用对 `attention budget`，不是用满 `context window`。

</div>
