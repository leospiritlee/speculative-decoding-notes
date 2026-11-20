# 第7章 采样一致性与无损保证

**一句话**：`lossless` 靠 `modified rejection sampling` 与 `detailed balance`：`speculative decoding` 与 `speculative sampling` 在采样设定下给出同样的 `output distribution`。

**为什么需要它**：你必须证明投机之后的生成分布和原来逐 token 采样完全一致，否则不敢上线。这不是「看起来像」的问题，而是分布级的等价证明。两条独立提出的技术路线给出了同样的无损保证，理解它们为什么等价，是这一章的目标。

**核心结构**：设 `target distribution` 为 p、`draft distribution` 为 q。对每个候选 token，以概率 min(1, p(x)/q(x)) 接受；若拒绝，则从 `residual distribution`——正比于 (p(x) − q(x)) 的正部并归一化——重新采样一个 token 来填补；若整段候选都被接受，还可以从 p 多采一个 token 作为奖励。`detailed balance` 解释了接受判据与修正分布相乘后，草稿提议的偏置为何被精确抵消，最终边际分布等于 p。`total variation distance` 衡量 p 与 q 的差距，也解释接受率高低。`greedy decoding` 下这一切退化为更简单的规则：目标在该位置的 `argmax` 与草稿一致才接受，否则用目标的 `argmax` 替换。

**关键设计**：贪心设定严格无损，采样设定必须配修正分布——「拒绝就重抽草稿」不满足无损。主流有两种等价表述：Leviathan 等人的 `speculative decoding` 与 Chen 等人的 `speculative sampling`，二者在贪心之外的采样设定下给出同样保证。`typical acceptance` 这类放松严格无损的策略，综合报告口径下缺乏统一的评测标准，不同文档口径不一，引用时须回溯一手报告。

**拆解**：
### 1. 接受判据 min(1, p/q)
候选按目标与草稿概率之比接受，概率比越接近一越容易留下。这是无损性的第一道门。

### 2. 修正分布：拒绝后从残差重采
`residual distribution` 取 (p − q) 的正部归一化，保证补出来的 token 不把整体分布带偏。少了这一步，无损就不成立。

### 3. 详细平衡：偏置为何精确抵消
`detailed balance` 把接受与修正配对，证明草稿提议的偏置被精确抵消，最终边际分布回到 p。这是等价证明的核心。

### 4. 贪心特例：`argmax` 一致才接受
`greedy decoding` 下退化为 `argmax` 比较，不一致就用目标的结果替换。因为它最干净，常被用来做无损性的第一道自检。

**今天怎么做**：用固定随机种子对比开启与关闭投机的 `output distribution`，统计两者生成差异是否落在采样噪声范围内；贪心设定下要求输出逐 token 完全一致，不一致就回头查修正分布实现。

<div class="chapend" markdown="1">

**容易误解的地方**：以为「输出看起来差不多」就算无损。无损是分布级的等价，必须固定种子逐 token 比，或做分布统计，而不是靠肉眼相似。

**记住这一句**：无损不是近似：接受判据加 `residual distribution` 保证边际分布等于目标；贪心最干净。

</div>
