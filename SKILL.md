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

代码即事实。本技能不从需求文档出发，而是**从代码逆向出真实存在的领域模型**，把散落、重复、名实不符的概念收敛成一套清晰的术语与边界。

## 与 `refactor` 技能的分工

| | `refactor` | 本技能 |
|---|---|---|
| 你已知要做什么 | ✅ 它的主场 | 也适用（用它执行具体操作） |
| 你不知道该叫什么、该切在哪 | ❌ 不覆盖 | ✅ 本技能的核心 |
| 范围 | 单文件 / 单类 / 一次操作 | 整个项目（>100 文件亦适用） |
| 对外契约 | 一律不动 | **允许破坏，但必须带迁移方案** |

**不要重复造轮子。** 具体重构操作（提取函数、以多态取代条件、搬移字段等）一律执行
[references/catalog-*.md](references/catalog-composing.md) 中的手法。本技能负责回答
**该叫什么、该切在哪、按什么顺序、怎么不炸**。

> [!NOTE]
> **本技能是自包含的。** `references/process.md`、`references/safety.md`、
> `references/catalog-*.md`、`references/language-profiles.md` 是本技能内的通用层副本，
> 上游来源是姊妹技能 `refactor`（仓库 `code-refactoring-skill`）。
> **通用层以 `refactor` 仓库为准**；本技能自有的六份参考文档
> （`behavior-freeze` / `semantic-audit` / `domain-extraction` /
> `naming-normalization` / `migration` / `session-state`）以本仓库为准。

> [!IMPORTANT]
> **本技能的第一原则：没有行为基线，不许动任何一行代码。**
> 代码即事实，意味着「现状」必须先被钉死，否则你无法证明功能没变。

---

## 快速索引

| 你现在需要什么 | 参考文件 |
|---|---|
| 冻结现状、建立行为基线（**永远从这一步开始**） | [behavior-freeze.md](references/behavior-freeze.md) |
| 逐模块审计语义、产出术语表与漂移清单 | [semantic-audit.md](references/semantic-audit.md) |
| 从代码逆向出领域模型与模块边界 | [domain-extraction.md](references/domain-extraction.md) |
| 把术语表落到代码上、安全地批量重命名 | [naming-normalization.md](references/naming-normalization.md) |
| 破坏性变更的迁移方案（版本、脚本、弃用、回滚） | [migration.md](references/migration.md) |
| 跨会话保持进展（>100 文件必读） | [session-state.md](references/session-state.md) |
| 判定与删除死代码（两条取证路线） | [dead-code-removal.md](references/dead-code-removal.md) |
| 语义类坏味道清单 | [smells.md 家族 7](references/smells.md)（姊妹技能 `refactor`） |
| 语义红线与契约迁移安全规则 | [safety.md §9](references/safety.md) |
| 具体重构手法 | [references/catalog-*.md](references/catalog-composing.md) |
| 语言惯用法与测试命令 | [language-profiles.md](references/language-profiles.md) |

---

## 五个阶段总览

```
阶段 0  行为冻结    →  BEHAVIOR-CONTRACT.md          （硬性前置）
阶段 1  语义审计    →  GLOSSARY.md / SEMANTIC-MAP.md / 漂移清单
阶段 2  目标模型    →  TARGET-MODEL.md / RENAME-MAP.md
阶段 3  架构重划    →  分批的架构改动（用 catalog-architecture.md）
阶段 4  分批迁移    →  Expand → Migrate → Contract，每批门禁
```

阶段 0 不是可选项。阶段 1 与阶段 2 之间**必须先定词再动代码**——这是本技能与
「边看边改」式重构最本质的区别。

---

## 决策树

**「这个项目要从头重构」／「语义很乱，代码和概念对不上」**（范围是整个项目）
→ 完整走五阶段。**先读 [session-state.md](references/session-state.md)** 建立状态文件，
再读 [behavior-freeze.md](references/behavior-freeze.md) 冻结现状。
若项目 >100 文件，审计必须分模块切片，**一次会话只处理一个切片**。

**已有行为基线，要做语义审计**（只想知道「这些概念到底是什么」）
→ 读 [semantic-audit.md](references/semantic-audit.md) +
[domain-extraction.md](references/domain-extraction.md)。产出三件套，**不改代码**。

**概念清楚了，要落到代码上**（改类名/模块名/字段名）
→ 读 [naming-normalization.md](references/naming-normalization.md)。
先产出 `RENAME-MAP.md` 并确认，再执行。破坏性重命名必须读
[migration.md](references/migration.md)。

