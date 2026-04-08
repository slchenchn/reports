<!--
1. 如果图片路径不存在，那表示是这是一个placeholder，如果你是Gemini，需要你生成图片放到对应的位置上
2. harness engineering是什么，怎么演进的，作用是什么，如何使用，未来如何发展
3. 不能做成枯燥的列表/流水账，要娓娓道来，但不能变成纯讲故事，略微有故事性即可

-->

# Harness Engineering 调研报告

如果说 Prompt Engineering 解决的是“这次怎么让模型答对”，Harness Engineering 解决的是“怎么让 agent 在真实工程里持续做对”
当 agent 开始读仓库、跑命令、改代码、提 PR、做回归验证后，瓶颈就不再是单轮 prompt，而是上下文是否可见、约束是否可执行、反馈是否足够快、失败后能不能恢复

## 先给结论

- Harness Engineering 不是 prompt 的加强版，而是把 agent 放进一个可收敛的工程控制系统
- 真正决定上限的不是模型一次能写出多漂亮的代码，而是仓库是否足够 legible、约束是否物理化、反馈环是否闭合
- 这件事更像平台工程而不是应用开发，工程师的主要工作从“写逻辑”转成“设计环境、编码规则、管理熵”

## 从 Prompt 到 Harness

[Mitchell Hashimoto](https://mitchellh.com/writing/my-ai-adoption-journey) 对 Harness Engineering 的定义很朴素，当 agent 重复犯某种错，就把这类错误变成系统性预防措施，而不是继续手动兜底
[OpenAI](https://openai.com/index/harness-engineering/) 则把这件事推进成了工程范式，在一个 agent-first 的产品开发流程里，团队用 5 个月做出约 100 万行代码和约 1500 个 PR，平均 3.5 PR/工程师/天，人的主要工作不再是亲手写每一段代码，而是让 agent 有能力把代码写出来

| 范式 | 主要对象 | 典型手段 | 关心的问题 |
| --- | --- | --- | --- |
| Prompt Engineering | 单轮输入 | prompt 模板、few-shot、输出格式约束 | 这次回答怎么更准 |
| Context Engineering | 信息装载 | 检索、索引、MCP、文档路由、知识压缩 | 模型能不能看到做决定所需的信息 |
| Harness Engineering | 整个执行系统 | hooks、lint、tests、planner、evaluator、progress file、observability | agent 能不能长时间、自主、可恢复地把任务做完 |

<figure style="text-align:center;">
  <img src="harness_engineering_evolution.png" alt="Three rounded boxes arranged from left to right. The first box is Prompt Engineering and is labeled micro-management. The second is Context Engineering and is labeled knowledge routing. The third is Harness Engineering and is labeled system constraints plus feedback loops. Arrows connect the three boxes, showing that control progressively moves from single-turn prompting to system-level orchestration and closed-loop execution" style="width:80%;" />
  <figcaption>从微操 prompt，到路由知识，再到把 agent 放进一个带约束和反馈的系统里</figcaption>
</figure>

这三者不是替代关系，而是叠加关系
Prompt 负责局部表达，Context 负责可见性，Harness 负责让执行过程收敛

## Harness 到底在管什么

一个常见比喻是 Model 像 CPU，Harness 像 OS 或 Runtime，这个类比不必抠得太细，但责任边界很清楚
模型负责生成，Harness 负责让生成结果可见、可检验、可恢复、可积累

从 [ThoughtWorks 对 Harness 的梳理](https://martinfowler.com/articles/harness-engineering.html) 来看，Harness 本质上是一个控制回路，核心是两类控制手段

| 方向 | 作用 | 典型实现 |
| --- | --- | --- |
| Guides（前馈控制） | 在 agent 动手前收窄搜索空间 | AGENTS.md、ARCHITECTURE.md、schema、skills、bootstrap script、LSP、MCP |
| Sensors（反馈控制） | 在 agent 动手后发现偏差并推动自修复 | lint、typecheck、结构约束检查、过滤测试、浏览器自动化、日志指标查询、review agent |

ThoughtWorks 还把这些控制分成确定性和推理性两类，这个区分很有用，因为它直接对应成本和可信度

| 类型 | Guides | Sensors |
| --- | --- | --- |
| 计算型（确定性） | schema、脚本、codemod、LSP、MCP | lint、类型检查、依赖边界检查、coverage、结构测试 |
| 推理型（概率性） | 设计原则、review rubric、任务拆解模板 | LLM-as-judge、语义重复检测、trace analyzer、架构评审 agent |

<img src="harness_control_loop_placeholder.png" alt="A systems diagram for Harness Engineering in a coding workflow. On the far left is Task Spec, Architecture, and ExecPlan as versioned repository artifacts. They feed into a Planner agent that produces a structured work plan and explicit verification strategy. The Planner hands work to an Executor agent connected to tools for repository read and write, shell execution, browser automation, and observability queries. The Executor writes code changes, progress notes, and telemetry annotations back into the repository. On the right side are Sensors: formatting, lint, typecheck, filtered tests, dependency boundary checks, screenshot or DOM checks, latency and error-budget checks, and policy checks. Sensor outputs feed into an Evaluator agent that classifies failures as context gap, tool gap, architectural violation, flaky environment, or missing recovery path. One arrow loops back to the Executor for immediate repair. A second arrow loops upward to Harness Maintainers who update AGENTS.md, hooks, linters, schemas, docs, or scripts so the same class of error becomes less likely in future runs. The visual should look like an engineering control loop, not a marketing diagram, with clear left-to-right data flow and two labeled loops called local repair and system hardening">

这也是为什么 [Birgitta Böckeler](https://martinfowler.com/articles/harness-engineering.html) 会把 Harness 解释成一种 cybernetic governor
它不是替模型思考，而是通过前馈和反馈，把一个高方差系统压到可控区间里

## OpenAI 五条原则，真正落在工程上是什么意思

OpenAI 这篇文章里最有操作性的部分，不是“纯 Agent 写了多少代码”，而是下面这五条原则怎么落到了仓库和运行时里

| 原则 | 真正意思 | 对应工程动作 |
| --- | --- | --- |
| What the agent can't see doesn't exist | 不在 repo 内、不可检索、不可版本化的知识，对 agent 来说等于不存在 | 把 Slack 讨论、设计决策、schema、计划文档沉到仓库里 |
| Ask what capability is missing | 错误优先解释为环境缺陷，不优先解释为模型不行 | 补工具、补脚本、补结构化 artifact，而不是继续手动补洞 |
| Mechanical enforcement over documentation | 文档可以表达意图，但不能守住底线 | 用 lint、结构测试、CI 规则把边界物理化 |
| Give the agent eyes | 代码本身不足以验证 UI、性能和运行时行为 | 接入浏览器自动化、日志、指标、trace、截图 |
| A map, not a manual | context 很贵，巨型说明书会挤掉真正相关的信息 | 薄 AGENTS.md 只做索引，细节放到分层 docs 里按需加载 |

OpenAI 的 `ExecPlan`，Anthropic 的 `progress file + feature list + init script`，本质上是同一种东西
它们都是 durable artifact，让 agent 在上下文重置、长任务切片、多人并行时还能重新定位自己，而不是每次都从聊天记录里捞状态

## 四根真正 load-bearing 的柱子

| 支柱 | 解决什么问题 | 典型实现 | 做坏了会怎样 |
| --- | --- | --- | --- |
| Context Engineering | agent 看不到关键约束和知识 | 薄入口文档、索引化 docs、references、schema、本地化知识库 | 搜索乱撞、重复试错、上下文污染 |
| Architectural Constraints | agent 知道规则，但不一定会守 | 单向依赖、物理分层、强类型、DI、结构 lint | 局部能跑，整体架构快速漂移 |
| Feedback Loops | agent 改了代码但不知道真错在哪里 | hooks、过滤测试、browser checks、review agent、observability | 幻觉修复、自信退出、死循环 |
| Entropy Management | 高频提交会不断堆积 slop 和冗余抽象 | cleanup agent、doc gardening、规则提升、周期性重构 PR | 一开始跑得快，几周后质量雪崩 |

这里有三个容易被低估的点

- Context 不是越多越好，真正重要的是可路由、可验证、可重启
- 反馈速度通常比反馈完美更重要，毫秒级到秒级的确定性信号比分钟级的大而全流水线更能改变 agent 行为
- 熵管理不能靠“周五统一打扫”，OpenAI 的经验很直接，高吞吐系统需要常驻的后台清理能力

## 采用路径，不必一上来就上满配

[Mitchell Hashimoto](https://mitchellh.com/writing/my-ai-adoption-journey) 给出的六阶段采纳框架比较实用，因为它不是从“要不要 all in”开始，而是从“先把哪个环节交给 agent”开始

| 阶段 | 关键动作 | 真正获得的能力 |
| --- | --- | --- |
| 1 | Drop the Chatbot | 从问答工具切到可读写文件、可执行命令的 agent |
| 2 | Reproduce Your Own Work | 用 double-step 找到 agent 和人类思路的差距 |
| 3 | End-of-Day Agents | 把夜间和离线时间变成并行产能 |
| 4 | Outsource the Slam Dunks | 先把高确定性任务稳定外包给 agent |
| 5 | Engineer the Harness | 把失败案例沉淀成规则、脚本、hook 和文档 |
| 6 | Always Have an Agent Running | 人和 agent 进入并行流水线，而不是轮流上工 |

他在 [Ghostty 的一个非平凡功能开发复盘](https://mitchellh.com/writing/non-trivial-vibing) 里给了一个很好的微观样本，16 个 session，约 2 个日历日，token 成本 $15.98
这类案例最有价值的地方不在于“绝对更快”，而在于 agent 可以在你切走做别的事情时继续推进，这改变的是注意力调度，而不只是编码速度

Hashimoto 还有一个很实用的 insight，view model 质量会直接放大或削弱 agent 能力
把接口改干净、把类型命名拉直、把 tagged unions 等中间层建好，本质上是在改善环境的 ambient affordances，也就是让系统更 legible、更 harnessable

## 行业案例到底在说明什么

| 案例 | 做了什么 | 结果 | 真正说明了什么 |
| --- | --- | --- | --- |
| [OpenAI](https://openai.com/index/harness-engineering/) | 把 repo 变成 system of record，用自定义 lint、结构测试、浏览器和 observability 给 agent 建完整运行时 | 约 100 万行代码、约 1500 PR、5 个月、3.5 PR/工程师/天 | 当约束和反馈足够硬，人的时间会从写代码迁移到设计 leverage |
| [LangChain](https://blog.langchain.com/improving-deep-agents-with-harness-engineering/) | 不换模型，只改 harness，包括 self-verify、LoopDetectionMiddleware、PreCompletionChecklistMiddleware 和 reasoning sandwich | Terminal Bench 2.0 从 52.8 提到 66.5 | Harness 调优本身可以比换模型更直接地改成功率 |
| [Vercel d0](https://vercel.com/blog/we-removed-80-percent-of-our-agents-tools) | 把 15+ 个专用工具砍到 `ExecuteCommand + ExecuteSQL` 两个 | 平均耗时 274.8s 降到 77.4s，成功率 80% 到 100% | 过度工具化会替模型做太多决策，反而压缩了它的推理空间 |
| [Anthropic 2025 长任务 harness](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) | 用 initializer agent、JSON feature list、progress file、init script 解决跨 context window 的失忆问题 | agent 能按 feature 增量推进，并在重启后快速定位状态 | 文件系统里的结构化 artifact 是长任务的持久记忆 |
| [Anthropic 2026 多 agent harness](https://www.anthropic.com/engineering/harness-design-long-running-apps) | 把 planner、generator、evaluator 分开，让 evaluator 用 Playwright 实际点页面和验收 | solo 20 分钟 $9 的结果明显不完整，full harness 6 小时 $200 交付质量显著更高，后续在更强模型上又去掉了部分 sprint 结构 | Harness 复杂度不是越多越好，而是要随着模型边界移动，能删就删，不能僵化 |

把这些案例放在一起看，会看到一个反直觉但稳定的结论
Harness 的价值不只是“补模型短板”，更是在重新分配系统里的认知负担，哪些由模型承担，哪些由工具承担，哪些由结构化 artifact 承担，哪些必须留给人类

## 常见失败模式与反模式

| 症状 | 根因 | 常见补救 |
| --- | --- | --- |
| 上下文越长，agent 越钝 | 无关搜索结果、长日志、过多工具输出占满注意力 | 只注入任务相关上下文，用 sub-agent 做 context firewall，用过滤测试替代全量测试 |
| AGENTS.md 越写越长，效果反而越差 | 巨型说明书不可验证、不可维护、信号权重失真 | 把 AGENTS.md 缩成目录，把细则下沉到分层 docs |
| 局部实现都不错，整体系统却不成形 | agent 优化局部 completeness，不跟踪 global coherence | 里程碑 gate、feature list、sprint contract、先连线后打磨 |
| agent 总说自己做得很好 | 自我评估天然偏正向 | 独立 evaluator、few-shot 校准、显式评分 rubric |
| 测试全绿，但实际行为很差 | 测试覆盖了 agent 容易生成的路径，没有覆盖真实用户路径 | browser automation、approved fixtures、人工定义关键旅程 |
| Harness 用久了越来越没感觉 | 规则在编码旧模型的短板，模型变了，规则没变 | 定期压力测试 harness，删除失效脚手架，避免历史包袱固化 |
| 合并速度上来了，代码库却越来越脏 | 吞吐提升了，熵管理没有同步升级 | 单独的 cleanup lane、质量评分、自动化重构 PR、规则提升 |

最近的 [Natural-Language Agent Harnesses](https://arxiv.org/abs/2603.25723) 也提示了一个风险
Harness 可以被外化成更可移植的自然语言 artifact，但这不等于“多写 agentfile 就一定更好”，泛泛而谈的最佳实践很容易稀释真正有用的仓库特定约束

## 一个更实用的最小落地顺序

不是所有团队都需要 OpenAI 或 Anthropic 那种全套编排，一个能开始产生复利的最小版本，通常按下面顺序搭更合理

1. 先解决可见性
   让 agent 看见架构边界、关键命令、测试入口和主要文档位置，薄 AGENTS.md 比百科全书更有效
2. 再解决确定性反馈
   把 format、lint、typecheck、过滤测试、依赖边界检查尽量左移到本地和 hook 里
3. 再解决重启和交接
   给长任务加 ExecPlan、progress file、feature list、init script，让 agent 能从文件系统恢复状态
4. 最后再加重型能力
   浏览器自动化、observability、review agent、multi-agent evaluator 这些东西有用，但它们只有在前三层打稳后才真的值回成本
5. 单独留一条熵管理通道
   把重复 review 意见升级成规则，把重复修复升级成脚本，把 stale docs 交给后台 agent 清理

一个比较合理的文档布局可以长这样

```text
AGENTS.md                               # 薄入口，只做协作约束和索引
ARCHITECTURE.md                         # 系统边界、分层、不变量
docs/
├── design-docs/                        # 设计决策与核心原则
├── exec-plans/                         # 活跃计划、已完成计划、技术债
├── generated/                          # schema、接口、自动生成参考资料
├── product-specs/                      # 用户旅程和功能规格
├── references/                         # 外部资料压缩版
├── DESIGN.md                           # 视觉和交互约束
├── FRONTEND.md                         # 前端工程约定
├── PLANS.md                            # 中长期路线图
├── PRODUCT_SENSE.md                    # 产品判断口径
├── QUALITY_SCORE.md                    # 质量门禁和评分标准
├── RELIABILITY.md                      # SLO、降级、可观测性
└── SECURITY.md                         # 威胁模型、密钥、合规
```

这套结构真正想解决的不是“文档齐全”
而是让 agent 能用最低的 token 成本找到对当前任务最有决定性的约束

## 我的判断

- Harness 是新的工程杠杆，模型当然重要，但模型能力只能给出潜在上限，真正决定稳定产出的还是 Harness 是否把高方差执行压成了可收敛流程
- 未来最值钱的不是“会不会写 prompt”，而是能不能把失败经验编码进 repo、hooks、linters、tests 和 artifact，把一次性的经验变成可复用的系统能力
- maintainability harness 和 architecture fitness harness 已经相对可做，[ThoughtWorks](https://martinfowler.com/articles/harness-engineering.html) 讲的 behaviour harness 仍然最难，因为“测试通过”还远不等于“产品行为真的对”
- 高吞吐 agent 团队里的 merge philosophy 会和传统工程不同，等待往往比修复更贵，所以前提不是更保守，而是把检测、回滚和熵管理做得更硬
- Harness 的假设一定会过期，Anthropic 和 LangChain 的经验都说明，很多 guardrail 是为今天的模型缺陷临时补位，模型一变，load-bearing 的地方就会跟着移动

所以 Harness Engineering 不是一次性的基础设施建设，而是一套持续重写“模型边界和系统边界交界面”的工程实践
谁能更快地把 agent 的失败翻译成可执行约束、可观察信号和可恢复流程，谁就更可能在 agent-first 的开发范式里拿到真正的复利
