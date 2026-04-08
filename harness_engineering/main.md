<!--
如果图片路径不存在，那表示是这是一个placeholder，需要你生成图片放到对应的位置上

-->

# Harness engineering调研报告

## 起源
harness engineering这个术语是由社区（尤其是 Mitchell Hashimoto 等人）先提出/使用，OpenAI 在 2026 年把它“系统化并带火”

OpenAI的文章[Harness engineering: leveraging Codex in an agent-first world](https://openai.com/index/harness-engineering/)描述了一个0 人写代码、全靠 agent（Codex）开发百万行系统的实践，关键数据如下：

| 指标              | 数值                     |
| ----------------- | ------------------------ |
| 时间周期          | 5 个月                   |
| 团队规模          | 最初 3 人，后扩展到 7 人 |
| 代码总量          | 约 100 万行              |
| 人工手写代码      | 0 行                     |
| Pull Request 总数 | 约 1,500 个              |
| 人均日吞吐量      | 3.5 个 PR/人/天          |
| 效率提升          | 约 10 倍                 |

OpenAI把这一套方法论称为 harness engineering。也即把工程重点从“写代码”转为：
- 设计环境
- 设计反馈 loop
- 设计约束系统
  

从prompt engineering，到context engineering，再到harness engineering的转变，本质上是承认模型的能力逐步提高，人类的“微操”必要性逐步降低，以至于只对结果监督是最优解。

<figure style="text-align:center;">
  <img src="assets/harness_engineering_evolution.png" alt="从 Prompt Engineering 到 Context Engineering，再到 Harness Engineering" style="width:80%;" />
  <figcaption>从 Prompt Engineering 到 Context Engineering，再到 Harness Engineering</figcaption>
</figure>


## 定义
Harness Engineering 是一种新兴的软件工程范式，旨在设计、构建和迭代一套完整的运行环境与制度体系，以引导和约束 AI 智能体，使其能够自主、可靠地完成复杂长周期任务。
工程师从代码编写者转变为系统设计师与环境架构师

model ≈ CPU

harness ≈ OS / runtime

Agent = Model+ Harness

> Humans steer, agents execute

## 四大支柱
### 上下文工程（Context Engineering）
核心原则是给智能体一张“地图”，而不是一本 1000 页的说明书。AGENTS.md/CLAUDE.md 应作为内容目录，保持精简（100-150行），详细内容链接到 docs/ 目录。

### 架构约束与护栏（Architectural Constraints）
通过强制执行不变量而非对实施过程进行微观管理。严格分层架构：Types → Config → Repo → Service → Runtime → UI，依赖方向严格控制。

### 反馈循环
code review、自动测试、CI/CD
例如Claude Code的Hook机制

### 熔管理与垃圾收集（Entropy Management）
AI 会复制代码库中已存在的模式，包括不良模式，导致“AI 残渣”积累。需要建立定期“垃圾收集”机制，将“黄金原则”编码到代码库中。


## 最佳实践
### 渐进式披露
代码库知识库应采用结构化的 docs/ 目录，作为记录系统使用方式、约束、原则等文档。
而不是使用一个巨大的CLAUDE.md文件

智能体从小而稳定的切入点开始，被指导下一步该去哪里查看，而不是一开始就被淹没在海量信息中。这实现了“渐进式披露”——智能体根据需要逐步获取信息，而非一次性加载所有上下文。


### 先合并后修复

| 维度     | 传统      | agent 时代        |
| -------- | --------- | ----------------- |
| 修改成本 | 高        | 低                |
| 等待成本 | 低        | 高                |
| 最优策略 | 严格 gate | 快速 merge + 修复 |

**总结**

- 错误很贵 → 必须在 merge 前消灭。

**传统**

- 防错优先（prevent bugs before merge）

**现在**

- 容错优先（allow bugs, fix fast）

**要点**：在高吞吐量环境下，纠错成本低、等待成本高。

high throughput开发范式中，要**防止过早抽象**
> Three similar lines of code is better than a premature abstraction
>
> —— Claude Code



### 一开始效率低
一开始效率更低是正常的。因为此时agent 没有一个“可执行的工作环境”


### 快速迭代
每遇到一个问题并解决，都要考虑让AI挖掘前因后果，沉淀经验，并思考是否能泛化到更广泛的场景中。


## 总结
> 显而易见的是：构建软件仍然需要纪律，但纪律更多地体现在支撑结构上，而不是代码上。保持代码库一致性的工具、抽象和反馈回路变得越发重要。
> 
> 我们当前最棘手的挑战集中在设计环境、反馈回路和控制系统方面，帮助智能体实现我们的目标：大规模构建和维护复杂、可靠的软件。
> 
> —— OpenAI