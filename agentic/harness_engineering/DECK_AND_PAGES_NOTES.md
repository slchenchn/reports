# Harness 幻灯片与 GitHub Pages 修改备忘

本文记录对 `deck.html` 与仓库发布相关的排错与版式经验，便于以后改同类全屏幻灯或静态站时复用。

---

## 1. GitHub Pages：`upload-pages-artifact` 与 `AGENTS.md`

**现象**：Actions 里 `actions/upload-pages-artifact` 打 tar 时报错，例如：

`tar: ./AGENTS.md: File removed before we read it`

**原因**：仓库里的 `AGENTS.md` 若是 **Git 符号链接**（`git ls-tree` 显示 mode `120000`），且指向本机绝对路径（如 NFS），在 Linux runner 上目标不存在，`tar` 遍历/读取时会异常。

**处理**：

- 改为普通文件（例如短说明 + 指向 `CLAUDE.md` 的相对链接），或改用 **相对路径** 的 symlink（如 `CLAUDE.md`），不要用机器专属绝对路径。

---

## 2. GitHub Pages：Jekyll 与 `.nojekyll`

**背景**：从分支发布时，GitHub 默认跑 Jekyll；仓库里大量 Markdown、复杂 HTML 时容易构建失败。

**处理**：在站点根目录放空的 **`.nojekyll`**，跳过 Jekyll，按静态文件原样发布。

**注意**：若改为只用 `docs/` 作站点根，需同步调整资源路径或目录结构。

---

## 3. 全屏幻灯 CSS：「方框里下半截全空」

常见成因不是「内容少」，而是 **人为把容器撑高** 再 **把块级内容顶对齐**，中间就全空。

| 成因 | 说明 |
|------|------|
| `min-height: clamp(...vh...)` 过大 | 卡片实际内容高度远小于 `min-height`，视觉上「框很高、字挤在上沿」。 |
| `justify-content: space-between` | 在 **flex 竖排**里只有少量子元素时，会把第一项顶到最上、最后一项沉到最下，中间大块留白（`future-card`、早期的 `loop-card` 等）。 |
| `align-items: start` + 多列 grid | 每列高度各自按内容算，**同一行里**会出现「某一格明显更矮」的情况（失败模式四宫格、middleware 一行三格等）。 |

**处理思路**：

- 需要 **紧凑**：去掉不必要的大 `min-height`，改用 `min-height: 0`，flex 用 `justify-content: flex-start` + 合理 `gap`。
- 需要 **同一行等高外框**：grid 用 `align-items: stretch`（或默认 stretch），子项 `height: 100%` / 作为 grid 子项被拉伸；**内部**仍可用 flex + `flex-start` 让文字顶对齐，空白留在框内下方而不是整块变矮。
- **双栏两个 panel 等高**：父级 `align-items: stretch`，子 panel `display: flex; flex-direction: column; justify-content: flex-start`。窄屏单列时改回 `align-items: start`，避免竖排时第二块被拉得过长。

本项目里 **Cold Start** 双 panel 使用修饰类 **`two-column--dual-panels`** 承载上述行为。

---

## 4. Grid 子项溢出：长英文标识符

**现象**：如 `PreCompletionChecklistMiddleware` 在窄列里 **水平溢出**。

**原因**：Grid 子项默认 `min-width: auto`，子内容最小宽度会撑开列，拒绝缩小。

**处理**：

- 在会进 grid 的卡片容器上设 **`min-width: 0`**（以及需要时的 **`min-height: 0`**）。
- 标题长词：`overflow-wrap: anywhere`、`word-break: break-word`，必要时略减小标题 `clamp` 上限并收紧 `line-height`。

---

## 5. 标题性句子字号：`.quote` 与 `quote--lede`

**现象**：左侧金句若用全局 **`.quote`**，字号是 `clamp(1.6rem, 4.55vw, 4.2rem)` 量级，和右侧列表对比过强，或整页显得「一句话比标题还大」。

**处理**：对 **段落级说明句**使用 **`quote--lede`**（较小一档的 `clamp`），与双栏左列结论句、折旧页 guardrail 等保持一致语义。

---

## 6. 表格列宽（范式四列表）

若四列使用悬殊的 `fr` 比例，容易出现「有的列特别窄、有的特别宽」的观感。

**处理**：表格式布局可用 **`grid-template-columns: repeat(4, minmax(0, 1fr))`** 做等分；`minmax(0, 1fr)` 避免长文撑破网格。

---

## 7. 闭环页箭头与 `loop-layout`

`loop-card` 去掉大 `min-height` 后，若 `loop-layout` 仍 `align-items: center`，箭头与卡片对齐可能需单独处理。

**处理**：`loop-layout` 可对卡片 **`align-items: start`**，箭头单独 **`align-self: center`** 并加一点 **`margin-top`**，兼顾高度与视觉居中。

---

## 8. 本地验收：没有 IDE Browser MCP 时

当前工作区若未启用 **cursor-ide-browser** MCP，无法在对话里直接 DevTools。

**可行做法**：

- 本地 `python -m http.server` 提供目录，用 **Playwright / Puppeteer** 打开 `deck.html`，逐屏截图或对关键节点读 **`getComputedStyle`（如 `minHeight`、`offsetHeight`）**，比纯凭描述猜 CSS 更准确。
- 典型探针：middleware 卡片高度、`loop-card` 的 `min-height` 是否仍被算成上百像素等。

---

## 9. 修改时建议自测的页面索引（`deck.html`）

| 关注点 | 大致位置 |
|--------|----------|
| 四列表格列宽 | 带 `paradigm-table` 的幻灯 |
| 起源双卡、标签字号 | `origin-card` / `.tag` |
| 闭环双卡高度 | `loop-card`、`loop-layout` |
| LangChain 三卡、长标题 | `middleware` / `middleware-list` |
| 失败模式四宫格等高 | `failure-grid` / `failure-card` |
| Cold Start 双 panel 等高 | `two-column--dual-panels` |
| 金句字号 | `.quote` vs `quote--lede` |
| 未来三卡紧凑 | `future-card` / `future-grid` |

---

## 10. 原则小结

1. **先区分**：要「框随内容变矮」还是「同一行外框等高」；二者用的 grid/flex 组合不同。  
2. **`min-height` + `space-between` 是留白大户**，全屏幻灯里尤其要警惕。  
3. **Grid 里一定要有 `minmax(0, …)` / 子项 `min-width: 0`**，否则中英文长词容易把布局撑坏。  
4. **层级字号分开**：展示型 `.quote` 与说明型 `quote--lede` 不要混用。  
5. **CI 与静态站**：符号链接、Jekyll、artifact 打全仓 tar 都会在「本机看着正常」时仍失败，要在仓库层面消除环境差异。
