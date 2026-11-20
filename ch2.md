# 第2章 草稿、验证、接受：一次交易的三个角色

**一句话**：`speculative decoding` 用便宜的 `draft model` 连猜若干 `speculative token`，`target model` 用一次 `parallel verification` 批量裁决，再由 `rejection sampling` 修正分布。

**为什么需要它**：你想让目标模型一次吐多个 token，又不能改变它本该给出的输出。直接改解码顺序会破坏分布，硬并行又会泄露未来信息。投机解码给出的交易是：把「算得快」换成「先验后验」——先由便宜的角色提出候选，再由昂贵的角色一次性裁决，最后补一个修正项保证公平。三个角色缺一个，这笔交易就不成立。

**核心结构**：第一个角色是提议者，通常是一个 `draft model`，它按自回归方式连续猜出 γ 个候选，`draft length` 也叫 `num_speculative_tokens`。第二个角色是裁决者，即 `target model`：它把 γ 个候选当作已经给定的序列，用 `teacher forcing` 做一次并行验证，同时得到每个位置的 `acceptance probability`。第三个角色是修正者，`modified rejection sampling` 在被拒绝的位置补一个 token，保证整体分布不偏。`Draft & Verify` 这条路线把三件事讲得最直白；Hugging Face 的 `assistant model` 参数是这条链路最易上手的运行入口。

**关键设计**：验证阶段之所以能并行，关键在于候选已经给定——`teacher forcing` 把「一个接一个生成」变成「一次性给若干位置打分」，原本的 `sequential dependency` 在验证里被拆开。但并行验证只负责判定，不负责兜底：若某个候选被拒，必须从修正分布重采，而不是简单重跑草稿。把验证和修正混成一件事，是无损性最常见的破口。

**拆解**：
### 1. 三个角色各司其职
草稿只负责提议，不决定输出；目标只负责裁决，不替草稿猜；修正采样在拒绝时补位。三者边界清楚，才谈得上无损。

### 2. 草稿阶段：一次连猜 γ 个
`draft model` 按自回归方式生成 `speculative token`，一条候选链长度 γ 决定一次验证能看到多少位置。γ 不是越大越好，后面会讲边际递减。

### 3. 验证阶段：强制教学把串行变并行
`target model` 把整条候选当给定序列做 `teacher forcing`，一次前向给出所有位置的概率。这是用一次权重搬运换多个判定的核心动作。

### 4. 接受与修正：拒绝时补一个，不是重来
`rejection sampling` 在被拒位置按修正分布补 token，前面的接受结果保留。若每次拒绝都从头重来，收益会被完全吃掉。

**今天怎么做**：用 `assistant model` 在 Hugging Face 上跑一条最小投机链路，固定随机种子，记录每一步 `mean accepted tokens` 与是否走了修正分支。先看清三个角色各自在哪一行代码里，再谈优化。

<div class="chapend" markdown="1">

**容易误解的地方**：以为草稿的输出会直接给用户。草稿只是候选，任何一个位置都要经目标裁决，被拒的位置还要经修正采样补。

**记住这一句**：草稿提议、目标裁决、修正采样兜底；三件事缺一件就不叫 `lossless acceleration`。

</div>
