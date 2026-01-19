# 大模型量化技术分享

> **目标读者**：对量化有初步了解但缺系统框架，主要诉求是工程落地——在既定推理框架（优先 vLLM）与目标硬件约束下，拿到可复现的量化配方（选型、校准、导出格式、kernel/后端适配、评测与回滚），并能解释质量/性能波动的原因。

---

## 目录

1. [量化基础原理](#1-量化基础原理)
2. [量化框架选型](#2-量化框架选型)
3. [推理框架与部署](#3-推理框架与部署)
4. [算法选型指南](#4-算法选型指南)
5. [场景化量化策略](#5-场景化量化策略)
6. [实战案例：DeepSeek-R1](#6-实战案例deepseek-r1)
7. [踩坑经验与调试技巧](#7-踩坑经验与调试技巧)
8. [附录：常见问题解答](#8-附录常见问题解答)

---

## 1. 量化基础原理

量化过程分为两个核心阶段：

### 1.1 Pre-processing（预处理）

在浮点域对权重做等效变换，如缩放、旋转等。

**SmoothQuant 公式**：

$$
Y=XW=(X \cdot diag(s)^{-1}) \cdot (diag(s) \cdot W)=(X \cdot R^{-1}) \cdot (R \cdot W)
$$

> 若 R 为 Hadamard 矩阵，则该公式即为 **QuaRot** 量化方法。

<figure style="text-align:center;">
  <img src="smoothquant.png" alt="smoothquant" style="width:80%;" />
  <figcaption>SmoothQuant 原理示意</figcaption>
</figure>

### 1.2 Rounding（舍入）

将浮点数值 cast 成整数，包含两个关键步骤：

| 步骤 | 说明 | 优化方法 |
|------|------|----------|
| **Clamp** | 限制最大数值，压缩动态范围，提高量化 grid 精度 | 以 MSE 或 KL-div 为指标选择 clamp 数值 |
| **Rounding** | 将浮点数舍入到整数域 | 四舍五入是 MSE 度量下的最优方法，但可通过训练找到全局最优舍入策略 |

<figure style="text-align:center;">
  <img src="adaround.png" alt="adaround" style="width:40%;" />
  <figcaption>AdaRound 舍入优化</figcaption>
</figure>

**技术趋势**：
- 主流 INT 量化方法聚焦 pre-processing 阶段
- [TesseraQ](https://github.com/Intelligent-Computing-Lab-Panda/TesseraQ) 等方法关注 rounding 阶段
- FP 量化因 ulp 不固定，rounding 阶段需要更多关注

---

## 2. 量化框架选型

| 框架 | 定位/特点 | 支持推理框架 | 使用注意 |
|------|-----------|--------------|----------|
| [LightCompress](https://github.com/ModelTC/LightCompress) | 学术导向，覆盖面广 | vLLM / SGLang / AutoAWQ | 文档更新滞后 |
| [llm-compressor](https://github.com/vllm-project/llm-compressor) | vLLM 官方工具 | vLLM / SGLang | 量化效率一般；高级需求需二次开发 |
| [GPTQModel](https://github.com/ModelCloud/GPTQModel) | 专注权重量化，社区活跃 | Transformers / vLLM | 非 GPTQ 或全量化场景支持不足 |
| [ModelOpt](https://github.com/NVIDIA/Model-Optimizer) | NVIDIA 官方工具链 | TensorRT-LLM | 需适配才能用 vLLM 推理 |
| [neural-compressor](https://github.com/intel/neural-compressor) | Intel 官方工具 | OpenVINO / ONNX Runtime / PyTorch | 软件模拟方法丰富 |
| [bitsandbytes](https://github.com/bitsandbytes-foundation/bitsandbytes) | 训练/推理通用，适配 HuggingFace | Transformers / PEFT | 8bit 与 4bit（含 QLoRA） |

---

## 3. 推理框架与部署

### 3.1 推理框架对比

| 框架 | 侧重点 | 上手难度 | 模型支持 | 调度/工程特性 |
|------|--------|----------|----------|---------------|
| **TRT-LLM** | 极致性能、企业级场景 | 高 | 依赖 NVIDIA 生态 | 性能最优但流程复杂 |
| **vLLM** | 学术/工程通用 | 低 | 模型覆盖更广 | 吞吐与易用性均衡 |
| **SGLang** | 学术/工程通用 | 低 | 模型覆盖较少 | 调度效率高、代码量少 |

> **推荐**：考虑到芯片部门选择了 vLLM 作为推理框架，建议后续量化工作优先选择 vLLM。

### 3.2 量化框架与推理框架的依赖关系

- **算法层面**：基本独立
- **工程层面**：**强依赖**推理端是否支持对应的量化产物格式/权重布局，以及是否具备匹配的 kernel 与算子融合路径

### 3.3 高效推理配置

**TP（Tensor Parallelism）配置原则**：TP 应与模型大小/显存匹配，过大过小都会影响效率。

| 模型 | 硬件 | 推荐配置 |
|------|------|----------|
| Qwen3-8B | 8 卡 A800 | TP=1，部署 8 个实例 |
| Qwen3-32B | 8 卡 A800 | TP=4 |

### 3.4 性能分析

参考：[Profiling vLLM](https://docs.vllm.ai/en/stable/contributing/profiling/#profile-with-pytorch-profiler)

---

## 4. 算法选型指南

### 4.1 主流算法对比

| 方法 | 适用场景/组合建议 |
|------|-------------------|
| **GPTQ** | 适合与旋转方法配合 |
| **AWQ** | 适合 WnA16 场景（如 W4A16） |
| **SmoothQuant** | 仅在 NVFP4 量化中有优势 |

### 4.2 推荐组合

| 量化类型 | 推荐组合 | 备注 |
|----------|----------|------|
| **INT 量化** | OSTQuant + GPTQv2 | 建议在目标模型/数据集/推理后端上复测 |
| **NVFP4 量化** | SmoothQuant + GPTQv2 | 关注推理端是否有隐式 fallback |

### 4.3 量化粒度选择

| 粒度 | 适用对象 | 说明 |
|------|----------|------|
| **per-channel** | 权重量化 | 按通道粒度 |
| **per-token** | 激活量化 | 按 token 维度自适应，LLM 首选 |
| **per-group** | 小模型 | 通过二级量化逼近，参考 [QServe](https://arxiv.org/abs/2405.04532) |

<figure style="text-align:center;">
  <img src="qserve.png" alt="qserve" style="width:80%;" />
  <figcaption>QServe 二级量化示意</figcaption>
</figure>

### 4.4 权重量化 vs 全量化

**决策准则**：

| 条件 | 推荐方案 |
|------|----------|
| 浮点算力不足（如 x6000） | 全量化（W4A8 / W8A8） |
| 推理负载不高，memory-bound | 仅权重量化（W4A16 / W8A16） |

> W4A16 / W8A8 在不少任务上**有机会**达到接近无损的精度（依赖校准/数据集/实现细节）。

---

## 5. 场景化量化策略

### 5.1 MoE 模型量化

- **优势**：相比同参数量的 Dense 模型，MoE 模型更易量化
- **问题排查**：若长文能力（GPQA/Code）下降而短文能力（CEval）不变，考虑不量化 attn 部分

### 5.2 大模型 vs 小模型

| 模型规模 | 量化难点 | 解决思路 |
|----------|----------|----------|
| **小模型** | weight 难量化，无结构性 pattern | 使用更细粒度量化（per-group） |
| **大模型（80B+）** | activation 难量化 | per-token 或更细粒度量化有效；工程挑战在于量化效率和资源占用 |

### 5.3 混合精度量化

**按层优先级排序**（从易到难）：

1. routed-experts / gate / mlp
2. shared-experts
3. attn.o_proj
4. attn.qkv_proj
5. lm_head / embedding

**按 block 位置**：优先给后面的 block 更高精度

**Tensor 内混合精度**：参考 [ARCQuant](https://arxiv.org/abs/2601.07475)

### 5.4 长上下文量化

> TODO：待补充（可从 KV cache/attention 的数值稳定性、RoPE scaling/位置编码策略、长上下文评测集构造与稳定复现切入）

### 5.5 多模态对齐量化

> TODO：待补充（建议覆盖：视觉编码器/投影层是否量化、对齐层保精度策略、多模态评测与回归集）

### 5.6 DiT 扩散模型量化

> TODO：待补充（建议从 VAE/文本编码器/DiT 主干分别量化的敏感度、生成质量指标与回归集、采样器与 CFG 对数值稳定性的影响切入）

---

## 6. 实战案例：DeepSeek-R1

**参考文档**：[DeepSeek-R1 量化报告](https://mx4lbik1jc.feishu.cn/wiki/PrQZwdj2hiU4TskDjgDcGx8Pnoe)

### 6.1 量化方案

**核心方法**：QuaRot（仅需旋转变换）

### 6.2 为什么不用 GPTQ？

| 原因 | 说明 |
|------|------|
| **资源限制** | 1T 内存 + 8×A800 做 data-free 量化已是极限 |
| **速度瓶颈** | 每个 expert 因输入不同，需单独做一次 GPTQ |

<figure style="text-align:center;">
  <img src="spinquant.png" alt="spinquant" style="width:80%;" />
  <figcaption>SpinQuant 原理</figcaption>
</figure>

<figure style="text-align:center;">
  <img src="hadamard.png" alt="hadamard" style="width:80%;" />
  <figcaption>Hadamard 变换</figcaption>
</figure>

---

## 7. 踩坑经验与调试技巧

### 7.1 量化结果检查

**必查项目**：weight-name、dtype、shape、数值范围

> 开发检查脚本，很多问题仅通过这些基础检查就能发现。

### 7.2 推理框架日志

⚠️ **关键提醒**：vLLM / SGLang 可能只打 warning 就自动 fallback，而不报错！

**典型案例**：A800 无法推理 NVFP4-W4A4，vLLM 会自动 fallback 成 NVFP4-W4A16。

### 7.3 量化损失可视化

<figure style="text-align:center;">
  <img src="gptqv2_loss.png" alt="gptqv2_loss" style="width:80%;" />
  <figcaption>GPTQv2 量化损失可视化</figcaption>
</figure>

### 7.4 数值精度问题

**建议**：精度要求高的场景，使用 CPU 做 dtype cast。

**原因**：CUDA 对 FP8 的 cast 可能存在问题，例如 168.1 会 cast 成 160（预期应为 176）。

### 7.5 稳定评测

| 问题 | 解决方案 |
|------|----------|
| 温度系数对 thinking 模型影响大 | 使用固定温度系数 |
| 采样过程增大复现难度 | 固定 seed=42 |
| 浮点运算非结合性导致结果不同 | 参考 [Defeating Nondeterminism](https://thinkingmachines.ai/blog/defeating-nondeterminism-in-llm-inference/) |
| 小数据集波动大 | 对数据集进行倍增 |

> 条件允许时，建议使用[全量测试集](https://mx4lbik1jc.feishu.cn/wiki/St9fwFNB7i9XYgkkCP1c7vMynOb)进行测试。

---

## 8. 附录：常见问题解答

### Q1: scale / zero offset 是什么？怎么存储？

scale / zero 用于将浮点域数值映射到整数域：

- **量化**：`q = round(x / scale) + zero_point`
- **反量化**：`x_hat = (q - zero_point) * scale`

存储方式详见：[量化 Survey](https://mx4lbik1jc.feishu.cn/wiki/OHfpwaOcaiMv93kLZX9cHFsqnQg)

### Q2: Q/DQ 显式量化 vs llm-compressor 产物有何不同？

| 类型 | 说明 | Forward 公式 |
|------|------|--------------|
| **Q/DQ (fake-quant)** | 先量化反量化，再用高精度运算 | $Y = dq(q(X)) \cdot dq(q(W^{T}))$ |
| **real-quant** | llm-compressor 产物 | $Y = X \cdot W^{T}$（直接低精度运算） |

**差异**：
- 存在微小数值偏差
- 多数 PTQ 方法对 fake/real quant 不敏感
- 部分训练方法（如 TesseraQ）可能更敏感
- **推荐**：优先使用 real-quant，推理效率更高

### Q3: W4A16 / W8A8 需要魔改 vLLM 吗？

需要，改动与推理执行的 kernel 内核相关：
- 权重打包
- 算子融合
- 量化/反量化位置

### Q4: 量化结果中的 config / tokenizer 是什么？

这些是 Hugging Face 格式模型的标准组成部分，非量化独有。量化配置通常写在 `config.json` 中，便于 vLLM 选择正确的推理方法。

### Q5: 模型评测能否模块化？

详见：[LLM Benchmark 评估工作流设计文档](https://mx4lbik1jc.feishu.cn/wiki/St9fwFNB7i9XYgkkCP1c7vMynOb)

### Q6: 不同模型如何选择量化方法？

目前没有自动选择的方法，需要根据模型特点和目标场景进行实验验证。

---

*文档版本：2601 | 最后更新：2026-01*
