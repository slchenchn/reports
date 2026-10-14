# OpenClaw 技术研究报告


## 执行摘要


<figure style="text-align:center;">
  <img src="assets/openclaw_architecture_2.png" alt="OpenClaw 架构图" style="width:80%;" />
  <figcaption>OpenClaw Architecture</figcaption>
</figure>


OpenClaw 的核心形态是一个 **单体进程内的"网关 + Agent Runtime**：对 Operator 客户端（负责发起控制、查看状态与审批的控制端，如 CLI、Web UI、macOS app）与 Node（负责提供设备/执行能力的节点端，如 iOS/Android node、headless node）而言，统一通过 Gateway WebSocket 协议接入，该协议同时承载控制平面与节点传输；而 WhatsApp、Telegram、Discord、Slack 等 chat channels 则通过各自的平台适配器接入。内部则将对话回合映射为一次"嵌入式 Pi Agent 会话运行"，并以 **lane-aware FIFO 队列** 实现"同一会话串行、跨会话限并发"的确定性调度。

该系统的核心算法/优化主要包含：

- **检索/记忆**：Markdown 作为真源，SQLite 作为索引与缓存载体；向量相似度 + FTS5(BM25) 的混合检索、可选 MMR 多样性重排与时间衰减（指数衰减）。
- **消息分块**：基于 min/max 字符阈值的分块器，按段落/换行/句子/空白的优先级选择断点，并显式避免在 Markdown 代码围栏内断开；当被迫在围栏处截断时会"闭合并重开围栏"以维持格式有效性。
- **可靠性与控制**：WebSocket 握手挑战（nonce）+ 角色/作用域/能力声明；协议版本协商与 TypeBox → Schema 生成；对副作用操作引入幂等键要求；执行审批与命令级策略叠加。

技术栈上，OpenClaw 主仓库是 **ESM TypeScript**，主仓库 `package.json` 显示包版本为 `2026.3.9`。

---

## 当前取得的成就

- **开源影响力**：GitHub 史上增长最快的开源项目， **304k stars**、**57.5k forks**、**18,060 commits**
- **云厂商与平台承接**：至少包括 **AWS**、**腾讯云**、**火山引擎**、**Kimi**、**华为云**、**百度智能云**、**微信侧内测的 QClaw**
- **社区活动**：真格基金 / 火山引擎 / 多个海外社区 Meetup
- **商业生态信号**： OpenAI、Vercel、Blacksmith、Convex 等 sponsors。

---

## 技术栈与依赖分解

### 构建系统与工程组织

- **monorepo**：pnpm workspace 聚合 `ui`、`packages/`*、`extensions/*`。
- **插件系统**：官方文档明确插件以 TypeScript 模块形式加载，并通过 `jiti` 在 runtime 加载（是热加载吗？）；插件运行在网关进程内。

### 依赖关系图

```mermaid
graph TD
  A[OpenClaw Core TypeScript/Node] -->|runs on| N[Node.js >=22.12.0]
  A -->|pkg mgr| P[pnpm 10.23.0 workspace]

  A --> PI[Pi SDK runtime<br/>@mariozechner/pi-*]
  A --> GW[Gateway WS control plane]
  A --> PL[Plugin runtime in-process]
  A --> MEM[Memory subsystem]
  A --> CH[Channel adapters]
  A --> OUT[Outbound delivery + streaming]

  GW --> WS[ws WebSocket]
  GW --> SCHEMA[@sinclair/typebox + ajv]
  GW --> PROTO[Protocol versioning + codegen]

  PL --> JITI[jiti dynamic TS loader]
  PL --> HOOK[src/plugins/types.ts hooks/types]

  MEM --> MD[Markdown files as source of truth]
  MEM --> SQLITE[SQLite store]
  MEM --> VEC[sqlite-vec vec0 virtual table]
  MEM --> FTS[FTS5/BM25 when available]
  MEM --> LEMB[node-llama-cpp optional local embeddings]

  OUT --> CHUNK[EmbeddedBlockChunker]
  OUT --> IMG[sharp]
  OUT --> PDF[pdfjs-dist]
  OUT --> WEB[playwright-core]

  CH --> TG[grammy Telegram]
  CH --> SL[@slack/bolt]
  CH --> WA[@whiskeysockets/baileys]
  CH --> DC[@discordjs/voice]

  style A fill:#fff,stroke:#333
```

