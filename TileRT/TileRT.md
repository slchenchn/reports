# TileRT 调研报告：面向超低延迟的 Tile 级推理引擎

## TileRT 概述

TileRT（Tile-Based Runtime）是 [tile-ai](https://github.com/tile-ai) 团队推出的一款专为超低延迟 LLM 推理设计的实验性 runtime。


### 性能表现

在典型的 BatchSize=1 场景下，TileRT 展现了显著的延迟优势：

<figure style="text-align:center;">
  <img src="utps.png" alt="TileRT Performance Comparison" style="width:80%;" />
  <figcaption>测试配置：BatchSize=1, Prompt/Decode=1K/1K, 硬件环境：8×NVIDIA B200</figcaption>
</figure>

---

## 核心应用场景

TileRT 适合需要**低延迟、高 TPOT，但并发量不高**的场景，如：

- **高频交易**：毫秒级响应决定交易成败
- **交互式 AI 助手**：提升用户交互体验
- **实时决策系统**：如自动驾驶、具身智能，低延迟保障系统可靠性
- **长时运行 Agent**：需要快速响应的 AI Agent 场景
- **AI 编程助手**：开发者更关心单次请求的响应速度而非吞吐量
---

## 核心设计理念

首先明确：
1. TileLang 是 DSL（Domain Specific Language），而 TileRT 是 runtime。简单来说，DSL 承担"代码表达"，runtime 承担"代码执行"。TileRT 的编译器技术未来将整合到 [TileLang](https://github.com/tile-ai/tilelang) 项目中。
2. TileRT 主要解决的是多设备并行推理中计算通信互相等待的问题，对单卡推理优势不大

TileRT 采用编译器驱动的方法：前端（TileLang）编译器将 LLM 算子分解为可独立调度的 tile 级任务，**在compile-time确定最优调度方案**，随后生成定制化的 runtime 执行这些任务，以高度重叠的方式在**多个**设备间调度计算、I/O 和通信，从而减少等待时间，提高硬件利用率。



### 1. Tile 级细粒度并行（Tile-level Execution）

TileRT 最核心的设计理念是细粒度 tile 级任务执行。与传统执行方式的对比如下：

| 维度       | 传统层级执行                   | TileRT Tile 级执行                     |
| ---------- | ------------------------------ | -------------------------------------- |
| 执行粒度   | 层级算子执行（整层完成后推进） | 细粒度 tile 级任务（拆分至算子子步骤） |
| 执行流程   | 串行：计算 → I/O → 通信        | 重叠：计算 ∥ I/O ∥ 通信（并发执行）    |
| 同步机制   | 设备同步屏障（层间强制等待）   | 异步 tile 调度（满足依赖即执行）       |
| 硬件利用率 | 操作间大量空闲                 | 最大化利用（减少空闲时间）             |
| 任务覆盖   | 仅按层拆分                     | 覆盖 MLA/FFN/MoE 等子任务              |
| 队列管理   | 无专门任务队列                 | 三类队列调度（计算/I/O/通信）          |

#### Warp Group 级任务调度

根据官方 PPT，TileRT 的调度粒度是 **Warp Group**：

```
┌─────────────────────────────────────────────────────────────────────┐
│                      TileRT 引擎设计                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  算子级模型定义：                                                   │
│       ┌──────┐                                                      │
│       │ Q*K  │──→ Norm                                              │
│       └──┬───┘                                                      │
│    GEMM ─┤                     算子拆解为 tile 级任务               │
│          └──→ Indexer          每个任务可以被调度到一个 warp group 上│
│                                                                     │
│  执行时间线：                                                       │
│  ────────────────────────────────────────────────────► 时间         │
│  │ T3 │ T3 │ T3 │ T3 │ T3 │ T3 │                    │ barrier      │
│  ├────┼────┼────┼────┼────┼────┤                    │              │
│  │T0  │T1  │T2  │T0  │T1  │T2  │ ...               │              │
│  │T1  │T2  │T0  │T1  │T2  │T0  │                    │              │
│  └────┴────┴────┴────┴────┴────┘                    │              │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │ WG0  WG1  WG2 │ WG0  WG1  WG2 │ WG0  WG1  WG2 │ ...        │    │
│  │     CTA      │      CTA      │      CTA      │             │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  异构 Worker 虚拟化（warp/block specialization）                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**关键设计**：
- **Warp Group 调度**：每个 tile 任务被调度到一个 warp group 上执行
- **异构 Worker 虚拟化**：通过 warp/block specialization，不同的 warp 专门化执行不同类型的任务（计算、I/O、通信）
- **CTA 内协同**：多个 warp group 在同一 CTA（Cooperative Thread Array）内协同工作

### 2. 三级异步调度队列

TileRT 内部维护了三类独立的硬件队列：
*   **计算队列（Compute Queue）**：负责 Tile 级的矩阵运算。
*   **I/O 队列（I/O Queue）**：负责权重布局转换与内存搬运。
*   **通信队列（Comm Queue）**：负责多卡间的分布式 Partial Sum 传输。

这种设计允许在计算当前 Tile 的同时，通信引擎正在传输上一个 Tile 的结果，从而实现了**计算与通信的完全掩盖（Communication Hiding）**。

### 3. 面向 DeepSeek-V3 的原生优化

TileRT 对 DeepSeek-V3 引入的先进架构进行了专项加速：
*   **MLA (Multi-head Latent Attention)**：针对 MLA 的低秩投影特性，重新设计了 Tile 化读取模式，显著降低了 KV 缓存的访存开销。
*   **Sparse MoE**：优化了 MoE 专家路由后的 Tile 分发逻辑，确保在专家负载不均时仍能保持高效的任务流水。
*   **FP8 适配**：深度集成 FP8 量化算子，在 B200 硬件上充分释放张量核心（Tensor Core）的算力。

### 4. 易于验证的双模式设计

TileRT 支持双执行模式以确保优化过程中的正确性：
*   **Golden 模式**：基于标准 PyTorch 算子的参考实现，用于正确性验证。
*   **TileRT 模式**：基于 `torch.ops.tilert.*` 的高度优化实现，调用底层 Tile 级执行引擎。

### 5. TileLang 与 TileRT 的协作机制

TileRT 的调度能力依赖于 [TileLang](https://github.com/tile-ai/tilelang) 编译器提供的 tile 级任务描述。根据官方 PPT，两者的协作流程如下：

**TileLang 编译流程**（Input Model → Compiler-generated Runtime）：

| 阶段 | 组件                              | 说明                                                                                                                                                          |
| :--- | :-------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1    | Graph Partitioner                 | 将模型图分割为可调度单元                                                                                                                                      |
| 2    | Tile-level Compile-time Scheduler | 分析算子依赖图，确定可并行的任务组；生成 warp group 分配方案和 specialization 策略；确定流水线阶段和 barrier 位置；与 Auto Tuner/Profiler/Cost Model 协同优化 |
| 3    | Task Assemble & Codegen           | 组装任务并生成代码，使用 Tile 级 Micro-kernel 库（模板化 kernel 自动生成）                                                                                    |
| 4    | **Compiler-generated Runtime**    | 最终输出：针对特定模型定制化生成的 TileRT runtime（包含调度逻辑的代码）                                                                                       |

> ✅ **关键：TileRT runtime 是 TileLang 编译器生成的！** 调度框架（哪些任务可以并行、warp 如何分配、依赖关系图）在 compile-time 确定并嵌入到生成的代码中。

#### Runtime（Compiler-generated）的职责

根据 PPT，TileRT 的 Runtime 是**编译器生成的**（Compiler-generated Runtime），而非通用的动态调度器。根据官方公众号文章，runtime 在框架内动态执行（依赖满足即触发，无需等待整层完成）：

| 职责                    | 说明                                       |
| :---------------------- | :----------------------------------------- |
| **执行静态调度**        | 按照 compile-time 确定的顺序执行 tile 任务 |
| **依赖跟踪与异步触发**  | 检查前置任务是否完成，依赖满足时立即执行   |
| **重叠执行**            | 计算/IO/通信并发进行                       |
| **Barrier 同步**        | 在必要的同步点执行 barrier                 |
| **Warp Specialization** | 不同 warp 执行专门化的任务类型             |
| **资源管理**            | 管理 shared memory、寄存器等资源           |

> ✅ **综合理解**：compile-time 生成调度规则，runtime 按规则动态执行。检查逻辑和触发规则是 compile-time 生成的，但执行时机取决于 runtime 的实际依赖完成情况。

#### 未公开的技术细节

由于 TileRT 核心代码仅以 `.so` 形式发布，且官方公众号和 PPT 信息有限，以下细节尚不明确：

| 类别 | 未知内容 |
|:---|:---|
| **任务表示** | Tile 任务的具体数据结构、DAG 格式 |
| **依赖机制** | 依赖检查的具体实现方式（轮询/信号/硬件同步原语） |
| **Warp Specialization** | 如何动态分配不同warp的角色 |
| **Auto Tuner** | 调优参数空间、Cost Model 的具体指标 |
| **Micro-kernel 库** | Kernel 模板的设计、如何针对不同硬件生成代码 |

#### 与 NPU 编译器、传统 GPGPU 的对比

TileLang/TileRT 的设计哲学介于传统 GPGPU 和 NPU 编译器之间，体现了"重编译、轻运行"的趋势：

| 维度 | 传统 GPGPU（CUDA） | TileLang/TileRT | NPU 编译器（如昇腾 CANN） |
|:---|:---|:---|:---|
| **调度决策时机** | Runtime（硬件 warp scheduler） | Compile-time 为主 + runtime 依赖触发 | Compile-time（完全静态） |
| **内存管理** | 程序员手动管理 / 库封装 | 编译期规划 shared memory 布局 | 编译期分配 buffer、规划数据搬运 |
| **硬件映射** | 程序员指定 grid/block/thread | 编译器映射到 warp group | 编译器映射到 AI Core/Cube Unit |
| **编译产物** | PTX/SASS kernel | Compiler-generated runtime | 静态执行图 + 轻量 runtime |
| **Runtime 职责** | Kernel launch + 硬件调度 | 按规则执行，依赖满足即触发 | 按图执行，几乎无决策 |
| **灵活性** | 高（动态 shape、分支） | 中（静态 shape，动态依赖触发） | 低（固定执行路径） |
| **优化负担** | 程序员 + 库开发者 | 编译器 | 编译器 |
| **适用场景** | 通用计算、动态负载 | LLM 推理（静态模型结构） | 固定模型部署 |

**设计哲学**：把复杂性前推到编译期，换取 runtime 的确定性和低开销。TileLang 在 GPU 上实现了类似 NPU 的静态调度优势，同时保留了一定的动态触发能力。



---

## 部署限制与挑战

虽然 TileRT 在延迟上具有代差优势，但其目前仍处于实验阶段，存在以下局限：

1.  **权重预处理开销**：为了实现高效的 Tile 化访问，模型权重必须进行**物理布局转换（Layout Transformation）**。这意味着无法直接加载 standard checkpoints，需要使用项目提供的转换工具。
2.  **硬件生态狭窄**：当前高度依赖 NVIDIA Blackwell (B200) 的硬件特性，对旧代 GPU 或非 NVIDIA 硬件支持有限。
3.  **开发门槛高**：扩展新算子需要深入理解 TileLang DSL 以及 Tile 级的依赖管理，对普通开发者不友好。
4.  **模型通用性**：目前主要针对 DeepSeek-V3/V3.2 系列模型进行深度硬编码优化，尚未实现通用模型的一键适配。
6. **项目状态**：仍为实验性预览版，稳定性和功能覆盖度有待提升

---

## 相关生态项目

| 项目                                                              | 角色定位           | 相互关系                                       |
| :---------------------------------------------------------------- | :----------------- | :--------------------------------------------- |
| **[TileLang](https://github.com/tile-ai/tilelang)**               | 编译器 DSL         | 为 TileRT 提供算子级的 Tile 表达能力           |
| **[TileScale](https://github.com/tile-ai/tilescale)**             | 分布式框架         | 负责多机多卡环境下的策略切分与调度             |
| **[TileLang-Ascend](https://github.com/tile-ai/tilelang-ascend)** | 国产化适配         | 针对华为昇腾 NPU 优化的 TileLang 变体          |
| **TileRT**                                                        | 执行引擎 (Runtime) | 负责将编译器生成的 Tile 任务高效落地到硬件执行 |

---

## 参考资料

*   **GitHub**: [tile-ai/TileRT](https://github.com/tile-ai/TileRT)
*   **官方公众号文章**: [性能远超 vLLM 和 SGLang！TileRT：编译器驱动下的 Tile-Based Runtime](https://mp.weixin.qq.com/s/5T-93n5kk7UbHXj_I3NIvw)
*   **技术分享 PPT**: [小红书 - TileRT 引擎设计](https://www.xiaohongshu.com/explore/694f9f33000000002103d560)
*   **TileLang 论文**: [TileLang: A Composable Tiled Programming Model for AI Systems](https://arxiv.org/abs/2504.17577)
*   **TileLang 文档**: [tilelang.com](https://tilelang.com/)
*   **TileScale**: [tile-ai/tilescale](https://github.com/tile-ai/tilescale) - 分布式 tile 级通信框架
*   **Models**: [DeepSeek-V3.2-Exp Optimized Weights](https://huggingface.co/collections/Tile-AI/tilert)
*   **CUDA 异步执行**: [CUDA Programming Guide - Asynchronous Execution](https://docs.nvidia.com/cuda/cuda-programming-guide/02-basics/asynchronous-execution.html)
*   **相关研究**: [Mirage Persistent Kernel](https://arxiv.org/abs/2512.22219) - SM 级任务图与 in-kernel 调度