# 全项目语义级重架构（Project Rearchitecture）

一个面向 **AI 迭代生成、缺少文档、语义混乱** 的既有项目的重架构技能。

> **代码即事实。** 这类项目通常是用 AI 一轮轮对话堆出来的，没有完整文档，
> 概念在迭代中漂移，命名和实际语义对不上。本技能**先从代码逆向出真实存在的领域模型**，
> 归一命名与概念，再重划架构边界——**全程以行为契约冻结保障功能不变**。

适用于 Claude Code、Cursor、Aider、Continue、GitHub Copilot、OpenAI Assistants，
或任何接受系统提示（system prompt）的 LLM。

---

## 它解决什么问题

| 症状 | 本技能的对应能力 |
|---|---|
| 同一个名字在不同地方指不同的事 | D1 同名不同义检测 → 拆解 |
| 同一件事有 5 个名字（`user`/`account`/`customer`/`member`） | D2 同义不同名检测 → 归一 |
| `validate()` 里在写数据库；`Utils` 里装核心业务规则 | D3 名实不符检测 → 先拆副作用后改名 |
| 有完整实现但业务上不存在的旧流程残留 | D4 幽灵概念检测 → 安全删除 |
| 业务规则散在 5 个文件里，没有载体 | D5 缺失概念检测 → 引入抽象 |
| 项目没有测试，不敢动 | 行为冻结：特征化测试 + 黄金样本 + 契约快照 |
| 一次会话装不下整个项目的分析 | 模块切片 + 状态落盘 + 跨会话恢复 |
| 语义正确与对外接口兼容冲突 | 允许破坏性变更，**强制四件套迁移方案** |

---

## 与 `refactor` 技能的分工

| | [`refactor`](https://github.com/chenyilo/code-refactoring-skill) | 本技能 |
|---|---|---|
| 你已经知道要做什么重构 | ✅ 它的主场 | 也适用 |
| 你不知道该叫什么、该切在哪 | ❌ 不覆盖 | ✅ **本技能的核心** |
| 范围 | 单文件 / 单类 / 一次操作 | 整个项目（>100 文件亦适用） |
| 对外契约 | 一律不动 | **允许破坏，但必须带迁移方案** |
| 无测试项目 | 仅提示风险 | ✅ 先建立行为基线 |

**分工原则**：本技能决定**该叫什么、该切在哪、按什么顺序、怎么不炸**；
`refactor` 技能的 catalog 提供**具体怎么改**的手法。两者配合使用。

---

## 五阶段流程

```
阶段 0  行为冻结    →  BEHAVIOR-CONTRACT.md          （硬性前置，不可跳过）
阶段 1  语义审计    →  GLOSSARY.md / SEMANTIC-MAP.md / DRIFT-REPORT.md
阶段 2  目标模型    →  TARGET-MODEL.md / RENAME-MAP.md
阶段 3  架构重划    →  分批的结构改动
阶段 4  分批迁移    →  Expand → Migrate → Contract
```

**顺序不可倒置。** 跳过阶段 1 直接改名的结果，通常是在第三轮又改回来。

> [!IMPORTANT]
> **第一原则：没有行为基线，不许动任何一行代码。**
> 代码即事实，意味着「现状」必须先被钉死，否则无法证明功能没变。

---

## 安装

把本仓库放进你的代理能够读取的技能目录。以 Claude Code 为例：

```bash
git clone git@github.com:chenyilo/project-rearchitecture.git \
  ~/.claude/skills/project-rearchitecture
```

通用做法：把整个仓库目录放到你的 agent 的 skills 目录下，
`SKILL.md` 会自动被识别（名称 `project-rearchitecture`）。

手工方式：把 `PROMPT.md` 风格的内容（即 `SKILL.md` 的正文）粘进代理的系统提示或规则文件。
本技能是**自包含**的，不依赖外部文件。

---

## 文件结构

```
project-rearchitecture/
├── SKILL.md                          技能主入口（决策树、触发词、核心契约）
├── references/
│   ├── behavior-freeze.md            阶段 0：行为冻结与基线建立
│   ├── semantic-audit.md             阶段 1：语义审计（D1–D5 漂移分类与评分）
│   ├── domain-extraction.md          阶段 2A：领域模型与边界切分
│   ├── naming-normalization.md       阶段 2B：命名归一（RENAME-MAP 与批次编排）
│   ├── migration.md                  阶段 4：破坏性变更四件套与三道闸
│   ├── session-state.md              跨会话状态管理（>100 文件必读）
│   ├── process.md                    通用重构流程（同步自 refactor 仓库）
│   ├── safety.md                     安全协议 §1–§9（同步自 refactor 仓库）
│   ├── smells.md                     坏味道清单 家族 1–7（同步自 refactor 仓库）
│   ├── language-profiles.md          语言惯用法与测试命令（同步自 refactor 仓库）
│   └── catalog-*.md                  7 份重构手法目录（同步自 refactor 仓库）
└── templates/
    ├── BEHAVIOR-CONTRACT-template.md  行为契约
    ├── GLOSSARY-template.md           术语表
    ├── SEMANTIC-MAP-template.md       语义地图
    ├── DRIFT-REPORT-template.md       漂移清单与优先级
    ├── TARGET-MODEL-template.md       目标模型与边界
    ├── RENAME-MAP-template.md         重命名映射与批次
    ├── MIGRATION-PLAN-template.md     迁移方案（四件套）
    └── REARCH-STATE-template.md       跨会话状态
```

> **自包含说明**：`process.md`、`safety.md`、`smells.md`、`language-profiles.md`
> 与 `catalog-*.md` 是通用层的自包含副本，上游来源是
> [`chenyilo/code-refactoring-skill`](https://github.com/chenyilo/code-refactoring-skill)。
> **这四类通用文件以上游仓库为准**；本技能自有的六份参考文档以本仓库为准。

---

## 触发词

**中文**：全项目重构、重新架构、重架构、语义重构、语义级重构、语义混乱、
命名混乱、概念不清、概念漂移、代码和概念对不上、名实不符、逆向建模、
反推领域模型、领域模型梳理、术语表、统一语言、从头重构、重写架构、
AI 生成的项目整理、这个项目太乱了想重做底层

**English**：`rearchitecture`、`re-architect`、`semantic refactor`、`domain model`、
`ubiquitous language`、`code as truth`、`reverse engineer the domain`、
`restructure the whole project`、`the code and the concepts don't match`、
`naming is inconsistent`、`vibe-coded cleanup`

> 「把这段代码清理一下」属于 `refactor` 技能的触发词。本技能的触发点必然是
> **项目级**或**语义级**的。

---

## 设计原则

1. **代码即事实** —— 不从需求文档出发，从代码反推语义。
2. **取证不推断** —— 每条结论必须有证据（数据形状 / 调用关系 / 行为断言），命名只是线索。
3. **先冻结再动** —— 没有可复现的行为基线，不许改业务代码。
4. **先定词再改代码** —— 术语表冻结后才开始改名。
5. **一次一个语义批次** —— 不在一个批次里混合「命名归一」与「结构重划」。
6. **破坏必须带迁移** —— 版本提升、迁移脚本、弃用期、回滚步骤，缺一不可。
7. **结论必须落盘** —— 术语表、语义地图、状态文件都是仓库里的版本化文件，不是聊天记录。
8. **审计阶段禁止编辑业务代码** —— 防止「审计着审计着就顺手改了」。

---

## 贡献

欢迎提交 Issue 与 PR。改进建议请附带一个真实项目中的反例——
本技能的方法论都来自真实的重架构失败模式。

## 许可证

MIT
