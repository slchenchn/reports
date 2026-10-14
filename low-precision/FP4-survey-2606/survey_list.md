<!--
写作指引：
- 这是一份 2026 年上半年 FP4 相关工作的速览清单，给做大模型训练/推理/量化的人快速扫一遍前沿用
- 只收 2026 年（arXiv ID 前缀 26xx）发表的论文，以及 2026 年的深度技术博客；2025 年的奠基工作（NVFP4 pretraining 2509.25149、FP4 fully-quantized training 2505.19115、MR-GPTQ 2509.23202 等）只在背景里点一句
- 每条按贡献 + 硬数字 + 约束/失败经验来写，不复述 abstract 套话
- 收录与剔除原则：实打实用 FP4/NVFP4/MXFP4 浮点格式的才进正文，纯 INT4/聚类量化（即便标题挂 4-bit）不算；个别 borderline 会在条目里直接标注
- 数字与日期已逐条核验过源页面；arXiv 日期以 ID 月份为准（2601=1月 … 2606=6月）
- 待补图：格式对比图、2026 H1 时间线图（见正文 placeholder）
-->

# FP4 进展速览（2026 H1）

FP4 在 2025 年还停留在"能不能训得动"的论证阶段（Microsoft 的 [Optimizing LLM Training Using FP4](https://arxiv.org/abs/2501.17116)、[FP4 All the Way](https://arxiv.org/abs/2505.19115)、NVIDIA 的 [Pretraining LLMs with NVFP4](https://arxiv.org/abs/2509.25149)），到 2026 年上半年已经分化成几条平行战线同时推进：从头预训练的稳定性配方、旗舰模型把 FP4 当默认精度落地、QAT/QAD 把精度找回来、PTQ 在格式约束下抠 scale 和 rotation、以及 kernel/serving 把理论 TFLOPS 真正吃下来。下面这份清单按这几条线组织，截至 2026-06

<img src="fp4-timeline-2026h1.png" alt="2026 上半年 FP4 关键工作时间线：横轴为 1 月到 6 月，纵轴分四条泳道（训练方法、旗舰模型、QAT/QAD、推理PTQ/kernel）。1 月 Quartet II、ARCQuant、NVFP4 QAD；2 月 Dissecting Outlier Dynamics、vLLM GPT-OSS 部署博客；3 月 Practical FP4 MoE on Hopper、Megatron-Core MoE、IF4、Unveiling MXFP4、FP4 sensitivity、BATQuant；4 月 HiFloat4、Nemotron Super、DuQuant++；5 月 AMD MXFP4 pretraining、nGPT-4bit、MXFP4-RL 分解、SOAR；6 月 Nemotron Ultra、UFP4/Shrinkage Bias、CKA-QAD、ScaleSweep。每个节点标注机构 logo">

## 总表（速查）

时间按 arXiv ID 月份对齐（2601=1 月 … 2606=6 月）；单位只取一个最具代表性的；备注是一句话抓重点；最后一列只在开源时填，有代码就只给代码

| # | Title | 单位 | 会议/期刊 | 时间 | 备注 | 代码/模型 |
|---|-------|------|-----------|------|------|-----------|
| 1 | [Quartet II](https://arxiv.org/abs/2601.22813) | IST-DASLab | arXiv | 2026-01 | NVFP4 预训练，MS-EDEN 无偏梯度估计，误差比 SR 低 2x+；Blackwell kernel 相对 BF16 ≤4.2x | [IST-DASLab/Quartet-II](https://github.com/IST-DASLab/Quartet-II) |
| 2 | [Pretraining LLMs with MXFP4 on Native FP4 Hardware](https://arxiv.org/abs/2605.09825) | AMD | arXiv | 2026-05 | MI355X 原生 FP4；定位 Wgrad 量化是收敛瓶颈，唯确定性 Hadamard 能稳住 | — |
| 3 | [HiFloat4 Format for LM Pre-training on Ascend NPUs](https://arxiv.org/abs/2604.08826) | Huawei | arXiv | 2026-04 | Ascend 的 HiF4 三级分层缩放，免随机舍入；loss 误差 0.85-1.19%，优于 MXFP4 | — |
| 4 | [Normalized Architectures are Natively 4-Bit](https://arxiv.org/abs/2605.06067) | NVIDIA / Technion | arXiv | 2026-05 | nGPT 超球面约束天然适配 NVFP4，免 RHT/动态 scaling；点积 SNR +7dB | — |
| 5 | [Practical FP4 Training for Large-Scale MoE on Hopper](https://arxiv.org/abs/2603.02731) | — | arXiv | 2026-03 | 无原生 FP4 的 Hopper 上用 MXFP4 压激活+EP 通信；671B 显存 -14.8%、吞吐 +12.5% | — |
| 6 | [Rethinking Shrinkage Bias in FP4 Pretraining (UFP4)](https://arxiv.org/abs/2606.20381) | Ant Group | arXiv | 2026-06 | 指出 E2M1 的 Shrinkage Bias 逐层乘性累积，主张改用均匀 4-bit 网格 UFP4 | — |
| 7 | [Dissecting Outlier Dynamics in NVFP4 Pretraining (CHON)](https://arxiv.org/abs/2602.02047) | HKUST(GZ) | arXiv | 2026-02 | NVFP4 outlier 纵向解剖 + Hot-Channel Patch；loss gap 0.94%→0.58% | — |
| 8 | [Nemotron 3 Ultra / Super](https://arxiv.org/abs/2606.15007) | NVIDIA | arXiv | 2026-06 | 550B NVFP4 预训练（迄今最大），loss gap <0.4%；Super(2604.12374) 为首个 NVFP4 预训练 Nemotron | 模型/数据 HF 开源 |
| 9 | [Scalable Training of MoE Models with Megatron Core](https://arxiv.org/abs/2603.07685) | NVIDIA | arXiv | 2026-03 | 88 页 MoE 训练系统报告，含 NVFP4 两级 microscaling recipe（FP4 为配方组件） | [NVIDIA/Megatron-LM](https://github.com/NVIDIA/Megatron-LM) |
| 10 | [Quantization-Aware Distillation for NVFP4 (QAD)](https://arxiv.org/abs/2601.20088) | NVIDIA | arXiv | 2026-01 | KL 蒸馏恢复 NVFP4 精度，比 QAT 稳、对数据覆盖鲁棒；Nano 9B AIME25 反超 BF16 | — |
| 11 | [Beyond Output Matching (CKA-QAD)](https://arxiv.org/abs/2606.05682) | — | arXiv | 2026-06 | 指出 KL-only QAD 掩盖内部表征退化，加层级 CKA 正则；Qwen3-4B AIME25 +3.8 | — |
| 12 | [Attn-QAT: 4-Bit Attention with QAT](https://arxiv.org/abs/2603.00040) | UCSD (Hao AI Lab) | arXiv | 2026-03 | 首个 attention FP4 QAT；反向重算须与前向同精度，B200 相对 FA-4 1.39x | — |
| 13 | [Decomposing MXFP4 Error for LLM RL](https://arxiv.org/abs/2605.20402) | — | arXiv | 2026-05 | MXFP4 误差三分解（scale bias/deadzone/grid noise）对应 RL 三种失效路径 | — |
| 14 | [Adaptive Block-Scaled Data Types (IF4)](https://arxiv.org/abs/2603.28765) | MIT (HAN Lab) | arXiv | 2026-03 | 每 16 值组 FP4/INT4 自适应，复用空闲 scale 符号位，零额外存储；训练/PTQ 均胜 NVFP4 | [mit-han-lab/fouroversix](https://github.com/mit-han-lab/fouroversix) |
| 15 | [Unveiling the Potential of Quantization with MXFP4](https://arxiv.org/abs/2603.08713) | Meta | arXiv | 2026-03 | 纯软件 OAS+MBS，把 MXFP4 与 NVFP4 端到端精度差从 ~10% 压到 ~1%，GEMM 开销 6.2% | — |
| 16 | [Diagnosing FP4 Inference: Layer/Block Sensitivity](https://arxiv.org/abs/2603.08747) | Penn State | arXiv／ICLR 2026 | 2026-03 | NVFP4/MXFP4 逐层逐 block 敏感度图：MLP 投影最敏感，敏感度不集中在末层 | — |
| 17 | [ARCQuant: Augmented Residual Channels for NVFP4](https://arxiv.org/abs/2601.07475) | Tianjin University | arXiv | 2026-01 | 给激活补 NVFP4 residual channels 做误差补偿，保统一格式直接走标准 GEMM | [actypedef/ARCQuant](https://github.com/actypedef/ARCQuant) |
| 18 | [Benchmarking PTQ of LLMs under Microscaling FP](https://arxiv.org/abs/2601.09555) | Huawei | arXiv | 2026-01 | MXFP PTQ 系统评测（7+ 算法/15 benchmark）；scale factor 是 MXFP4 主误差源 | — |
| 19 | [DuQuant++: Fine-grained Rotation for MXFP4](https://arxiv.org/abs/2604.17789) | — | arXiv | 2026-04 | rotation block 对齐 microscaling group(B=32)，单次 rotation；PPL 优于 MR-GPTQ | [Hsu1023/DuQuant-v2](https://github.com/Hsu1023/DuQuant-v2) |
| 20 | [SOAR: Scale Optimization for NVFP4](https://arxiv.org/abs/2605.12245) | SJTU | arXiv | 2026-05 | NVFP4 PTQ：闭式联合优化 scale + 量化/反量化 scale 解耦离散搜索 | — |
| 21 | [ScaleSweep: NVFP4 PTQ via Block Scale Init](https://arxiv.org/abs/2606.07618) | Peking University | arXiv | 2026-06 | block scale 候选扫描替代 AbsMax（推出搜索上下界），激进量化保 >93% 全精度 | — |
| 22 | [BATQuant: Outlier-resilient MXFP4 Quantization](https://arxiv.org/abs/2603.16590) | Huawei / USTC | arXiv | 2026-03 | 诊断全局旋转跨 block 转移 outlier，改 block-wise affine；多模态恢复 96.43% | — |

深度博客与工具链（详见第六节，平台代会议/期刊）：

| Title | 单位/平台 | 时间 | 备注 | 链接 |
|-------|-----------|------|------|------|
| Using NVFP4 Low-Precision Model Training | NVIDIA | 2026-02 | Llama 3 8B 1T token NVFP4 预训练，B200 相对 BF16 ≤1.59x，关键是末四层保 BF16 | [link](https://developer.nvidia.com/blog/using-nvfp4-low-precision-model-training-for-higher-throughput-without-losing-accuracy/) |
| Train Models Faster with JAX/MaxText Using NVFP4 | NVIDIA | 2026-06 | JAX/MaxText NVFP4 配方，FP8→NVFP4 加速 1.31-1.73x（405B 在 GB300 1.73x） | [link](https://developer.nvidia.com/blog/train-models-faster-with-jax-and-maxtext-using-nvfp4-on-nvidia-blackwell/) |
| 3 Ways NVFP4 Accelerates AI Training and Inference | NVIDIA | 2026-02 | 官方综述：512 卡训 405B 仅 64.6 分钟（比 FP8 快 1.9x），前瞻 Rubin | [link](https://developer.nvidia.com/blog/3-ways-nvfp4-accelerates-ai-training-and-inference/) |
| Accelerating LLMs with NVFP4 Quantization | Red Hat | 2026-02 | NVFP4 PTQ 工程实践，精度恢复随规模上升（70B+ ~99%）；LLM Compressor + vLLM | [link](https://developers.redhat.com/articles/2026/02/04/accelerating-large-language-models-nvfp4-quantization) |
| GPT-OSS Optimizations on Blackwell | vLLM / NVIDIA | 2026-02 | gpt-oss-120b(MXFP4 MoE) 推理 max-throughput +38%，接 FlashInfer MXFP4 MoE kernel | [link](https://blog.vllm.ai/2026/02/01/gpt-oss-optimizations.html) |
| TFLOPS Gap: FP4 MoE Kernel Engineering | HuggingFace | 2026-01 | 同硬件下 vLLM/SGLang/FlashInfer 三套 NVFP4 MoE kernel 差 ~145 TFLOPS | [link](https://huggingface.co/blog/apsys/blackwell-nvfp4-comparison) |
| Optimizing NVFP4 Grouped GEMM – Worklog | 个人 (Mufeez Amjad) | 2026-03 | kernel 竞赛复盘，238us→23.8us（~10x），TMA+warp spec+cluster multicast | [link](https://mufeezamjad.com/blog/nvfp4-group-gemm) |

---

## FP4 的两种主流格式（先对齐口径）

两种格式权重都是 **E2M1**（1 符号 + 2 指数 + 1 尾数 = 4 bit），差别全在 block 粒度和 scale 上，这决定了后面几乎所有 PTQ/训练技巧的取舍：

- **NVFP4**（NVIDIA）：16 元素一个 block，block scale 用 FP8 E4M3，外面再叠一个 per-tensor FP32 global scale。小 block + 高精度 scale，动态范围保得好，但每权重多约 0.5 bit 开销，且只跑在 Blackwell（B200/B300/GB200/GB300/RTX 5090/RTX PRO 6000）
- **MXFP4**（OCP 标准）：32 元素一个 block，block scale 是 E8M0 的 power-of-two。元数据省（每权重约 0.25 bit），生态更宽（AMD MI355X / ROCm 7.x 原生，Hopper 可软件模拟，GPT-OSS 原生权重），但 power-of-two scale 和大 block 在 4-bit 下精度更难做

经验上同等校准质量 NVFP4 通常优于 MXFP4，但 MXFP4 配 MR-GPTQ 这类校准能补回大部分差距；reasoning/数学这类任务两种 FP4 都还难可靠替代 FP8。Blackwell 上 NVFP4 算术吞吐约为 FP8 的 2-3x、显存约 1.8x，相对 FP16 显存约 3.5x

<img src="nvfp4-vs-mxfp4-scaling.png" alt="NVFP4 与 MXFP4 的两级缩放结构对比示意图。左侧 NVFP4：一行 16 个 E2M1 值组成一个 micro-block，共享一个 FP8 E4M3 block scale，整个 tensor 再乘一个 FP32 per-tensor global scale，标注每权重 ~0.5 bit scale 开销。右侧 MXFP4：一行 32 个 E2M1 值组成一个 block，共享一个 E8M0 power-of-two block scale，无 per-tensor scale，标注 ~0.25 bit 开销。底部对比表：block size 16 vs 32、scale 类型 E4M3 vs E8M0、硬件 Blackwell-only vs Blackwell+MI355X+Hopper-emulated">

---

## 一、预训练与训练方法

这条线 2026 的主题是：FP4 从头训不再是"能不能收敛"，而是"哪个算子、哪种格式、哪种 rounding 才是稳定性的真正瓶颈"，结论高度收敛到 **weight gradient 路径 + 格式几何**

**[Quartet II: Accurate LLM Pre-Training in NVFP4 by Improved Unbiased Gradient Estimation](https://arxiv.org/abs/2601.22813)**
`2026-01 · IST-DASLab · arXiv 2601.22813`
原始 Quartet 的续作，提出无偏量化算子 MS-EDEN：把随机性从单个 FP4 值上移到 microscale factor，配合 Randomized Hadamard Transform 和对 group scale 的无偏 stochastic rounding，量化误差比普通 SR 低 2x 以上。线性层全 NVFP4，前向反向所有主要 matmul 都改进了梯度估计，在 1.9B / 38B token 上验证，Blackwell kernel 相比 BF16 最高 4.2x，代码开源在 [IST-DASLab/Quartet-II](https://github.com/IST-DASLab/Quartet-II)

**[Pretraining Large Language Models with MXFP4 on Native FP4 Hardware](https://arxiv.org/abs/2605.09825)**
`2026-05 · AMD · arXiv 2605.09825`
在 MI355X 原生 FP4 上对 Llama 3.1-8B 做全流程 MXFP4 预训练，逐级开启 Fprop/Dgrad/Wgrad 的受控实验定位到 **Wgrad（weight gradient）量化是收敛退化的主因**。一旦量化 Wgrad，stochastic rounding 和随机 Hadamard 都救不回，只有**确定性** Hadamard 旋转能稳住——作者据此判断不稳定来自敏感梯度路径上的结构化 micro-scaling 误差，而非随机性不足。端到端相对 FP8 加速 9-10%，token 开销增加约 8-9%

**[HiFloat4 Format for Language Model Pre-training on Ascend NPUs](https://arxiv.org/abs/2604.08826)**
`2026-04 · Huawei · arXiv 2604.08826`
华为给 Ascend 设计的 FP4 格式 HiF4，用三级分层缩放（E6M2 一级 + E1 二/三级 scaler + S1P2 数值，每 64 元素 block 仅 32 bit 元数据），靠格式本身的数值稳定性省掉 MXFP4 必需的随机舍入和 truncation-free scaling，只需 RHT + nearest rounding。三个模型各训 50B token，相对 BF16 的 loss 误差 0.85%-1.19%，全面优于 MXFP4 的 1.44%-1.79%；MoE 上 FP4 计算覆盖率到 95.9%

**[Normalized Architectures are Natively 4-Bit](https://arxiv.org/abs/2605.06067)**
`2026-05 · NVIDIA / Technion · arXiv 2605.06067`
一个角度很不一样的结论：nGPT（权重和隐表示约束到单位超球面）天然对低精度鲁棒，能直接端到端 NVFP4 训练，不用 RHT 也不用动态 per-tensor scaling。机制是超球面约束放大元素积的弱正相关，让信号在 hidden 维度建设性累加而量化噪声相互抵消，点积 SNR 从标准 GPT 的 18.6 dB 提到 26 dB，loss landscape 平坦约 3.5 倍。在 3B/30B hybrid Mamba-Transformer MoE 上 NVFP4 相对误差约 0%

**[Practical FP4 Training for Large-Scale MoE Models on Hopper GPUs](https://arxiv.org/abs/2603.02731)**
`2026-03 · arXiv 2603.02731`
没有原生 FP4 tensor core 的 Hopper 上怎么吃 FP4 的红利：核心计算仍走 FP8，只把激活和专家并行（EP）通信用 MXFP4 压。关键是直接 FP8↔FP4 量化加 scaling-aware 的行优先到列优先转换，绕开 FP4↔BF16↔FP8 的精度往返。671B 规模上峰值激活显存降 14.8%、吞吐升 12.5%，收敛与 FP8 baseline 持平

**[Rethinking Shrinkage Bias in LLM FP4 Pretraining: Geometric Origin, Systemic Impact, and UFP4 Recipe](https://arxiv.org/abs/2606.20381)**
`2026-06 · Ant Group · arXiv 2606.20381`
直接质疑 E2M1 这条主线：非均匀格式的 bin 几何不对称带来系统性负向舍入误差（Shrinkage Bias），逐层乘性累积，还被广泛用于离群值抑制的 RHT 进一步放大。作者提出 UFP4，换成 E1M2/INT4 风格的**均匀 4-bit 网格**并对所有训练算子施加 RHT，在 Dense 1.5B / MoE 7.9B / MoE 124B 上 loss 退化都优于 E2M1 基线，主张未来加速器应把均匀 4-bit 网格作为一等训练原语

**[Dissecting Outlier Dynamics in LLM NVFP4 Pretraining](https://arxiv.org/abs/2602.02047)**
`2026-02 · HKUST(GZ) / Alibaba / U Toronto · arXiv 2602.02047`
对 NVFP4 预训练做 outlier 的纵向解剖：集中在 Softmax、linear-attention gating、SwiGLU，post-QK 操作最敏感；训练早期是瞬态尖峰，后期收敛成一小撮持久 hot channel。提出 Hot-Channel Patch 在线识别并用高效 kernel 回注残差（整套叫 CHON），在 GLA-1.3B / 60B token 上把相对 BF16 的 loss gap 从 0.94% 压到 0.58%

<!-- 另有 NYU 的 Understanding Quantization of Optimizer States (arXiv 2603.16731)，分析低精度 optimizer state 的 stalling 机制和 reset 时机，但非 FP4-specific，未进正文 -->

---

## 二、旗舰模型里的 FP4 实践

FP4 在 2026 不再是论文里的玩具，几个最大的开放模型直接把它当训练/推理的默认精度

**[Nemotron 3 Ultra / Super: Open Efficient MoE Hybrid Mamba-Transformer](https://arxiv.org/abs/2606.15007)**
`2026-06 · NVIDIA · arXiv 2606.15007（Super 见 2604.12374）`
Ultra 是 550B 总参 / 55B 激活的 MoE hybrid Mamba-Attention，base 用 NVFP4 分两阶段共 20T token 预训练，是目前最大规模的稳定准确 NVFP4 训练。配方很说明问题：多数 linear 走 NVFP4，但**末尾 15%（16 层）、Mamba 输出投影、QKV/attention 投影、MTP、embedding 留高精度**，相对 BF16 段 loss gap 平均 <0.4%。Super（120B/12.7B，2604.12374）是家族里首个 NVFP4 预训练的模型，引入 LatentMoE，推理吞吐相对 Qwen3.5-122B 最高 7.5x

**[Scalable Training of Mixture-of-Experts Models with Megatron Core](https://arxiv.org/abs/2603.07685)**
`2026-03 · NVIDIA · arXiv 2603.07685`
88 页的 MoE 训练系统报告，FP4 是其中一节：NVFP4 用 E2M1 两级 microscaling（per-tensor FP32 + per-block E4M3 8-bit），作为 reduced-precision recipe 的一部分维持收敛。系统侧的 Parallel Folding、grouped GEMM、通信 overlap 更值得读——端到端 DeepSeek-V3-685B 在 GB300 上 1233 TFLOPS/GPU。报告未单列 NVFP4 vs BF16/FP8 的 loss 差，FP4 在这里是配方组件而非主角

---

## 三、微调 / QAT / QAD

PTQ 在 4-bit 下掉点明显时，2026 的主流回答是 QAT/QAD，而且重点从"对齐 logits"转向"对齐内部表征"和"保护敏感算子"

**[Quantization-Aware Distillation for NVFP4 Inference Accuracy Recovery](https://arxiv.org/abs/2601.20088)**
`2026-01 · NVIDIA · arXiv 2601.20088`
提出 QAD：用 KL 散度把全精度 teacher 蒸馏到 NVFP4 student，比传统 QAT 在 SFT/RL/model-merging 多阶段 post-training 流程下更稳、对数据覆盖更鲁棒。Nemotron Nano 9B V2 在 AIME25 上 QAD 71.5%（BF16 71.1%，反超 +0.4），QAT 仅 67.1%，QAD 领先 4.4 个点；蒸馏 token 量 0.3B-6B 不等

**[Beyond Output Matching: Preserving Internal Geometry in NVFP4 LLM Distillation](https://arxiv.org/abs/2606.05682)**
`2026-06 · arXiv 2606.05682`
对上一条 QAD 的批判性跟进：用 CKA 分析发现 KL-only QAD 会把末层表征相似度从 PTQ 的 0.983 拉到 0.740，输出对齐掩盖了内部表征退化，RL 后训练模型尤其严重。CKA-QAD 在 TopK-KL 损失上加层级 Gram 矩阵的 CKA 正则，Qwen3-4B-Thinking 的 AIME25 从 68.5% 提到 72.3%，开销很小（step time +0.5%、显存 +7%）

**[Attn-QAT: 4-Bit Attention With Quantization-Aware Training](https://arxiv.org/abs/2603.00040)**
`2026-03 · Hao AI Lab @ UCSD · arXiv 2603.00040`
第一篇系统做 attention 的 FP4 QAT——attention 激活重尾、对 FP4 的极小动态范围最敏感，是端到端 FP4 计算最后一块硬骨头。关键发现是把 FP4 前向直接接高精度 FlashAttention 反向会训崩，两条稳定原则：反向重算 attention 概率要用与前向一致的低精度、保存高精度辅助输出以解决 FA 梯度里隐含的精度假设。B200 上相对 FlashAttention-4 加速 1.39x、FP4 达 1801 TFLOPS

**[Decomposing MXFP4 Quantization Error for LLM RL: Reducible Bias, Recoverable Deadzone, and an Irreducible Floor](https://arxiv.org/abs/2605.20402)**
`2026-05 · arXiv 2605.20402`
把 MXFP4 误差精确三分解——scale bias（power-of-two 取整）、deadzone truncation（小值置零）、grid noise（4-bit 网格舍入）——并论证每项各主导一种 RL 失效路径：scale bias 在反向乘性累积伤梯度、deadzone 恶化 rollout、grid noise 抬高策略熵。对应给 Macro-block scaling + Outlier Fallback + AQN 三个修正，Qwen3-30B-A3B 反超 BF16 +1.0%；grid noise 是消不掉的下限

---

## 四、格式与硬件协同

**[Adaptive Block-Scaled Data Types](https://arxiv.org/abs/2603.28765)**
`2026-03 · MIT HAN Lab · arXiv 2603.28765`
针对 NVFP4 在每组近最大值处量化误差偏大，提出 IF4：每 16 值组在 FP4/INT4 间自适应选择，复用 NVFP4 中空闲的 scale 符号位标记类型，**零额外存储**，同框架推广到 IF3/IF6。训练（W4A4G4）loss 低于 NVFP4，配 MS-EDEN 差距更大；PTQ 在 Qwen 上 WikiText-2 PPL 8.27 vs NVFP4 8.37。还给了 IF4 MAC 单元的硬件开销（延迟 +4.7%、面积 +66.6%），证明可在下一代加速器落地，代码在 [mit-han-lab/fouroversix](https://github.com/mit-han-lab/fouroversix)

---

## 五、推理后训练量化（PTQ）

PTQ 这条线最拥挤，但绕来绕去都在解同一个矛盾：FP4 的 fine-grained block scaling 让传统的 rotation/smoothing/mixed-precision 三板斧失效，得为格式量身定制

**[Unveiling the Potential of Quantization with MXFP4: Strategies for Quantization Error Reduction](https://arxiv.org/abs/2603.08713)**
`2026-03 · Meta · arXiv 2603.08713`
两个纯软件、不改硬件的技巧：Overflow-Aware Scaling 在 power-of-two 块缩放下扩大有效动态范围，Macro Block Scaling 用更粗粒度分配高精度 scale 保 outlier。把 MXFP4 与 NVFP4 的端到端精度差从约 10% 压到平均 1% 以内，GEMM 开销仅约 6.2%——意义在于让生态更宽的 MXFP4 逼近 NVFP4 精度

**[Diagnosing FP4 Inference: A Layer-wise and Block-wise Sensitivity Analysis of NVFP4 and MXFP4](https://arxiv.org/abs/2603.08747)**
`2026-03 · Penn State · arXiv 2603.08747`
做混合精度前必看的敏感度地图：每次只把某类投影或某个 block 量化到 FP4、其余 FP16。结论是 MLP 投影最敏感（down > up > gate > attention 的 q/k/v），attention 投影可低精度；敏感度**不集中在末尾层**，早期 block 在 MXFP4 下也可能很敏感。Qwen2.5-0.5B 上 MXFP4 基线 PPL 约 36.71 vs NVFP4 约 21.63

**[ARCQuant: Boosting NVFP4 Quantization with Augmented Residual Channels for LLMs](https://arxiv.org/abs/2601.07475)**
`2026-01 · Tianjin University · arXiv 2601.07475`
思路很巧：给激活补一组同样按 NVFP4 量化的 residual channels 做误差补偿，把补偿融进 reduction 维度，从而严格保持统一 NVFP4 格式、直接复用标准 GEMM kernel，几乎不增开销，绕开 rotation 破坏 block isolation、smoothing 压不住 4-bit、mixed-precision 违反硬件统一精度三个痛点。RTX 5090 上相对 FP16 最高约 3x，代码 [actypedef/ARCQuant](https://github.com/actypedef/ARCQuant)

**[Benchmarking Post-Training Quantization of LLMs under Microscaling Floating Point Formats](https://arxiv.org/abs/2601.09555)**
`2026-01 · Huawei · arXiv 2601.09555`
7+ 种 PTQ 算法 × 15 benchmark × 3 个 LLM 族的系统评测。结论：MXFP8 近乎无损，MXFP4 掉点明显且仍是难点；scale factor 被定位为 MXFP4 主要误差源，一个简单 pre-scale 优化就能显著缓解；多模态 LLM 的量化敏感度由 language model 主导而非 vision encoder

**[DuQuant++: Fine-grained Rotation Enhances Microscaling FP4 Quantization](https://arxiv.org/abs/2604.17789)**
`2026-04 · arXiv 2604.17789`
把 DuQuant 适配到 MXFP4：让 rotation block size 与 microscaling group（B=32）对齐，用 outlier-aware 细粒度 rotation 打散集中在 channel 的 outlier，避免单个 outlier 抬高整块 scale。每 group 独立 scale 使单次 rotation 即可取代原来的 dual rotation。LLaMA3-8B W4A4 WikiText2 PPL 7.07（带 GPTQ 补偿 6.88），优于 MR-GPTQ 7.29，代码 [Hsu1023/DuQuant-v2](https://github.com/Hsu1023/DuQuant-v2)

同一时期还有几篇专攻 NVFP4/MXFP4 的 scale 与 outlier 处理，思路互补，值得一并扫：

- **[SOAR: Scale Optimization for Accurate Reconstruction in NVFP4 Quantization](https://arxiv.org/abs/2605.12245)**（`2026-05 · SJTU`）用闭式解联合优化 global 与 block-wise scale，并把量化/反量化 scale 解耦做离散搜索，Qwen3-8B W4A4 平均 +0.56 vs RaZeR，同显存无额外硬件
- **[ScaleSweep: Accurate NVFP4 PTQ via Block Scale Initialization](https://arxiv.org/abs/2606.07618)**（`2026-06 · PKU`）指出现有 scale 初始化基本沿用 AbsMax，离最优有明显差距，改成按 MSE/WMSE 扫描 block scale 候选并推导出搜索上下界，权重+激活+KV+query 激进量化下保 >93% 全精度
- **[BATQuant: Outlier-resilient MXFP4 Quantization via Learnable Block-wise Optimization](https://arxiv.org/abs/2603.16590)**（`2026-03 · Huawei / USTC`）诊断出全局正交旋转会把 outlier 能量跨 MXFP block 转移、产生双峰分布，改用 Block-wise Affine Transformation 限制在 block 粒度内，多模态 benchmark 恢复到全精度的 96.43%

<!-- 剔除说明：CodeQuant(2604.10496)、IBM 的 MoE 泛化保证(2604.06515)、QIG(2603.17809) 标题或场景挂 4-bit，但实际用的是 INT4/聚类/W4A8 整数量化，不涉及 FP4 浮点格式，未进正文 -->

---

## 六、深度博客与工具链

论文之外，2026 上半年这批厂商/团队博客信息密度很高，尤其是把 FP4 真正部署上线踩过的坑

**训练侧配方**

- [Using NVFP4 Low-Precision Model Training for Higher Throughput Without Losing Accuracy](https://developer.nvidia.com/blog/using-nvfp4-low-precision-model-training-for-higher-throughput-without-losing-accuracy/)（`NVIDIA · 2026-02`）Llama 3 8B 在 1T token 上 NVFP4 预训练，B200 相对 BF16 最高 1.59x，稳定性关键是**最后四层 transformer 保 BF16**，recipe 已进 NeMo Megatron Bridge
- [Train Models Faster with JAX and MaxText Using NVFP4 on Blackwell](https://developer.nvidia.com/blog/train-models-faster-with-jax-and-maxtext-using-nvfp4-on-nvidia-blackwell/)（`NVIDIA · 2026-06`）JAX/MaxText 的 NVFP4 配方，FP8→NVFP4 预训练加速 1.31-1.73x，Llama 3.1 405B 在 GB300 上 1.73x，五项核心技术（16 元素 micro block、两层缩放、选择性 RHT、2D weight scaling、随机舍入）讲得很细
- [3 Ways NVFP4 Accelerates AI Training and Inference](https://developer.nvidia.com/blog/3-ways-nvfp4-accelerates-ai-training-and-inference/)（`NVIDIA · 2026-02`）官方综述口径：512 卡 Blackwell Ultra 训 Llama 3.1 405B 仅 64.6 分钟，比 FP8 快 1.9x；前瞻 Rubin（35/50 PFLOPS）

**推理量化实践与工具链**

- [Accelerating LLMs with NVFP4 Quantization](https://developers.redhat.com/articles/2026/02/04/accelerating-large-language-models-nvfp4-quantization)（`Red Hat · 2026-02`）NVFP4 PTQ 的工程实践，精度恢复随规模上升（70B-235B 约 99%、7B-14B 约 95-98%），小模型校准不稳定，工具链 LLM Compressor + vLLM

**kernel 与大规模 serving**

- [GPT-OSS Performance Optimizations on Blackwell: Pushing the Pareto Frontier](https://blog.vllm.ai/2026/02/01/gpt-oss-optimizations.html)（`vLLM/NVIDIA · 2026-02`）gpt-oss-120b（原生 MXFP4 MoE）在 Blackwell 上 max-throughput +38%，接 FlashInfer 的 MXFP4 MoE kernel 吃原生 FP4 tensor core，给了具体调优 flag
- [TFLOPS Gap: Why FP4 MoE Kernel Engineering Matters on Blackwell](https://huggingface.co/blog/apsys/blackwell-nvfp4-comparison)（`HuggingFace · 2026-01`）同硬件同模型下 vLLM/SGLang/FlashInfer 三套 NVFP4 MoE kernel 差约 145 TFLOPS，SGLang 靠 sm_100a 定制 CUTLASS schedule + padding 强制对齐让 TMA 走快路径
- [Optimizing NVFP4 Grouped GEMM on Blackwell – Worklog](https://mufeezamjad.com/blog/nvfp4-group-gemm)（`个人博客 · 2026-03`）GPUMode kernel 竞赛复盘，从 238us 优化到 23.8us（约 10x），TMA + warp specialization + cluster multicast + 深流水，作者还坦白了离最快提交 16us 的差距来源