> 注：图中外部平台 SDK 节点仅用于说明依赖"类别"；具体启用与否由配置与插件决定，在此不展开部署/场景。

---

## Agent 工作区配置层

OpenClaw 将一组位于 Agent workspace 中的 Markdown 文件作为**标准文件面（standard workspace file map / bootstrap files）**。这些文件会在会话启动、系统提示词拼装、首次引导或 heartbeat 轮询等不同阶段参与注入，共同决定 Agent 的行为、人格、用户画像、启动流程与记忆入口。
它们构成的是 **prompt-facing 的工作区配置层**，但并不覆盖全部 runtime 与 security 配置；例如工具权限、工具 profile、provider 级限制、路由与渠道白名单，主要仍由 `~/.openclaw/openclaw.json` 控制。

### 工作区文件职责

| 文件           | 是否属实   | 更准确的官方含义                                                                          | 关键纠偏                                                           |
| -------------- | ---------- | ----------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| `AGENTS.md`    | 是         | Agent 的 operating instructions，定义规则、优先级、如何使用 memory；每个 session 都会加载 | 不只是"职责声明"，更像工作守则 + 操作约束                          |
| `SOUL.md`      | 是         | Persona / tone / boundaries；作为持续注入的角色设定，影响 system prompt                   | "个性化提示词"这个说法基本正确                                     |
| `TOOLS.md`     | 部分正确   | 记录本地工具、设备别名、SSH 主机、环境约定等使用说明                                      | **不直接控制工具权限**；真正 allow/deny/profile 在 `openclaw.json` |
| `IDENTITY.md`  | 是         | 记录 agent 的 name / vibe / emoji / avatar；可同步到 `agents.list[].identity`             | 不只是"展示"，还影响 mention pattern、默认 ack reaction、头像      |
| `USER.md`      | 是         | 用户档案、称呼、时区、上下文偏好；每 session 加载                                         | "用户偏好，上下文先验"基本正确                                     |
| `HEARTBEAT.md` | 是（可选） | heartbeat 轮询时读取的小型 checklist                                                      | 不是通用会话配置，而是**定时巡检任务的提示补丁**                   |
| `BOOTSTRAP.md` | 是         | 首次启动时的一次性 onboarding ritual；完成后删除                                          | "一次性消费"正确，而且官方强调它不应在后续重启反复生成             |
| `MEMORY.md`    | 是（可选） | 长期记忆文件；正常会话可注入，同时也是 `memory_search` 的索引源之一                       | **不是唯一 RAG 源**；`memory/**/*.md` 同样是主要记忆真源           |

### `AGENTS.md` 的核心内容

`AGENTS.md` 是 Agent 在当前 workspace 中的总工作守则，作用类似 operating manual。官方模板中的核心内容主要包括：

- **首次运行规则**：若存在 `BOOTSTRAP.md`，优先完成首次 onboarding，再删除该文件，避免后续重复消费。
- **每次会话的启动动作**：启动时依次读取 `SOUL.md`、`USER.md`、当天与前一天的 `memory/YYYY-MM-DD.md`；主会话额外读取 `MEMORY.md`。
- **记忆写入原则**：要求把值得保留的信息落到文件，而不是依赖模型隐式记忆；原始事件与近期上下文写入 daily memory，长期稳定事实再沉淀到 `MEMORY.md`。
- **安全边界**：默认不得泄露隐私、不得擅自执行破坏性操作、优先使用可恢复删除而不是直接 `rm`。
- **外部动作策略**：读文件、整理工作区、查询资料等内部动作可直接执行；对外发消息、发邮件、公开发布内容等离开本机边界的动作应先询问。
- **群聊行为规范**：不是每条消息都必须响应；被点名、能提供有效帮助、或需要纠正重要错误时再发言；支持 reaction 的平台优先使用 reaction 降低噪声。
- **工具与技能约定**：技能通过 `SKILL.md` 扩展 agent 的任务能力，本地环境与工具使用说明通常由 `TOOLS.md` 补充。
- **Heartbeat 约定**：收到 heartbeat poll 时先查看 `HEARTBEAT.md`；可执行巡检、整理、记忆维护等低打扰任务；无事可做则返回 `HEARTBEAT_OK`。

因此，`AGENTS.md` 不是单纯的身份声明，而是将**启动流程、读写记忆约定、安全边界、外部动作策略、群聊行为与 heartbeat 行为**统一收拢到一个工作规则文件中。

### `BOOT.md`：容易被忽略的启动文件

