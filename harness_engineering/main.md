<!--
1. 如果图片路径不存在，那表示是这是一个placeholder，如果你是Gemini，需要你生成图片放到对应的位置上
2. harness engineering是什么，怎么演进的，作用是什么，如何使用，未来如何发展
3. 不能做成枯燥的列表/流水账，要娓娓道来，但不能变成纯讲故事，略微有故事性即可

-->

# Harness Engineering 调研报告

## 背景与演化
### 概念提出
Harness Engineering 的概念最早由 Mitchell Hashimoto 在博客 [My AI Adoption Journey](https://mitchellh.com/writing/my-ai-adoption-journey) 中正式提出，他将其定义为：
> anytime you find an agent makes a mistake, you take the time to engineer a solution such that the agent never makes that mistake again

OpenAI 随后在其 [实践文章](https://openai.com/index/harness-engineering/) 中展示了完全依赖 Agent (Codex) 零手写代码完成百万行系统重构的极端案例，将这一概念从个人实践系统化为工程范式：

| 指标         | 核心数据                    |
| ------------ | --------------------------- |
| 周期与人力   | 5 个月，3-7 人团队          |
| 代码与产出   | ~100 万行代码，~1500 PR     |
| 吞吐量与收益 | 3.5 PR/人/天，效率提升 ~10x |

这种开发模式将工程重心从"直接编写业务逻辑"转移到"构建基础设施"：
- 设计高容错的运行环境
- 构建闭环反馈回路
- 建立确定性的架构约束

### 范式演进
范式的演进本质上反映了模型能力的溢出，人类微操干预的必要性降低：
- **Prompt Engineering**：微操输入，要求模型一次性输出正确结果
- **Context Engineering**：知识路由，提供外部挂载点以丰富模型视野
- **Harness Engineering**：系统级约束与自动化执行，关注环境流转，最终达到 Humans steer, agents execute

三者之间是叠加而非替代关系：Prompt Engineering 用于探索，Context Engineering 用于对齐，Harness Engineering 用于自主运行

<figure style="text-align:center;">
  <img src="harness_engineering_evolution.png" alt="工程范式演进：微操代价上升，系统约束接管" style="width:80%;" />
  <figcaption>工程范式演进：微操代价上升，系统约束接管</figcaption>
</figure>

## 核心定义

Harness Engineering 是一种纯 Agent 驱动的软件工程范式
核心是将 Agent 视作计算核心 (Model ≈ CPU)，而 Harness 则是提供上下文调度、异常捕获和状态流转的操作系统 (Harness ≈ OS/Runtime), Agent = Model + Harness
工程师的工作流随之改变，角色彻底转变为环境架构师和策略制定者

[Martin Fowler / Birgitta Bockeler (ThoughtWorks)](https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html) 进一步用控制论 (Cybernetics) 对 Harness 进行了理论化：Harness 本质上是一个控制论调节器 (cybernetic governor)，由前馈控制 (Guides) 和反馈控制 (Sensors) 组成。这映射了 Ashby 的必要多样性法则 (Law of Requisite Variety)：调节器必须拥有与被调节系统至少同等多样性的控制手段

## OpenAI 的五条原则

OpenAI 在百万行项目中提炼出五条核心原则，它们构成了 Harness Engineering 最具实操性的指导框架：

### 1. "What the Agent Can't See Doesn't Exist"

所有决策必须以 Markdown、Schema 和 ExecPlan 的形式推入仓库。AGENTS.md 保持在约 100 行，作为索引路由指向更深层的信息源。在此框架下，Codex 单次运行可持续超过 7 小时——这只有在上下文完整且稳定时才能实现

### 2. "Ask What Capability Is Missing, Not Why the Agent Is Failing"

当 Agent 产出错误时，不归咎于模型能力不足，而是将每个失败重新框架为环境缺陷。团队构建了带 OpenTelemetry 集成的自定义并发助手，而非引入外部依赖——偏好"无聊技术" (boring technology)，即 API 稳定、训练数据覆盖充分的技术栈，可预测性优先于精巧性

### 3. "Mechanical Enforcement Over Documentation"

固定分层领域结构 (Types → Config → Repo → Service → Runtime → UI) 并通过自定义 Linter 进行依赖验证——Linter 本身也由 Codex 编写。效果是：Agent 即使在长时间无人值守运行中也无法意外违反结构规则

### 4. "Give the Agent Eyes"

将 Chrome DevTools Protocol 接入 Agent 运行时，提供 DOM 快照、截图和导航能力。接入 Victoria Logs (LogQL) 和 Victoria Metrics (PromQL) 实现可观测性查询。Vector 作为 fan-out 路由器驱动整个可观测性栈。每个 git worktree 创建临时可观测性栈，任务完成后销毁。这使得可以下达诸如"确保服务启动在 800ms 以内完成"、"关键用户旅程中没有 span 超过 2 秒"等量化指令

### 5. "A Map, Not a Manual"

ARCHITECTURE.md 捕捉结构和边界。架构不变量以排除方式表达 ("这里不存在某物") 而非规定方式。团队尝试过"巨型 AGENTS.md"方案，失败了——精简的路由映射才是有效的

**ExecPlans：可重启的任务契约**

ExecPlan 是定义在 PLANS.md 中的自包含设计文档。通过标准是"初学者应该能读完它并端到端地实现功能"。ExecPlan 是活文档——应该始终可以仅从 ExecPlan 出发重新启动工作，不依赖任何其他状态。这对 Agent 跨上下文重置的可靠性至关重要

## 落地支柱

### Context Engineering (上下文工程)
拒绝向 Agent 倾倒全局代码，采用"渐进式披露"策略
根目录的 `AGENTS.md` 等文档仅作为轻量级索引路由，具体模块规则分散在 `docs/` 目录，按需加载，避免 Context 污染和冗余 Token 消耗

[ThoughtWorks](https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html) 对控制手段做了进一步分类：

|                  | 前馈 (Guides)           | 反馈 (Sensors)                 |
| ---------------- | ---------------------- | ----------------------------- |
| **计算型** (确定性) | LSP 集成、MCP Server、架构文档 | ESLint、Semgrep、类型检查器、覆盖工具 |
| **推理型** (概率性) | 为 LLM 优化的 Linter 消息   | LLM-as-Judge 语义重复检测、过度工程检测 |

### Architectural Constraints (架构约束)
用强类型、依赖注入和物理隔离替代 Prompt 中的软约束
例如强制执行 Types → Config → Repo → Service → Runtime → UI 的单向依赖，在 CI 阶段直接阻断 Agent 的越权调用，而非寄希望于模型遵守文档口头约束

Birgitta Bockeler 提出了"环境可供性" (Ambient Affordances) 概念：强类型语言、清晰的模块边界和成熟的框架天然创造了这些可供性——没有它们，某些控制手段根本无法构建

### Feedback Loops (反馈循环)
将 Code Review、单元测试和 Lint 检查自动化并接入 Agent 的自迭代循环
借鉴类似 Claude Code 的 Hook 机制，使 Agent 每次改动都能立即获得确定性的失败信号并自行触发修正

反馈速度分层：

| 层级              | 响应时间   | 示例                      |
| ----------------- | --------- | ------------------------ |
| PostToolUse Hook  | 毫秒级    | 即时格式检查、安全扫描       |
| Pre-commit Hook   | 秒级      | Lint、类型检查              |
| CI Pipeline       | 分钟级    | 集成测试、依赖验证           |
| Human Review      | 小时-天级  | 架构审查、产品判断           |

### Entropy Management (熵管理)
高频度的 Agent 提交极易积累"AI 残渣"和冗余抽象
需要引入定期的自动重构流水线 (Garbage Collection)，将共性的最佳实践固化为 Lint 规则或底层基类，防止技术债雪崩

OpenAI 的经验证明了这一点：他们最初尝试将每周五作为手动清理日，发现消耗了 20% 的工程时间却跟不上生成速度。最终转向周期性的后台清理任务——Agent 扫描偏离"黄金原则"的代码，更新质量评级，并自动开启针对性的重构 PR

## Mitchell Hashimoto 的六阶段采纳框架

Hashimoto 基于自身从 Ghostty 开发中获得的经验，提出了一个渐进式的 AI 采纳路径：

| 阶段 | 名称 | 核心行为 |
|------|------|----------|
| 1 | Drop the Chatbot | 放弃聊天界面，转向有文件读写和程序执行能力的 Agent |
| 2 | Reproduce Your Own Work | "Double-step"技术：先手动完成任务，再让 Agent 在不看人类代码的情况下复现 |
| 3 | End-of-Day Agents | 在工作日最后 30 分钟启动 Agent 做深度调研、探索和 PR 分类 |
| 4 | Outsource the Slam Dunks | 将高确定性任务委派给 Agent，同时关闭桌面通知避免上下文切换 |
| 5 | Engineer the Harness | 将每一个 Agent 错误编码为系统性预防措施 |
| 6 | Always Have an Agent Running | "If I'm coding, I want an agent planning. If they're coding, I want to be reviewing." |

**Ghostty 案例：** 他在 [Ghostty](https://mitchellh.com/writing/non-trivial-vibing) 上用 16 个 session、3 个日历日、$15.98 token 成本实现了一个非平凡特性。AGENTS.md 中的每一行都对应一个过去的 Agent 错误——"it almost completely resolved them all"

**关键洞察——View Model 质量决定 Agent 效能：** "The cleanliness of a UI frontend and business logic backend is often dictated by the quality of the view model in between." 手动重构接口（如切换到 tagged unions、重命名类型）能大幅提升后续 Agent 在前后端工作上的表现。干净的接口让 Agent 能力倍增

## 行业实践案例

### LangChain：纯 Harness 改进，13.7 个百分点提升

[LangChain](https://blog.langchain.com/improving-deep-agents-with-harness-engineering/) 在 Terminal Bench 2.0 上将得分从 52.8% 提升到 66.5%——从 Top 30 开外跃升至 Top 5——模型 (GPT-5.2-Codex) 零更换，仅改变 Harness。关键技术：

- **Reasoning Sandwich**：planning 阶段使用 xhigh reasoning，implementation 阶段降至 high，verification 阶段回升至 xhigh。纯 xhigh 反而只有 53.9%（因超时）
- **LoopDetectionMiddleware**：追踪每个文件的编辑次数，超过阈值后触发"consider reconsidering your approach"
- **PreCompletionChecklistMiddleware**：在 Agent 退出前拦截，强制其对照任务规格运行验证
- **Trace Analyzer Skill**：自动从 LangSmith trace 中衍生并行错误分析 Agent，类似 ensemble boosting

这是"Harness 才是护城河，而非模型"最有力的量化证据

### Vercel d0：从 15 个工具到 2 个，成功率从 80% 到 100%

[Vercel](https://vercel.com/blog/we-removed-80-percent-of-our-agents-tools) 将其 text-to-SQL Agent 从 15+ 个精密工具 (GetEntityJoins, LoadCatalog, RecallContext 等) 精简为 2 个 (ExecuteCommand + ExecuteSQL)：

| 指标     | 15+ 工具方案 | 2 工具方案 |
| -------- | ----------- | --------- |
| 平均耗时  | 274s        | 77s       |
| 成功率    | 80%         | 100%      |
| Token 消耗 | ~102k      | ~61k      |

核心教训："We were constraining reasoning because we didn't trust the model to reason." 过度工具化反而限制了模型的推理能力

### Anthropic：三 Agent GAN 架构

[Anthropic 研究团队](https://www.anthropic.com/engineering/harness-design-long-running-apps)测试了一个受 GAN 启发的 Planner-Generator-Evaluator 架构：

- **Solo Agent**：20 分钟，$9——核心功能损坏
- **Full Harness**：6 小时，$200——交付完善、功能丰富的应用

Generator 和 Evaluator 在每个 Sprint 前协商完成标准 (sprint contracts)。Evaluator 通过 Playwright 进行主动测试，实际点击运行中的应用。关键发现：当 Opus 4.6 展现出更强的规划能力后，团队移除了 sprint 结构而保留 planner/evaluator 角色——Harness 的设计空间不会随模型进步而缩小，而是发生位移

### Anthropic Ralph Loop：长任务双 Agent 模式

用于长时间运行工作的轻量双 Agent 模式：Initializer Agent 设置环境（init script、progress file、feature list、初始 commit），Coding Agent 在后续每个 session 中读取 git log 和 progress file 来自我定位，选择最高优先级的未完成 feature 工作。使用 JSON feature list 因为"model is less likely to inappropriately change or overwrite JSON files"——文件系统作为跨上下文重置的持久记忆

## 失败模式与反模式

### Context 退化 ("Dumb Zone")

性能随上下文长度增加而可测量地退化——即使在简单任务上。不相关的 grep 结果和工具调用成为累积性干扰物。[HumanLayer](https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents) 提出的对策：Sub-agent 作为"上下文防火墙" (context firewall)，父 Agent 只接收带源引用的浓缩响应

### 过度规格化悖论

[ETH Zurich 对 138 个 agentfile 的研究](https://arxiv.org/html/2603.25723v1)发现：
- LLM 生成的 agentfile 对性能有**负面**影响——生成内容倾向于泛泛而谈的最佳实践，缺乏针对具体代码库的关键约束，反而稀释了有效信号
- 人工编写的文件在设计不佳时也仅带来约 4% 的提升——设计不佳指缺乏可执行性的描述性文档，Agent 读了但无法据此做出更好的决策
- 目录列表：零收益——Agent 已经可以通过文件系统工具自行探索目录结构，静态目录列表只是重复信息且会随代码库变化而过时
- Agent 处理指令时多消耗 14-22% 的推理 token，但解决率没有提升——指令增加了 Agent 需要内化的信息量，但这些信息对实际任务分解和执行没有提供增量价值
- 过度的工具引导反而导致更差的结果——强行规定 Agent 使用特定工具序列会剥夺其根据运行时状态灵活调整策略的能力，等于用编排脚本的思路约束了一个推理系统

### 全局一致性缺失

[Eric Mann](https://eric.mann.blog/the-agentic-harness-problem-why-ai-agents-need-better-guardrails-than-code-reviews/) 在构建 `tss-ceremony` 时遭遇了典型失败：Agent 在隔离组件上交付了精美、正确的工作，但核心签名仪式（场景 5-11）从未被构建——P3 优先级的额外功能闪闪发光，而核心价值主张仍是 placeholder stub

根因："They optimized for local completeness without tracking global coherence." 这与初级工程师的行为模式惊人地一致

对策：里程碑完成门控 (milestone completion gates)、生产者-消费者层间的显式数据契约、"先连线后打磨" (wiring before polish) 原则

### 自我评估偏差

[Anthropic 发现](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) Agent 评估自己的工作时"tend to respond by confidently praising the work -- even when, to a human observer, the quality is obviously mediocre"。解决方案：独立的评估 Agent，配合 few-shot 示例和详细的评分细则，校准为偏怀疑态度

### 测试套件中毒

运行完整 5+ 分钟的测试套件导致 4,000+ 行通过测试的信息在 Agent 的上下文中产生幻觉。解决方案：仅运行与当前任务相关的过滤测试

### Harness 过拟合

模型对特定 Harness 的后训练会导致在其他 Harness 中表现不佳。在 Terminal Bench 2.0 上，Opus 4.6 在 Claude Code Harness 中排名 #33，但在陌生 Harness 中排名 #5，位差达 ±4。这意味着模型评估在一定程度上是 Harness 评估

### Harness 漂移

随着 Harness 增长，Guides 和 Sensors 逐渐失步。非确定性控制更难测试。指令与反馈信号之间出现矛盾。开放性问题："If sensors never fire, is that a sign of high quality or inadequate detection mechanisms?"

## 实践经验与反思

### 文件架构图
```
AGENTS.md                               # Agent 行为约束与协作入口（宜精简，链到 docs）
ARCHITECTURE.md                         # 系统分层与模块边界总览
docs/                                   # 渐进式披露：详细文档根目录
├── design-docs/                        # 设计决策与方案沉淀
│   ├── index.md
│   ├── core-beliefs.md
│   └── ...
├── exec-plans/                         # 可执行计划（任务拆解与跟踪）
│   ├── active/
│   ├── completed/
│   └── tech-debt-tracker.md
├── generated/                          # 由工具/流水线自动生成的文档
│   └── db-schema.md
├── product-specs/                      # 产品功能与体验规格
│   ├── index.md
│   ├── new-user-onboarding.md
│   └── ...
├── references/                         # 外部资料压缩版，供 Agent 快速检索
│   ├── design-system-reference-llms.txt
│   ├── nixpacks-llms.txt
│   ├── uv-llms.txt
│   └── ...
├── DESIGN.md                           # 交互与视觉设计约束
├── FRONTEND.md                         # 前端工程约定（框架、目录、状态）
├── PLANS.md                            # 路线图与里程碑（中长期）
├── PRODUCT_SENSE.md                    # 产品判断与取舍原则
├── QUALITY_SCORE.md                    # 质量门禁与评分口径
├── RELIABILITY.md                      # SLO、容错、降级与可观测性
└── SECURITY.md                         # 威胁模型、密钥与合规要求
```

### 效率悖论与冷启动阵痛
在 Harness 环境建立初期，整体效率往往低于人工开发
因缺乏配套的自动化验证、明确的 Lint 规则和清晰的领域边界，Agent 会频繁陷入修复循环或产出幻觉代码，基础设施的完备度直接决定了 Agent 的产出上限

OpenAI 报告的总效率提升约 [10x](https://openai.com/index/harness-engineering/)，但这是 Harness 成熟后的稳态数据。在冷启动阶段，基础设施的构建开销使实际产出低于纯人工开发。吞吐量随团队规模增长而非线性提升，因为更好的 Harness 设计对每一位工程师产生复合价值——这是传统开发中不存在的网络效应

### 高吞吐范式下的取舍：先合并后修复
在极高并发的产出下，传统工程严格的 Gatekeeper 模式会成为瓶颈
- 传统模式：防错优先，单次修改成本高，准入卡点严格
- 高吞吐模式：容错优先，依赖快速 Merge + 快速发现回滚，纠错成本远低于等待阻塞的成本

这种模式下必须容忍局部的代码丑陋，防止过早抽象 (Three similar lines of code is better than a premature abstraction)

[OpenAI Latent Space 播客](https://www.latent.space/p/harness-eng)中 Ryan Lopopolo 分享了极端实践：每日消耗约 10 亿 token（约 $2-3k/天），合并前零人工 Code Review（仅合并后监控），构建循环约束在 1 分钟以内。其团队（7 人）维护了 500+ npm 包——专门为 Agent 并行工作而设计

### 经验固化闭环
Agent 每解决一个复杂 Bug，其推理过程和修复策略必须被要求沉淀至知识库或转换为持续集成测试
未被泛化吸收的 Case 只是单点修复，无法提升系统的整体鲁棒性

Anthropic 的关键洞察：**"Every component in a harness encodes an assumption about what the model can't do on its own, and those assumptions are worth stress testing... because they can quickly go stale."** Harness 的组件是对模型短板的编码，而这些短板会随模型进步而改变，因此 Harness Engineering 本质上是持续进化的实践，而非一次性的基础设施建设

## 总结

在大规模的纯 Agent 开发中，代码的一致性不再由开发者手动 review 保证，而是完全依赖底层基础设施的纪律性
当前算法工程的挑战已经从"如何让模型写对一段逻辑"，转向了"如何设计一套容错约束系统，让模型在不断犯错和自动纠错中必然收敛到可用状态"

三个核心判断浮现：

1. **Harness 是护城河，模型不是。** LangChain 仅改 Harness 就提升 13.7 个百分点，Opus 4.6 在不同 Harness 中排名从 #33 到 #5——模型评估在一定程度上就是 Harness 评估
2. **Harness 的假设有保质期。** 每个组件都编码了"模型做不到什么"的假设，这些假设需要持续压力测试，因为它们会随模型进步而过时
3. **全局一致性是未解问题。** Agent 在局部优化上已经出色，但跨模块、跨系统的全局连贯性仍然是 Harness Engineering 最大的开放挑战

