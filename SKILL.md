---
name: project-rearchitecture
description: >
  面向 AI 迭代生成、缺少文档、语义混乱的既有项目，做全项目级语义与架构重架构。
  代码即事实：先从代码逆向出领域模型与术语表，归一命名与概念，再重划架构边界，
  并以行为契约冻结保障功能不变。支持破坏性变更，但必须配套迁移方案。
  触发词：全项目重构、重新架构、语义重构、rearchitecture、语义混乱、命名混乱、
  概念不清、代码和概念对不上、逆向建模、领域模型、术语表、从头重构、重写架构、
  semantic refactor、domain model、ubiquitous language、code as truth。
user_invocable: true
auto_model_invocable: true
---

# 全项目语义级重架构（Project Rearchitecture）

面向「用 AI 一轮轮对话堆出来、没有文档、概念漂移」的既有项目：**在功能不变的前提下，把底层的领域语义与架构边界重新做一遍。**

代码即事实。这里不从需求文档出发，而是**从代码逆向出真实存在的领域模型**，把散落、重复、名实不符的概念收敛成一套清晰的术语与边界。

> [!IMPORTANT]
> **第一原则：没有行为基线，不许动任何一行代码。**
> 现状必须先被钉死，否则你无法证明功能没变。

## 适用与不适用

**适用**：范围是整个项目；你不知道该叫什么、该切在哪；概念在迭代中漂移，代码与业务语义对不上；项目缺文档、缺测试、超过 100 个文件。

**不适用**：范围是单文件或单类；你已知要做什么操作（提取函数、以多态取代条件、搬移字段）。这类改动照常规重构流程走 `references/process.md`，直接执行即可，不需要这里的前置阶段。

---

## 快速索引

| 你现在需要什么 | 参考文件 |
|---|---|
| 冻结现状、建立行为基线（**永远从这一步开始**） | [behavior-freeze.md](references/behavior-freeze.md) |
| 逐模块审计语义、产出术语表与漂移清单 | [semantic-audit.md](references/semantic-audit.md) |
| 从代码逆向出领域模型与模块边界 | [domain-extraction.md](references/domain-extraction.md) |
| 把术语落到代码上、安全地批量改名 | [naming-normalization.md](references/naming-normalization.md) |
| 破坏性变更的迁移方案（版本、脚本、弃用、回滚） | [migration.md](references/migration.md) |
| 跨会话保持进展（>100 文件必读） | [session-state.md](references/session-state.md) |
| 判定与删除死代码 | [dead-code-removal.md](references/dead-code-removal.md) |
| 语义漂移类坏味道清单 | [smells.md 家族 7](references/smells.md) |
| 改名与删除的红线、契约处置规则 | [safety.md §9](references/safety.md) |
| 具体重构手法 | [catalog-composing.md](references/catalog-composing.md) 等 `catalog-*.md` |
| 语言惯用法与测试命令 | [language-profiles.md](references/language-profiles.md) |

---

## 五个阶段

```
阶段 0  行为冻结    →  BEHAVIOR-CONTRACT.md          （硬性前置）
阶段 1  语义审计    →  GLOSSARY.md / SEMANTIC-MAP.md / DRIFT-REPORT.md
阶段 2  目标模型    →  TARGET-MODEL.md / RENAME-MAP.md
阶段 3  架构重划    →  分批的架构改动（catalog-architecture.md）
阶段 4  分批迁移    →  Expand → Migrate → Contract，每批门禁
```

阶段 0 不是可选项。阶段 1 与阶段 2 之间**必须先定词再动代码**——跳过这一步直接改名，通常会在第三轮又改回来。

---

## 决策树

**「这个项目要从头重构」／「语义很乱，代码和概念对不上」**（范围是整个项目）
→ 完整走五阶段。先读 [session-state.md](references/session-state.md) 建立状态文件，再读 [behavior-freeze.md](references/behavior-freeze.md) 冻结现状。项目超过 100 个文件时审计必须分模块切片，一次会话只处理一个切片。