除上述文件外，`Agent Workspace` 文档还列出一个容易与 `BOOTSTRAP.md` 混淆的文件：

- `BOOT.md`：**可选启动清单**，在 gateway restart 时、且 internal hooks 启用时执行

它和 `BOOTSTRAP.md` 的区别很重要：

- `BOOTSTRAP.md` 是**一次性出生引导**
- `BOOT.md` 是**每次启动时的自动启动任务清单**

两者处于完全不同的生命周期。

### 注入时机与生效范围

这些文件在系统中的作用时机并不相同。默认情况下：

- `AGENTS.md`、`SOUL.md`、`TOOLS.md`、`IDENTITY.md`、`USER.md`、`HEARTBEAT.md` 会作为 Project Context 的一部分参与注入。
- `BOOTSTRAP.md` 只在 brand-new workspace 的首次引导阶段注入；引导结束后应删除。
- `MEMORY.md` 或 `memory.md` 在文件存在时可被注入正常会话。
- `memory/*.md` 的日记型记忆**不会自动全量注入**，而是通过 `memory_search` / `memory_get` 按需检索，因此它们更像"外置记忆库"而不是常驻 prompt。
- 文档还特别指出：**subagent session 默认只注入** `AGENTS.md` **和** `TOOLS.md`，因此主会话和子会话的上下文面并不完全相同。

这意味着，上述文件更准确地说是 **"Agent 工作区中的 prompt-facing 配置层 + 记忆层"**，而不是完整的系统配置全集。

### 工具权限与安全边界的真实控制面

`TOOLS.md` 更适合承载"如何使用工具/本地环境长什么样"这类提示词信息；真正生效的工具权限与安全边界主要由 `openclaw.json` 控制，其中包括：

- `tools.profile`：基础 allowlist，例如 `minimal`、`coding`、`messaging`、`full`
- `tools.allow` / `tools.deny`：全局工具白名单/黑名单，deny wins
- `tools.byProvider`：按 provider / model 进一步收紧工具权限
- `gateway.tools.allow` / `gateway.tools.deny`：针对 HTTP `/tools/invoke` 的附加限制

因此：

- `TOOLS.md` 是**提示词层面的工具说明书**
- `openclaw.json` 才是**runtime 层面的实际权限与安全边界**

### 配置层边界

如果将这些文件理解为**Agent 工作区内、直接塑造 agent 行为/人格/记忆/启动流程的核心 Markdown 文件**，这个范围是成立的。
但从整个 OpenClaw 系统看，核心配置面并不止这些文件，至少还包括：

- `openclaw.json`：真正的 runtime、security、tool policy、gateway、channels、models 配置中心
- `BOOT.md`：官方工作区文件图里存在，但常被社区帖子漏掉
- `memory/` 目录：记忆系统的重要主体，不应被 `MEMORY.md` 单文件替代

因此，这组文件更适合被定义为：**OpenClaw Agent Workspace 的核心文档配置层**，而不是整个系统的完整核心配置。

> 依据：OpenClaw 官方文档 `Agent Workspace`、`Agent Runtime`、`System Prompt`、`Agent Bootstrapping`、`Heartbeat`、`Memory`、`Tools`、`Configuration Reference`、`agents` CLI。

---

## 记忆系统

### 会话与状态数据结构

- **会话历史**：Compaction 文档说明"将较老对话总结为 compact summary entry，并保存在 session JSONL history 中"，即会话记录以 JSONL 形式持久化，并在需要时进行压缩。
- **记忆真源**：Memory 文档强调"记忆是 Agent 工作区中的 Markdown 文件；文件是真源"，并给出默认两层结构（`memory/YYYY-MM-DD.md` 日志 + `MEMORY.md` 长期记忆）。
- **记忆索引**：官方文档指定每 Agent 一个 SQLite 文件（默认 `~/.openclaw/memory/<agentId>.sqlite`），并在可用时使用 sqlite-vec 的 `vec0` 虚表加速向量检索。

### `MEMORY.md` 的准确定位：长期记忆入口，不等于全部记忆系统

`MEMORY.md` 适合被理解为长期记忆入口，而不是全部记忆系统本身。更精确的说法是：

