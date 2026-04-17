<!-- 写作目标：梳理 LLM 推理服务限流策略，重点是 reject（429）而非排队，面向内部部署 AI coding agent 场景 -->

# LLM 推理服务限流策略

## 为什么要限流

限流的核心目标不是省算力，是**保体验**。LLM 推理的关键体验指标：

- **TTFT (Time To First Token)**：用户发请求到看到第一个 token 的时间，直接决定"卡不卡"的感知
- **TPOT (Time Per Output Token)**：每个输出 token 的生成间隔，决定流式输出的流畅度
- **E2E Latency**：端到端延迟，从请求发出到完整响应结束

当并发超过系统承载能力时，所有在线请求的 TTFT 和 TPOT 都会劣化——KV cache 压力增大、prefill 和 decode 互相抢占、调度队列堆积。这时候与其让所有人一起慢，不如拒掉一部分请求（返回 429），保住剩余请求的体验质量

## 排队 vs Reject

这两件事在架构上分属不同层：

| 机制 | 实现层 | 行为 | 适用场景 |
| --- | --- | --- | --- |
| 排队 (Queuing) | 推理引擎内部（vLLM scheduler） | 请求进入 waiting queue，等 KV cache 有空位后调度 | 短暂的负载波动，用户愿意多等几秒 |
| 拒绝 (Reject 429) | API Gateway / 中间件 | 直接返回 HTTP 429，不进入推理引擎 | 系统已过载，继续排队只会让所有人体验崩溃 |

vLLM 自身不提供原生的 429 限流能力，它的设计哲学是"你给我多少请求我就排多少"。所以 reject 必须在 vLLM 前面的 gateway 层实现

本文关注的是 **reject 策略的设计**——什么时候拒、拒谁、拒多少

## 业界实践

### OpenAI / Anthropic

这两家都不承诺延迟 SLA，做的是**流量管控**而非体验保障：

- OpenAI 按 RPM（Requests Per Minute）+ TPM（Tokens Per Minute）双维度限流
- Anthropic 更细一层，把 TPM 拆成 ITPM（Input Tokens Per Minute）和 OTPM（Output Tokens Per Minute）分别限流——因为 prefill 和 decode 消耗的算力完全不同
- 两家都不公开承诺 TTFT/TPOT 的具体数值

对我们的参考价值有限。我们的场景是自己掌控算力，目标是保延迟体验，不是控流量

### GitHub Copilot / Cursor

2025 年中都转向了 **credit pool 模式**：

- 按月分配 token 消费额度，不同模型消耗不同 credit
- 超额后降级到低成本模型（Auto 模式），而非硬拒
- 本质是成本控制手段，不涉及延迟保障

对我们的参考价值同样有限——我们不做计费，做体验

### 阿里云百炼

阿里的方案按服务等级分了三档，**有明确的延迟 SLA**：

| SKU | TTFT | TPOT |
| --- | --- | --- |
| InferX Base | < 2s | < 70 ms |
| InferX Pro | < 1.5s | < 50 ms |
| InferX Max | < 1s | < 30 ms |

高并发场景推荐购买**预置吞吐（Provisioned Throughput）**，独占资源不受公共池限流影响

**核心思路**：用分层 SLA 定义"好体验"的标准，通过资源隔离保障

### 腾讯混元

- 默认单账号并发限制 **5 路**
- 性能目标：TTFT < 2s，TPOT < 50 ms

**核心思路**：直接限并发数，简单粗暴但有效

### 小结

| | OpenAI / Anthropic | Copilot / Cursor | 阿里 / 腾讯 |
| --- | --- | --- | --- |
| 本质 | 流量管控 | 成本控制 | 体验保障 |
| 延迟 SLA | 无 | 无 | 有 |
| 限流指标 | RPM + TPM | credit pool | 并发数 / 吞吐配额 |

我们的场景（公司内部部署开源 LLM 做 coding agent）最接近阿里/腾讯——**需要承诺体验**，而不是控流量或控成本

## 我们的限流策略设计

### 场景特点：Coding Agent

我们的用户场景是 Claude Code 这类 coding agent，而非代码补全。这决定了几个关键假设：

- **请求频率不高**：agent 一轮对话可能生成几千 token 的计划、再执行工具调用、再生成代码，单个 session 的请求间隔在秒级到十秒级
- **单次请求很重**：agent 的 context window 使用率很高，经常带着完整文件内容 + 对话历史，input token 量大（几千到几万），output token 量也大
- **对 TTFT 的容忍度较高**：用户发完指令后在等 agent 思考，2-3 秒的 TTFT 可以接受
- **对 TPOT 有一定要求**：agent 输出通常较长（代码+解释），如果 TPOT 太高，几千 token 的输出要等很久
- **并发度可预估**：公司开发者数量有限，同时高峰在工作时间，流量模式可预测

### 延迟 SLA 目标

参考阿里 InferX Base 和腾讯的数据，结合 coding agent 对延迟的实际容忍度：

| 指标 | 目标值 | 依据 |
| --- | --- | --- |
| TTFT (p95) | < 3s | agent 场景用户可以等几秒，阿里 Base 档也是 < 2s，我们宽松一些 |
| TPOT (p95) | < 50 ms | 对标腾讯和阿里 Pro 档，保证 1000 token 输出在 50s 内完成 |

注意是 **p95** 而非均值。均值达标但 p95 爆炸 = 每 20 个请求有一个体验很差

