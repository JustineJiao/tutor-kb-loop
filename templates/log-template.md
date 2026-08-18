# 循环轮次记录（<知识库名>）

> 由 tutor-kb-loop 的 loop 工作流每轮追加。读末尾 2 条做收敛判定：
> **连续 2 轮无新增 P0/P1 + P2 全部显式处置（closed/blocked/declined）→ 已收敛，转低频维护。**
> 受阻项（blocked）不算未收敛。

## R## · YYYY-MM-DD

| 字段 | 内容 |
|:--|:--|
| 驱动者 | 领域导师团（tutor-kb-loop） |
| 缺口台账 | 本轮新增 X 条（P0: a / P1: b / P2: c）；处理前剩余 P0: x / P1: y |
| 下载/转换 | 下载 X 篇 → 转换 X 篇（blocked: N，原因：付费墙/OCR/…） |
| 消化 | source Y → Z（净值；置信度沿用 llm-wiki） |
| 画像反馈 | N 位导师各补 X 处；board.md 补 Y 处（登记于 feedback-log.md） |
| 新增缺口 | （本轮消化暴露的新缺口，若有） |
| 遗留缺口 | 剩余 P0/P1 清单 + P2 处置状态（已补/暂缓/不追） |
| 收敛判定 | 未收敛 / ✅ 已收敛（连续 2 轮无新增 P0/P1 + P2 全部显式处置） |

---

## 反馈日志（feedback-log.md）

| 写回文件 | 改动节 | 补入内容（摘要） | 来源 source | 日期/轮次 |
|:--|:--|:--|:--|:--|
| tutor/[person]-perspective/SKILL.md | 人物时间线 | 新增某成果数据 | `wiki/sources/...` | YYYY-MM-DD · R## |
| tutor/board.md | 技术路线×方法矩阵 | 新增某路线证据 | `wiki/sources/...` | YYYY-MM-DD · R## |

> 规则：只补带来源的事实性更新；模型名/一句话定义改动须强证据 + 人工确认；
> 个人事实 → 个人画像；团队解释/领域判断 → board.md；同一事实只落一处。