- `memory/**/*.md` 用于记录按时间或任务沉淀的原始记忆与过程日志，`MEMORY.md` 用于沉淀经过筛选、具有长期复用价值的稳定记忆。
- `MEMORY.md` 是**长期、整理后的核心记忆**，适合存放稳定偏好、长期决策、重要背景。
- `memory/YYYY-MM-DD.md` 是**逐日原始日志**，记录近期事件、上下文、未沉淀事实。
- `memory_search` 的索引范围默认覆盖 `MEMORY.md` + `memory/**/*.md`；所以 RAG 源是"长期记忆 + 日志记忆"的组合，而不是单个文件。
- 官方还强调 Markdown 文件才是真源，SQLite 只是派生索引与缓存；索引可重建，文件才是长期真相。
- 是否写入 memory，以及写入 `memory/YYYY-MM-DD.md` 还是 `MEMORY.md`，主要由系统提示词与工作区规则驱动模型自行判断；OpenClaw 提供的是文件真源、记忆检索与静默 memory flush 提醒，而不是固定的后台归档器。
- 关于"何时提醒模型写记忆"，OpenClaw 采用的是**每轮常驻规则 + compaction 前额外提醒**的方式：工作区规则文件会在每个 agent turn 参与 system prompt 组装，因此记忆写入原则不是只在 session 开头出现一次；此外在 session 接近 auto-compaction 时，系统还会触发一次 silent memory flush，额外提示模型把值得长期保留的信息写入记忆文件后再压缩上下文。

所以从系统设计看，`MEMORY.md` 更像是"始终值得优先看的 curated memory"，而 `memory/` 目录才是完整回忆库。

### 检索算法

官方 Memory 文档把检索拆成四层：向量相似度、BM25 关键字、加权融合、可选后处理（时间衰减 + MMR）。

**混合检索融合（Hybrid BM25 + Vector）**：

这一步的目标是把"关键词命中"和"语义相近"结合起来：BM25 更擅长找 ID、命令名、变量名等字面匹配，向量检索更擅长找"说法不同但意思接近"的内容。

1. **候选池**：向量取 top `maxResults * candidateMultiplier`（余弦相似度），BM25 取 top `maxResults * candidateMultiplier`（FTS5 BM25 rank，越小越好）。
2. **BM25 rank → 分数**：`textScore = 1 / (1 + max(0, bm25Rank))`。
3. **加权融合**：`finalScore = vectorWeight * vectorScore + textWeight * textScore`（配置层会将两者归一化使其"像百分比"）。

**MMR（Maximal Marginal Relevance）重排**：

MMR 的作用不是单纯追求"最相关"，而是在相关性之外压制重复结果，避免返回多条内容高度相似的 chunk。

- 迭代选取使 `λ * relevance − (1−λ) * max_similarity_to_selected` 最大的条目；相似度用 tokenized 文本的 Jaccard 相似度度量。

**时间衰减（Temporal Decay）**：

时间衰减用于给较新的记忆更高权重，避免半年前的一条老记录因为措辞更完整而长期压过最近更新的信息。

- 文档给出指数乘子：`decayedScore = score × e^(-λ × ageInDays)`，其中 `λ = ln(2) / halfLifeDays`。

### sqlite-vec 加速与回退

`sqlite-vec` 本质上是 SQLite 的向量检索扩展；有它时，向量距离计算尽量在数据库内部完成，没有它时再退回 JS 内存侧计算。

- 官方文档：若 sqlite-vec 可用，则将 embedding 存入 SQLite `vec0` 虚表并在 DB 内做距离检索；若不可用回退到 JS 内存中做余弦相似度并排序。
- PingCAP 的深挖文章（第三方，但引用到目录与数据表结构）强调这种"把向量距离计算下推到数据库"可以保持 Node 进程轻量，并通过 FTS5 + vec 扩展组成单文件 RAG 栈。

---

## 调度与自动化机制

### 车道队列与并发调度

官方 "Command Queue" 描述了一个 **lane-aware FIFO**：每条 lane 有并发上限；`runEmbeddedPiAgent` 先以 session key 进入 `session:<key>` lane，保证同 session 同时只有一个活跃运行；随后再进入全局 lane（默认 `main`）限制总体并发。

源码中，`runEmbeddedPiAgent` 计算 session lane 与 global lane 的方式非常直接：

```ts
const sessionLane = resolveSessionLane(...);
const globalLane = resolveGlobalLane(...);
```

其中 lane 规范在 `src/agents/pi-embedded-runner/lanes.ts`：若 key 不以 `session:` 开头则自动加前缀。

