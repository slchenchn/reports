<!-- 写作目标：梳理 LLM 推理服务限流策略，重点是 reject（429）而非排队，面向内部部署 AI coding agent 场景
写作约束：
- 不解释 TTFT、TPOT、E2E latency 的定义
- 排队由 vLLM 完成，本文只讨论网关层硬 reject 新请求
- 首版只做全局限流，不做 per-user
- 外部参考只看提供 Coding Plan 的厂商
- 策略核心是基于 LiteLLM 实时观测 TTFT/TPOT 异常后触发全局 reject
- 不讨论错误码、客户端约定、灰度方案和工程实现 -->

# LLM 推理服务限流策略

<!-- ## 问题定义

排队是 vLLM scheduler 的事。请求进入 vLLM 后会在 waiting queue 里等 KV cache 空位，这层我们不动

我们要解决的是上一层的 admission control：**当系统体验已经开始恶化时，网关还要不要继续把新请求送进 vLLM**

答案是不要。继续喂请求只会拉长 queue、拖慢所有在线请求。正确做法是在网关层硬 reject，截断 queue 增长

vLLM 自身也没有成熟的 reject 机制。[Issue #13395](https://github.com/vllm-project/vllm/issues/13395)（2025-02）提出 busy 时 reject request，被 `Closed as not planned`；[RFC #18826](https://github.com/vllm-project/vllm/issues/18826)（2025-05）提出 waiting queue 上限，至今未合并 -->

## 定义
本文讨论的是基于http 429这种硬拒绝的限流策略，而不是基于排队等待的限流策略。

## 外部参考
[阿里云百炼](https://help.aliyun.com/zh/model-studio/coding-plan)、[Kimi](https://platform.moonshot.cn/)、[智谱](https://zcode.z.ai/docs/configuration)、[火山方舟](https://www.volcengine.com/article/37554)、[MiniMax](https://platform.minimaxi.com/docs/guides/pricing-codingplan) 这几家的 coding plan，基本没有给出数值化的 TTFT / TPOT 承诺，只有Kimi-K2-turbo（比基础版消耗更大）写了输出速度 `60~100 Token/s`。


## 我们的策略

### 核心思路

1. 先只做全局限流，不做 per-user，现阶段没有成熟可复用的 per-user 做法
2. 限流不依赖预先压测得到的并发阈值，而是基于实时观测的 TTFT、TPOT、latency等指标，动态调整限流阈值

### 触发条件

观测 60 秒滑动窗口内的 p95 值，任一指标越过阈值即触发保护：

| 指标     | 异常阈值 | 依据                                                                        |
| -------- | -------- | --------------------------------------------------------------------------- |
| p95 TTFT | > 20s     |    |
| p95 TPOT | > 200ms  |  |

为什么用 p95 而不是均值：均值对少量极端值不敏感，p95 能捕捉到"已经有不少请求开始变慢"的信号

为什么用 60 秒窗口：太短（如 10s）容易因为单个大请求误触发；太长（如 5min）反应迟钝，保护不及时

### 恢复条件

触发保护后，不能在指标刚掉到阈值以下就立刻恢复放流——否则会在保护态和正常态之间高频抖动（刚恢复就涌入新请求，指标又恶化，再触发保护）

采用 **滞回（hysteresis）机制**：

- **进入保护态**：60s 窗口 p95 TTFT > 20s 或 p95 TPOT > 200ms
- **退出保护态**：60s 窗口 p95 TTFT < 10s **且** p95 TPOT < 100ms，**持续 30s**

也就是说，恢复阈值是触发阈值的约 60%，且需要稳定保持 30 秒。这保证系统真正恢复健康后才重新放流，而不是刚从过载中喘过气就被新流量打回去

### 保护态下的行为

正常时：请求正常进入 LiteLLM → 转发到 vLLM → vLLM 内部排队和调度

保护态时：
- 已在 vLLM queue 中的请求继续排队和执行
- 新到达网关的请求直接 reject（429），不送入 vLLM
- 持续监控指标，等满足恢复条件后退出保护态


## 阈值总表

| 参数          | 值           | 说明                                |
| ------------- | ------------ | ----------------------------------- |
| 观测窗口      | 60s  | 平衡灵敏度和稳定性                  |
| 统计口径      | p95          | 捕捉尾部劣化，对均值噪声不敏感      |
| TTFT 触发阈值 | 20s           | agent 场景体验恶化边界              |
| TPOT 触发阈值 | 200ms        | 长输出场景下体验不可接受边界        |
| TTFT 恢复阈值 | 10s           | 触发阈值的 60%                      |
| TPOT 恢复阈值 | 100ms         | 触发阈值的 60%                      |
| 恢复持续时间  | 30s          | 低于恢复阈值需持续 30s 才退出保护态 |
