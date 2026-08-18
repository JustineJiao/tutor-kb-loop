---
name: tutor-kb-loop
version: 0.1.0
description: 导师画像×知识库互补循环优化：缺口审查→下载→转换→消化→反馈写回画像，螺旋收敛。当用户要求「跑一轮互补循环」「缺口审查」「知识库还缺什么文献」「文献缺口台账」「收敛判定/循环状态」「下载文献 X」「反馈写回画像」时使用。编排于 llm-wiki 之上，消费其消化/查询/状态能力并回写导师画像。
---

# tutor-kb-loop — 知识库 × 导师视角 互补循环

## 核心理念

- **导师视角 = 脑**：导师画像定义领域判断标准与理想态（该有什么基因/路线/数据/缺口）。
- **知识库 = 体**：提供实证事实与数据（哪些发现验证过、哪些带置信度标注）。
- **互为检验**：视角缺事实支撑会飘，事实缺视角组织会散。
- **循环**：视角定义理想态 → 缺口审查找差距 → 补文献 → 新事实反馈写回画像 → 视角锐化 → 找新差距 → 螺旋收敛。

**一句话定位**：导师视角定义领域「应该有什么」，知识库提供「有什么」；差距即文献缺口，补齐后反哺视角，视角锐化后再找新差距——两者互相喂养、螺旋收敛。

## 依赖 skill / 工具

| 依赖 | 用途 | 回退方案 |
|------|------|---------|
| `llm-wiki` | Step 4 消化（ingest/batch-ingest）、知识库定位（cwd/.wiki-schema.md/`~/.llm-wiki-path`） | 直接按 wiki schema 手动消化 |
| 文献检索/下载（如 paper-search MCP / WebSearch） | Step 2 下载全文 | 用户手动提供 PDF |
| anydoc（llm-wiki 内置 venv：`C:/Users/13986/.venvs/anydoc/Scripts/python.exe`） | Step 3 PDF→Markdown（表格还原优先） | `markdown-converter`（markitdown） |
| `nuwa-skill`（可选） | 建库前置：新领域蒸馏 Top5 导师画像；骨架包「待蒸馏」引导 | 跳过蒸馏，用现有导师包 |

## 工作流总览

**不触发**：仅要求"总结这篇文章"、"查询某概念"（走 llm-wiki）。

| 用户意图关键词 | 工作流 |
|---|---|
| "跑一轮互补循环" / "执行一轮" | **loop**（主循环 5 步串行） |
| "缺口审查" / "知识库还缺什么文献" | **gap-audit**（Step 1 单独执行） |
| "下载文献 X" / "补这篇文献" | **fetch**（Step 2+3） |
| "反馈写回画像" / "更新导师画像" | **feedback**（Step 5） |
| "循环状态" / "收敛了吗" / "台账" | **status**（读台账+轮次日志） |

---

## 通用前置检查（除 status 外均先执行）

1. **定位知识库**（复用 llm-wiki 逻辑）：
   - 当前工作目录含 `.wiki-schema.md` → 用它作为知识库根路径
   - 否则读 `~/.llm-wiki-path`（`C:\Users\13986\.llm-wiki-path`）
   - 两者都没有 → 提示用户先运行 llm-wiki 的 init，或直接给出知识库路径
2. **读取 `.wiki-schema.md`** 确认语言（`语言：中文`/`语言：English`，缺省 zh），输出语言随库语言
3. **懒初始化**（首次运行自动补建，不阻塞流程，完成后告知用户）：按「文件落位表」检查，缺失即补建 `tutor/loop/`、`tutor/board.md`、`raw/pdfs_original/`、`raw/pdfs_md/`、`wiki/reviews/`
4. **扫描导师包**：列出 `<知识库>/tutor/*-perspective/`
   - 有 `SKILL.md` → **完成包**（可读，作为缺口审查来源）
   - 无 `SKILL.md` → **骨架包**（只有调研底稿，标「待蒸馏」，可引导走 nuwa-skill）
   - 导师包留库内作数据源，**只读取、不激活、不复制注册**（避免双份漂移）

