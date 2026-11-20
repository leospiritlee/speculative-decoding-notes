# 第4章 草稿从哪来：五条来源的取舍

**一句话**：`draft model` 有五条来源——独立小模型、`self-speculative decoding`、`knowledge distillation`、`prompt lookup` 与 `inference with reference` 检索式，以及 `feature level drafting` 特征层路线。

**为什么需要它**：显存已经很紧，还要不要塞第二个模型？这是选草稿时最现实的问题。接受率最高的方案未必能部署，能部署的方案接受率未必够。草稿来源的本质，是在显存、训练成本与接受率之间做三方取舍，先看清你缺哪一样。

**核心结构**：第一条是独立 `draft model`：选一个比目标小一到两个数量级的模型，与目标共享 `tokenizer`。综合报告口径下参数量常取目标的几十分之一到几分之一，越小越便宜但分布偏差越大。第二条是 `self-speculative decoding`：截断目标前若干层或 `early exit`，`Draft & Verify` 用目标前几层加轻量适配头做草稿，省掉第二个检查点。第三条是 `DistillSpec` 的 `knowledge distillation`，训练小模型拟合目标分布。第四条是 `training-free` 的 `n-gram matching` 与 `prompt lookup`，`inference with reference` 用参考文本做无损加速。第五条是 `feature level drafting`，`EAGLE` 在中间 `feature` 层做自回归预测。

**关键设计**：独立草稿接受率高，但要额外驻留显存，是小批量场景的主要开销来源；自草稿省显存，却受截断造成的信息损失限制；蒸馏路线接受率高，代价是额外训练流程；检索式零训练零模型，却强依赖输入与输出的重复结构，开放生成几乎无效。综合报告口径下，蒸馏过的草稿相比同尺寸独立草稿可显著抬升接受率，而特征层草稿是当前最强路线之一。没有一条来源是普适最优。

**拆解**：
### 1. 独立小模型：拿显存换接受率
它最直观、接受率最高，但要在显存里常驻第二个模型。小批量服务里，这份驻留就是主要额外成本。

### 2. 自草稿：复用目标自己的前几层
`self-speculative decoding` 不引入第二个检查点，适合显存紧张又不想维护额外模型的场景。代价是共享骨干带来的信息损失。

### 3. 蒸馏：让草稿分布贴近目标
`knowledge distillation` 直接优化 `draft distribution` 与 `target distribution` 的差距，接受率随之上升。换来的是一条训练流程。

### 4. 检索与特征层：不训练和在特征层预测
`prompt lookup` 从历史里匹配 `n-gram`，`feature level drafting` 则在连续特征上预测。前者零成本但挑任务，后者最强但要改目标中间层。

**今天怎么做**：列一张草稿来源对比表，逐项标注「是否额外驻留、是否需要训练、对重复结构的依赖、预期接受率」。先用最小代价的一条跑通，再决定是否升级到特征层。

<div class="chapend" markdown="1">

**容易误解的地方**：以为草稿越小越好。太小分布偏差大，`acceptance rate` 掉下来，省下的草稿成本会被更低的接受数吃回去。

**记住这一句**：草稿来源是显存、训练成本与接受率的三方取舍；先看你缺哪样，再选哪条路。

</div>
