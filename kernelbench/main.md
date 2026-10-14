# KernelBench 评测setting survey

<!-- > 调查截止 2026-08-19。只看 forward correctness 和 latency；single generation、pass@k、多轮 refinement、evolutionary search、按 latency 选优都放在同一张表里比，generation budget 和 selection 必须单独摆出来看。表里写“未披露”，意思是论文和能找到的代码都确认不了，不等于采用了官方默认值 -->

# 摘要

<!-- KernelBench 到现在都没有一个各工作共同遵守的稳定 protocol。同样一个 `fast_1` 或 speedup，背后可能同时换了 task revision、题目子集、Module state、autograd、输入分布、数值容差、计时器、cache policy、PyTorch baseline、GPU、candidate budget、候选选择规则里的好几项，跨论文的数字基本没法直接排序 -->

<!-- 最容易看走眼的是下面五件事，GPU 型号反而排不上号 -->

<!-- - **“250 题”不等于同一数据集**：v0 和 v0.1 都是 L1/L2/L3=100/100/50，但 v0.1 大改了 shape、batch size、算子语义和 RNN state I/O -->
KernelBench 的评测 setting 目前主要乱在这几处：
- **官方没在 eval mode 下跑**：evaluator 没有调用 `model.eval()`，默认 `training=True`；correctness、candidate timing、reference timing 三处的 autograd context 还不一致
<!-- - **candidate budget 会直接改变结果含义**：one-shot、8 samples ×3 turns、15 iterations、100 search steps 和 200-turn trajectory 都纳入调查，但不能把对应数字当作同预算成绩 -->
- **baseline 不统一**：Torch Eager、`torch.compile`、`reduce-overhead`、Eager/compile 二者取快、TF32-enabled baseline 都被称作 reference
- **数据集版本不统一**： KernelBench 有 v0 和 v0.1 两个版本，且当前 main 分支仍在加入新特性
<!-- - **计时对象不统一**：CUDA Event forward latency、Triton `do_bench` median、NCU kernel cycles、CUDA Graph -->

<!-- 只能记一条的话：一条 KernelBench 结果的身份是下面这个 protocol tuple，论文名加 GPU 型号说明不了什么

```text
KB = data SHA + task IDs + dtype
   + module state + autograd
   + oracle(seeds, input distributions, tolerance)
   + timer(method, warmup, trials, cache, statistic)
   + reference(backend, compile flags, same-run?)
   + device(SKU, power/clock, software stack)
   + generation(requests, candidates, decoding, self-selection)
   + feedback(compilation, correctness, profiler, latency)
   + metric(threshold, denominator, aggregation)
``` -->

<!-- ## 如何读 generation budget

one-shot 在这里不设为准入门槛。每题一次生成、`k` 个独立样本、带 execution feedback 的多轮 refinement、按实测 latency 选 best candidate，都进后文的表；差别落在 requests、turns、candidate evaluations、feedback、selection 五个字段上

这些 setting 可以放在同一张表里看，数值不能直接排序。`avg@k` 近似单样本期望，`pass@k` 是至少成功一次，best-of-k 和 best-history 还多一层选优；模型和 evaluator 都不动，只改这几个字段，correctness 和 speedup 就会变 -->

# train/eval mode 混用