**要重划模块与分层边界**
→ 读 [domain-extraction.md](references/domain-extraction.md) 的边界切分法，
然后读 [safety.md](references/safety.md) §8 架构重构协议，
用 `catalog-architecture.md` 与 `catalog-organizing.md` 的手法执行。

**发现的是单点坏味道**（一个长函数、一处重复）
→ 别用本技能。直接用 `refactor` 技能。

**怀疑某段代码是死代码，想删掉**
→ 读 [dead-code-removal.md](references/dead-code-removal.md)。**先确认取证路线**：
有线上日志/APM → 运行时取证；无 → 纯本地入口可达性。
**纯本地模式下删除范围必须收窄**（只删 T1 高置信、T5、T6），
且 `grep` 零命中**不构成**删除依据。

**要删 feature flag 后面的代码**
→ **不要直接删。** 先关开关、观察一个完整周期，再决定是否删代码。
见 [dead-code-removal.md](references/dead-code-removal.md) §D.4 T2。

**用户没说要改契约，但你发现必须改**
→ 停下来问。读 [migration.md](references/migration.md) §1 判定破坏性，
给出迁移方案后再动手。

---

## 触发词

**项目级：** `rearchitecture`、`re-architect`、`semantic refactor`、`domain model`、
`ubiquitous language`、`code as truth`、`reverse engineer the domain`、
`restructure the whole project`、`the code and the concepts don't match`、
`naming is inconsistent`、`vibe-coded cleanup`、`AI-generated codebase cleanup`

**中文：** 全项目重构、重新架构、重架构、语义重构、语义级重构、语义混乱、
命名混乱、概念不清、概念漂移、代码和概念对不上、名实不符、逆向建模、
反推领域模型、领域模型梳理、术语表、统一语言、从头重构、重写架构、
AI 生成的项目整理、这个项目太乱了想重做底层

**注意区分：** 「把这段代码清理一下」是 `refactor` 的触发词，不是本技能的。
本技能的触发点必然是**项目级**或**语义级**的。

---

## 核心契约（贯穿全程，不可协商）

1. **功能不变，语义可以变。** 领域概念、命名、模块边界、分层可以彻底重做；
   用户在外部能观察到的行为（除已批准并配了迁移方案的破坏性变更外）必须一致。
2. **先冻结，再审计，再定词，最后才动代码。** 顺序不可倒置。
   跳过阶段 1 直接改名的结果，通常是在第三轮又改回来。
3. **一次一个语义批次。** 一个批次 = 一个可独立验证、独立回滚、测试全绿的变更集。
   绝不在一个批次里同时改命名和改结构。
4. **破坏性变更有四件套，缺一不可**：版本提升、迁移脚本、弃用期、回滚步骤。
   详见 [migration.md](references/migration.md) §3。
5. **任何结论都要落盘。** 术语表、语义地图、重命名映射、状态文件都是仓库里的
   版本化文件，不是聊天记录。跨会话必须靠它们恢复。
6. **删代码是存在性变更，不是优化。** 死代码必须走证据化判定
   （[dead-code-removal.md](references/dead-code-removal.md)）：
   `grep` 零命中不算证据；纯本地模式只允许删高置信度的不可达代码；
   删除必须单独成批、独立提交、登记观察项。

---

## 智能体执行要点

1. **绝不猜测文件路径**：读取或编辑前，先用搜索/列举工具确认真实位置。
2. **优先定向搜索**：用 `grep_search` 摸清标识符分布与调用点数量，不要整目录阅读。
   统计调用点数量是判断重命名影响面的主要手段。
3. **审计阶段禁止编辑业务代码**。阶段 0–2 只产出文档，不碰实现。
   这条纪律能防止「审计着审计着就顺手改了」。
4. **每批次结束更新状态文件**：`REARCH-STATE.md` 与 `AUDIT-PROGRESS.md`。
   会话可以中断，状态不能丢。
5. **编辑后先做语法/类型检查，再跑测试**（例如 `tsc --noEmit`、`cargo check`、
   `mypy`），最后跑行为基线。三者顺序不要颠倒。
6. **省 Token**：不要整份读取 catalog 文件。先 grep 定位，再只看目标行区间。
7. **重命名前必须统计调用点**：超过 20 个调用点属于黄线，超过 50 个属于红线，
   详见 [safety.md](references/safety.md) §6 大型代码库协议。

---

## 本技能不做什么

- 不在建立行为基线之前修改任何业务代码
- 不在一个批次中混合「命名归一」与「结构重划」两类改动
- 不做没有迁移方案的破坏性变更
- 不因为「代码很难看」就推断它的语义是错的——**先取证，再判定**
- 不把审计结论留在聊天里而不落盘
- 不替代 `refactor` 技能执行单点重构操作