### 限流维度

采用 **两层限流**：

**第一层：Per-user 限流**——保公平性，防止少数用户挤占所有算力

| 维度 | 指标 | 原理 |
| --- | --- | --- |
| 请求频率 | RPM per user | 防脚本循环调用、IDE 插件 bug 导致的重复请求 |
| Token 吞吐 | TPM per user（input + output） | 一个满 context window 的请求算力消耗可能是短请求的 50 倍，光限 RPM 不够 |

**第二层：全局限流**——保系统稳定性，当整体负载超过承载能力时做 load shedding

| 维度 | 指标 | 原理 |
| --- | --- | --- |
| 系统负载 | 全局并发数 / vLLM queue depth | 不论单用户是否超限，系统过载时必须拒绝新请求保护在线请求体验 |

两层的触发关系是 **OR**：任意一层触发就 reject

### 限流算法

**Per-user 限流用 Token Bucket**

Token bucket 的核心参数是 `rate`（令牌补充速率）和 `burst`（桶容量/突发上限）:

- 桶以固定速率 `rate` 持续补充令牌
- 每次请求消耗对应数量的令牌（RPM 桶每请求消耗 1，TPM 桶消耗该请求的 token 数）
- 桶满后多余令牌丢弃
- 桶空时拒绝请求

这个算法的好处是天然允许突发——开发者不会匀速发请求，可能连续发 3-4 个请求后休息一会儿。bucket 的 `burst` 参数控制突发上限，`rate` 控制持续速率

Token bucket 相比 fixed window 没有窗口边界的双倍流量问题，相比 sliding window 实现更简单且语义更直观

**全局限流用 Queue Depth 阈值**

直接读 vLLM 暴露的 Prometheus metrics（`vllm:num_requests_running` + `vllm:num_requests_waiting`），超过设定阈值就 reject。这个方法最直接地反映系统真实负载，不需要预估 capacity

### 限流阈值

阈值需要通过压测标定，但可以给出设定方法和参考范围

#### 全局并发阈值

**方法**：在目标硬件上跑不同并发数的 benchmark，记录 TTFT/TPOT 随并发的变化，找到延迟 SLA 被突破的拐点

一个典型的模式是：低并发时 TTFT/TPOT 几乎不变，超过某个拐点后急剧恶化（因为 KV cache 开始紧张、scheduling 开始排队）

```
并发数:     1    4    8   12   16   20   24   28   32
p95 TTFT: 0.3  0.4  0.5  0.8  1.2  2.0  3.5  6.0  12.0  (秒)
                                         ↑
                                    拐点在这里
```

全局并发上限 = 拐点并发数 × 80%（留 buffer），上例中拐点 ~20，上限设 16

#### Per-user RPM

Coding agent 场景下单用户的正常 RPM 很低：

- agent 一次 tool call chain 可能 3-5 个 LLM 请求，间隔几秒
- 一分钟内一个用户正常使用大约 5-15 个请求
- 设 **RPM = 30**，是正常使用的 2-3 倍，主要防 bug 和滥用

Token bucket 参数：`rate = 30/min = 0.5/s`，`burst = 10`（允许瞬间最多 10 个请求）

#### Per-user TPM

这个和模型的 context window 强相关：

- 假设模型 context window 128k token
- Coding agent 一次请求的典型 token 量：input 10k-50k（代码文件+对话历史），output 1k-5k
- 一次请求平均消耗约 20k-30k token
- 一分钟内正常使用 5-10 个请求 ≈ 100k-300k token

设 **TPM = 500k**，覆盖绝大多数正常使用，少数几个满 context 请求也能容纳

Token bucket 参数：`rate = 500k/min ≈ 8.3k/s`，`burst = 150k`（允许突发约 3 个大请求）

#### 阈值总表

| 维度 | 参数 | 建议值 | 说明 |
| --- | --- | --- | --- |
| 全局并发上限 | max concurrent requests | 需压测标定 | 取 p95 TTFT < 3s 的拐点 × 80% |
| Per-user RPM | rate | 30/min | 正常使用的 2-3x |
| Per-user RPM | burst | 10 | 允许短暂突发 |
| Per-user TPM | rate | 500k/min | 覆盖高强度使用 |
| Per-user TPM | burst | 150k | 约 3 个大 context 请求 |

以上数值是 **起步值**，需要上线后根据真实使用数据调优：
- 429 rate > 5%：阈值太紧，影响开发者生产力，应放宽
- 429 rate < 0.1% 且延迟 SLA 经常被突破：阈值太松，应收紧

### 降级策略（可选）

除了直接 429 拒绝，还可以考虑更柔性的降级：

- **模型降级**：系统过载时自动路由到更小的模型（如从 70B 切到 7B），保证有响应但质量略降
- **优先级差异化**：区分不同团队或使用场景的优先级，过载时先拒低优先级请求

这些属于 graceful degradation，初期可以先做硬拒（429），后续再迭代

## 待定事项

- [ ] 具体模型 + 硬件配置下的并发-延迟曲线需要实测，确定全局并发上限
- [ ] Per-user RPM/TPM 阈值上线后需根据真实 429 rate 和延迟数据迭代
- [ ] 是否需要按团队/角色做差异化优先级
- [ ] 限流策略灰度方案——先 shadow mode（只记录不拒绝）观察，再逐步开启