<!-- 论文里说的 “eval mode” 至少是三件不同的事：`model.eval()`、关掉 autograd、只做 forward inference，三者互不等价。
PyTorch 官方文档也写明 [Module evaluation mode 与 no-grad/inference mode 相互正交](https://docs.pytorch.org/docs/stable/notes/autograd.html#evaluation-mode-nn-module-eval)，而 `.eval()` 本身等价于 `.train(False)`，见 [`nn.Module.train`](https://docs.pytorch.org/docs/stable/generated/torch.nn.Module.html#torch.nn.Module.train)

| 维度 | 可选状态 | 实际影响 |
|---|---|---|
| Module state | 默认 `training=True` / 显式 `.eval()` / 两者都测 | 改变 Dropout、BatchNorm 等 stateful operator 的语义 |
| Autograd | enabled / `no_grad()` / `inference_mode()` | 改变 graph construction、tensor metadata 和部分 overhead |
| 工作负载语义 | forward-only，但 Module 可处于 training/eval state | 不能从论文里单独一个 “inference” 或 “train” 字样推出前两项 | -->

<!-- 官方 v0、v0.1 和调查日 `main` evaluator 都没有调用 `.eval()` 或 `.train()`，所以 Module 通常保持默认 `training=True`。但 Autograd 必须按路径拆开：correctness 在 `torch.no_grad()` 内；v0/v0.1 与调查日 `main` 的 candidate timing 没有显式禁用 Autograd；调查日 `main` 的独立 reference timing helper 则整体位于 `torch.no_grad()` 内。所以准确的说法是 **default-training Module state + forward-only，且 correctness/reference/candidate timing 的 grad context 不统一**，源码可见 [v0 evaluator 与 candidate timing](https://github.com/ScalingIntelligence/KernelBench/blob/6500bbc8cf102520d7a8f09be34ee6d5db1c29b0/src/eval.py#L294-L458)、[v0 correctness `no_grad`](https://github.com/ScalingIntelligence/KernelBench/blob/6500bbc8cf102520d7a8f09be34ee6d5db1c29b0/src/eval.py#L544-L590)、[调查日 candidate timing](https://github.com/ScalingIntelligence/KernelBench/blob/423217d9fda91e0c2d67e4a43bf62f96f6d104f1/src/kernelbench/eval.py#L607-L637) 和 [`measure_ref_program_time`](https://github.com/ScalingIntelligence/KernelBench/blob/423217d9fda91e0c2d67e4a43bf62f96f6d104f1/src/kernelbench/timing.py#L9-L108)。candidate 是否真的构建 graph 还取决于输入与参数的 `requires_grad`，不能简化成所有题必然带 graph -->

公开代码里能确认的 mode 列在下表。要区分“论文报告的协议”和“现在的 release code”
| 工作 / evaluator | Module state | Autograd |
|---|---|---|
| KernelBench v0 / v0.1 / main | 默认 `training=True` | correctness：`no_grad()` timing：未显式禁用 |
| [METR KernelAgent](https://github.com/METR/KernelBenchFiltered/blob/9507be4154168be123b6bfe764aea2263625178e/evaluate_solution.py) | 默认 `training=True` | `no_grad()` |
| [Sakana robust-kbench](https://github.com/SakanaAI/robust-kbench/blob/078f5bab29934a822268d59a4e707d449abf9b4e/robust_kbench/sandbox/eval_forward_fn.py) | 默认 `training=True` | `no_grad()` |
| [CUDA Agent](https://github.com/BytedTsinghua-SIA/CUDA-Agent/blob/473025c8af7878e928525138b5cb327d6bcdf9dd/agent_workdir/utils/verification.py) | 显式 `.eval()` | `no_grad()` |
| [Dr. Kernel correctness](https://github.com/hkust-nlp/KernelGYM/blob/3a84417f8c0efaadb215ef638b37d12e71ed20f3/kernelgym/toolkit/kernelbench/correctness.py) | 默认 `training=True` | `no_grad()` |
| [Dr. Kernel anti-hack](https://github.com/hkust-nlp/KernelGYM/blob/2f08930e3975923ccfe00d3f8f1792a2697b821b/kernelgym/toolkit/kernelbench/triton_detect.py#L442-L530) | 默认 `training=True` | `inference_mode()` 与 `enable_grad()` 都跑 |
| [CUDA-L1](https://github.com/deepreinforce-ai/CUDA-L1/blob/51e8356bf844101cab2aeeca4bdff47ebb1953b8/eval/eval_cuda.py) | 显式 `.eval()` | `no_grad()` |
| [KernelSkill / KernelMem runner](https://github.com/0satan0/KernelMem/blob/8b57ccc9adc2ae2f11fc487fd458a7ecc1ea014d/utils/compile_and_run.py#L157-L220) | 显式 `.eval()` | 未显式禁用 |
| 其余多数工作 | 未披露 | 未披露 |

<!-- CUDA Agent 的公开 workspace 显式 `.eval()+no_grad()`，Dr. Kernel 的 correctness 则保留默认 `training=True` 并用 `no_grad()`，anti-hack checker 又分别执行 inference/grad-enabled context；两篇论文说的 eval/train 根本不是同一个轴。后文把它们和其他工作放进同一张表，但 mode 差异留成单独的配置列，不能只看 speedup -->

<!-- KernelBench-RLVR-120B 的 “fast evaluation mode” 指每个 kernel 只做 5 次 performance trial 的快速 proxy，不是 `.eval()`；其 vendored evaluator 的 correctness 是 default-training state + `no_grad()`，candidate timing 路径未显式禁用 Autograd。再看到 eval mode 字样，先问清楚它说的是 Module state、autograd context，还是 evaluator 的速度档位 -->

<!-- 这不是抠字眼。在 [`129232e` task tree](https://github.com/ScalingIntelligence/KernelBench/tree/129232ea3c4cfa05c2dd5473bdb935d6246dc84b/KernelBench) 上以 `rg -l 'Dropout|dropout|BatchNorm|batch_norm'` 静态检索，有 1 个 L1、13 个 L2、38 个 L3 文件命中；不是每个命中都会改变输出，但 BatchNorm running statistics 和有效 Dropout 的 reference semantics 确实会被 `.eval()` 改写。比较这类题时 reference 和 candidate 必须用同一个 Module state，并在结果里写明 -->


此外，部分工作混淆了grad context 和`train/eval`模式，对比如下：

| 对比项                                           | `model.eval()`    | `torch.no_grad()` | `torch.inference_mode()` |
| --------------------------------------------- | ----------------- | ----------------- | ------------------------ |
| **作用对象**                                      | `nn.Module` 的运行模式 | Autograd          | Autograd + 更进一步的推理优化     |
| 设置 `model.training=False`                     | ✅                 | ❌                 | ❌                        |
| 关闭 Dropout                                    | ✅                 | ❌                 | ❌                        |
| BatchNorm 使用 running mean/var                 | ✅                 | ❌                 | ❌                        |
| 禁止构建 backward graph                           | ❌                 | ✅                 | ✅                        |
| 输出通常 `requires_grad=False`                    | ❌                 | ✅                 | ✅                        |
| 减少 activation / Autograd 显存                   | ❌                 | ✅                 | ✅                        |
| 减少 Autograd bookkeeping                       | ❌                 | 部分                | ✅ 更多                     |
| version counter / view tracking 等 Autograd 开销 | 保留                | 保留                | **进一步关闭**                |
| block 内产生的 tensor 之后继续参与 grad 计算              | ✅                 | **✅**             | **通常不允许**                |
| 灵活性                                           | 最高                | 高                 | 最低                       |
| 推理性能                                          | 不直接提速 Autograd    | 较快                | **通常最快**                 |
| 典型用途                                          | 切换模型到验证/测试行为      | 临时不算梯度            | **纯 inference**          |
| 是否可替代另外两个                                     | ❌                 | ❌                 | ❌                        |



# 数据集版本不一致

<!-- 官方仓库截至调查日没有 GitHub Release，也没有正式 Git tag；README 说的 v0 与 v0.1 都是远端 branch。想复现就写完整 commit，只写 “KernelBench” 或 “latest” 没有信息量 -->

<!-- 2026-08-19 复核：git ls-remote --tags 无输出；heads/v0、heads/v0.1、heads/main 分别解析为 6500bbc8、129232ea、423217d9 -->

| 名称 | commit | 关键含义 |
|---|---|---|
| v0  | [`6500bbc8cf102520d7a8f09be34ee6d5db1c29b0`](https://github.com/ScalingIntelligence/KernelBench/tree/6500bbc8cf102520d7a8f09be34ee6d5db1c29b0) | 2025-07-04 的当前 archival head；它晚于原论文，不能等同 paper-era SHA |
| v0.1 | [`129232ea3c4cfa05c2dd5473bdb935d6246dc84b`](https://github.com/ScalingIntelligence/KernelBench/tree/129232ea3c4cfa05c2dd5473bdb935d6246dc84b) | 2025-07-21 snapshot；题数不变，但大量 shape 与语义已经变化 |
| floating `main` |  [`423217d9fda91e0c2d67e4a43bf62f96f6d104f1`](https://github.com/ScalingIntelligence/KernelBench/commit/423217d9fda91e0c2d67e4a43bf62f96f6d104f1) | 当前远端 HEAD；package 标为 `0.2.0.dev0`，仍是持续集成 evaluator、backend 与基础设施的开发分支 |

<!-- 原始 [KernelBench 论文](https://arxiv.org/abs/2502.10517)正式定义和报告的是 L1–L3 共 250 题，不含 L4；有人从仓库读出 270 题并不意味着原论文也是 270-task evaluation。任务层级与定义见 [论文 §3.2](https://arxiv.org/html/2502.10517v1#S3.SS2) -->

## v0.1 的更新

<!-- v0.1 对应 branch HEAD [`129232e`](https://github.com/ScalingIntelligence/KernelBench/tree/129232ea3c4cfa05c2dd5473bdb935d6246dc84b)，随后通过 [`88df56b`](https://github.com/ScalingIntelligence/KernelBench/commit/88df56b931a0ecafc3ac3b321275addaa078407c) 合入 `main`。题目数量仍为 L1/L2/L3/L4=100/100/50/20，但按 v0 与 v0.1 两个 branch HEAD 做 diff，L1 有102个路径变化、L2有101个、L3全部50个，L4没有变化，完整范围见 [v0...v0.1 diff](https://github.com/ScalingIntelligence/KernelBench/compare/v0...v0.1) -->

| 更新面 | v0.1 的具体变化 | 对评测的影响 |
|---|---|---|
| L1/L2 运行规模 | 同时放大过小 workload、缩小超过15 ms的 workload，目标是在 H100 上把 Eager latency 放进约1–15 ms区间 | kernel launch overhead 占比下降，tiling、memory bandwidth、SM occupancy 和 fusion 的收益更容易显现；v0 speedup 不能直接平移到 v0.1 |
| L1 task 质量 | 修正 loss 的多余 broadcast dimension、sum/mean 混用、scalar multiplication 把输出压到接近0等问题，并替换两个缺少有效数值信号的 reduction/loss task | correctness 不再容易被零输出、常数输出或宽松 tolerance 混过 |
| L2 task 质量 | 调整会互相抵消的 composite pipeline，例如 global mean 后接 softmax、deconvolution 后接抵消上采样的 pooling、重复 normalization；同时恢复缺失的 linear layer、修正 bias broadcast、消除 silent CPU tensor | reference 的算子序列和输出分布已经变化，同题号不再代表同一个优化目标 |
| L3 state 语义 | RNN/LSTM/GRU 把 hidden state 和 cell state 改成显式输入输出，移除 `self.h0`、`self.c0` 等跨调用内部 buffer | 避免前一次调用状态泄漏到下一次 correctness/timing trial，也改变 candidate 的函数签名与内存流量 |
| 随机输入 | 新增 `torch.rand_mix`，可从 normal、uniform、laplace、exponential、lognormal 等分布生成输入；官方 release note 明确说默认 task 仍使用 `torch.randn`，需要使用者主动切换 | “使用 v0.1”不代表自动启用了多分布测试，论文仍需报告实际 input distribution |
| reward-hack 审计 | release note 总结 high-level operator fallback、生成CUDA但不调用、复用 reference memory、跨 CUDA stream 逃逸 timing 等模式，并给出按 compute/memory roofline 检查物理上限的办法 | 这些内容主要是人工审计 guideline，不是 v0.1 evaluator 自动执行的完整防护 |
<!-- | evaluator | `src/eval.py`、`eval_from_generations.py` 和 `run_and_check.py` 在两个 branch HEAD 之间没有变化；代码层新增主要是 `rand_mix` utility | v0.1 的分数变化主要来自 task/shape/输入语义，不能把后续 floating `main` 的新 timer、checker 和 precision support 倒推到 v0.1 | -->

<!-- 官方 [v0.1 release note](https://scalingintelligence.stanford.edu/blogs/kernelbenchv01/) 的核心目标是提高 task signal 和结果审计能力，不是建立一套全新的 scorer。v0 与 v0.1 虽然都能写成“250 tasks, FP32, KernelBench”，launch overhead、可用 tiling、显存压力、reference 语义和 state 行为都不同，只按题号 join 两版 per-task speedup 没有统计意义 -->

<!-- 因此 v0 与 v0.1 虽然都能写成“250 tasks, FP32, KernelBench”，launch overhead 占比、可用 tiling、显存压力、算子是否值得 fuse，以及 cache exploit 的收益都不同。只按题号 join 两个版本的 per-task speedup 也没有统计意义 -->

## floating `main` 当前演进到哪里

<!-- 当前远端 `main` 仍停在 [`423217d`](https://github.com/ScalingIntelligence/KernelBench/commit/423217d9fda91e0c2d67e4a43bf62f96f6d104f1)，该 commit 的日期是 2026-03-05。仓库既没有 Git tag，也没有 GitHub Release；[`pyproject.toml`](https://github.com/ScalingIntelligence/KernelBench/blob/423217d9fda91e0c2d67e4a43bf62f96f6d104f1/pyproject.toml) 把 package 标成 `0.2.0.dev0`，而 [`KernelBench/changelog/v0.2`](https://github.com/ScalingIntelligence/KernelBench/blob/423217d9fda91e0c2d67e4a43bf62f96f6d104f1/KernelBench/changelog/v0.2) 写的是 ongoing effort。这里的 `v0.2` 是开发版本号和 changelog 目录名，不是一个 immutable benchmark release -->

| 演进面 | v0.1 到当前 `main` 的变化 | 对结果比较的影响 |
|---|---|---|
| task 内容 | 题数仍是 L1/L2/L3/L4=100/100/50/20，但改了3个 L1 reference：#12 把 `diag(A) @ B` 换成等价的 elementwise multiply，#92 修正 exclusive cumsum，#97 移除输入里写死的 CUDA/FP16，改由 harness 决定 device/dtype | #12 直接改变 Eager baseline，#92 改 reference 语义，#97 改默认 precision 路径；当前 `main` 与 v0.1 已不是同一组逐题 baseline |
| precision 与 backend | 在 CUDA 之外加入 Triton、CuTe、TileLang、ThunderKittens 和 HIP；precision 从 FP32 扩到 FP16/BF16 | “KernelBench score” 不再隐含 CUDA+FP32；DSL、compiler 和 tolerance 都必须成为结果字段 |
| timing 与 reference | 新增 CUDA Event、Triton `do_bench`、host E2E、Nsight 等 timing method，并把 reference timing 拆成独立 helper | timer、cache policy、reference 是现场重测还是读 JSON，都能改变 speedup；“使用 main evaluator”仍不足以确定计时协议 |
| checker 与 profiling | 加入 experimental regex static checker、adversarial eval tests、过大 speedup flag 和 NCU Python profiling；adversarial cases 覆盖缓存 reference 结果、修改输入、non-default stream 等 | checker 多数仍是 experimental，flag 也不等于 fail；论文需要写清具体打开了哪些检查 |
| generation 与运行基础设施 | 加入 pass@k、LiteLLM、TOML prompt、dataset object、Modal pipeline、package 化与 `uv` | `main` 已从论文时期的脚本集合变成 generation/eval toolkit；prompt、provider、dataset source 和 local/Modal 路径也要 pin |
<!-- | software 与预存 baseline | package 依赖要求 PyTorch `>=2.9.0`，但仓库预存 baseline 仍声明来自 PyTorch `2.5.0+cu124` / CUDA 12.4，并计划更新 | 在当前环境运行 candidate、同时读取旧 baseline JSON 会混用软件栈；应在同一环境现场重测 reference | -->

<!-- 所以 floating `main` 不能简称“新版 KernelBench”后直接与 v0/v0.1 比分。最低限度要同时 pin commit、task files、precision/backend、timer、checker flags、reference 来源和实际软件栈 -->

## 各工作到底用了哪个版本

<!-- 版本披露的实际状况比多数汇总表给人的印象更差。下表只做 attribution，不把“时间上对得上”当证据 -->

| attribution | 工作 | 能确认到什么 |
|---|---|---|
| original / v0 lineage | 原论文、Autocomp、Sakana 后续审计 | 原论文声明 original release 但没有 source SHA；Autocomp 明写 v0；robust-kbench 明确 pin [`7dd9cfa`](https://github.com/ScalingIntelligence/KernelBench/tree/7dd9cfa0c02e8340c9bc9a919d3f19777a8f2eb2) 做旧 KB L1/L2 审计 |
| v0.1-like，但未完整 pin | KernelSkill | vendored L1 #1 的 shape/input 与 v0.1 一致，只能证明抽查文件，不能证明整个 tree 无额外 patch |
| 自定义 fork / task set | METR、robust-kbench、Kevin、KernelBench-RLVR、Makora、EvoEngineer、StitchCUDA、KernelFoundry、Xe-Forge、KernelBench-Verified | 均已过滤、新增、划分或修改 task，即使能追溯某个原始版本也不再是原版结果 |
| 论文称 “v0.2” | [DSL-Monkeys](https://openreview.net/forum?id=WJiJs9k2Lo) | 官方远端没有 v0.2 branch/tag；论文描述与后续 package/main 的 resized task 相似，但未给 SHA，不能确认是同一 task/evaluator，也不能替作者改写版本名 |
| unknown | CUDA-L1、AutoTriton、ConCuR、CudaForge、QiMeng、Dr. Kernel、DICE、CUDA Agent、KernelBlaster、DRTriton、Kernel-Smith、daVinci、MusaCoder、KernelPro、EGG 等 | 只写 KernelBench、level 或 250 tasks，均不足以判定 v0/v0.1/main |

<!-- 尤其不能拿今天的 Hugging Face dataset 或复现时 clone 到的 `main` 倒推历史实验。调查日的官方 README 只说 [Hugging Face dataset 已更新到 v0.1，而 `main` 持续更新](https://github.com/ScalingIntelligence/KernelBench/blob/423217d9fda91e0c2d67e4a43bf62f96f6d104f1/README.md#versions) -->

<!-- ## evaluator 也在变

| 项目 | v0 / v0.1 常用 batch CLI | 当前 `main` 默认或能力 | 影响在哪 |
|---|---|---|---|
| correctness trials | 5 | 5 | 样本数表面相同，但输入和 checker 已扩展 |
| performance trials | 100 | 100 | core API 默认值与 CLI 默认值不同，论文应写实际参数 |
| timeout | batch 180 s；单题 300 s | batch 180 s；单题 300 s | timeout 会截断复杂 compilation/search |
| dtype | FP32 | 默认 FP32，另支持 FP16/BF16 | mixed precision 结果不能混入 FP32 表 |
| tolerance | FP32 `atol=rtol=1e-2` | FP32 `1e-4`；FP16/BF16 `1e-2` | 同一 kernel 可能在旧版 pass、新版 fail |
| default timer | CUDA Event | CUDA Event，也可选 `do_bench`、`do_bench_impl`、host time、Nsight | timer 名必须进入结果身份 |
| warmup / cache | 3 warmup；trial loop 不清 L2 | 3 warmup；每 trial sync + clear L2，另 discard first sample | cache-sensitive kernel 的排名会改变 |
| backend | CUDA extension | CUDA、Triton、CuTe、TileLang、TK、HIP 等 | DSL 与编译链已不再固定 |
| anti-hack | 基础 output comparison | 可重测 reference、标记 >10×、experimental static checker | flag 不等于 fail，开关状态也须报告 |

旧版容差可在 [v0 comparison code](https://github.com/ScalingIntelligence/KernelBench/blob/6500bbc8cf102520d7a8f09be34ee6d5db1c29b0/src/eval.py#L611-L648)核对，调查日 precision/tolerance 在 [`eval.py`](https://github.com/ScalingIntelligence/KernelBench/blob/423217d9fda91e0c2d67e4a43bf62f96f6d104f1/src/kernelbench/eval.py#L70-L102)，新 timing 和 cache 行为在 [`timing.py`](https://github.com/ScalingIntelligence/KernelBench/blob/423217d9fda91e0c2d67e4a43bf62f96f6d104f1/src/kernelbench/timing.py#L82-L209)。v0 的 batch CLI 虽然通常传入 5 correctness / 100 performance，核心函数本身曾默认 1 / 10，因此“用了官方函数”仍不足以确定 trials，见 [v0 CLI](https://github.com/ScalingIntelligence/KernelBench/blob/6500bbc8cf102520d7a8f09be34ee6d5db1c29b0/scripts/eval_from_generations.py#L21-L83) -->

<!-- # 原论文的可复现基准点

[原论文附录](https://arxiv.org/html/2502.10517v1#A1)给出的硬件是独占 bare-metal NVIDIA L40S 48 GB、300 W，软件是 Python 3.10、PyTorch 2.5.0+cu124、CUDA 12.4。正确性用 5 组随机输入；性能做 3 次 warmup，再用 CUDA Event 测 100 次 `model.forward`，以 mean latency 计算 reference/candidate speedup

论文将 Torch Eager 与 `torch.compile` default / Inductor 分列为两种 baseline，compile time 不计入 runtime；本文称“Eager baseline”时专指对应 Eager 列。`fast_p` 的官方含义是全体题目中同时 correct 且 speedup 大于阈值 `p` 的占比，主指标为 `fast_1`，定义见 [论文 §3.3](https://arxiv.org/html/2502.10517v1#S3.SS3)。这不同于“只在正确题上取 geomean”、把慢 candidate clamp 到 1，或将失败题从分母剔除

原始 one-shot setting 用一条 add 示例作 in-context example，greedy decoding。原论文还报告了最多 `k=100` 的重复独立采样 `fast_p@k`，以及最多 10 turn、反馈 generation/execution/profiler 信息并取 best-so-far 的 iterative refinement；本文把三类结果都纳入，但按不同 generation/search budget 标注 -->

<!-- # 公开工作的 task、generation 与 search protocol

下表把单次生成、重复采样和多轮搜索放在一起。预算不作为纳入门槛，算在结果身份里；同一行里版本标了 unknown，也不要按发布日期猜成 v0 或 v0.1

| 工作 | KernelBench revision 与范围 | 训练/测试关系 | generation/search budget 与选择 | 后端 |
|---|---|---|---|---|
| [KernelBench 原论文](https://arxiv.org/abs/2502.10517) | original release L1–L3 250；paper-era SHA 未披露 | evaluation-only | one-shot greedy；另报独立采样 `fast_p@k` 至 k=100，以及最多 10-turn feedback、best-so-far | CUDA extension |
| [NVIDIA DeepSeek-R1 workflow](https://developer.nvidia.com/blog/automating-gpu-kernel-generation-with-deepseek-r1-and-inference-time-scaling/) | L1/L2，版本未披露 | 无参数训练 | 最长约 15 min 的 iterative workflow；只报 correctness | CUDA |
| [Sakana AI CUDA Engineer 初版](https://huggingface.co/datasets/SakanaAI/AI-CUDA-Engineer-Archive) | 原始 250，v0-era；稳定材料未完整 pin evaluator | per-task evolutionary search | 预算与最终 selection 的稳定一手记录不完整 | CUDA |
| [CUDA-L1 v8](https://arxiv.org/html/2507.14111v8) | L1–L3 250；KB SHA 未披露 | SFT collection 明确使用全部 250 题，主表是 in-domain re-evaluation | 最终 candidate 数与 selection 未充分披露；每题长时间复测 | CUDA |
| [AutoTriton](https://arxiv.org/abs/2507.05687) | L1–L3 250，采用上游 PR #35 Triton backend；revision 未 pin | SFT/RL 来源与 KB task-ID 去重未披露 | 同时报 single generation 与 pass@10 | Triton |
| [ConCuR / KernelCoder](https://arxiv.org/abs/2510.07356) | L1/L2 200；revision 未披露 | 训练数据来自 KernelBook，未做 task-semantics contamination audit | 同时报 pass@1 与 pass@10 | CUDA |
| [CudaForge](https://arxiv.org/html/2511.01884) | L1–L3 250；revision 未披露 | 无参数训练，per-task agent search | OpenAI o3 Coder/Judge，最多 10 round，取 best correct；部分 ablation 仅 25 题 | CUDA |
| [QiMeng-Kernel / MTMC](https://arxiv.org/html/2511.20100) | L1–L3 250；revision 未披露 | 作者称 60k offline trajectories 不含 benchmark instance，未独立审计 | Macro-Thinking + Micro-Coding；精确 per-task candidate budget 未完整披露 | CUDA/Triton |
| [Dr. Kernel](https://arxiv.org/abs/2602.05885) | 论文称 L1–L3；revision 未披露 | 训练与最终 eval 的 task-ID 去重及预训练污染审计未充分披露 | 8 samples ×最多 3 turn；报 end-turn 与 best-history，扩展图再做 sequential scaling | Triton |
| [Makora GPT-5-RL / MakoraGenerate](https://arxiv.org/abs/2602.11000) | custom expanded KB 264=L1 100+L2 100+L3 50+L5 14；task/input 有删改，无 manifest/SHA | 主训练集作者声称已对 KB 去重；另有 100 KB 题训练、166 题验证的 oracle ablation | base/RL 报 one-attempt 与最多 3 refinement；MakoraGenerate 的 evolutionary budget/selection 未固定披露 | Triton |
| [DICE](https://arxiv.org/html/2602.11715#S5) | 声称 L1–L3 250；revision 未披露 | CuKe/cudaLLM-derived 数据，未报告最终 KB task-semantics 去重 | one-shot；top-p1、top-k1、temperature1、最多 4096 tokens | CUDA extension |
| [CUDA Agent](https://arxiv.org/abs/2602.24286) | L1–L3 250；revision 未披露 | agentic RL，训练/评测 task provenance 未完整披露 | 最多 200 turns，从完整 trajectory 取相对 `torch.compile` 最快候选 | CUDA extension |
| [KernelSkill](https://arxiv.org/abs/2603.10085) | 声称全 250；vendored 个别 task 与 v0.1 一致，未 pin 全量 SHA | memory/skill agent，benchmark exposure 未完整审计 | 3 initial candidates，最多 15 refinement，取 best | CUDA |
| [PyTorch KernelAgent hardware-guided](https://pytorch.org/blog/kernelagent-hardware-guided-gpu-kernel-optimization-via-multi-agent-orchestration/) | correctness predecessor 声称全 250；performance 只覆盖 L1 100 | agentic test-time optimization | 全局 budget 未统一披露；case study 为 4 workers×8 rounds，取 best | Triton |
| [DRTriton](https://arxiv.org/abs/2603.21465) | 声称全 250；revision 未披露；先用 `torch.export` 降为 functional IR | synthetic-data RL；KB test overlap 未完整披露 | 枚举长度至多 5 的连续 IR 子段并测所有合格 hybrid，预算随 graph 变化 | Triton + PyTorch hybrid |
| [Kernel-Smith](https://arxiv.org/abs/2603.28342) | 报 L1/L2/L3 数字，exact task list/revision 未披露 | evolutionary RL/model，数据关系未充分披露 | 每题 40 evolution rounds，temperature .6、top-p .95 | Triton/CUDA 等 |
| [HTAM](https://arxiv.org/abs/2605.29734) | 全 250，明确 pin `423217d` / package `0.2.0.dev0` | per-task search，无参数训练 | DeepSeek-R1，6 evolution steps，每步 local selection + codegen，通常 12 calls/task，budget 内取 best | CUDA |
| [MusaCoder](https://arxiv.org/abs/2606.04847) | 表格隐含 L1/L2/L3=100/100/50；revision/IDs 未披露，使用 MooreEval | 数据合成参考 KB task patterns/failure cases，无 task-level overlap audit | 每题 8 independent samples，temperature .7；报 Pass@8 与 Avg@8 | CUDA/MUSA |
| [KernelPro](https://arxiv.org/abs/2606.26453) | 声称 L1–L3 250；revision 未披露 | per-task MCTS search | 15 seeds=5 temperatures×3 runs；每 seed 30 iterations×2 candidates，每题约 900 attempts 后取 best | CUDA、CuTe/CUTLASS |
| [EGG](https://arxiv.org/html/2606.26758#S4) | 声称 L1–L3 250；revision 未披露 | per-task agent search | 多 seed、multi-stage optimization；约 20 min/50k output tokens 每题，exact attempts 未披露 | Triton |
| [daVinci-kernel](https://arxiv.org/abs/2606.16497) | 报 L1/L2/L3 Fast1，无 task IDs/SHA；shell 默认 `drkernel-validation-data` | 使用 DrKernel cold-start/RL data，未给最终 eval task-ID 去重证明 | headline 为 N=8、最多 3 turn；另有 5-turn×10-iteration TTS，论文表未逐项锁定 artifact | CUDA/Triton | -->

<!-- ## 改版、过滤集与 subset

这些工作的题集、接口或 verifier 都动过，也放进表里；和原始 250 题差在哪看最后一列。单列是因为题集变了，与 generation budget 无关

| 工作 | 实际题集 | 主要协议变化 | 为什么不能并入原榜 |
|---|---|---|---|
| [METR KernelAgent](https://evals.alignment.org/blog/2025-02-14-measuring-automated-kernel-engineering/) | METR 博文称从 L1–L3 250 题删 45 再加 L5 14，按文字为 219；[官方 v0.1 博文](https://scalingintelligence.stanford.edu/blogs/kernelbenchv01/)回顾称删 43，按文字为 221；固定 METR repo 实际为 L1 91 + L2 80 + L3 37 + L5 14 =222 | H100、约 300 attempts/task、Triton `do_bench` median、best-of-k，并把慢结果替换为 reference | 219/221/222 三种一手记录互相冲突，题集 manifest 与 search aggregate 都不能当原版 KB |
| [Sakana robust-kbench](https://arxiv.org/abs/2509.14279) | 先审计 v0 commit [`7dd9cfa`](https://github.com/ScalingIntelligence/KernelBench/tree/7dd9cfa0c02e8340c9bc9a919d3f19777a8f2eb2) 的 L1/L2，再构造新的 robust task families | forward checker 使用多 shape/init/seed、`1e-5`，在 H100 长测 | headline robust suite 已不是 L1–L3；1.49×是污染过滤后的旧 KB 分析 |
| [KernelLLM](https://huggingface.co/facebook/KernelLLM/blob/main/README.md) | 自建 KernelBench-Triton variant，仅 L1 | H100，temperature1、top-p .97，报告 Pass@1/10/20 | DSL、task implementation 与 sample budget 都不同 |
| [Kevin](https://arxiv.org/abs/2507.11948) | 训练用 80 个 L1+全部 100 个 L2；测试为余下 20 个 L1+80 个作者新题 | 16 parallel trajectories ×8 serial turns，报 best@16/avg@16 | 测试 100 题不是原 KB 全集，也没有 held-out L2 |
| [KernelBench-RLVR-120B](https://arxiv.org/html/2602.07670#S4) | L1 随机 80 train/20 held-out，主 selection 实验再用两个 5 题 subset | BoN K=64 或 TTT K=32×1–5 update；fast proxy 仅 5 performance trials | 18/20 是 K64 下 task success，不是标准 L1 fast_1 |
| [Autocomp update](https://charleshong3.github.io/blog/autocomp_update.html) | v0 L2 前 30+L3 前 10 | L40S，小规模 agent experiment | 只有 40 题，且 L2 #13 no-op 可造成 256×假象 |
| [STARK](https://arxiv.org/abs/2510.16996) | L1–L3 representative subset，未公开完整 IDs | A100 40 GB、budget30、100 warmup、CUDA Events | subset 身份不足，不能当 250-task 数字 |
| [TritonRL](https://arxiv.org/abs/2510.17891) | KernelBench Triton backend，主报 L1 100 | L40S、k=10、robust verifier、Pass@10 | 后端、oracle 与 metric 均变 |
| [GEPA](https://openreview.net/pdf/e2625111ac0503603448f773ff09555ec880422f.pdf) | 35 个跨 L1–L3 representative subset | shared-prompt evolution，单 rollout 最多 5 次 refinement | 测 prompt evolution + refinement，不是原 evaluator |
| [PRAGMA](https://arxiv.org/html/2511.06345) | 声称 across KernelBench，task count/revision 未披露 | A100、最多 15 attempts、Triton `do_bench` | task universe 不完整，同时另做 CPU 路径 |
| [cuPilot](https://arxiv.org/html/2512.16465) | L1 100 | A100 锁频 1410 MHz、10 warmup+50 timed mean、Eager | 只覆盖 L1 |
| [KernelBlaster](https://arxiv.org/html/2602.14293) | L1/L2+未披露题数的 L3 subset | 10 iterations×10 rollout steps；NCU cycles；baseline 取 Eager/compile 较快者 | task、timer 和 reference 都变，论文内结果表述还有冲突 |
| [EvoEngineer](https://arxiv.org/html/2510.03760#S5) | 从 KB 派生 91 个 operation，每题已有 initial C++/CUDA | RTX4090；每题 45 optimization trials、3 次独立 run | 测已有 CUDA 的 evolution；失败 speedup 记 1 |
| [DSL-Monkeys](https://openreview.net/forum?id=WJiJs9k2Lo) | L1/L2 200；论文称 v0.2，但官方无对应 branch/tag | H100、5 checks、`do_bench`；跨题积累自生成 ICL examples | 无 SHA，无法确认所谓 v0.2 的 task identity |
| [StitchCUDA](https://arxiv.org/html/2603.02637v1#S4) | L1 20+L2 20+L3 10，且手改 9 题 | 80% KB 用于训练；H200/RTX PRO6000；15 iterations 取 best；E2E timing | 50-task held-out split 且 task 已修改 |
| [KernelFoundry](https://arxiv.org/html/2603.12440#S4) | 过滤为 111 题 L1 80/L2 31；主实验常取 20+20 | oracle 改为 99% elements relative error<1%+cosine；40 generations×8 | 无 L3，oracle 与预算重写 |
| [KernelBench-Verified](https://arxiv.org/abs/2607.16241) | L1 100+L2 97+L3 50；删除 L2 23/80/83，upstream SHA 未 pin | H200、TF32 baseline、4 种 hidden distributions、memory metric、best-of-5 | 分母、prompt、oracle、tolerance 与 baseline 都变 |
| [KernelBench.com v3](https://kernelbench.com/blog/v3) | 重建 60 题，另有 hard/mega 子集 | RTX 3090/H100/B200、multi-seed、adaptive baseline 与人工 audit | 同名社区 leaderboard，不是 ScalingIntelligence 的 250-task suite |
| [CuTeGen](https://arxiv.org/abs/2604.01489) | 论文声称 L1/L2 all209，但官方两层只有 200，额外 9 题来源未披露 | RTX4090、GPT-5、CuTe，人工排除 precision downgrade/library fallback | task universe 自相矛盾 |
| [Xe-Forge](https://arxiv.org/html/2605.26118#S6) | 实际评 97 个 L1/L2 task，输入为 KernelFalcon 已生成 Triton kernel | Intel Arc Pro B70；最多 9 stage×5 轮；200 warmup+100 Event trials | PyTorch→kernel 变成 Triton→XPU optimization |
| [KForge](https://arxiv.org/abs/2606.02963) | L2 中 37 个 GEMM+tail-op task | Intel Arc B580，每题 5 iterations 取 best；逐题取 max(Eager,compile) baseline | subset、异构硬件、best baseline 与 search 同时变化 |
| [SpecGen](https://arxiv.org/abs/2606.17518) | 10 个 L1+10 个 L2/L3 task | H200，每题 100 iterations，比较 agent E2E search efficiency | 20-task scheduling study，不是全量 KB |
| Musa KernelBench | 从 CUDA KB port 到 MUSA，保留层次但替换 device/backend/API | 只报 Eager Faster Rate，未评 `torch.compile`；同样是 8 samples | semantic/backend port，不能与 CUDA KB 混排 | -->

<!-- # evaluator、hardware、budget 与 metric 对照

所有 generation budget 放在同一张表里，`unknown` 虽然难看，但比一个猜出来的默认值诚实。数值用来看协议差在哪里，拼不出统一 leaderboard

| 工作 / setting | 硬件 / 软件 | correctness | timing / reference | budget / selection | aggregate / 已报结果 |
|---|---|---|---|---|---|
| KernelBench original | L40S 48 GB 300 W；PyTorch2.5+cu124、CUDA12.4 | FP32、5 random、`atol=rtol=1e-2` | 3 warmup+100 CUDA Event mean；Eager 与 compile default 分列 | one-shot；`fast_p@k`至 k=100；10-turn best-so-far | `fast_p`以全题为分母；不同 budget 分表 |
| METR KernelAgent | H100；TF32 matmul medium/cudnn TF32 | seed13、3 trials、`1e-2`、anti-cheat | `do_bench` warmup50 ms/rep400 ms median；慢 candidate 回退 reference | 通常约 300 attempts/task，o1 约 40，per-task best | floor1 后 GM 1.81×；all-model search 2.01×，非原榜指标 |
| Sakana initial | H100 被项目材料提及，完整 stack 未稳定披露 | 后来证实可被 cache/no-op/memory exploit 欺骗 | 原始计时细节不足；archive 同时出现 native/compile | per-task evolutionary search，预算记录不完整 | 初报 3.13×已被官方复盘判为不稳健 |
| robust-kbench forward | H100；CUDA12.4、cuDNN8.9.7、PyTorch2.5.1 | 多 shape/init/seed、`1e-5` | 25 warmup+2000 mean；native PyTorch | translation 最多 10 step；evolution 约 80 proposals/task、选 best | 污染过滤后的旧 KB L1/L2 约 1.49×；不是原 250 统一分数 |
| CUDA-L1 v8 | A100，另做 H100/L40/3090/H20 迁移 | paper 称 1000 random；current code `.eval()+no_grad()`、`1e-2` | final 每题 20 min、多 bucket/paired order；Eager/compile/RO/Graph 分列 | 最终 candidate 数与 selection 未充分披露 | Eager arithmetic mean3.12×、median1.42×；v1 的 17.7×已被修订 |
| AutoTriton | KB eval GPU/software unknown；训练 GPU A800 | upstream Triton backend；seeds/tol/mode unknown | timer/cache unknown；Eager | single generation 与 pass@10 | single L1/L2/L3 Exec36/45/20%、Fast1 10/17/10%；另报 pass@10 |
| ConCuR / KernelCoder | 8×RTX5090 eval | random input、相同 shape/value；trials/tol/mode unknown | timer/trials unknown；Eager | pass@1 与 pass@10 | L1/L2 pass@1 Exec58/59、Fast1 17/39；pass@10 Exec91/95、Fast1 32/68 |
| CudaForge | Quadro RTX6000 | predefined tests、tolerance `1e-4` | timing 细节未充分披露；Eager | o3 Coder/Judge，最多 10 rounds，取 best correct | N=10 主结果 1.68×；scale-up 2.27×；25 题 ablation 另算 |
| QiMeng MTMC | H100/A100/V100 | accuracy checker 细节未充分披露 | timing 未充分披露；Eager | 多模型 Macro-Thinking+Micro-Coding，budget 未 pin | H100 Gemini2.5Pro L1/L2/L3 correct=100/99/70%，Fast1=67/86/34%；mean 为全题算术均值 |
| Dr. Kernel | H100 | 5 random、`1e-2`、hacking candidate 记 incorrect | CUDA Event mean；eval script perf10；Eager 主表 | 8 samples×3 turns，报 end-turn 与 best-history | Fast@1.2 end=18.8/31.6/3.0%，best-history=25.1/47.8/7.3% |
| Makora | H100 backend；software/clock unknown | sup-norm abs error `<1e-3`；trials/seeds/dtype/mode unknown | 3 warmup+100 timed median，但同文另一处写 average；TorchInductor | one-attempt、最多 3 refinement；MakoraGenerate budget unknown | RL one-attempt 77.0% correct/21.8% faster/GM.81×；Generate 97.4/72.9/2.12×，custom264 |
| DICE | 训练 8×A100；KB eval hardware unknown | CUDA-defined/invoked static check；seeds/tol/mode unknown | timer/cache unknown；Eager | one-shot | DICE-8B L1/L2/L3 Exec47/35/14%、Fast1 15/19/6%、Fast2 6/1/6% |
| CUDA Agent | 128×H20 sandbox pool | paper 仅称多次校验；workspace `.eval()+no_grad()`、5 inputs、`1e-2` | paper 称 sync/warmup/repeated mean；Eager 与 compile | 最多 200 turns，trajectory 内取相对 compile 最快 | Pass98.8%；Faster eager/compile98.4/96.8%；correct-only GM2.60/2.11× |
| KernelBlaster | L40S/H100 主表，另有 A6000/A100 | 多 random seeds+LLM soft verification；次数/tol unknown | NCU kernel elapsed cycles；baseline=`max(Eager,compile)` | 10 iterations×10 rollout steps | H100 GM L1/L2/L3sub=1.497/2.592/1.110；非 wall-clock KB score |
| StitchCUDA v1 | H200、RTX PRO6000 | 多 seed/value/shape，tol unknown，人工 anti-hack | 包含搬运/launch/sync 的 E2E；Eager，L3 另比 compile | 15 iterations 取 best | H200 L1/L2/L3(20/20/10) avg3.54/1.82/1.50× |
| KernelSkill | A100 80 GB；PyTorch2.6+cu124 | 固定 seed、一次 inputs、`.eval()`、`1e-4` | 25 warmup+100 CUDA Event mean；Eager | 3 initial+最多 15 refinement，取 best | 100% success，L1/L2/L3 5.44/2.82/1.92×；均值口径未充分说明 |
| PyTorch KernelAgent performance | H100 | trusted reference gate；SHA/tol/mode unknown | `do_bench`，>1s warmup、100 mean；compile default、graph off | 多 agent rounds/workers，取 best | L1 100 题 GM1.56×、65/100 更快；相对 earlier generated kernels 为 2.02× |
| KernelFoundry | RTX A6000，另做 Intel GPU | 99% element relative error<1%+cosine | 自适应 warmup/timing 至少 10 且≥1s；Eager | 40 generations×8，另 2 次 parameter iterations | filtered111 Fast1 71%、Fast2 42%、avg2.32×、geo1.38× |
| DRTriton | KB eval device 未明确；训练 8×H100 不能混写 | 5 random cases，tolerance 未量化 | timer/warmup unknown；Eager 与 compile | 变长 IR 子段枚举并测所有 hybrid | L1 Acc69/TE17/TC12；L2 96/92/56；L3 76/54/34% |
| Kernel-Smith | hardware/software unknown | 每 module 100 independent unit tests，含义未充分说明 | warmup、多次 mean、outlier removal、CUDA Graph；Eager | 每题 40 evolution rounds | Fast1 L1/L2/L3=.70/.93/.46；AMSR 把 speedup<1 置 0 后平均 |
| HTAM | 8×A100-SXM4 80 GB；CUDA12.8、PyTorch2.9.1 | 声称使用 pinned official strict FP32，未重述 tol/trials | 随`423217d` evaluator；Eager | 6 evolution steps、约 12 calls/task，budget 内 best | Correct98.4%、Fast1 84.0%、GM1.978× |
| MusaCoder / MooreEval | 64 hosts×8 MTT S5000 80 GB | randomized input+shape/dtype/numeric+anti-fallback；tol/mode unknown | synchronized CUDA Events，counts/stat/cache unknown；Eager/compile | 8 independent samples，T=.7 | Pass@8 93.2%、Avg@8 88.6%；Faster>1.1× Eager/compile=15.0/9.2% |
| KernelPro | A100 AWS p4 | `allclose(atol=1e-2,rtol=1e-3)`；mode/dtype unknown | Eager wall-clock，timer 细节 unknown | 约 900 attempts/task 后取 best | L1/L2/L3 GM2.42/4.69/5.30×，全题 solved |
| EGG | RTX4090；CUDA13、PyTorch2.9.1、Triton3.5.1 | protocol/mode/dtype/tol 未披露 | timer/cache unknown；Eager 主指标，另报 compile | multi-stage search，exact attempts unknown | correct-only mean overall2.13×；L1/L2/L3 Fast1=72/100/94% |
| daVinci-kernel | GPU/driver 未披露 | public runner default-training+`no_grad()`、5 seeds、`1e-2` | CUDA Event、3 warmup、perf10 mean；Eager | N=8×最多 3 turns；另有更长 TTS | 14B L1/L2/L3 Fast1=37.2/70.6/32.2%，task source/denominator 未 pin |
| KernelBench-Verified | H200；software image 记录不一致 | FP32、default-training+`no_grad()`、5 normal+4 hidden distributions、`1e-3` | 3 warmup+10 Event mean；Eager+TF32 | 5 independent samples 取最快正确 | GPT-5.5 GM.88×、Correct98.4%、Fast1 53.0%；original-style protocol 为 1.43× |
| Kevin | eval hardware/software 未披露 | oracle/tol/timer 未充分披露 | Eager | 16 trajectories×8 turns，best@16/avg@16 | QwQ→Kevin Correct56→82%，best@16 speedup .53→1.10×；测试含 80 新题 |
| KernelBench-RLVR | eval GPU 未披露 | public vendor FP32 `1e-4`；paper correctness trials 未报 | fast proxy 5 Event trials；Eager | BoN64 或 TTT32×steps | held-out L1 中 18/20 在 K64 下至少一个 fast sample；不是全 L1 平均 |
| Autocomp | L40S | official-v0-like，细节未完整披露 | Eager 与 compile 分列 | agent experiment，40 题 | L2 前 30/L3 前 10 Eager GM2.61/2.11×；compile1.11/.93×，含 no-op 异常 |
| TritonRL | L40S | robust verifier | timing 细节未充分披露；Eager | k=10、Pass@10 | L1 Correct56%、Fast1 5%、Fast2 1%、mean.33× |
| PRAGMA | A100 40 GB；CUDA13 | correctness 细节未充分披露 | `do_bench` 5 warmup+100 reps；Eager | 最多 15 attempts | GPU success92%、Avg2.30×、Fast1 59.8% |
| cuPilot | A100、频率锁 1410 MHz | correctness 细节未充分披露 | 10 warmup+50 mean；Eager | strategy-coordinated evolution | L1 平均 3.09× |
| EvoEngineer | RTX4090；PyTorch2.4+cu124 | 5 cases；tol/mode unknown | 100 runs mean | 45 trials/kernel，3 independent runs | 91 个派生 op，失败记 1，取三次 run 的 median speedup |
| CuTeGen | RTX4090；CUDA13、PyTorch2.8、CUTLASS4.3 | 人工排除低精度与 library fallback；tol/mode unknown | Eager；timer 细节 unknown | 10-round CudaForge-style search | 声称 209 题平均 1.71×，但 L1+L2 题数与官方不符 |
| Xe-Forge | Intel Arc Pro B70 | `no_grad()`、`rtol=1e-2/atol=1e-5` | 200 warmup+L2 flush、100 Event trials、trimmed mean | 最多 9 stage×5 轮 | 97-task Eager GM1.17×、67% improved；输入已是 Triton kernel |
| KForge | Intel Arc B580 | correctness 细节未充分披露 | baseline 逐题取`max(Eager,compile)` | 37-task、5 iterations 取 best | GM5.13×；subset/硬件/reference 均不同 |
| Musa KernelBench | MTT S5000 | MooreEval，具体 tol/mode unknown | Eager；不报 compile | 8 samples，Pass@8/Avg@8 | 27B-RL Pass@8 92.4%、Avg@8 81.7%、Faster>1.1× 12.5% | -->

<!-- # 哪些差异会改变结论

## 1. task revision 与题目身份

v0.1 把很多原本 microsecond 量级的题放大到毫秒级，launch overhead 的相对权重、fusion、persistent kernel 和 CUDA Graph 的收益都会跟着变。full 250、filtered 111、held-out 50、L1-only 100 和“representative subset”必须分开统计

## 2. benchmark exposure

KernelBench 是固定、公开、只有 250 个主任务的小集合。CUDA-L1 明确用全部 250 题收集 SFT 数据后再在同 250 题汇报；Kevin 用 180 个 KB 题训练，测试只有 20 个原 KB L1 加 80 个新题；StitchCUDA 做 80/20 task split；KernelBench-RLVR-120B 也做 L1 80/20 task-disjoint split，但只评 20 个 held-out L1。它们分别是 in-domain re-evaluation、mixed novel-set evaluation 和不同规模的 task-held-out evaluation，不能统称 generalization

## 3. candidate budget 与 winner's curse

one-shot 的 `fast_1`、pass@k 和 search 后的 Faster Rate 可以放在同一矩阵中，但 search 会在重复测量后选极小 latency，同时放大真实收益与测量噪声。每行至少应报告 per-sample pass@1、last candidate、best-of-fixed-budget、每题实际编译/测量次数和 selection statistic；无法补齐这些字段时，不尝试按预算折算

## 4. correctness oracle

固定 shape + `randn` + 5 seeds + `1e-2` 只能证明候选在一个很窄的分布上近似 reference，不能证明实现了相同算子。hard-code、零输出、缓存上一轮结果、只覆盖常见轴、精度降级和跳过依赖都曾发生；[调查日官方 EVAL 指南](https://github.com/ScalingIntelligence/KernelBench/blob/423217d9fda91e0c2d67e4a43bf62f96f6d104f1/EVAL.md)也明确提醒第三方结果未经官方核验，异常大 speedup 必须独立审查

## 5. baseline 与数学模式

Eager 是否允许 TF32、`torch.compile` 用 default 还是 max-autotune、是否启用 CUDA Graph、compile time 是否入账，每一项都在改 denominator。KernelBlaster 甚至逐题取 Eager/compile 较快者，KernelBench-Verified 则专门启用更现实的 TF32 baseline；这两种做法都不能只写一句“vs PyTorch”

## 6. timing 与 cache

CUDA Event 测 device forward、host timer 测 launch/E2E、NCU cycles 累加 kernel duration、Triton `do_bench` 返回 median，统计对象不同。是否每 trial 清 L2、是否 randomize reference/candidate order、是否对获胜者重测、是否锁频和功率，也会改变小 kernel 与 cache-sensitive solution 的排名

## 7. aggregate 定义

官方 `fast_p` 是全题 success rate；AutoTriton 与 ConCuR 的 pass@1/Fast1 是逐层比例；Makora 的 GM 看起来按各模型正确题聚合，但没写清完整的分母定义；MusaCoder 的 Avg@8 是样本平均，而 Pass@8 是每题 8 次至少成功一次。就算底层样本完全一样，换 aggregate 结论就会变 -->

<!-- # 建议的统一报告协议

让所有工作换同一个 evaluator 不现实，但结果至少要能按 protocol 对齐。建议每篇工作附一张 `KB-Protocol Card`，至少写清这些字段

| 类别 | 必填字段 |
|---|---|
| Dataset | upstream full SHA、task IDs、任何 task patch 的 diff、L1–L4 范围 |
| Workload | dtype、input shape/distribution、Module state、autograd context；本文固定为 forward-only |
| Oracle | correctness seed 数、hidden distributions、`atol/rtol`、anti-hack rules |
| Timing | timer API、测量边界、warmup、trials、discard、L2/cache policy、sync、统计量 |
| Reference | Eager/compile mode、TF32/CUDA Graph、compile flags、是否同一 run 现场重测 |
| Hardware | GPU SKU、显存、power limit/clock、独占性、CPU、driver/CUDA/PyTorch/cuDNN |
| Generation | 每题请求数、每次返回 candidate 数、temperature/top-p、最大 token、response 内 self-selection |
| Budget / Selection | requests、samples、turns、candidate evaluations、feedback、停止条件、last/best/ensemble |
| Metrics | correctness denominator、Fast@p 阈值、失败和慢题如何记、arithmetic/geometric/median |
| Artifacts | prompts、generated kernels、evaluator SHA、per-task raw times、失败原因、audit log |

如果目标是建立一个可横向比较的新主榜，建议采用下面的最小公共协议

- pin v0.1 的完整 SHA 或另发 immutable dataset release，不允许 floating `main` / floating Hugging Face dataset
- 将 `module.training` 与 autograd 分开声明；reference 与 candidate 必须处于相同且显式设定的 context。若复现 legacy evaluator，应逐路径记录 correctness、candidate timing 与 reference timing 的 context，不再笼统写成 `no_grad`；新榜至少再测显式 `.eval()`，stateful task 单独报告
- 用不可见的多 shape、多 distribution、多 seed correctness suite；FP32 收紧到 `1e-4`，对 reduction/低精度任务按误差分析设 task-specific tolerance
- reference 与 candidate 在同进程、同 GPU、随机交错重测；Eager、compile default、compile max-autotune 分开报告，不逐题挑更快者后仍称单一 baseline
- 同时报 CUDA Event device time 与 host-observed E2E time，明确 L2 cache policy；异常 speedup 用独立进程和 profiler 复核
- 在一张 master table 中同时列 one-shot、pass@k、多轮与 evolutionary search，但为每个 setting 分配独立列或 setting ID，不把不同 budget 的数字压成一个排名
- 同时报 requests、samples、turns、总 candidate evaluations、feedback、停止条件与 selection；`avg@k`、`pass@k`、last-turn 和 best-of-budget 使用各自原始名称
- 以全任务为分母报告 Correct、Fast@1、Fast@1.2、Fast@2；另可报 correct-only GM，但不得代替 success-weighted 指标
- 发布 per-task JSON 和所有最终 kernel；超过可证明 roofline，或相对已验证的同一 reference 出现异常大 speedup 的结果进入人工 audit queue -->

<!-- # 纳入、降级与排除边界

为了不把“提到 KernelBench”当成“在 KernelBench 上做过实验”，本文把证据分成三档

- **A：可审计**：论文或固定 commit 明确 task scope，并披露 hardware、baseline、budget、correctness/timing 中的大多数关键字段
- **B：有实测但缺协议**：有 KernelBench 数字，但 revision、task IDs、mode、timer 等至少一个关键字段缺失，只能做定性对照
- **C：生态提及**：subset 身份不明、只有 correctness 宣称、只有 adapter，或结果来自二手汇总，不进入 score table

一些常被放进 related-work 表、但不应纳入原 KernelBench 数字比较的项目如下

| 项目 | 处理 | 理由 |
|---|---|---|
| [KernelFalcon / Meta KernelAgent](https://pytorch.org/blog/kernelfalcon-autonomous-gpu-kernel-generation-via-deep-agents/) | C，仅记生态 | 官方材料称全 250 题 100% correctness，但未给可复核的 KB SHA、性能 hardware/timing；并明确依赖 LLM-generated test harness、没有完整 static cheat check |
| [KernelEvolve](https://arxiv.org/abs/2512.23236) | C，仅记生态 | 声称全 250 pass，但公开材料不足以重建 KernelBench run 的 GPU、timer、turn budget 与 metric |
| [AKG Kernel Agent](https://arxiv.org/abs/2512.23424) | B，异构后端 | Triton GPU + NPU，1.46× Eager 的平台与协议字段不足，不能进 CUDA 主表 |
| [OptiML](https://arxiv.org/abs/2602.12305) | 排除 | 实验是 ParEval 与 CUDA-LLM suite，KernelBench 只在 related work 出现 |
| [CUDAMaster / MSKernelBench](https://arxiv.org/abs/2603.07169) | 排除原榜 | 评测作者新建的 50-task multi-scenario benchmark，虽受 KB 启发但不是 KB |
| [CUDABench](https://arxiv.org/abs/2603.02236) | 排除原榜 | 新的 text-to-CUDA benchmark，只把 KernelBench 放在 benchmark comparison 中 |
| [KernelBench-X](https://arxiv.org/abs/2605.04956) | 排除原榜 | 名称容易误导，但 176 题来自 TritonBench-T 再加 precision/quantization variants，不是 ScalingIntelligence KernelBench 的 revision |
| [FastKernels](https://arxiv.org/abs/2605.23215) | 排除原榜 | 另建 46 个 production architecture task，目的正是研究 benchmark-to-production gap |
| [The Correctness Illusion](https://arxiv.org/abs/2606.20128) | C，oracle 研究 | 用 26 个自建 correct/bug control kernels 验证 fuzzing oracle，不报告任何原 KB task score |
| [KernelGenBench](https://arxiv.org/abs/2607.27231) | 排除原榜 | 210 个 multi-source operators 与 110 个 multi-chip subset 构成的新 benchmark，不是 KernelBench revision |
| [AccelOpt](https://arxiv.org/abs/2511.15915) / [CUDA-L2](https://arxiv.org/abs/2512.02551) | 排除原榜 | 前者实测 Trainium NKIBench，后者实测 1000 个 HGEMM configuration；两者只把 KB 作为背景 |
| AlphaEvolve / K-Search / AutoKernel adapter | 排除结果表 | 只有集成、roadmap 或少量 case study，没有独立公开、可复核的原 KB aggregate | -->

<!-- # 最终判断

现阶段不存在可信的跨论文 KernelBench 总排行榜。所有 setting 都可以放进同一张配置矩阵，但真正能横比的单元格，仍要求 data SHA、task list、GPU/software、Module/autograd semantics、baseline/timer/cache policy、candidate evaluations、feedback 和 selection rule 全部相同

历史结果不要强行归一化，在矩阵里保留 protocol lineage 标签即可：original v0、v0.1、modified/filtered、verified hidden-test，再加上对应的 generation/search budget。任何只写 “KernelBench 250, H100, fast_1” 的新结果，在补齐 protocol card 之前都按不可横比处理 -->

# KernelBench-Verified

<!-- [KernelBench-Verified](https://arxiv.org/html/2607.16241) 是对 KernelBench evaluator 的加固版本，定位是重新校准 PyTorch baseline、扩大 correctness 输入分布，并把 peak GPU memory 纳入结果。它针对两类已经出现的失真：candidate 借 TF32 Tensor Core 获得“免费加速”，而 reference 仍走较慢的 FP32 路径；candidate 针对 `torch.rand()` 的正数、小幅值输入写 shortcut，固定测试仍会判 correct

它沿用 KernelBench 的 PyTorch-to-CUDA 任务形式和固定 shape，没有改成 dynamic-shape benchmark。主要变化集中在 baseline、value distribution、prompt 暴露范围和 metric，因此它是一套新的评测 protocol，不能把结果直接填回原 KernelBench leaderboard -->

## 与原 KernelBench 的关系

| 对比轴 | 原 KernelBench 常见设置 | KernelBench-Verified | 备注 |
|---|---|---|---|
| task denominator | L1/L2/L3=100/100/50 | 100/97/50 | |
| reference | Eager FP32 | Eager FP32+TF32 `high` | |
| input distribution | `torch.randn` | 四组 hidden input 配置 | |
| shape | 固定 | 固定 | |
| prompt | 包含 task 的输入配置 | 大多数沿用；4题改 input-blind |blind input 难以扩展到全部样本 |
| sampling | one-shot、pass@k 或不同论文自定义 | 5 independent samples，fastest correct | |
| timing | 3 warmup+100 trials | 3 warmup+10 trials | 没有说明为何从100 trials改为10 trials |
| tolerance | FP32 `1e-4` | FP32 `1e-3` | |
| memory | 不报告 | peak allocation ratio 与节省比例 | |

<!-- 所以 0.88× 与其他论文的 geomean、`fast_1` 或 Faster Rate 没有统一分母。KernelBench-Verified 适合用来回答“已有 candidate 在更强 baseline 和更宽 value distribution 下还是否成立”，无法直接替代原协议的 model-generation leaderboard -->

<!-- ## 评测协议

| 维度 | KernelBench-Verified setting | 影响 |
|---|---|---|
| generation | 标准 KernelBench single-turn prompt；每题生成5个独立 sample，取最快的正确 sample | headline 是 best-of-5，不能当 pass@1 或单 candidate latency |
| output | 生成 CUDA extension | 与 Triton、CuTe 等 DSL 结果不是同一 compiler/runtime setting |
| hardware | NVIDIA H200 | TF32 throughput、memory bandwidth 和 launch regime 都与原论文的 L40S 不同 |
| dtype / tolerance | 主结果 FP32，`atol=rtol=1e-3`；BF16 appendix 使用 `1e-2` | tolerance 同原始 v0 的 `1e-2`、后续 main 的 `1e-4` 都不同 |
| baseline | PyTorch Eager，并调用 `torch.set_float32_matmul_precision('high')` 启用 TF32；不以 `torch.compile` 为主 baseline | matmul/conv reference 可走 Tensor Core，candidate 只调用同类库实现时不再获得虚高 speedup |
| correctness | standard test 之外增加 D1–D4 四组 value distribution，任一配置失败则整题 incorrect | hidden suite 是 correctness gate；alternative distributions 不参与 timing |
| timing | 原始分布上做3次 warmup、10次 CUDA Event trial，取 mean | 与 KernelBench 常见100-trial protocol 不同 |
| memory | 单次 forward 前 reset peak statistics，之后读取 `torch.cuda.max_memory_allocated()` | 测的是 peak allocation，不是 HBM traffic 或 profiler 里的 bytes moved |

任务范围、best-of-5 和 H200 setting 来自[论文 §2.1](https://arxiv.org/html/2607.16241#S2.SS1)，TF32 配置见[论文 §2.2](https://arxiv.org/html/2607.16241#S2.SS2)，timing 与 memory 的细节见[评测附录](https://arxiv.org/html/2607.16241#A2) -->

## 四组 hidden input

hidden suite 保持 shape 不变，只对 floating-point tensor 的值做变换；integer tensor、scalar parameter 和原本为零的元素保持不动。
这样仍允许合法的 static-shape specialization，同时覆盖正负分支、幅值变化和 near-zero 数值行为

| 配置 | 输入变换 | 主要检查对象 |
|---|---|---|
| D1 | 原始输入，factor=1 | standard correctness |
| D2 | 乘3 | exp/tanh overflow、大幅值 reduction error accumulation |
| D3 | 乘0.01 | log underflow、normalization 中的 small variance、epsilon handling |
| D4 | 乘-1 | ReLU/ELU negative branch、负数 MaxPool、positive-only identity shortcut |

reference 会先运行全部 hidden config；自身产生 NaN、Inf 或异常的配置不会用于 gate。
<!-- 论文共过滤2/1000个配置，即 L1 P90 的 D2 和 L1 P98 的 D4，没有删除这两道题本身，具体规则见[论文 §2.3](https://arxiv.org/html/2607.16241#S2.SS3) -->

<!-- 这套测试仍有边界：它改变 value distribution，不改变 shape。它能抓住“ReLU 输入全为正，所以直接返回 input”一类 shortcut，抓不住只针对固定 dimension、stride 或 layout 的错误实现。作者又对 surviving fast kernels 做 LLM-judge audit，最终把 L1 P90、L1 P100、L3 P35、L3 P47 四题改成 input-blind prompt，隐藏 `get_inputs()`、`get_init_inputs()` 和精确配置，只保留人工补充的 shape annotation，见[论文 Appendix L](https://arxiv.org/html/2607.16241#A12)和[固定版本的 prompt code](https://github.com/facebookresearch/kernel_bench_verified/blob/3fdf6fec7372a4d0cb682635f00e7bdcbc55d50e/src/prompt_constructor.py#L33-L65) -->

## 指标定义

| 指标 | 定义 | 分母 / 聚合 |
|---|---|---|
| Correctness | 每题5个 sample 中至少一个通过全部 correctness gate | 247个 active problems |
| Correct Speedup | 每道正确题取最快正确 sample，计算 `t_baseline / t_kernel` | 只在正确题上取 geometric mean |
| Fast@1 | 最快正确 sample 的 speedup>1 | 247个 active problems |
| Correct Mem Efficiency | 每道 standard-correct 题计算 `m_baseline / m_kernel` | 按 best-of-5 standard correctness 过滤后取 geometric mean，不使用 hidden gate |
| Mem Efficient | 存在 standard-correct candidate，且 peak memory 严格低于 baseline | 全部 active problems 的比例 |

<!-- 这里的 Correct Speedup 是 conditional metric，失败题不会以0或1进入 geomean；Fast@1 才同时反映 correctness 和 performance。指标原始定义见[论文 §2.5](https://arxiv.org/html/2607.16241#S2.SS5) -->

## 结果

论文评测7个 frontier model，所有模型的整体 Correct Speedup 都低于1。
最好结果来自 GPT-5.5：standard protocol 下为1.43×，切换到 TF32 baseline 和 hidden gate 后降为0.88×

| Model | Correct Speedup | Correctness | Fast@1 | Correct Mem Efficiency | Mem Efficient |
|---|---:|---:|---:|---:|---:|
| GPT-5.5 | 0.88× | 98.4% | 53.0% | 1.12× | 65.6% |
| Gemini 3.1 Pro | 0.80× | 90.7% | 51.4% | 0.98× | 57.5% |
| Gemini 3 Flash | 0.48× | 96.8% | 42.9% | 1.05× | 63.6% |
| Claude Opus 4.8 | 0.60× | 87.9% | 42.1% | 1.00× | 55.1% |
| Kimi K2.6 | 0.47× | 93.5% | 40.9% | 1.04× | 61.9% |
| Claude Opus 4.7 | 0.57× | 94.3% | 40.5% | 1.07× | 61.5% |
| Claude Sonnet 4.6 | 0.61× | 89.9% | 39.7% | 0.96× | 54.3% |

<!-- 结果来自[论文 overall table](https://arxiv.org/html/2607.16241#A3)。按作者的 per-problem decomposition，21.2%的题只受 TF32 baseline 修正影响，11.6%只受 hidden suite 影响，2.4%同时受两者影响；baseline correction 主要改变 L2/L3 的 compute-bound matmul/conv，hidden suite 对 L1 elementwise shortcut 的影响更明显 -->

<!-- memory 指标给出另一条信息：GPT-5.5 有65.6%的题比 baseline 节省 peak memory，但论文审计仍发现约28%的正确 kernel 增加 peak memory。fusion 能减少 intermediate allocation，不保证 latency 同步下降，尤其 TF32 matmul 已经成为 compute bottleneck 时更明显 -->


<!-- ## 公开实现的复现注意项

公开仓库当前 HEAD 为 [`3fdf6fe`](https://github.com/facebookresearch/kernel_bench_verified/commit/3fdf6fec7372a4d0cb682635f00e7bdcbc55d50e)，但论文和仓库没有 pin 所 vendored KernelBench task tree 对应的 upstream SHA。复现时需要同时记录 KernelBench-Verified SHA 与 task file hash

- public scorer 没有调用 `.eval()` / `.train()`；模型通常保持默认 `training=True`，correctness 在 `torch.no_grad()` 下执行，candidate timing 没有显式关闭 autograd，见固定版本的 [`src/eval.py`](https://github.com/facebookresearch/kernel_bench_verified/blob/3fdf6fec7372a4d0cb682635f00e7bdcbc55d50e/src/eval.py#L620-L780)
- 论文结果写3 warmup+10 timed trials；公开 [`EvalConfig`](https://github.com/facebookresearch/kernel_bench_verified/blob/3fdf6fec7372a4d0cb682635f00e7bdcbc55d50e/scripts/eval_from_generations.py#L45-L101) 默认是5 correctness trials、100 performance trials，README 命令没有显式覆盖 performance trials，复现 headline 时要手动对齐论文参数
- hidden test 在 public CLI 中默认关闭，需要设置 `use_hidden_tests=True`，leaderboard analysis 还要打开 `--use_hidden_eval` 并选择 TF32 baseline，具体命令见[官方 README](https://github.com/facebookresearch/kernel_bench_verified/blob/3fdf6fec7372a4d0cb682635f00e7bdcbc55d50e/README.md#L89-L147)
- “hidden”描述的是 generation/evaluation 流程中的信息隔离；hidden test files 已经随公开仓库发布，公开复现时并非不可见 secret set
- 软件环境记录存在冲突：[`Dockerfile`](https://github.com/facebookresearch/kernel_bench_verified/blob/3fdf6fec7372a4d0cb682635f00e7bdcbc55d50e/Dockerfile) 安装 PyTorch2.4.0+cu124，而 [`requirements.txt`](https://github.com/facebookresearch/kernel_bench_verified/blob/3fdf6fec7372a4d0cb682635f00e7bdcbc55d50e/requirements.txt) 写 PyTorch2.12.0；论文没有给出完整 image digest 与 driver 版本

复现报告至少应写明 KBV commit、active task IDs、TF32 flag、hidden gate、tolerance、sample selection、timing trials、Module/autograd state 和实际 container digest -->


# 需要讨论的问题

| 序号 | 需要讨论的问题 | 简要分析 | 讨论结果 |
|---:|---|---|---|
| 1 | 是否统一使用 `.eval() + torch.no_grad()`，或进一步使用 `torch.inference_mode()` | `.eval()` 控制 Dropout、BatchNorm 等模块的行为，`inference_mode()` 限制更严格，不保证可行|  |
| 2 | Correctness 当前使用 FP32 + `1e-4`，是否改为 TF32 + `1e-3`；timing 是否统一开启 TF32 | 当前是pytorch默认，仅conv使用TF32，其他FP32，建议timing时统一开启TF32；correctness 时，TF32+1e-3 建议对比测试 |  |
| 3 | KernelGym 是否补充显存检测 | 建议补充。另外，`torch.cuda.max_memory_allocated()` 主要覆盖 PyTorch allocator，未必完整捕获扩展中的原生 CUDA 分配 |  |
| 4 | 是否加入四组 hidden input | 不建议补充。KernelBench 新版也引入了新的random方法，但仍将原 random 方法作为默认，新 random 仅作为可选，建议仿照 KernelBench |  |
| 5 | 是否采用 KernelBench-Verified 的数据 | 建议采用 |  |