更完整的队列行为（steer/followup/collect/steer-backlog 等模式、debounce、cap、drop 策略）在文档里定义为"消息到运行回合"的调度语义（本质上是对队列输入流的合并与抢占策略）。

**伪代码（车道队列核心）**：

```text
function enqueue_run(sessionKey, globalLane="main"):
  laneSession = "session:" + normalize(sessionKey)
  enqueue(laneSession, () => enqueue(globalLane, () => run_attempt(sessionKey)))
```

该设计的关键性质：

- **互斥**：同一 session 串行化（避免并发触碰同一会话文件/日志/共享资源）。
- **限并发**：全局 lane 对总并发封顶（避免 LLM 调用风暴与上游限流）。

### Heartbeat 与 Cron

围绕 OpenClaw 的自动化，社区里最容易混淆的是：把 heartbeat 理解成"agent 内部 `while true` 循环"，再把 cron 理解成"heartbeat 每次 tick 顺手检查的日历规则"。源码和官方文档显示，这个说法只对了一部分，需要做几处关键纠偏。

#### Heartbeat：Gateway 驱动的周期性巡检

OpenClaw 的 heartbeat 由 Gateway 的 heartbeat runner 定期触发，核心配置是 `agents.defaults.heartbeat.every`（默认 `30m`）。触发后，系统会调用 `runHeartbeatOnce(...)`，解析目标 session、生成 heartbeat prompt、再跑一次正常的 agent turn。

更准确地说，heartbeat 是：

- **Gateway 侧的周期性调度**，不是 LLM 内部常驻线程；
- **默认跑在 main session**，但也可以通过 `heartbeat.session` 指到某个明确 session key；
- **复用正常 agent turn 管线**，因此依然会进入 system prompt、workspace files、工具调用、reply 解析等常规路径；
- **有跳过机制**：若主队列正忙、处于 quiet hours、或 `HEARTBEAT.md` 存在但实际上为空，则这次 heartbeat 可被跳过，以节省 API 调用。

此外，heartbeat 的 prompt 合同也非常明确：如果没有需要上报的内容，应返回 `HEARTBEAT_OK`。实现里还会对这类纯 ACK 结果做 strip/prune，避免无信息量 heartbeat 把主会话 transcript 污染得越来越长。

因此，heartbeat 更接近：

> **主会话中的周期性巡检 turn**

而不是一个抽象的：

> `while True: sleep(interval); agent.step()`

后者作为概念类比可以帮助理解，但并不等同于 OpenClaw 的实际实现。

#### Cron：持久化任务调度器

OpenClaw 的 cron service 独立存在于 Gateway 内部，任务会持久化到 `~/.openclaw/cron/jobs.json`。它支持三类 schedule：

- `at`：一次性时间点；
- `every`：固定间隔；
- `cron`：5/6 段 cron 表达式，可带时区。

所以，如果从调度表达力看：

- **heartbeat** 的确主要是 **interval-based**；
- **cron** 则不是单纯的 absolute timestamp，它同时覆盖 **one-shot timestamp、fixed interval、calendar rule** 三类计划任务。

这意味着"heartbeat 用相对 interval，cron 用绝对 timestamp"这个理解只能算 **部分正确**。更准确的表述是：

- heartbeat：**周期性巡检机制**
- cron：**持久化任务调度机制**

#### Cron 的执行模式：main 与 isolated

这是最重要的纠偏点之一。OpenClaw 的 cron job 分为两类：

1. **Main session job**
   - `sessionTarget: "main"`
   - `payload.kind: "systemEvent"`
   - 执行方式不是直接起独立 agent turn，而是先把文本作为 system event 入队，再根据 `wakeMode` 选择立刻唤醒 heartbeat，或等待下一次 heartbeat。

2. **Isolated job**
   - `sessionTarget: "isolated"`
   - `payload.kind: "agentTurn"`
   - 在 `cron:<jobId>` 命名空间下启动独立 agent turn；
   - isolated run 会强制 `forceNew`，每次执行都拿 fresh session id，避免历史上下文与 delivery metadata 污染新一轮任务。

因此，"heartbeat 在主 session，cron 在隔离 session"并不是普遍真理。只有 **isolated cron** 才符合这个说法；而大量 reminder/wakeup 型 cron，其实是 **投递到主 session** 并借由 heartbeat 消化的。

#### 协作模型：两个并列机制

很多介绍会写成：

```text
heartbeat tick -> check cron -> execute task
```

但 OpenClaw 的实现更接近下面这个模型：

