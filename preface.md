## 序：从一次串行等待说起

自回归解码有一个让人无奈的规矩：第 t 个 token 必须等前 t-1 个都生成完才能开始。批量小、算术强度低，图形处理器大部分时间不在算，而在等权重从显存搬进计算单元。你堆算力，尾延迟几乎不动——瓶颈从来不在算力，在 `memory bandwidth`。

投机解码换的是这件事的组织方式：让一个便宜的草稿先连续猜出若干候选，再让昂贵的目标模型用一次并行前向同时验证这些候选。验证时把候选当作已给定的序列做 `teacher forcing`，一次目标前向就能判定多个位置的接受或拒绝。只要草稿猜得够准，就等于用一次目标前向换出多个 token。这不是近似方法，不牺牲质量，所以叫 `lossless acceleration`。

这本专题精读写这条链路的十二道关：先看清自回归瓶颈，再走草稿—验证—接受的核心交易、接受率与期望加速、草稿来源、多 token 预测与 `Medusa`、树式验证、采样一致性、框架集成、批量并发、质量回归、优化叠加，最后落到什么时候不该用。每关回答同一个问题：一次目标前向换多个 token，换来多少加速，又欠下什么账。

**分工**：与《机器学习系统精读》不同，那本讲训练与推理的系统底座；与《端侧AI推理精读》也不同，那本讲压缩部署；与《推理模型精读》更是两件事，那本讲让模型回答前先想。这本只讲 `speculative decoding` 这条链：一次前向怎么合法地多吐几个 token，以及在真实服务里收益为什么常常偏离论文数字。

## 投机解码全景图

```text
+----------------------------------------------------------------------+
| SPECULATIVE DECODING: ONE FORWARD, MANY TOKENS                       |
|                                                                      |
| [1] bottleneck     -> [2] draft + verify  -> [3] acceptance          |
| [4] draft sources  -> [5] Medusa / MTP     -> [6] Lookahead / EAGLE  |
| [7] losslessness   -> [8] serving engines -> [9] batching            |
| [10] mismatch      -> [11] composition    -> [12] boundaries         |
|                                                                      |
| draft guesses cheaply; target verifies all guesses in one pass       |
+----------------------------------------------------------------------+
```

这张图不是模块调用顺序，而是「一次目标前向想多吐几个 token，代价依次落在哪里」：先问瓶颈是不是访存，再问草稿从哪来、怎么验、怎么保证分布不变、框架怎么接、并发怎么摊，最后问什么条件下这笔交易根本不划算。越往后，代价越隐性，也越需要在上线前想清楚。

## 十二章速查表

| 关 | 主题 | 一句话 |
|---|---|---|
| 1 | 自回归瓶颈 | 解码卡在 `memory bandwidth`，不在算力 |
| 2 | 核心交易 | `draft model` 提议，`target model` 并行裁决，修正采样兜底 |
| 3 | 接受率 | `acceptance rate` 决定收益，边际递减还要扣草稿代价 |
| 4 | 草稿来源 | 独立、自草稿、蒸馏、检索与特征层五条路 |
| 5 | 多 token 预测 | `Medusa head` 与辅助头把提议能力内建进目标 |
| 6 | 树式验证 | `draft tree` 一次验多支，`tree attention mask` 是因果红线 |
| 7 | 无损保证 | `modified rejection sampling` 让边际分布等于目标 |
| 8 | 框架集成 | `vLLM`、`TensorRT-LLM`、`SGLang` 只给入口，收益看在线负载 |
| 9 | 批量并发 | `throughput` 与 `TPOT` 分开测，甜点区是小批量低延迟 |
| 10 | 错配与质量 | `total variation distance` 一大，`acceptance rate` 就掉 |
| 11 | 优化叠加 | `quantization`、`KV cache` 与并行是叠加项，不是替代品 |
| 12 | 适用边界 | 短输出、低接受率、强约束场景开销摊不薄 |

## 三层读法：机制、工程、判断

看到一个新方法，先问它属于哪一层。`speculative decoding`、`speculative sampling`、`modified rejection sampling`、`detailed balance`、`greedy decoding` 是机制层，论文讨论无损性怎么成立；`Medusa`、`EAGLE`、`Lookahead Decoding`、`draft tree`、`num_speculative_tokens` 是算法与旋钮层，给出可运行的提案与验证形态；`vLLM`、`TensorRT-LLM`、`SGLang`、`in flight batching`、`prefix caching`、`TPOT` 是工程层，告诉你线上收益为什么和论文不同。

三层不能互相替代。机制漂亮不等于工程上跑得稳，框架能跑也不等于这笔账划算。综合报告口径建议按「先确认负载是不是解码受限、再量草稿与目标分布差距、最后才选提案器与投机长度」的顺序推进，每一步都用自己的模型与真实并发做对照，而不是照搬别人的接受率。

## 一个可验收的出口

读完十二关，出口标准不是记住某个模型名，而是能对着一个具体服务回答四句话：你的负载是不是 `memory bound`、批量多大；草稿与目标的 `acceptance rate` 按位置怎么衰减；你选的提案器要不要额外驻留显存；开启投机后 `time to first token`、`TPOT` 与 `throughput` 分别怎么变。四句都答得出，说明这条链路已经能用于判断，而不只是读过一篇综述。

对做服务的人还要多一句：投机的收益是概率性的，单次请求方差很大，高百分位延迟比平均接受数更能暴露问题。这一句答不出来，前面四句答得再好也算不对工程账。

## 语料边界与口径

一手论文逐字留存，用于机制与术语的精确写法：Fast Inference from Transformers via Speculative Decoding、Accelerating Large Language Model Decoding with Speculative Sampling、Medusa、Lookahead Decoding、EAGLE、EAGLE-2、EAGLE-3、SpecInfer、Draft & Verify、Better & Faster Large Language Models via Multi-token Prediction、Blockwise Parallel Decoding、DistillSpec、Online Speculative Decoding、Inference with Reference、Token Recycling 与一篇综合综述。官方文档给出接入形态：vLLM、TensorRT-LLM、SGLang 与 Hugging Face 的生成接口。

综合报告只补工程判断，比例、成本、吞吐、加速比与延迟数字一律标注「综合报告口径」。十二章的顺序、场景与速查表属于「延伸，非语料原文」，不加行内代码标记；行内代码只用语料中确实出现的标识符。安全与合规内容只作工程提示，不构成法律意见。