# 语义地图：<项目名>

> 由 `project-rearchitecture` 技能在阶段 1 产出。记录**每个概念在代码中的全部投影**。
> 这是 `RENAME-MAP` 与架构切分的依据。

- 生成日期：
- 覆盖切片：
- 关联文件：`GLOSSARY.md` / `DRIFT-REPORT.md`

---

## 使用说明

1. 本表是**事实记录**，不是设计文档。只写代码里实际存在的东西。
2. 每个概念必须给出证据位置（文件:行）。无证据的行不写入。
3. 「是否权威」列标记该处是否为该概念的真值来源（判定规则见
   [semantic-audit.md](../references/semantic-audit.md) §1.5）。

---

## 概念 → 代码投影总表

### <概念中文名>（建议标识符 `<NewName>`）

| 投影位置 | 类型 | 具体标识符 | 是否权威 | 证据 |
|---|---|---|---|---|
| `src/orders/order.ts:12` | 类型定义 | `class Order` | ✅ | 领域层正式声明 |
| `migrations/003.sql` | 数据形状 | `orders` 表 | ✅ | 表结构 |
| `src/api/routes.ts:88` | 对外 API | `GET /api/orders` | | 路由注册 |
| `src/orders/dto.ts:5` | 传输对象 | `OrderDTO` | | 序列化定义 |
| `src/legacy/flow.py:200` | 旧实现残留 | `OrderFlow` | | 疑似幽灵概念，见 DRIFT #7 |

**统计**：类 2 个 / 文件 5 个 / 表 1 个 / 路由 1 个 / 字段 14 个
**当前命名**：`Order`、`OrderDTO`、`OrderFlow`、`Purchase`（混用）
**漂移**：D2（`Order` 与 `Purchase` 同义不同名）；D4（`OrderFlow` 疑似幽灵）

---

## 反向索引：文件 → 概念

> 用于判断一个文件是否承担了多个概念（职责混杂的信号）。

| 文件 | 涉及概念 | 职责是否单一 | 备注 |
|---|---|---|---|
| `src/orders/order.ts` | Order | ✅ | |
| `src/utils/misc.ts` | Order / Customer / Billing | ❌ | 三个概念混杂，见 DRIFT #12 |

---

## 数据聚合索引：表 → 代码

> 这是限界上下文切分的主要依据。

| 表/集合 | 读写位置 | 读 | 写 | 关联表（同事务） |
|---|---|---|---|---|
| `orders` | `src/orders/*` | ✅ | ✅ | `order_items`、`payments` |

---

## 跨切片概念

> 贯穿多个切片的概念，通常是核心域，应优先获得清晰边界。

| 概念 | 涉及切片 | 是否核心域 | 备注 |
|---|---|---|---|
| | | | |

---

## 证据不足的概念（候选，未确认）

| 标识符 | 位置 | 缺什么证据 | 处置 |
|---|---|---|---|
| | | 无表、无调用关系 | 标 `UNRESOLVED`，见 GLOSSARY 待定项 |