**已有行为基线，只想弄清「这些概念到底是什么」**
→ 读 [semantic-audit.md](references/semantic-audit.md) 与 [domain-extraction.md](references/domain-extraction.md)。产出文档，不改代码。

**概念清楚了，要落到代码上**（改类名／模块名／字段名）
→ 读 [naming-normalization.md](references/naming-normalization.md)。先产出 `RENAME-MAP.md` 并确认，再执行；涉及对外契约的读 [migration.md](references/migration.md)。

**要重划模块与分层边界**
→ 读 [domain-extraction.md](references/domain-extraction.md) 的边界切分法，再读 [safety.md](references/safety.md) §8 架构重构协议，用 `catalog-architecture.md` 与 `catalog-organizing.md` 的手法执行。

**怀疑某段代码是死代码**
→ 读 [dead-code-removal.md](references/dead-code-removal.md)。先确认取证路线：有线上日志／APM → 运行时取证；没有 → 入口可达性判定，此时删除范围必须收窄。

**要删 feature flag 后面的代码**
→ 不要直接删。先关开关、观察一个完整周期，再决定是否删代码。见 [dead-code-removal.md](references/dead-code-removal.md) §D.4 T2。

**用户没说要改契约，但你判断必须改**
→ 停下来问。读 [migration.md](references/migration.md) §1 判定破坏性，给出迁移方案后再动手。

---

## 触发词

**中文：** 全项目重构、重新架构、重架构、语义重构、语义级重构、语义混乱、
命名混乱、概念不清、概念漂移、代码和概念对不上、名实不符、逆向建模、
反推领域模型、领域模型梳理、术语表、统一语言、从头重构、重写架构、
AI 生成的项目整理、这个项目太乱了想重做底层

**English：** `rearchitecture`、`re-architect`、`semantic refactor`、`domain model`、
`ubiquitous language`、`code as truth`、`reverse engineer the domain`、
`restructure the whole project`、`the code and the concepts don't match`、
`naming is inconsistent`、`vibe-coded cleanup`、`AI-generated codebase cleanup`

触发点必然是**项目级**或**语义级**的。「把这段代码清理一下」不属于这里。

---

## 核心契约

1. **功能不变，语义可以变。** 领域概念、命名、模块边界、分层可以重做；用户在外部能观察到的行为（除已批准并配了迁移方案的破坏性变更外）必须一致。
2. **顺序不可倒置：先冻结，再审计，再定词，最后动代码。**
3. **一次一个语义批次。** 一个批次 = 一个可独立验证、独立回滚、测试全绿的变更集；绝不在一个批次里同时改命名和改结构。
4. **破坏性变更四件套缺一不可**：版本提升、迁移脚本、弃用期、回滚步骤。见 [migration.md](references/migration.md) §4.3。
5. **任何结论都要落盘。** 术语表、语义地图、改名映射、状态文件都是仓库里的版本化文件，不是聊天记录。
6. **删代码是存在性变更，不是优化。** `grep` 零命中不算删除依据；入口可达性判定下只允许删高置信度的不可达代码；删除必须单独成批、独立提交、登记观察项。
7. **不因为「代码很难看」就推断它的语义是错的。** 先取证，再判定。

---

## 执行要点

1. **绝不猜测文件路径**：读取或编辑前，先用搜索或列举工具确认真实位置。
2. **优先定向搜索**：用 `grep_search` 摸清标识符分布与调用点数量，不要整目录阅读。
3. **审计阶段禁止编辑业务代码。** 阶段 0–2 只产出文档。
4. **每批次结束更新状态文件**：`REARCH-STATE.md` 与 `AUDIT-PROGRESS.md`。
5. **编辑后先做语法／类型检查，再跑测试**（`tsc --noEmit`、`cargo check`、`mypy`），最后跑行为基线。顺序不要颠倒。
6. **省 Token**：不要整份读取 catalog 文件，先 grep 定位再只看目标行区间。
7. **改名前统计调用点**：超过 20 个属黄线，超过 50 个属红线，见 [safety.md](references/safety.md) §6 大型代码库协议。