```text
Gateway
 ├─ Heartbeat runner
 │   └─ 定期或被唤醒时，在主 session 发起 heartbeat turn
 │
 └─ Cron service
     ├─ 计算哪些 job due
     ├─ main job: enqueue system event -> wake heartbeat
     └─ isolated job: 直接启动 cron:<jobId> agent turn
```

也就是说：

- **cron scheduler 自己计算 due job**，不是 heartbeat 顺手维护的一张附表；
- **main-target cron** 会借助 heartbeat 完成主会话语义；
- **isolated cron** 则完全可以绕过 heartbeat，直接起独立运行。

两者真正的关系不是"heartbeat 是调度器，cron 是规则"，而是：

- heartbeat：**主会话巡检/唤醒执行机制**
- cron：**持久化定时任务调度机制**
- 两者通过 **system event + wake** 这座桥梁在 main-session 场景下耦合

#### 任务分流方式

这里并不存在一个单独的、写死在代码里的"自然语言任务分类器"。OpenClaw 更像是把选择权交给模型：

- system prompt 里显式暴露了 `cron` 工具，并明确写着它用于 **reminders / wake events**；
- `HEARTBEAT.md` 本质只是 workspace 里的普通文件，想用 heartbeat 做自动化，通常是让 agent 去**编辑 checklist**，而不是调用一个专门的 heartbeat 创建 API；
- 官方模板和文档也明确建议：**精确定时、一次性提醒、独立任务用 cron；多个周期检查打包进 `HEARTBEAT.md`。**

所以，当用户说"帮我添加一个定时任务"时，模型通常会优先走 cron，原因很实际：

- 这是现成的一等工具接口；
- reminder/schedule 的 tool affordance 很强；
- 不需要模型先推断一个 checklist 应该怎样改写；
- 可直接表达 `at` / `every` / `cron` / `wakeMode` / `delivery` / `sessionTarget`。

反过来，如果你明确说的是：

- "把这件事加入 heartbeat checklist"
- "以后每次 heartbeat 顺便检查这个，不需要精确时刻"

那模型才更可能去改 `HEARTBEAT.md`，让任务进入 heartbeat 体系。

#### 小结

对 OpenClaw 的 heartbeat 与 cron，较为严格的总结应是：

- **Heartbeat**：Gateway 驱动的、默认运行在主会话中的周期性 agent turn；通过固定 prompt 和 `HEARTBEAT_OK` 合同，实现低噪声的巡检与提醒。
- **Cron**：Gateway 内建、持久化的任务调度器；支持 `at` / `every` / `cron` 三种 schedule，并可选择 main-session system event 或 isolated agent turn 两种执行语义。
- **两者的衔接点**：main-session cron job 通过 `enqueueSystemEvent + wake heartbeat` 进入主会话；isolated cron job 则直接在 `cron:<jobId>` 里运行。
- **任务分流方式**：主要不是硬编码 classifier，而是模型依据系统提示、文档约定和工具可供性，在 `cron` 工具与 `HEARTBEAT.md` 编辑之间自行选择。

---

## 文本分块与流式输出

这里的分块主要面向 **流式输出到 UI 与各类 chat channels**，而不是 RAG 建库阶段的文档切片。若只是单一网页 chat，前端通常可以直接把 token 增量渲染出来；但 OpenClaw 需要把同一份长回复稳定投递到多种客户端与消息渠道，因此必须控制单次输出大小、寻找适合消息落点的自然断点，并避免在 Markdown 尤其代码围栏中间切断，导致下游渲染异常或平台兼容性问题。

官方 "Streaming and Chunking" 给出分块器 `EmbeddedBlockChunker` 的设计：

- **低水位**（minChars）：缓冲不足不发；
- **高水位**（maxChars）：优先在 maxChars 前选择断点，否则硬切；
- **断点优先级**：段落 → 换行 → 句子 → 空白 → 硬切；
- **代码围栏**（fence）：不在围栏内切；被迫切时会闭合并重开围栏。

源码 `src/agents/pi-embedded-block-chunker.ts` 进一步落实为：

- 在选择断点前解析 `fenceSpans`；
- `breakPreference` 默认 `"paragraph"`；
- 当触达 `maxChars` 且位置落在 fence 内，返回 `fenceSplit`（close/reopen 行）。

**伪代码（分块断点选择）**：