---

## 字段映射表（流程文档概念 → 现状画像字段）

> 流程文档引用的部分字段在 nuwa-skill 画像中不存在，按此表映射；不存在且无映射的落 `board.md`。

| 流程文档概念 | nuwa-skill 画像实际字段 | 落点 |
|:--|:--|:--|
| 科研思维 / 学术决策 / 表达风格 | 核心心智模型 / 决策启发式 / 表达DNA | 个人画像 SKILL.md |
| 关键数据锚点（部分） | 核心心智模型下的证据、决策启发式下的案例 | 个人画像对应章节 |
| 技术路线×矩阵 | （无）→ 新建 | `tutor/board.md` |
| 交叉视角 | 智识谱系（部分承载） | `tutor/board.md` |
| 工作清单 | （无）→ 建议 nuwa-skill 扩展 | `tutor/board.md`（当前落点） |
| 诚实边界 | 诚实边界（✅ 存在） | 个人画像 SKILL.md 末尾 |
| 时间线 | 人物时间线（✅ 存在） | 个人画像 SKILL.md |
| 联合画像 + 加载器 | （无此机制；WorkBuddy 单次激活一个 skill） | `tutor/board.md`（数据文件，不激活） |

> **扩展建议（不落地，仅提议）**：nuwa-skill 的 `skill-template.md` 未来可增「关键数据锚点」「工作清单」两节，届时字段映射表可简化。

---

## 工作流 1：loop（主循环 5 步串行）

触发："跑一轮互补循环"。执行顺序：

1. **通用前置检查**（含懒初始化、扫描导师包）
2. **Step 1｜缺口审查** → 走 gap-audit 工作流，产出/更新 `tutor/loop/ledger.md`。**完成判据**：台账每条含完整字段（ID/文献标识/归属导师方向/为什么需要/来源可追溯性/优先级/状态），四路来源均已读取
3. **Step 2+3｜下载 + 转换** → 走 fetch 工作流，逐条处理台账中 `open` 项。**完成判据**：台账中所有 `open` 项已推进（`downloaded`/`converted`）或标 `blocked`/`declined`
4. **Step 4｜消化** → 走 llm-wiki 的 ingest/batch-ingest（原文与转换稿落位 `raw/pdfs_original/`、`raw/pdfs_md/` 后，把 `raw/pdfs_md/` 下文件喂给 llm-wiki），产物进 `wiki/sources/`，置信度沿用 llm-wiki 标准（EXTRACTED/INFERRED/AMBIGUOUS/UNVERIFIED）。**完成判据**：所有 `converted` 项已入库，`wiki/sources/` 有对应页
5. **Step 5｜反馈写回** → 走 feedback 工作流。**完成判据**：本轮所有 ingested 的新事实已按归属路由写回，`tutor/loop/feedback-log.md` 已逐条登记
6. **收敛判定** → 读 `tutor/loop/log.md` 末尾 2 条：连续 2 轮无新增 P0/P1 + P2 全部显式处置 → 输出「已收敛，转低频维护」；否则输出「未收敛」及剩余项
7. **登记轮次** → 按 `templates/log-template.md` 追加本轮记录

---

## 工作流 2：gap-audit（缺口审查）

触发："缺口审查" / "知识库还缺什么文献"。**输出**：更新 `tutor/loop/ledger.md`。

### 缺口来源（四路合并，缺一不可）

