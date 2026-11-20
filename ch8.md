# 第8章 框架集成：入口、旋钮与版本

**一句话**：`vLLM`、`TensorRT-LLM`、`SGLang` 都提供 `speculative decoding` 接入，`num_speculative_tokens` 与 `online serving` 的调度共同决定线上收益。

**为什么需要它**：论文里的接受率到你线上服务就变了，因为工程集成有开关、有调度、有版本差异。框架只提供入口，真正的旋钮是投机长度与并发形态。部署前不核对版本与约束，很容易启动失败或收益归零。

**核心结构**：`vLLM` 用一个统一接口切换独立草稿模型、`n-gram` 匹配、`EAGLE` 与 `Medusa` 等不同提案器。`TensorRT-LLM` 支持草稿模型、`Jacobi` 或 `Lookahead Decoding` 类解码、`Medusa` 以及 `rejection sampling`，并与 `in flight batching` 配合。`SGLang` 包含 `EAGLE`、`n-gram` 等提案方式，强调与服务调度、`prefix caching` 的协同。Hugging Face 的生成接口通过 `assistant model` 参数启用投机解码，实现贪心与采样下的无损算法，适合研究与原型验证。

**关键设计**：综合报告口径下，投机解码在小批量、`memory bandwidth` 受限的 `online serving` 上收益最大；批量变大后验证成本上升、每请求摊到的收益下降，需要按实际并发重新调 `num_speculative_tokens`。某些实现要求草稿与目标共享 `tokenizer` 与内核配置，部署前必须核对版本文档。一致的观测指标包括每步 `acceptance rate`、`mean accepted tokens`、`time to first token`、`TPOT`、`throughput` 与 `speedup`，缺一项都无法定位收益去哪了。

**拆解**：
### 1. vLLM：统一提案器接口
`vLLM` 把独立草稿、`n-gram`、`EAGLE`、`Medusa` 收进一个接口，切换提案器不必换整套服务。这是它适合作实验平台的原因。

### 2. TensorRT-LLM：与在飞批处理叠加
`TensorRT-LLM` 在英伟达生态里提供投机解码，与 `in flight batching` 叠加后收益受 `batch size` 影响明显，要在吞吐与单请求延迟间取舍。

### 3. SGLang：与前缀缓存协同
`SGLang` 强调 `prefix caching` 与投机的协同，高重复前缀的工作负载上两者能叠加收益，这要按真实提示核验。

### 4. 框架版本差异是真实约束
参数名、后端约束与共享词表要求随版本变化。照抄旧配置很容易启动失败，必须以实际部署版本文档为准。

**今天怎么做**：用固定提示与固定并发压测，分别记录 `time to first token`、`TPOT`、`throughput` 与 `acceptance rate`；换一个 `num_speculative_tokens` 再测一轮，确认旋钮确实起作用，而不是框架忽略了你的配置。

<div class="chapend" markdown="1">

**容易误解的地方**：以为打开投机开关就自动变快。批量与并发一变，最优投机长度就变，收益要看在线负载而不是默认值。

**记住这一句**：框架只给入口，收益看在线负载；先核对共享 `tokenizer` 与版本，再调 `num_speculative_tokens`。

</div>