```text
function pick_break(buffer, minChars, maxChars, preference):
  if len(buffer) < minChars: return NONE
  fenceSpans = parseFenceSpans(buffer)
  for mode in preference_order(preference):
    idx = find_safe_break(buffer, mode, fenceSpans, minChars, maxChars)
    if idx found: return idx
  if len(buffer) >= maxChars:
    if safe_at(maxChars): return maxChars
    if inside_fence(maxChars): return (maxChars, close_fence, reopen_fence)
    return maxChars
  return NONE
```

该算法的准确性/稳定性要点：

- "安全断点"的定义依赖 fence span 的解析质量；若 fence 解析失败，可能在 Markdown 中产生不闭合围栏，影响下游渲染与平台兼容。
- 复杂度：每次 `pickBreak` 都可能对窗口进行多次 `indexOf/lastIndexOf` 扫描与 fence 解析；在高频流式输出时会成为 CPU 热点候选（见后文优化）。

---

## 性能与优化技术

### 已采用的优化手段

- **并发控制与资源竞争消减**：lane-aware FIFO 队列确保单会话串行、跨会话限并发；并明确"无外部依赖/无后台 worker 线程，纯 TS + promises"。
  之所以采用这种轻量的进程内调度机制，而不是一开始就引入独立 worker/外部队列系统，主要是因为 OpenClaw 的核心负载大量表现为 I/O 密集型编排：模型调用、WebSocket 事件收发、会话/记忆文件读写、工具结果回传等都更适合由单个 Node.js 主进程统一协调。这样做可以减少跨进程通信与状态同步成本，使 session 状态、队列顺序、流式输出和权限/审批语义保持一致；代价则是 CPU 密集任务更容易阻塞事件循环，因此后续是否引入 worker pool 取决于真实 profiling 结果。
- **上下文引擎重用**：`run.ts` 明确"resolve once and reuse across retries"，避免在多次重试/失败转移中重复初始化上下文引擎。
- **向量检索加速**：sqlite-vec 路径把距离计算迁移到 SQLite，避免把所有 embedding 拉进 JS 计算。
- **嵌入缓存与异步索引**：Memory 文档描述 embedding cache（SQLite 缓存）、文件变更 debounce、后台异步 sync，且 `memory_search` 不阻塞索引更新（结果可轻微陈旧）。
- **分块器的"格式保持"**：围栏闭合/重开避免平台侧渲染异常，属于"正确性驱动的性能折中"（宁可多做 fence span 解析与分割，也要保证输出可消费）。

### 性能关键路径与潜在热点

结合 `runEmbeddedPiAgent` 所在文件规模（约 1.5k LOC）与其承担的职责，可将热点拆为三类：

- **控制面热点：协议/事件与校验**
  - Gateway WS 协议支持 roles/scopes/caps、版本协商与 schema 生成；这类路径通常包含 JSON schema 校验与路由分发。
  - 建议关注：Ajv 校验器是否预编译并缓存（避免每帧重复 compile）。
- **Runtime 热点：队列、重试、认证故障转移**
  - `run.ts` 包含多处 `await` 与失败转移逻辑（例如 auth profile 在冷却窗口内跳过、必要时探测与回退），属于"复杂状态机 + IO"混合热点。
  - 建议关注：把"纯 CPU 的字符串/数组处理"与"外部 IO/网络调用"分隔，以便更准确的 profiling。
- **内容处理热点：分块与检索**
  - 分块器需要做 fence spans 解析与多级断点搜索；流式输出高频触发时，`indexOf/lastIndexOf` 与正则（例如 `\s`）会占据显著 CPU。
  - 记忆检索在"sqlite-vec 不可用"路径会退化为 JS 侧对 N 个 chunk 做余弦相似度并排序（O(N·d) + O(N log N)），对大 corpus 明显敏感。

### 具体微优化建议

以下建议以"保持语义不变"为前提，且尽量指向可定位的代码区域/函数名：

- **分块器（EmbeddedBlockChunker）**
  - 增量维护 fence spans：当前实现多处对 buffer 调用 `parseFenceSpans(buffer)`；在 append 模式下可以考虑"追加式解析/缓存上一次 fence 状态"，将平均复杂度从"每次 O(n)"降到"每次 O(Δn)"（代价是实现复杂度上升与边界 bug 风险）。
  - 减少 window 切片与重复扫描：`window = buffer.slice(0, min(maxChars, len))` 后再做多轮扫描，可考虑在同一 pass 中同时寻找段落/换行/句子候选（但需保持 breakPreference 的优先级语义）。