1. **导师画像理想态**：读 `tutor/*-perspective/SKILL.md`（只读完成包）的「核心心智模型 / 决策启发式 / 时间线 / 价值观与反模式 / 智识谱系 / 诚实边界」+ `references/research/01-writings.md`、`05-decisions.md`（若存在）
2. **库内「待补」标注**：Grep `wiki/` 内 `[待创建`、`confidence: AMBIGUOUS`、`confidence: UNVERIFIED`；读 `wiki/reviews/` 下 `status: pending` 项
3. **已知最大缺口**：读 `tutor/loop/ledger.md`（遗留未处理项）+ `tutor/loop/log.md`（最近轮次）
4. **用户当前问题**：本次调用想解决什么、补什么

### 审查方法

1. 逐条对照画像的「核心心智模型 / 决策启发式」问：库里有没有对应 source 支撑？（用**内容特征词**匹配库内 source，不能只看文件名——库有文件名截断/占位件教训）
2. 对照 `board.md`（若无则对照 tutor/README）问：每个导师方向是否都有文献覆盖？哪块最薄？
3. 对照「诚实边界」问：已知缺口补上了吗？
4. 合并四路结果，去重（同一文献多路命中只记一条，标注多归属）

### 台账条目（按 `templates/ledger-template.md`）

每条含：ID | 文献标识（作者+年份+标题/主题）| 归属导师方向（领域整体/具体导师）| 为什么需要（补哪个锚点/交叉视角/清单项）| 来源可追溯性 | 优先级 P0/P1/P2 | 状态

- **来源可追溯性**：A=检索到原文 / B=库内推断缺 / C=仅凭模型记忆（C 级默认降为 P2，除非用户确认）
- **优先级**：P0=影响关键结论/决策 → P1=影响执行质量 → P2=打磨/低价值
- 骨架包对应的方向：在台账标注「待蒸馏（导师包无 SKILL.md）」为 P2 观察项，不阻塞循环

**完成判据**：四路来源全部读取；台账每条含完整字段；无未分类的缺口（每条至少归入一个导师方向 + 一个优先级）。

---

## 工作流 3：fetch（下载 + 转换）

触发："下载文献 X" / loop 中 Step 2+3。对台账中每条 `open` 项：

1. **检索**：用文献检索/下载工具（paper-search MCP 等）获取全文；付费墙尝试替代通道（OA 版本/预印本/作者主页）
2. **验证**：文献标识必须实际检索到（标题+作者+年份匹配），**不靠模型记忆**；检查文件魔数（`%PDF` 等）
3. **落位**：原文 → `<知识库>/raw/pdfs_original/`；文件名合规（去特殊字符）
4. **转换**：优先 anydoc（venv 路径见依赖表）转 Markdown；扫描件需 OCR（不支持则标 `blocked(OCR)`）；老 PDF 内嵌字体提取失败用轻量文本提取兜底；转换后抽查无乱码/截断
5. **落位**：转换稿 → `<知识库>/raw/pdfs_md/`；显式 UTF-8 编码
6. **更新台账**：状态机推进；失败标 `blocked` + 原因（付费墙/OCR/无法获取），**受阻项不算未收敛**。**完成判据**：本条 `open` 项已推进（`downloaded`/`converted`）或标 `blocked`/`declined`

---

## 工作流 4：feedback（反馈写回画像）

触发："反馈写回画像" / loop 中 Step 5。消化出的**新事实**按归属写回。**约束**：只补**带来源的事实性更新**，不改导师的**思维模型本体**。

### 归属路由（防跨智囊团写回冲突）

| 反馈内容 | 写回位置 |
|:--|:--|
| 导师本人的新成果 / 新数据 | 该导师个人画像 `tutor/[person]-perspective/SKILL.md` |
| 领域整体的新路线/新数据/新共识 | 联合画像 `tutor/board.md`（矩阵/交叉视角/工作清单/诚实边界） |
| 新发现的领域缺口 | `tutor/board.md` 的工作清单/诚实边界（作为下一轮 Step 1 起点） |

