# 基于 Subagent 的量化-校验-部署-评测全流程自动化实践

## 1. 核心概念：什么是 Subagent

**Subagent** 是一种架构模式，它将一个庞大复杂的 Agent 拆分为多个职责清晰、上下文隔离、且可并行协作的subagent。
这些 Subagent 由一个主 Agent 统一调度，**且每个agent有独立的上下文**，从而避免上下文污染，提升复杂任务的稳定性与工程化水平，同时有效降低 Token 成本。

![Subagent 架构示意图](https://contentstatic.techgig.com/thumb/msid-123923017,width-800,resizemode-4/Subagents-in-AI-The-Key-to-Smarter-Reliable-Systems.jpg?24456)

### Subagent vs. 单一 Agent (Monolithic)

| 维度 | Subagent 架构 | 单一 Agent (Monolithic) |
| :--- | :--- | :--- |
| **关注点分离** | ✅ **强**：每个 Subagent 职责明确，专注单一领域 | ❌ **弱**：推理、决策、执行逻辑混杂，容易顾此失彼 |
| **上下文污染控制** | ✅ **优秀**：Subagent 上下文隔离，仅回传结构化结果，保持主上下文纯净 | ❌ **差**：历史推理与临时草稿堆积，长对话后容易出现幻觉或遗忘 |
| **长任务可扩展性** | ✅ **高**：天然适合 Multi-step / Multi-role 任务，通过分治策略处理复杂流 | ❌ **低**：任务链路越长，上下文越长，性能退化越明显 |
| **并行潜力** | ✅ **天然支持**：搜索、扫描、Diff 等独立任务可并行执行 | ❌ **受限**：本质上是单线程推理，难以并发 |

---

## 2. 场景与动机：为什么选择 Agent 驱动量化与评测

相比于工作流，引入 Agent 主要解决了以下痛点：

1.  **灵活适配**： 传统脚本难以处理动态需求，例如临时跳过某些层的量化、动态调整部署超参、或适配新模型结构。
2.  **模块解耦**： 量化、部署、评测各环节天然弱耦合。它们不需要共享全部上下文，只需传递部分关键结果（如输出路径）

---

## 3. 全流程实践记录

### 3.1 RTN 量化

#### 原prompt
执行RTN量化，执行量化的示例：
```bash
bash bash scripts/csl_run.sh configs/csl/RTN/rtn.moonlight-16B-A3B.nvf4.w4a4.mse.only-mlp.ultrachat.yml
```

- config文件： configs/csl/RTN/
- project目录：/nfs/FM/chenshuailin/code/llmc/

执行量化时，各种命名要遵循已有的风格
环境使用 ${project}/.venv

### 3.2 校验

#### 原prompt
project目录：/nfs/FM/chenshuailin/code/llmc/
检查量化后的模型是否符合预期，参考代码：
- tools/nvfp4/validate_realquant.py
- tools/fp8/deepseek/validate_realquant.py

环境使用 ${project}/.venv

### 3.3 模型部署

#### 原prompt
目的：部署模型到目标机器。
project目录：/nfs/FM/chenshuailin/code/llmc/
部署模型的机器需要找用户要。
默认使用vllm部署，将vllm部署脚本复制到模型目录下，然后执行部署脚本。
- vllm部署脚本：scripts/vllm/run_vllm.sh
- sglang部署脚本：scripts/sgl/run_sgl.sh
一半情况下，根据可用显存和模型尺寸，需要自行决定张量并行尺寸，例如，32B的模型，NVFP4量化，则逛模型权重大概需要16GB。而总显存是权重的4倍会比较好，因此32Gx2（tp2）的显存足够部署。

不同机器间的磁盘是共享的，因此不需要scp模型权重。

#### 运行结果
*   **智能参数调整**：虽然规则建议 TP2，但 Agent 在检测到运行时报错后，主动尝试并成功切换到了 TP1 部署，展现了良好的错误恢复能力。

### 3.4 测试

#### 原prompt
目的：基于opencompass对部署的模型进行测试
project目录：/nfs/FM/chenshuailin/code/opencompass_0303_bk/
示例测试脚本：tools/2512/instruct_fullv2/moonlight-16B-A3B.fullv2.nvf4.w4a4.mlp-only.sh
示例关键文件：
- 总config：examples/eval_0303_fullv2_moonlight_it.py
- 模型config：opencompass/configs/models/openai/moonlight.py

需要用tools/minimal_reasoning_api_client.py获取部署的模型名称，然后修改模型config里的path变量和api_base，再启动测试
各种命名要遵循已有风格

环境使用 ${project}/.venv

---

## 4. 总结与反思

### 优点
1.  **鲁棒性提升**：Agent 展现出了意料之外的纠错能力，例如自动处理 vocab size 不匹配问题。
2.  **动态决策**：在部署参数（如 TP 设置）不合理导致失败时，能够自主尝试降级方案（TP2 -> TP1），无需人工干预。

### 存在问题
1.  **Subagent 协作衔接**：Subagent 之间的上下文传递和任务切换偶尔不够顺滑。
2.  **指令遵循度**：部分 Subagent 可能会忽略其特定的 System Prompt 约束，需要进一步优化 Prompt 工程或框架限制。
