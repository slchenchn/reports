# 企业 RSI 与 Memory 实践：大厂与 Startup 的官方博客和方法

这份调研同时关注两件事：企业怎样改进 Research Agent，以及历史经验怎样进入下一轮研究。Memory 部分具体看**保存什么、怎样召回、如何更新和遗忘，以及是否转化为后续任务的能力提升**

Weco、Sakana、Poetiq 和 Anthropic 提供自动研究的具体实验，NVIDIA、AMD、Alibaba 提供 KernelGen 工程参考。另补充 **Mem0 和 Letta** 的近期材料，重点看经验整理、替代、降权和后台 dreaming；这些机制可以支持 RSI，其自身尚未证明研究能力递归增强

## 材料范围与阅读入口

范围为 **2026-04-19 至 2026-10-19**，以企业官方博客、技术文章和作者维护的实现为主。旧项目只收录这个窗口里的新报告或明确版本更新；研究计划、实验结果与公开实现分别介绍

| 企业 / 团队 | 近期官方材料 | 发表时间 | 本文重点 |
|---|---|---|---|
| Weco AI | [4 Levels of RSI](https://www.weco.ai/blog/4-levels-of-recursive-self-improvement)、[AIDE²](https://www.weco.ai/blog/first-evidence-of-recursive-self-improvement) | 2026-07-10、07-14 | 改写 harness，按操作选择搜索历史，检验改进者本身是否更强 |
| Sakana AI | [RSI Lab](https://sakana.ai/rsi-lab/)、[MASS](https://pub.sakana.ai/mass/) | 2026-06-05、10-12 | workflow 搜索、轨迹筛选与经验的参数内化 |
| Poetiq | [RSI Perspective](https://poetiq.ai/posts/rsi_perspective/)、[Augur](https://poetiq.ai/posts/augur/)、[LLM Router](https://poetiq.ai/posts/llm_router/) | 2026-08-04、08-10、10-15 | 自动构建 harness、发现能力边界、优化模型路由 |
| Anthropic | [Automated Researchers](https://www.anthropic.com/research/automated-researchers-mitigate-alignment-failures)、[When AI builds itself](https://www.anthropic.com/institute/recursive-self-improvement) | 2026-08-28；后一篇采用 09-18 实证更新 | 持久实验记忆与共享论坛，每轮重置 session，再独立训练和验收 |
| OpenAI | [Research acceleration: The view inside OpenAI](https://openai.com/index/research-acceleration-view-inside-openai/) | 2026-09-06 | 从实际研究流程观察 Agent 的投入、成功率与人工介入 |
| Google DeepMind | [AlphaEvolve: Scaling impact across fields](https://deepmind.google/blog/alphaevolve-impact/) | 2026-05-07 | 算法搜索进入模型、编译器与硬件基础设施 |
| NVIDIA | [KDA² / KDA v0.6](https://nvlabs.github.io/kda/blog/2026-09-27-kda-for-kda/) | 2026-09-27 | kernel 搜索、诊断工具、知识维护与验收强化 |
| AMD | [GEAK v4](https://www.amd.com/en/developer/resources/technical-articles/2026/geak-v4.html) | 2026-07-23 | 确定性编排、两层优化与执行证据驱动的学习 |
| Alibaba | [Atrex Kernel Agent v0.3.0](https://github.com/alibaba/atrex-kernel-agent/releases/tag/v0.3.0) | 2026-09-23，release | 长 episode、失败记录、恢复和 supervisor 验收 |
| Recursive | [First Steps Toward Automated AI Research](https://www.recursive.com/articles/first-steps-toward-automated-ai-research) | 2026-06-11 | 训练算法、训练速度与 kernel 优化的结果公开 |
| Hillclimb | [DeepRSI](https://www.deeprsi.com/blog/) | 2026-10-12 | 长时 ML 研究任务、公开反馈与私有评分 |
| Mem0 | [Memory Decay](https://mem0.ai/blog/introducing-memory-decay-in-mem0)、[Dream](https://mem0.ai/blog/dream-background-memory-consolidation-for-ai-agents) | 2026-05-08、08-04；分别更新于 09-03、09-28 | 按访问历史降权，合并重复、标记替代、后台归纳 |
| Letta | [Trajectory](https://www.letta.com/blog/trajectory/)、[Evaluating Memory in Production Agents](https://www.letta.com/blog/evaluating-memory-in-production-agents/) | 2026-07-23、07-28 | 统一经验格式、后台 dreaming，分别评测记忆使用和生成 |

<!-- 日期：Sakana RSI Lab 按官方博客索引 https://sakana.ai/blog/ 的首发日期；MASS 按 https://sakana.ai/mass/ 的博客发布日期，论文首发为 2026-10-08。Anthropic When AI builds itself 使用文内 Update 9/18/2026，未将其原有统计全部视为 9 月新结果。Cursor multi-agent-kernels 与 Anthropic 最初的 Automated Alignment Researchers 均发表于 2026-04-14，超出当前窗口，未列为近期成果。AlphaEvolve 2025 首发及 Sakana Lab 回顾的旧论文不重新计为新方法 -->

### 从 Memory 看，先区分经验存在哪里、怎样被使用

搜索记录、跨任务知识库和模型权重承载的经验不同。下面按实际使用方式比较；只保存日志，还不足以说明下一轮 Agent 会读取并用好它

| 企业 / 系统 | 经验载体 | 怎样进入下一轮 | 更新与遗忘 |
|---|---|---|---|
| Weco AIDE² | 候选解树、执行结果、近期摘要 | 按当前操作选取摘要、完整解和反馈 | 裁剪当前上下文；未给出跨任务经验库的淘汰机制 |
| Sakana MASS | workflow、执行轨迹、共享模型权重 | 筛选轨迹后做 LoRA，模型继续承担三个角色 | 用训练内化经验；未专门研究参数遗忘 |
| Anthropic AAR | 个人 memory file、共享 survey / forum / leaderboard、代码快照 | 新 session 读取持久记录，沿用已有方法和结果 | 清空会话历史，保留研究记录；未描述自动淘汰经验的算法 |
| NVIDIA KDA | 跨任务 KernelWiki + 当前任务工作区 | Wiki 规则检索，工作区回看候选与执行证据 | Wiki 可纠错、撤回和检查时效；任务经验自动写回流程披露不足 |
| AMD GEAK v4 | 运行内记录、跨次 learned cards、参考知识库 | 按 kernel / GPU / regime 提出候选，再实际验证 | 只让已验证规律进入卡片；未给出收益驱动的删除策略 |
| Alibaba Atrex | 实时 journal、`live.json`、Supervisor 的版本记录 | 支持 episode 恢复和结果复核 | 区分活动状态与正式记录，筛选归档产物；跨任务召回机制未展开 |
| Mem0 | 带状态与替代关系的记忆条目、归纳记忆 | 相关性搜索，再做状态过滤与访问时效排序 | Merge / Supersede / Synthesize + 软降权，保留原记录 |
| Letta | 标准化轨迹、记忆文件、skills、消息历史 | 后台 dreaming 整理，Agent 按需查文件与历史 | 评测归纳、去重和修复陈旧内容；未给出统一的自动删除规则 |

**上下文裁剪、检索降权、内容替代和物理删除要分别看**：它们都能减少旧经验对当前决策的干扰，但保留的信息和重新找回的成本不同。Poetiq、Recursive 等材料对内部经验管理的披露较少，后文只介绍已经公开的部分

## 1. Weco：直接优化 Research Agent 的代码

### 两层循环怎样运行

AIDE² 的内层 Agent 做具体研究任务，外层 Agent 修改内层 harness 的源码。每次改写后，让候选 Agent 在相同成本约束下完成多个研究任务，以私有评分决定是否接受新版本

博客实验的外层使用 Claude Opus 4.7，内层使用 Gemini 3 Flash。内层承担大量重复评测，外层的源码改写调用只占整体成本的一小部分，因此两层采用不同模型

![AIDE²：外层改写 Agent，内层在多类研究任务上评测，再将聚合结果反馈给外层](assets/weco-aide2-outer-loop.png)

图中每张卡片代表一类任务中的多次研究运行，聚合评分回到外层。它评估的是候选 Agent 的优化能力，接受的产物是下一版 Agent 源码

### 它具体改出了什么

| 改进对象 | 博客描述的实现 |
|---|---|
| 搜索策略 | 每个初始 draft 的子树作为 bandit 的一个 arm，分配不同 lineage 的搜索预算；选中 lineage 后，再贪心选择其中的 parent |
| 逃离平台期 | 最佳 lineage 停滞后，复制全局最佳代码，以新策略开启新 lineage，再作为新的 arm 参与预算分配 |
| 上下文 | 按 draft、debug、improve、eval review 等操作分别组织输入，用近期尝试的简短摘要与一个完整解替代整段历史 |
| 执行输出 | Eval review 对输出去重、截取头尾并限制大小；debug 保留原始尾部；improve 另读近期分数的平台期信号 |
| 验收约束 | Agent 看到 public feedback，版本选择使用 private score；固定完整成本，避免靠增加调用数量取得表面收益 |

上下文缩短后，节省的 token 可以重新投入搜索。这里值得研究的是“给当前操作哪些信息”，以及搜索分支之间如何分配预算

### Memory：按操作读取历史，控制本轮上下文

候选解树保存代码、执行反馈、分数与派生关系，Prompt 构造器再按 draft、debug、improve 等操作选择不同内容

| 当前需要的信息 | 进入上下文的形式 |
|---|---|
| 近期尝试 | 最新 12 条单行摘要，加一个完整解 |
| 评测结果检查 | 输出去重、截取头尾，上限 32k |
| 调试 | 原始执行输出的尾部 |
| 是否进入平台期 | 最近 4 个分数形成的信号 |

这里由 harness 规则决定历史的读取范围，没有描述 BM25 或独立的语义检索模型。其作用是减少当前操作的输入负担；被截掉的内容是否继续保存在搜索记录中，与是否进入 Prompt 是两回事，博客没有把它定义为长期经验库的删除机制

### 最有价值的是它实际做了 Ignition Test

Weco 又让改进后的 Agent 坐到外层位置，检查它是否更会产生下一代 Agent。博客里两组最终到达相似水平，效率差异也没有达到统计显著，因此作者没有宣称已经实现 ignition

| 博客实验 | 报告结果 |
|---|---|
| 外层搜索 | 100 次改写评测，约 8 天 |
| 被接受的连续改进版本 | 7 个 |
| Prompt 大小 | 相对直接拼接历史，平均缩小约 16 倍 |
| 改进者再次改进自身 | 未给出足够强的 ignition 证据 |

另一个值得留意的细节：系统产生过统计式反作弊组件，但后续 mutation 把它改坏了，最终版本中该组件没有实际作用。**Agent 写出了某种机制，和该机制真正贡献收益，需要分别检查**

对 KernelGen，可先让外层只改搜索策略、上下文构造或 profiling 流程，保持生成模型和硬件 evaluator 固定，再让新旧外层从相同起点竞争

<!-- Weco 方法来自官方博客 §1、§2.4、§3.1；图片来自 https://www.weco.ai/assets/blogs/first-evidence-of-recursive-self-improvement/figB_outer_loop.png，未修改图中内容。博客 7 月的 reward-hacking 数值与 9 月论文口径不同，本文未混用；正式论文 https://arxiv.org/abs/2609.26457 -->

## 2. Sakana：让改进结果进入共享模型的下一轮训练

RSI Lab 的路线是把自动研究、程序演化与模型开发连接起来，MASS 是其中近期有明确循环设计的工作。合作方包括 UC Berkeley，作者有 Hyunin Lee、Yujin Tang、Matei Zaharia 等

### MASS 的内外循环

同一个模型分别承担 executor、workflow optimizer 和 evaluator

1. 固定模型权重，搜索团队 workflow，改变任务如何分工、结果怎样交接，以及哪些检查必须完成
2. Evaluator 比较候选结果与已有 champion，optimizer 根据反馈提出下一版 workflow
3. 对选中的 workflow 多次 rollout，由当前模型挑选较好的执行轨迹
4. 去掉训练输入中的 workflow 文本，对 orchestrator 与 subagent 轨迹做 LoRA 训练
5. 新模型再次承担三种角色，开始下一轮

去掉 workflow 文本这一点很关键：训练目标包含组织工作的行为，模型需要从任务本身学会怎样安排协作

| 实验设置或观察 | 公开结果 |
|---|---|
| 起点与环境 | Qwen3.6-27B，qwen-code |
| 完整循环 | 2 轮 |
| 四个公开 research benchmark | 第二轮 score / output token 为底座的 1.2–1.6 倍 |
| 固定底座执行不同代搜索出的 workflow | 第一轮搜索产物对底座搜索产物的胜率为 64% |
| 与外部 judge 的判断一致率 | 73% → 93% |

score / output token 没有计入完整的搜索与训练成本。Workflow transfer 固定了最终执行模型，但搜索时三个角色共同变化，尚不能单独归因于 optimizer

### Memory：从外部轨迹到模型权重

MASS 有两层经验载体：搜索时的 workflow、反馈历史和执行 workspace，以及训练后的共享模型权重。Optimizer 根据已有反馈继续改 workflow，外层再挑选执行轨迹，把其中的行为训练进模型

| 筛选与训练 | 做法 |
|---|---|
| 收集经验 | 每个任务将 champion workflow 执行 18 次，记录消息、工具调用与结果 |
| 选择经验 | 当前模型两两比较 workspace，拟合 Bradley–Terry 分数排序 |
| 保留数量 | 每个任务最多 16 条轨迹，15 条训练、1 条 validation |
| 内化方式 | 移除 orchestrator 输入里的 workflow 文本，对 orchestrator 与 subagent 对话做 LoRA |

同一模型的 evaluator 参与了经验选择，训练后权重又供三个角色共同使用。这里值得跟踪的是经验筛选与参数更新怎样互相影响；文章没有给出外部记忆的去重、过期淘汰或模型参数遗忘机制

### 额外值得关注的观察

博客在固定 executor 的情况下，分别增强 evaluator，以及同时增强 evaluator 和 optimizer。后者的搜索改善更明显，说明高质量反馈能否被转化为有效修改，可能比单纯提高打分能力更关键

对 KernelGen，一个直接的实验是固定 kernel 生成模型与真实 GPU evaluator，只训练或替换 optimizer，测它能否在新任务上更快找到有效方向

<!-- MASS 具体流程与两轮设置来自详细博客；公开实现入口 https://github.com/SakanaAI/mass，复现覆盖说明见 docs/reproduction.md 与 docs/experiment_coverage.md。RSI Lab 成立文章是路线说明，回顾的 2024–2025 成果不作为近半年新实验 -->

## 3. Poetiq：Self-Optimizing Optimizers 与自动生成 Harness

Poetiq 把模型当作系统中的可替换组件，重点改进代码、prompt、工具和搜索策略。其官方说法是：Metasystem 一边构建任务专用 harness，一边优化自身的代码和跨任务策略

需要分开看两种产物：任务 harness 是否更强，以及负责生成 harness 的 Metasystem 是否更会继续优化。公开博客对前者给了多组结果，对后者的完整版本谱系、接受规则和独立对照披露较少

从 Memory 看，官方提到跨任务策略库，Augur 也会积累不同模型、不同问题类型的能力地图。不过，条目格式、召回方法、旧策略何时替换和失效，并未充分公开；这些目前只能作为可复用资产的描述，无法据此复现其经验管理算法

### 已公开的三种方法形态

| 形态 | 输入与输出 | 技术关注点 |
|---|---|---|
| 自动生成 Coding Harness | 给定任务数据，生成并优化调用模型的系统 | 固定底座权重，修改推理与工具组织；检查跨模型迁移 |
| Augur 能力边界探测 | 自动生成并逐渐强化问题，定位模型失效类型 | 将缺知识、组合推理失败、来源有争议分开处理 |
| 自动优化 LLM Router | 给定查询分布、可用模型与成本要求，生成路由决策 | 同时选择模型与 reasoning effort，优化质量和成本的取舍 |

### Augur：先辨别失败类型，再决定怎样改进

Augur 不只比较两个模型的最终答案，还检查原子事实、工具能否澄清分歧，以及把多个事实组合成答案时是否出错

| 失败类型 | 博客描述的处理 |
|---|---|
| 缺少事实 | 生成用于补充知识的监督数据 |
| 原子事实都正确，组合答案错误 | 构造有可验证答案的 reasoning / RL 任务 |
| 外部来源互相矛盾 | 排除，避免把没有唯一答案的问题当训练样本 |

这个区分比“把失败轨迹都拿去训练”更具体。对 KernelGen，也可以区分缺少 API / 硬件知识、知道优化规律却不会正确应用，以及 verifier 或 baseline 本身存在问题

动态出题可以降低固定题库被记住的风险，仍需验证题目与答案的可靠性，并留出不参与数据生成和选择的终测

### 最新 Router 案例：优化过程已开始落到商业目标

10 月博客模拟三类企业查询分布，分别约束模型供应商、可用模型和隐私条件。Metasystem 据此优化路由，生成的一个方案使用决策树：通常直接按特征分支选择模型，仅在不确定时额外调用 LLM，文章报告这一 fallback 少于 1%

例如，它把 case-law 问题送给 Opus，把部分 statute / regulation 问题送给 Grok，而不是只按模型的统一榜单成绩或单 token 价格分配请求。另一个场景把 reasoning effort 也纳入选择，成本评估使用实际任务开销

| 口径 | 博客报告 |
|---|---|
| 自动优化耗时 | 12 小时 |
| 从收集数据到完整结果的人类工程投入 | 1 名工程师，2.5 周 |
| Economics Inc 模拟查询集，固定高 effort 模型 | 81.8% accuracy，成本 $32.80 |
| 同一查询集，Poetiq Router | 82.3% accuracy，成本 $27.83 |

这些成本对应文章中的查询集，不是单次调用价格。它说明可把“成本—质量”作为自动优化目标，也显示数据准备与评测工程仍占大量工作；尚未公开足以复现 Metasystem 自身持续改进的完整实现

<!-- Coding 结果来源 https://poetiq.ai/posts/recursive_self_improvement_coding/，2026-05-14；本文只使用其中固定模型、自动构建与跨模型应用的描述。Augur 的“无法被记忆”和 RSI 已到来的表述为企业主张，未当成已验证结论；本文未把动态生成数据同时用于训练与检查视为独立终测 -->

## 4. Anthropic：让 Agent 提出训练方法，再独立训练和验收

8 月的 Automated Researchers 工作研究如何缓解已有 benchmark 可以测量的对齐问题，覆盖欺骗、迎合、隐私、jailbreak 等类别。它展示了自动研究怎样实际改动目标模型的训练方法与权重

### 研究流程

![Anthropic AAR：提出方法、审查代码、训练模型、隔离评分，再将结果返回共享论坛](assets/anthropic-aar-harness.png)

| 环节 | 公开实现的做法 |
|---|---|
| 准备文献 | 4 个 librarian agents 整理共享 survey，记录方法与复现要点 |
| 并行研究 | 5 个 Claude Opus 4.8 AAR 针对同一种 failure 提出训练方法 |
| 固化提案 | 训练前写 mini-paper，明确 loss、数据构造和训练配置，并绑定不可变记录 |
| 代码审查 | Monitor 检查实际代码是否违反数据与蒸馏约束，另检查代码与提案是否一致；修改后重新审查 |
| 固定试验预算 | 每个方法使用 1 张 H200，约 30 分钟训练 |
| 独立评测 | 独立 evaluator 读取权重，返回公开 benchmark 成绩与通用能力是否退化 |
| 下一轮 | 共享 forum、leaderboard 和代码快照，每轮开启新 session，靠持久记录延续研究 |

### Memory：重置会话，保留实验状态与共享成果

| 载体 | 内容与用途 |
|---|---|
| 每个 AAR 的持久 memory file | 保存自己尝试过的方法与分数，让新 session 接着研究 |
| 共享 survey | Librarian 整理的方法、相关文献与复现要点，作为研究起点 |
| 共享 forum 与 leaderboard | 查看其他研究者的尝试、分数与代码快照，继续修改已有方法 |

每轮开启新 session，切断不断增长的对话历史，同时保留可继续执行的研究成果。这个设计让“清空上下文”和“丢掉实验经验”可以独立处理，也适合研究长期失败历史是否会抑制探索

报告描述了读取与共享这些记录的流程，没有给出向量召回、BM25 或独立 reranker 的具体实现，也没有报告自动合并、撤回过时方法的策略

### 验收设计值得单独借鉴

系统同时优化多项 benchmark，用几何平均鼓励各项共同提高，并对通用能力回归设置淘汰条件。论文另外检查未向 Agent 展示的 benchmark、Petri 多轮审计，以及向更大目标模型迁移

其中，部分泛化试验会先用 held-out benchmark 从领先方法中选出最佳方案，再做 Petri 和规模迁移。因此那个 held-out 在这些选择步骤中承担 validation 作用，Petri 才是不参与选择的进一步测试

这种设计可以迁移到 kernel：训练前固化优化假设与约束，候选运行后由隔离 evaluator 同时检查 correctness、性能与回归，原始数据和验收程序不交给搜索 Agent 修改

### 与 RSI 的关系

这里的 Agent 学会提出更好的训练方法，被更新的是目标模型；它尚未通过多代“改进后的研究者继续改进研究者”实验，证明自动研究能力递归增强

另一个官方材料 When AI builds itself 展示了内部研发自动化与研究判断的变化，适合了解公司的整体路线。代码量、使用量和研究产出之间仍有距离，不宜把它们直接换算成 RSI 的加速倍率

<!-- 方法详述：https://alignment.anthropic.com/2026/automated-alignment-researchers/，§2–§3、§5；两类 monitor、不可变提案、fresh session 和 selection 口径均取自该报告。图片为该报告 fig2.png，未改动。未将窗口外 2026-04-14 的 weak-to-strong 实验数值混入 8 月研究 -->

## 5. OpenAI：从实际研发流程衡量 Research Acceleration

9 月报告把内部研发拆成 Decide、Design、Build、Run、Analyze、Communicate，观察 coding agent 的使用、实验运行、任务成功与人工介入情况

它的价值在于测量整个研发过程，帮助识别“写代码更快之后，瓶颈转移到了哪里”

| 观察 | 官方报告 | 对 RSI 的意义 |
|---|---|---|
| 自动化投入 | 8 月中旬，Agent runtime 相当于每个人类工作日约 3.1 个 Agent 工作日 | 并发研究已经形成可观投入，仍需计算实际成本与有效产出 |
| 实验频率 | 每名活跃实验者的试验数量上升 | 同期 compute 也增加，无法单独归因于 Agent |
| 高层研究规划 | 在 Agent 输出 token 中仍占很小比例 | 研究决策与具体执行的自动化程度不同 |
| 长任务成功 | 成功的 4–8 小时任务中，超过一半至少有人类介入一次 | 任务完成与完全自主研究需要分别衡量 |

这篇没有公开完整的自改写算法或多代研究者训练闭环。对 KernelGen，更直接的启发是同时记录 Agent 的研究投入、人工介入、GPU 消耗和最终有效改进，而非只计生成了多少 kernel 或运行了多少实验

这份报告也没有展开 Research Agent 的 Memory 实现，不能用研发使用统计推断其经验库、召回或遗忘机制

<!-- OpenAI 原文 §1–§3 与 methods。3.1 个 Agent 工作日按原文标准 8 小时工作日计算，属于 runtime 量，不是 3.1 倍生产率；成功率来自可确定结果的会话分类，原文排除了不确定结果。未将路线或未来目标当成已经实现的算法 -->

## 6. Google DeepMind：AlphaEvolve 把算法搜索接回 AI 基础设施

5 月的更新展示 AlphaEvolve 从算法搜索进入更多实际系统，包括 TPU 电路设计、缓存策略、编译器、模型训练与科学计算

其基础模式是让模型生成程序候选，经可执行 evaluator 打分，再依据候选数据库中的结果继续演化。近期文章主要补充部署范围与案例，算法的首次发布早于本文窗口

候选数据库属于搜索过程的外部状态。5 月文章没有单独验证新的 Memory 管理方法，也没有说明如何将这些部署案例自动总结为跨任务经验，因此本文只把它列为搜索记录与真实系统反馈的参考

| 近期案例 | 官方报告的作用 |
|---|---|
| TPU 设计 | 搜索得到的电路方案进入下一代 TPU 设计 |
| 缓存策略 | 改进 cache replacement policy，缩短以往依赖大量人工的搜索过程 |
| Google Spanner | 调整 LSM-tree compaction heuristics，降低约 20% write amplification |
| Klarna 模型训练 | 提高训练速度，同时改善模型质量 |

它展示了明确的反馈通道：AI 发现的算法可以改善训练和运行 AI 的基础设施。至于这些改进带回下一轮后，是否让 AlphaEvolve 本身发现得更快，仍需单独测量

对 KernelGen，值得采用的是选择真实瓶颈、让 evaluator 可执行，以及把局部优化重新接入实际训练或 serving 系统

<!-- 2026 更新原文 Improving AI infrastructure / Scaling commercial applications；基础算法只作为背景，来源为 2025-05-14 官方首发，不列作新进展：https://deepmind.google/blog/alphaevolve-a-gemini-powered-coding-agent-for-designing-advanced-algorithms/。未将 2025 首发结果归入 2026 新实验 -->

## 7. NVIDIA、AMD、Alibaba：KernelGen 可直接借鉴的工程方法

### NVIDIA KDA：知识与工具更新，需要跟着验证条件一起改

9 月 KDA 博客把 workflow、profiling skills 和 KernelWiki 都作为可改进对象。公开工程案例还展示了 evaluator 较弱时会出现的假收益，例如利用输入分布或布局边界绕过通用实现

其后续验收加入隔离的随机分布、布局、fresh seeds 与真实模型轨迹。对 KernelGen，值得借鉴的是把“发现了一个优化”与“这个优化适用于哪些条件”一起记录，经验进入后续任务前保留重新执行的入口

Memory 要分两部分看

| 层次 | 怎样保存与使用 | 怎样更新 |
|---|---|---|
| 跨任务 KernelWiki | 从实现、PR、文档与案例整理知识；按架构、语言、症状和置信度过滤，再按关键词命中排序 | 修正知识页、撤回证据不足的实现，记录适用版本并检查时效 |
| 当前任务工作区 | 保存候选父子关系、benchmark 结果、失败原因和运行产物，支持比较分支与继续优化 | 每轮 correctness / 性能反馈更新候选状态，保留失败证据 |

KernelWiki 的公开查询脚本是规则检索：别名扩展后，标题、元数据和正文命中分别计分，没有内置 LLM reranker。PR 来源页的收集、准入和索引也有脚本规则；综合知识页的完整自动写作流程尚未公开

Wiki 已有纠错、撤回与时效告警，告警本身不会自动删除内容。**任务内发现怎样提炼、跨任务验收并写回 Wiki，仍是公开流程中的缺口**，不能把 Wiki 的存在直接视为自动经验积累已经闭合

构造步骤与规模可见[现有展示稿的 KDA 部分](speaker1.md#3-kda--kernelwiki经验怎样总结更新与撤回)

### AMD GEAK v4：确定性编排，LLM 做技术判断

GEAK v4 用 JavaScript Workflow 控制循环、并行、预算、验收与停止，LLM 负责提出策略和代码。系统层先 profile 真实 workload、按 Amdahl 影响排序，再调用 kernel 层优化，最后回到 serving 重新测量

经验也分两层：本次运行记录方向、独立 speedup、端到端变化与结论；跨次 learned cards 按 kernel 类型、GPU 与 regime 保存已验证规律。卡片可以增加下一轮的候选方向，不能代替实际测量或直接排除需要尝试的路径

另有一类参考知识：`perf_knowledge` 按算子、backend、GPU、dtype 和 regime 组织知识矩阵，`expert_skills` 提供人写的优化 recipe，附匹配条件、验证证据与已测结果。它与运行后生成的 learned cards 来源不同

这里明确了经验的字段与适用范围，文章没有把具体召回器写成 BM25 或某个 embedding 模型。其保守原则很有价值：历史经验帮助生成候选，当前设备上的执行证据决定是否接受；未公开统一的陈旧卡片删除策略

| 设计 | 对 KernelGen 的作用 |
|---|---|
| 先做配置与 backend 筛选，再写新 kernel | 用较小成本先验证优化空间 |
| 独立验证与端到端回测 | 避免单 kernel 加速没有改善实际 workload |
| 已验证经验只用于提出候选 | 降低错误知识过早限制搜索的风险 |
| JS 控制状态与停止 | 将预算和流程约束变成可执行逻辑 |

该架构中的递归调用是系统层复用 kernel 优化子流程，尚不能据此认定优化器本身在递归变强

### Alibaba Atrex：独立 Supervisor 接受结果，失败也进入记录

Atrex 的长 episode 在隔离 Git worktree 中进行 profile、研究、编辑与验证，结束时给出结构化 handoff。Supervisor 核对代码与 journal，在同一 GPU allocation 下按 incumbent、candidate、candidate、incumbent 的 ABBA 顺序比较

通过 correctness 且严格改善的候选才替换 incumbent；其他结果同样进入 `memory/vN.json`。中断时保留 worktree、实验记录和恢复信息，能够延续同一次 episode

#### Memory：实时状态、正式结果与归档范围

| 记录 | 更新者与时机 | 作用 |
|---|---|---|
| Episode journal | 探索过程中持续追加结构化实验 | 保存假设、修改、验证与失败信息 |
| `memory/live.json` | 每次 journal 追加后原子刷新 | 暴露活动 episode 的实时状态，不参与版本选择或晋升 |
| `memory/vN.json` | Supervisor 写入正式结果 | 保存 outcome 和已持久化的结构化实验，包含失败、中断结果 |
| Trace-retention manifest | 运行结束或可处理的中断时生成 | 声明后续知识提取所需的代码、正式 Memory、journal、评测、Wiki 查询与精简 profiler 报告 |

Manifest 不收录原始 Agent session JSONL、stdout / stderr、大型 profiler capture 等全部运行材料。这是按后续用途划定归档范围；它尚未展示根据经验收益动态保留或淘汰记忆的算法

这些记录支持恢复和复核，但 README 没有展开新任务怎样召回历史 episode、哪些字段会被注入 Prompt。能保存和恢复经验，与能跨任务复用经验，需要分别验证

这里的有效经验既包括成功优化，也包括失败原因和测量证据。被接受的主要产物仍是 kernel 与对应 manifest，Agent 的研究策略是否因这些记录在新任务上变好，需要额外对照

<!-- AMD 机制来自 GEAK v4 官方文章 Two-altitude architecture / Evidence-gated learning / perf_knowledge；Atrex release 与 long_horizon/README.md 静态核对，源码固定于 5ad94ffaa815f066eabce073d2a920a91b715b0a。ABBA 仅表示交错计时协议，不能替代更多重复、噪声估计与独立任务。本文没有运行这些系统 -->

## 8. Recursive：结果值得关注，研究系统的公开细节有限

6 月博客把模型训练质量、训练速度与 GPU kernel 优化作为同一自动研究系统的应用，公开了部分结果产物

| 任务 | 公开结果 |
|---|---|
| NanoChat Autoresearch | Validation BPB 从此前最优 0.9372 降至 0.9109 |
| NanoGPT Speedrun | 达到规定 loss 的训练时间从 79.7 秒降至 77.5 秒 |
| SOL-ExecBench | 联合优化 235 个 kernel，mean SOL score 从此前最优 0.699 提高到 0.754 |

Kernel 结果来自 NVIDIA B200，SOL score 的 1.0 是 benchmark 的解析性能估计。它说明自动研究能得到较好的优化产物；完整的搜索策略、研究上下文更新与研究者版本接受机制没有充分公开，本文不据此推断其内部 Memory 算法或递归改进效率

这个团队适合继续跟踪后续技术报告与系统发布。当前更适合检查公开产物和验证设计，而非照着博客复现整个 RSI 引擎

<!-- Recursive 为此处讨论的自动 AI research 团队，避免与其他同名企业混淆。数值来自 2026-06-11 官方文章，0.699 / 0.754 为 mean SOL score，不是 speedup；235 kernels 联合搜索不等同于独立未见任务泛化。未把“保留上下文、复用技巧”的一般描述扩写成未公开的具体算法 -->

## 9. Hillclimb：为 RSI 相关能力做长时评测

DeepRSI 包含真实 ML 研究衍生的开放优化问题，覆盖训练、GPU kernel、估计器和算法。每个模型以相同 Pi harness 运行，拿 public verifier 做搜索反馈，最终由 private verifier 评分

| 评测设置 | 公开说明 |
|---|---|
| 任务数量 | 15 |
| 每个模型的重复 | 2 次 |
| 时长 | 每次 48 小时 |
| 环境 | 无网络，提供任务代码与本地工具 |
| 中间结果 | 保存快照，对比 public progress 与 private outcome |

它适合检查“长时间搜索是否持续获得真实收益”，也能定位 public feedback 上升而 private outcome 没有改善的情况。该测试以固定 Agent 改进任务为主，名称中的 RSI 不代表它已经测量了多代改进者的递归增强

对 KernelGen，可以沿用快照与私有验收的思路，再增加新旧 optimizer 从相同起点、在同一完整预算下比较的实验

这里保存的快照主要用于评测与审计，文章没有将其作为 Agent 在后续任务中检索的经验库

## 10. Mem0：把经验整理做成独立的维护过程

Mem0 是 Agent Memory 基础设施公司，近期的 Dream 与 Memory Decay 对应两类问题：重复、冲突的记忆怎样整理，以及大量旧记忆怎样影响当前召回。公开案例以用户事实与偏好为主，迁移到 kernel 优化仍需加入硬件、版本和测量条件

### Dream：合并、替代、归纳分别处理

写入新记忆时，extraction 只能看到当前对话和少量相似记忆，跨会话重复与冲突容易漏掉。Dream 将维护拆成三个动作

| 动作 | 条件与处理 | 旧记录怎样影响后续读取 |
|---|---|---|
| Merge | 新记忆完整包含旧内容并有补充，将旧条目标记为 `merged`，指向更完整的条目 | 默认隐藏；`include_merged=true` 可找回 |
| Supersede | 同一事实有明确的新版本，将旧条目标记为 `superseded`，指向替代条目 | 默认仍会返回并带状态；`latest_only=true` 才排除 |
| Synthesize | 将相似记忆分组，多个独立观察支持同一模式时，新增高层概括并记录来源 ID | 原始记忆保留，概括与原记录并存 |

**Merge / Supersede 在新增记忆的 extraction 流程中判断，Synthesis 才是定期后台任务**。前两者依赖相关旧记忆已进入写入上下文；关系不明确时不修改，所以仍可能漏掉未召回的冲突

状态由 `lifecycle_state` 和 `replacement_memory_id` 表达，更新使用条件写入处理并发。原记录不会被物理删除，标为 `immutable` 或 `exclude_from_dream` 的条目跳过整理。文章没有公开语义判断所用的具体模型

当前 Synthesis 只处理单独归属于 `user_id` 的记忆，带有 `agent_id / run_id / app_id` 的记忆不在这项用户级整理的范围内。用于研究任务库时，还需要设计任务、硬件和版本之间的经验归属

这里有一个容易忽略的实现点：**标记过时，不代表默认检索就不会再看到它**。调用方若要当前有效事实，需要明确选择读取模式

### Memory Decay：依据访问历史调整召回排序

Decay 作用于相关性搜索之后的排序，扩大候选池，再根据访问时效缩放相关性分数，最后截取结果

| 项目 | 公开做法 |
|---|---|
| 时效信号 | 每条记忆最多保留最近 20 次访问时间；旧记录以最后更新时间回退 |
| 分数缩放 | 当天访问可获得约 1.5× 加权，闲置数周逐渐降至约 0.3× |
| 保留方式 | 降低排序权重，仍可召回，不隐藏或删除 |
| 未公开部分 | 没有给出完整的衰减公式，也没有在这篇文章中展开底层召回器配置 |

它把整理内容和调整读取优先级拆开，适合持续增长的经验库。但访问次数、时间新旧与技术正确性并不等价，错误经验也可能因频繁召回而被强化

对 KernelGen，可借鉴“保留旧版本和替代关系”的做法，例如编译器升级后把旧 recipe 标成特定版本适用，默认检索当前条件的条目。通用硬件规律、临时调试信息和已被反证的结论应采用不同规则，不能只按时间统一降权

这些材料说明 Memory 生命周期已经产品化，尚未提供“使用该机制后，研究者跨任务改进能力持续增强”的实验证据

<!-- Mem0 日期按文章正文：Memory Decay 2026-05-08 / update 09-03；Dream 2026-08-04 / update 09-28，博客索引日期有差异。Dream 当前读取语义另核对 https://docs.mem0.ai/platform/features/dream。Merge/Supersede 在 add 时、Synthesis 后台调度，不能统称所有动作离线进行；docs 说明 Synthesis 当前按单独 user_id 的记忆运行，带 agent_id/run_id/app_id 的记忆不在该用户的 synthesis 范围内，因此不能假定该产品直接提供跨研究任务的合并。本文介绍 Platform 功能，未认定开源版具有完全相同实现 -->

## 11. Letta：统一经验格式，后台整理，再检验 Memory 是否真的改变行为

Letta 做有持久状态的 Agent，其近期材料把经验处理拆成可读的轨迹格式、后台 dreaming 和 Memory 能力评测，适合回答“有了历史记录之后，还缺哪几步”

### Trajectory：先让历史经验可以被模型高效阅读

`@letta-ai/trajectory` 将不同 Agent 产生的会话统一成记录序列

| 记录 | 主要字段 |
|---|---|
| `meta` | 来源 harness、工作目录、Git branch、模型 |
| 消息 | `user / assistant / reasoning` 与内容，reasoning 仅在原系统暴露时保留 |
| 工具交互 | 工具名、参数与结果，用 `tool_call_id` 关联 |

格式转换去掉重复 payload、界面事件与内部 bookkeeping，可截断过长工具结果。文章中的采样会话约缩小到原 token 的五分之一，目的是让模型阅读历史并形成记忆，完整回放与审计仍需保留原始记录

处理顺序是：**标准化历史 session → 初次建立 Memory → 后台 dreaming 阅读近期 session、归纳持久经验 → 当前 Agent 按需查询记忆与历史文件**。同一个经验整理过程可以消费不同 harness 的轨迹

文章没有指定统一使用 BM25、向量检索还是某个 reranker，也没有公开 dreaming 所用的固定模型和完整的记忆淘汰规则，不能从“后台归纳”直接推断已解决经验筛选与遗忘

### 把“用好记忆”和“生成好记忆”分别验收

Context-Bench V2 的 Memory profile 包含上下文内记忆、MemFS 中的外部文件、可加载 skills 和消息历史。MemFS 是由 Git 管理版本的记忆文件系统；评测同时使用积累后较混乱的 profile 和整理后的版本

| 维度 | 检查什么 | 典型失败 |
|---|---|---|
| Usage：retrieval | 发现信息缺失，主动找相关记忆或 skill | 只依赖当前上下文，直接回答 |
| Usage：adherence | 实际遵循已保存的有效规则 | 取到了记忆，行为仍不执行 |
| Generation：generalization | 从具体反馈归纳可复用规则 | 已有三条带日期的纠正，又追加第四条 |
| Generation：hygiene | 修改旧内容、去重、清理陈旧信息并维持结构 | 同一规则散落三处，只改一处，留下冲突副本 |

这里最有启发的是后两项：**能写出好代码，与能把经历整理成好记忆，是需要分别测量的能力**。只统计新增了多少 memory 或 skill，会漏掉重复、冲突和错误泛化

其评测由真实 trace 中的失败模式构造场景，使用用户模拟器和 LLM judge，完整 benchmark 未公开。它提供 Memory 行为的检查方法，尚未用 kernel 执行实验建立这些指标与跨任务优化收益的联系

对 KernelGen，可以用同一组实验记录比较“只追加总结”和“修改、合并已有经验”，再测后续未见任务的 correctness、搜索成本与性能；让评测同时覆盖记忆使用和记忆维护

<!-- Letta 两篇均为 2026-07 官方文章。Trajectory schema 与 token 样例见正文；dreaming 对跨 harness 轨迹的支持见 How we're using trajectory。Context-Bench V2 为 private benchmark，Sonnet-5 模拟用户、GPT-5.6-Sol 评分；正文不据模型榜单推断 KernelGen 因果收益。清理旧规则是评测中的行为观察，未扩写为所有 Letta Agent 都自动执行的固定策略 -->

## 12. 企业路线放在一起，值得继续追什么

| 路线 | 代表企业 | 已公开的关键机制 | 下一步最值得核对 |
|---|---|---|---|
| 改写改进器的代码 | Weco、Poetiq | 外层产生新 harness，评测后接受 | 新 harness 再担任改进者时，是否更有效 |
| 把团队经验训练进模型 | Sakana | 自评选轨迹、共享权重更新、多轮继续 | 三个角色各自变强多少，完整成本是多少 |
| 自动发现训练方法 | Anthropic | 训练提案、代码审查、固定 GPU 预算、独立评测 | 研究方法迁移，以及研究者本身的下一代更新 |
| 把研究自动化嵌入公司研发 | OpenAI | 分阶段记录使用、实验与任务结果 | 人类介入、compute 变化与实际研究进展的关系 |
| 改进 AI 基础设施 | DeepMind、NVIDIA、AMD、Alibaba | 可执行反馈、真实系统回测、知识与失败记录 | 局部产物收益怎样进入下一轮研究能力 |
| 长时研究与评测 | Recursive、Hillclimb | 研究成果公开、长时任务与私有验收 | 结果产物、任务优化与改进者提升分别测量 |
| 整理与维护外部 Memory | Mem0、Letta | 状态替代、合并、降权、后台 dreaming、记忆维护评测 | 减少冲突和 token 负担后，是否改善未见任务的实际研究收益 |

从 Memory 主题出发，最值得做的对照是：固定模型、GPU 与总预算，让同一份历史经验分别经过**直接追加、按任务筛选、合并与替代、会话重置**，观察未见任务上的有效探索。遗忘带来的收益，要和信息丢失、重复试错的代价一起测

如果下一步只精读几份材料，建议按这个顺序

1. **Weco AIDE²**：源码改写、独立验收与 ignition test 的完整问题定义
2. **Mem0 Dream / Memory Decay**：直接看经验怎样合并、替代和降权，以及默认读取是否排除过时内容
3. **Letta Trajectory / Memory Evaluation**：经验输入格式、后台整理，以及“使用 / 生成”两类能力的验收
4. **Anthropic 8 月 AAR / AMD GEAK / Alibaba Atrex**：持久实验记忆与会话重置、经验适用条件、活动状态与正式结果的区别
5. **Sakana MASS**：把筛选过的执行经验训练进权重，并检查角色迁移
6. **Poetiq Augur / Router**：从失败类型到训练数据，再到自动优化商业目标；继续跟踪其经验管理与 Metasystem 的进一步披露

与 Kernel Agent 的具体论文和现有实现对照，可继续看[主调研](main.md)；需要展开参数更新与训练信号时，可看[模型训练专题](model_training.md)