- **记忆检索（Hybrid + 后处理）**
  - 若 sqlite-vec 不可用：在 JS 余弦相似度中使用 `Float32Array` 存放 embedding，并以手写循环计算 dot/norm，避免对象数组与高阶函数分配；可进一步尝试"partial top-k"（例如使用最小堆或 nth_element 思路）避免全量排序。
  - MMR 中的 Jaccard 相似度：对每个 chunk 反复 tokenization 会放大成本，建议缓存 token set（或 Bloom/bitset 近似）并在 re-rank 迭代中复用。
- **队列与长任务隔离**
  - 当前队列强调"纯 TS + promises、无 worker threads"；当单次回合涉及大量 CPU（PDF 解析、图像处理、浏览器自动化回调等）时，可能阻塞事件循环并影响心跳/其他会话。官方 issue 中已有"用 worker thread pool 承载 agent 执行"的提案，可作为进一步研究方向。

---

## 已公开安全事件与现实攻击面

2026 年 1 月到 2 月，OpenClaw 暴露出一轮密集安全问题，主要集中在 **gateway 控制面、命令拼接、`system.run` 审批/allowlist、`safeBins` 与 sandbox 执行边界**。

### 2026 年初公开披露的代表性核心漏洞

| CVE / ID            | 漏洞名称                           | 类型         | 更准确的影响说明                                                                                                                                                                   |
| ------------------- | ---------------------------------- | ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `CVE-2026-25253`    | ClawJacked / 1-Click RCE           | 远程代码执行 | 攻击者可利用 Control UI / `gatewayUrl` / token 相关链路，从恶意网页发起对本地 OpenClaw 的未授权控制。对受影响版本而言，**仅仅绑定 localhost 并不足以完全缓解浏览器中转式攻击面**。 |
| `CVE-2026-24763`    | Docker sandbox command injection   | 命令注入     | Docker sandbox 命令构建路径存在注入问题，攻击者可借特制输入把原本应在受控容器环境中的执行，扩展为任意命令执行。                                                                    |
| `CVE-2026-25157`    | macOS SSH path command injection   | 命令注入     | macOS 节点/SSH 命令组装路径存在注入风险，攻击者可通过恶意构造路径或参数，向 OpenClaw 通过 SSH 发起的命令中拼入额外命令。                                                           |
| `JVNDB-2026-004709` | `system.run` access control bypass | 权限绕过     | 该 JVN 记录对应的核心问题是 `system.run` 的 `rawCommand` / `command` 一致性与 allowlist/approval 校验缺陷；从 CVE 视角，更接近 `CVE-2026-26325` 这一类的审批/允许列表绕过。        |
| `CVE-2026-28363`    | `safeBins` validation bypass       | 安全策略绕过 | `tools.exec.safeBins` 的验证与解析存在缺陷，攻击者可把本应只允许 stdin-only 小工具的机制，扩展成危险命令执行入口，从而绕过原有 exec 安全预期。                                     |

- **生态层风险**：ClawHub 是公开 skill registry，且默认开放；第三方 skill 应按不可信代码对待，skill 投毒属于真实供应链风险。
- **部署面风险**：2026 年初公网暴露实例至少达到数万级；较强的一手公开基线约为 **21,639**，部分二级报道给出 **40,000 到 42,000+** 的估计。
- **总体判断**：OpenClaw 的主要安全挑战不在 prompt injection 本身，而在强执行面平台的认证、审批、allowlist、sandbox 与本机控制边界。2026 年初后，项目已明显进入持续加固阶段。

---

## 差距、风险与开放问题

### 记忆索引一致性与恢复策略

issue 报告在更新后 memory-core 索引/持久化出现"停止更新/变脏后无结果"等问题，提示索引状态机（dirty/flush/sync）在升级、多进程竞争或权限变化下可能缺少强一致恢复路径。需要进一步审计 `src/memory/*` 的 schema/versioning 与迁移策略。

### 事件循环阻塞与并行化开放问题

当前队列实现刻意避免 worker threads，并强调纯 promises；这使得 CPU 密集型任务（例如大规模 chunker 扫描、JS fallback 向量相似度、PDF/图像处理）可能阻塞事件循环，影响心跳/多会话响应。社区已有"将 agent 执行迁移到 worker pool"的提案，后续应以 profiling 数据决定是否引入多线程（并评估跨线程传递大对象的开销）。
