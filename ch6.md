# 第6章 树式验证：Lookahead 与 EAGLE

**一句话**：`Lookahead Decoding` 用 `Jacobi decoding` 迭代加 `n-gram matching` 构造候选，`EAGLE` 在 `feature` 层做自回归草稿；`draft tree` 配合 `tree attention` 让一次前向验证多条分支。

**为什么需要它**：线性投机一次只验一条候选链，草稿一弱，前面猜的后面全被拒，白做功。树状投机把多个候选摆成一棵 `token tree`，一次目标前向验证多条分支，命中率明显高于单链。代价是验证序列更长、注意力掩码更复杂，需要按接受率动态控制树的规模。

**核心结构**：`Jacobi decoding` 把自回归生成看成求 `fixed point`：从随机猜测的整段序列出发，反复用模型并行预测每个位置，直到连续两轮一致即收敛，天然并行且不依赖草稿模型。`Lookahead Decoding` 把雅可比迭代与历史 `n-gram matching` 结合，用 `n-gram` 表快速构造候选窗口，再由目标 `parallel verification`，接受与模型预测一致的前缀。`EAGLE` 在目标中间层加轻量自回归头预测 `feature`，再经冻结解码层还原成 token；`EAGLE-2` 用 `dynamic draft tree` 按 `global acceptance` 决定扩展哪些分支；`EAGLE-3` 引入 `training time test` 与多层特征融合。`SpecInfer` 是 `tree based speculative inference` 的代表。

**关键设计**：因果性是一票否决项：`tree attention mask` 必须保证每个节点只看它的祖先，否则引入未来信息、直接毁掉无损性，这是树投机最容易写错的地方。综合报告口径下，树大小要按接受率动态选择，过大的树会让验证本身成为瓶颈，反而吃掉收益。`token recycling` 把被拒候选回收进下一轮草稿，减少浪费。这些数字多来自作者自报，引用前应回溯一手技术报告。

**拆解**：
### 1. 雅可比迭代：不动点式并行
`Jacobi decoding` 不需要草稿模型，从猜测出发反复并行预测直到收敛。它的弱点是收敛轮数不稳定。

### 2. Lookahead：历史窗口当草稿
`Lookahead Decoding` 用 `n-gram` 表构造候选窗口，接受与模型预测一致的前缀。它无损且无需训练，适合不想引入额外模型的场景。

### 3. EAGLE：在特征层预测下一步
`feature level drafting` 利用连续、信息更丰富的中间 `hidden state`，接受率明显高于同等规模的 token 层草稿。`EAGLE-2` 与 `EAGLE-3` 分别在树的动态扩展与训练时测试上推进。

### 4. 树验证与因果掩码
`SpecInfer` 用 `tree attention mask` 在一个批次里验证整棵树。掩码写错会泄露未来信息，比接受率低严重得多。

**今天怎么做**：开一次 `EAGLE-2` 的 `dynamic draft tree`，记录不同树规模下的 `acceptance length` 与验证耗时，找到「再多一条分支反而不划算」的拐点，再核对掩码的因果性。

<div class="chapend" markdown="1">

**容易误解的地方**：以为树越大越好。树大让序列变长、注意力掩码变复杂、验证内核并行度被摊薄，可能把收益吃回去。

**记住这一句**：树式投机一次验多支，`tree attention mask` 是因果红线；扩展看接受率，不看直觉。

</div>