- 导师可能属于多个智囊团：写回前先检查该导师归属；**同一事实只落一处**，不复制
- **个人事实 → 个人画像，不写联合画像**；**团队间解释/领域判断 → 才写联合画像**
- 个人画像具体映射：新成果 →「人物时间线」追加；新证据 → 对应「核心心智模型」补充证据（不改模型名与定义）；新决策案例 →「决策启发式」追加案例；新局限 →「诚实边界」追加

### 思维模型本体保护

- 模型**名**与**一句话定义**的改动 → 需**强证据 + 人工确认**（强证据 = 一手来源直接陈述其思维/定位变化，如本人访谈/著作原文）
- 证据、应用案例、时间线条目 → 可增补，改动处加注释：
  `<!-- 来源: wiki/sources/<页> | confidence: EXTRACTED | 轮次: R## -->`

### 反馈日志（可追溯性）

每次写回按 `templates/log-template.md` 中的反馈日志表登记到 `tutor/loop/feedback-log.md`（写回文件 / 改动节 / 补入内容摘要 / 来源 source / 日期·轮次）。

---

## 工作流 5：status（状态与收敛判定）

触发："循环状态" / "收敛了吗"。**只读**，不改文件：

1. 读 `tutor/loop/ledger.md`：按 P0/P1/P2 计数、按状态机计数（open/downloaded/converted/ingested/fedback/closed/blocked/declined）
2. 读 `tutor/loop/log.md` 末尾 2 条轮次记录
3. 输出报告：
   - 剩余缺口清单（P0/P1 优先，含 ID/文献标识/状态/受阻原因）
   - **收敛判定**：连续 2 轮无**新增** P0/P1 + P2 全部显式处置（closed/blocked/declined）→ ✅ 已收敛，转低频维护；否则 ❌ 未收敛
   - 骨架包待蒸馏提醒（若有）
4. 若台账不存在 → 提示先跑 gap-audit

---

## 文件落位表

```
<知识库>/
├── tutor/
│   ├── README.md              # 现有导师索引（不动；可追加 board.md 指引）
│   ├── board.md               # 【懒初始化】联合画像：领域共识矩阵/技术路线×方法矩阵/交叉视角/工作清单/诚实边界/更新时间
│   ├── [person]-perspective/  # 现有导师包（只读数据源，不激活）
│   └── loop/
│       ├── ledger.md          # 【懒初始化】缺口台账
│       ├── log.md             # 【懒初始化】轮次记录（含收敛判定）
│       └── feedback-log.md    # 【懒初始化】反馈日志
├── raw/
│   ├── pdfs_original/         # 【懒初始化】文献原文 PDF
│   └── pdfs_md/               # 【懒初始化】文献转换 Markdown
├── wiki/reviews/              # 【懒初始化】llm-wiki 待审项（确认后单向入台账）
└── wiki/sources/              # llm-wiki 消化产物（复用，不另建）
```

## 状态机

```
open → downloaded → converted → ingested → fedback → closed
            │             │           │
            └── blocked（付费墙/OCR/无法获取，标原因）
            └── declined（用户判定不追）
```

- `blocked` / `declined` 均视为「显式处置」，参与收敛判定但不要求闭环到 closed
- 台账 ↔ llm-wiki review 机制为**单向衔接**：review 确认后入台账；台账不反向写 review

---

## 边界与免责（不重复造轮子）

- **复用**：llm-wiki 的 ingest/query/status、置信度标注、SHA256 缓存、anydoc 转换；nuwa-skill 的导师蒸馏与人物包结构；文献检索工具
- **独有增量**：四路缺口聚合、ledger 状态机、反馈路由+去重、收敛判定（含「无新增」维度）、`board.md` 联合画像、`tutor/loop/` 跨会话状态
- **不做**：不替代 llm-wiki 的消化管线；不替代 nuwa-skill 的蒸馏；不激活/复制导师包；不自动修改思维模型本体
- **红线**：删除/覆盖知识库核心数据、修改库用途文档等一律按库规拒绝；写回画像须带来源，无来源不写
