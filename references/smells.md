# 代码坏味道检测清单（Code Smell Detection Catalog）

在重构流程的阶段 2 使用本清单。系统地排查每一个家族。把发现分级为 Blocker / Major / Minor。在改动任何东西之前先呈现全部发现结果。

---

## 如何使用

1. 先完整读取目标代码
2. 对照下面的每个坏味道家族逐一检查——症状在可能的情况下都写得便于用 grep 检索
3. 记录每个检测到的坏味道：文件路径 + 行号 + 严重级别
4. 把每个坏味道映射到推荐的重构操作
5. 在动手之前把分级列表呈现给用户

---

## 家族 1：臃肿（Bloat）

已经膨胀到难以理解、或难以安全修改的代码。

---

### 过长函数（Long Method / Long Function）
**症状：**
- 函数体超过约 20 行逻辑（不含空行和注释）
- 多层嵌套（缩进超过 3 层）
- 存在多条做不同事情的 `return` / `throw` 路径
- 函数体内有写着"step 1"、"step 2"之类的注释
- 很难用一个动词短语说清这个函数"做什么"

**检测：** 统计函数开括号 `{` 与闭括号 `}` 之间的行数。留意 `// --- Part 2 ---` 这类注释分隔标题。

**严重级别：** 超过 50 行为阻断级；20–50 行为严重级；15–20 行为轻微级。

**推荐：** Extract Method/Function（提炼函数，catalog-composing.md）、Decompose Conditional（分解条件表达式，catalog-simplifying.md）

---

### 过大的类 / 上帝对象（Large Class / God Object）
**症状：**
- 类超过约 200–300 行
- 超过 7 个服务于不同概念目的的公共方法
- 只有部分方法使用的字段（人格分裂）
- 类名中包含 "Manager"、"Handler"、"Service"、"Util"、"Helper" 或 "God"
- 这个类"认识"太多其他类

**检测：** 统计公共方法数量。按"哪些方法使用了哪些字段"对字段分组——如果它们落入 2 个或更多互不相交的组，那它其实是两个类。

**严重级别：** 超过 500 行或明显承担多项职责为阻断级；200–500 行为严重级。

**推荐：** Extract Class（提炼类，catalog-organizing.md）、Move Method（搬移函数，catalog-organizing.md）

---

### 过长参数列表（Long Parameter List）
**症状：**
- 函数/方法的参数超过 3 个
- 有一个或多个参数是布尔标志
- 同一组参数在多个调用点被一起传递
- 名称或类型相似的参数出现多次

**检测：** 用 grep 查找带 4 个及以上逗号分隔参数的函数签名。

**严重级别：** 参数超过 4 个或包含布尔标志为严重级；3–4 个非标志参数为轻微级。

**推荐：** Introduce Parameter Object（引入参数对象，catalog-api.md）、Preserve Whole Object（保持对象完整，catalog-api.md）、Remove Flag Argument（移除标志参数，catalog-api.md）

---

### 数据泥团（Data Clumps）
**症状：**
- 同样的 3 个及以上变量/字段在多个函数或类中一起出现
- 反复一起传递同一组基本类型（例如 `firstName, lastName, email` 总是同时出现）
- 从这组中移除一个变量会让其余变量失去意义

**检测：** 查找那些相同变量名反复一起出现的函数调用。

**严重级别：** 严重级。

**推荐：** Introduce Parameter Object（catalog-api.md）、Replace Data Value with Object（以对象取代数据值，catalog-organizing.md）

---

### 基本类型偏执（Primitive Obsession）
**症状：**
- 用裸字符串表示电话号码、邮箱、货币、坐标、ID 等概念
- 用裸整数表示具有领域含义的常量（状态码、类型、优先级）
- 用并行数组代替对象数组
- 存在 `if (type === "premium")` 这类类型检查代码

**检测：** 查找与字面量值做比较、而这些字面量代表领域概念的字符串/数字比较。

**严重级别：** 业务逻辑依赖它时为严重级；否则为轻微级。

**推荐：** Replace Primitive with Object / Value Object（以对象/值对象取代基本类型，catalog-organizing.md）、Replace Type Code with Class/Enum（以类/枚举取代类型码）

---

## 家族 2：面向对象滥用者（Object-Orientation Abusers）

对面向对象原则的不当运用。

---

### 类型切换 / 重复的类型检查（Switch on Type / Repeated Type Checks）
**症状：**
- `switch/case` 或 `if/else if` 链在检查类型标签（`type`、`kind`、`role`）
- 同样的 `switch` 模式出现在多个地方
- 新增一种"类型"需要找出所有 switch 语句并逐一添加 case

**检测：** 用 grep 查找 `switch` 或 `if (x.type ===` / `if (x instanceof` 这样的判断链。

**严重级别：** 同一个 switch 出现在 2 处以上为严重级；孤立出现为轻微级。

**推荐：** Replace Conditional with Polymorphism（以多态取代条件表达式，catalog-simplifying.md）、Extract Class（catalog-organizing.md）

---

### 临时字段（Temporary Field）
**症状：**
- 实例字段只在一个方法中被赋值，在其他地方是 `null`/`undefined`
- 该字段只在某些代码路径下才有意义
- 存在 "// only set when in batch mode" 这类注释

**严重级别：** 轻微级。

**推荐：** Extract Class（catalog-organizing.md）、Introduce Special Case（引入特例，catalog-simplifying.md）

---

### 被拒绝的遗赠（Refused Bequest）
**症状：**
- 子类覆写方法只是为了抛出 `NotImplementedException` 或返回空操作
- 子类只用到父类提供的一小部分
- 继承被用于复用代码，而不是真正的 is-a 关系

**严重级别：** 严重级。

**推荐：** Replace Subclass with Delegate（以委托取代子类，catalog-inheritance.md）、Extract Interface（提炼接口，catalog-inheritance.md）

---

### 异曲同工的类（Alternative Classes with Different Interfaces）
**症状：**
- 两个类做同一件事，但方法名不同（`fetchUser` 与 `getUser`、`UserLoader` 与 `UserFetcher`）
- 只要重命名就能把其中一个换成另一个

**严重级别：** 轻微级。

**推荐：** Rename Method（函数改名，catalog-api.md）、Extract Interface（catalog-inheritance.md）

---

## 家族 3：变化阻碍者（Change Preventers）

一处改动会迫使其他不相关的地方也跟着改的代码。

---

### 发散式变化（Divergent Change）
**症状：**
- 一个类因为多种不同原因而修改（例如 `User` 类在认证变化时改、在 UI 变化时改、在数据模型变化时也改）
- 在 git 历史中，多种不同"种类"的改动落在同一个文件里

**检测：** 查看 `git log --oneline <file>`——如果提交信息横跨互不相关的关注点，就存在这个坏味道。

**严重级别：** 严重级。

**推荐：** Extract Class（catalog-organizing.md）、Move Method（catalog-organizing.md）

---

### 霰弹式修改（Shotgun Surgery）
**症状：**
- 一处概念上的改动需要跨多个文件编辑
- "加一个新字段"要动 8 个文件
- 改动零散地分布在代码库各处

**检测：** 问自己"如果我改了 X，还有什么会坏？"——用 Grep 统计受影响的文件数。

**严重级别：** 严重级。

**推荐：** Move Method（catalog-organizing.md）、Inline Class（内联类，catalog-organizing.md）、合并相关逻辑

---

### 平行继承体系（Parallel Inheritance Hierarchies）
**症状：**
- 每次为 A 创建一个子类，就必须同时为 B 创建一个子类
- 两个类继承体系彼此镜像
- 类名共享前缀：`PremiumOrder`/`PremiumInvoice`、`StandardOrder`/`StandardInvoice`

**严重级别：** 严重级。

**推荐：** Move Method（catalog-organizing.md）、把一个继承体系折叠进另一个

---

## 家族 4：可有可无之物（Dispensables）

毫无用处、应当删除的代码。

---

### 用注释解释显而易见代码（Explanatory Comments Over Obvious Code）
**症状：**
- 注释描述代码做了什么（WHAT），而不是为什么这么做（WHY）
- 注释用自然语言复述代码：在 `i++` 上写 `// increment i by 1`
- 在并不复杂的逻辑前堆砌大段注释
- 被注释掉的代码留在原处

**注意：** 好的注释解释 WHY——一个不显然的约束、一处已知 bug 的规避手段、一个微妙的不变式。解释 WHAT 的注释说明代码本身不够清晰。

**严重级别：** 轻微级（但预示着更深的清晰度问题）。

**推荐：** 通过重命名让意图显而易见、用描述性名称做 Extract Method（catalog-composing.md）、删除被注释掉的死代码

---

### 重复代码（Duplicate Code）
**症状：**
- 几乎完全相同的代码块，只差变量名或字面量值
- 带有细微差异的复制粘贴
- 同一算法在不同地方实现了两次
- 工具函数在每个文件里各自重新实现，而没有共享

**检测：** 从其中一处取出有辨识度的子表达式，用 Grep 搜索。

**严重级别：** 承载逻辑时为严重级；琐碎时为轻微级。

**推荐：** Extract Function（提炼函数，catalog-composing.md）、Move Method（catalog-organizing.md）、Parameterize Method（函数参数化，catalog-api.md）

---

### 冗赘类 / 死代码（Lazy Class / Dead Code）
**症状：**
- 只有 1–2 个琐碎方法、本可以内联掉的类
- 从未被调用的函数（用 Grep 确认调用点为零）
- 被 import 但符号从未使用的文件
- 永远开启或永远关闭的特性开关

**检测：** 用 Grep 搜索该类/函数名——如果没有调用点，它就是死代码。

**严重级别：** 轻微级（杂乱）；如果它制造了虚假的复杂度则为严重级。

**推荐：** Inline Class（catalog-organizing.md）、直接删除死代码

---

### 夸夸其谈通用性（Speculative Generality）
**症状：**
- 只有唯一一个具体实现的抽象基类
- "为将来使用"而存在、但实际总是被传 `null`/`undefined` 的参数
- 零调用者的钩子、回调或扩展点
- 在第二个用例出现之前就搭好的"通用"基础设施

**严重级别：** 轻微级。

**推荐：** Collapse Hierarchy（折叠继承体系，catalog-inheritance.md）、Remove Parameter（移除参数，catalog-api.md）、删除未使用的抽象

---

## 家族 5：耦合者（Couplers）

类或模块之间不恰当的耦合。

---

### 依恋情结（Feature Envy）
**症状：**
- 某个方法使用另一个对象的数据或方法比使用自己的还多
- 一个函数接收一个对象后立刻深入其字段，去做本该由该对象自己完成的工作
- 模式：`function processPayment(order) { order.items.forEach(...); order.user.email...; order.discount... }`

**严重级别：** 严重级。

**推荐：** Move Method（catalog-organizing.md）、先 Extract Method 再 Move（catalog-composing.md + catalog-organizing.md）

---

### 狎昵关系（Inappropriate Intimacy）
**症状：**
- 两个类互相访问对方的私有/内部状态
- 两个模块之间存在双向依赖
- 类 A 把 `this` 传给类 B，好让 B 回调 A 的内部实现

**严重级别：** 严重级。

**推荐：** Move Method/Field（搬移函数/字段，catalog-organizing.md）、Extract Class（catalog-organizing.md）、Hide Delegate（隐藏委托关系，catalog-organizing.md）

---

### 过长的消息链（Message Chains）
**症状：**
- `a.getB().getC().doThing()`——穿过多个对象连续调用
- 违反迪米特法则：你只应认识直接协作者
- 链条中段的变化会迫使每个调用点都跟着改

**检测：** 用 grep 查找在同一行出现 3 次及以上的 `.(`。

**严重级别：** 出现在多处为严重级；孤立出现为轻微级。

**推荐：** Hide Delegate（catalog-organizing.md）、Extract Method（catalog-composing.md）

---

### 中间人（Middle Man）
**症状：**
- 一个只会转发的类——每个方法都只是调用另一个对象上的同名方法
- 不添加任何行为、转换或保护的包装器
- 你"为了间接层"加了它，但这个间接层当前没有任何用处

**严重级别：** 轻微级。

**推荐：** Remove Middle Man（移除中间人，catalog-organizing.md）、Inline Class（catalog-organizing.md）

---

## 家族 6：架构违规（Architectural Violations）

职责越过架构分层边界的代码。这些坏味道表明某种模式（MVC、MVP、MVVM、Clean Architecture、Hexagonal）所期望的结构分离已经被侵蚀。它们在文件/模块级别被检测，而不是在行级别。

**一旦发现其中任何一条：** 在架构违规被处理之前，不要继续进行代码级重构，或者与用户明确商定暂不处理。在对这些坏味道采取任何行动之前，先读 `safety.md` §8。

---

### 臃肿的控制器 / 臃肿的路由处理器（Fat Controller / Fat Route Handler）
**症状：**
- Controller、路由处理器或 Activity/Fragment 超过约 50 行
- Controller 中包含实现业务规则（定价、资格、折扣）的 `if/else` 链
- Controller 为完成一个动作而按顺序调用 4 个以上服务
- Controller 直接发邮件、投递任务或进行计算
- 不启动完整的 Web 框架就无法对 Controller 做单元测试

**检测：**
```
# 统计 controller/handler 的行数：
grep -n "class.*Controller\|def.*route\|app\.(get\|post\|put\|delete)" <file>
# 在 controller 中查找业务规则关键词：
grep -n "discount\|eligibility\|calculate\|if.*status\|if.*role" controllers/
```

**严重级别：** 超过 50 行且含领域逻辑为严重级；业务规则在多个 controller 间重复为阻断级。

**推荐：** Extract Use Case / Interactor（提炼用例/交互器，catalog-architecture.md）、Push Business Logic to Domain（把业务逻辑下沉到领域层，catalog-architecture.md）

---

### 含业务逻辑的 UI（UI with Business Logic）
**症状：**
- View、Activity、Fragment 或 Component 计算派生状态（总计、过滤后的列表、格式化后的值）
- 事件处理器中包含领域校验（`if (age < 18) return error`）
- Component 直接调用仓储或数据库（没有 service/ViewModel 层）
- 展示逻辑与生命周期方法纠缠在一起，导致无法做隔离测试

**检测：**
```
# 在视图文件中查找领域逻辑关键词：
grep -rn "calculate\|discount\|validate\|\.filter\|\.reduce\|\.map" views/ components/ activities/
# 在视图文件中查找 repository/DB 的 import：
grep -rn "import.*Repository\|import.*dao\|import.*database" views/ components/
```

**严重级别：** 严重级。

**推荐：** Extract ViewModel（提炼 ViewModel，catalog-architecture.md）、Introduce Presenter（引入 Presenter，catalog-architecture.md）

---

### 贫血领域模型（Anemic Domain Model）
**症状：**
- 领域对象只包含字段和 getter/setter——零行为
- 所有逻辑都住在接收领域对象作为参数的 `*Service`、`*Manager` 或 `*Helper` 类里
- 同一条领域规则在多个 service 类中实现（由贫血导致的重复）
- 领域对象无法维护自身的不变式（非法状态可以被表达出来）

**检测：**
```
# 找出除访问器之外没有方法的领域类：
grep -n "def get_\|def set_\|public get\|public set" domain/
# 找出做逐实体计算的 service 类：
grep -n "def.*calculate\|def.*validate\|def.*compute" services/
```

**严重级别：** 业务规则重复为严重级；模型简单、规则很少为轻微级。

**推荐：** Push Business Logic to Domain（catalog-architecture.md）

---

### 分层违规（Layer Violation）
**症状：**
- 表现层直接 import data/repository/db 包（跳过领域层）
- 领域对象 import 了仓储、service 或 HTTP 客户端
- controller import 了具体的 ORM 实体或数据库行类型，并直接操作它
- 依赖箭头的方向与架构意图相反

**检测：**
```
# 找出表现层从数据层 import 的情况：
grep -rn "import.*repository\|import.*dao\|import.*entity" controllers/ views/ presenters/
# 找出领域层从基础设施 import 的情况：
grep -rn "import.*http\|import.*sql\|import.*orm" domain/ models/
```

**严重级别：** 严重级。

**推荐：** Fix Layer Violation（修复分层违规，catalog-architecture.md）、Introduce Repository（引入仓储，catalog-architecture.md）

---

### 分散的数据访问（Scattered Data Access）
**症状：**
- SQL 查询或 ORM 调用出现在 controller、service 以及 presenter 中——没有唯一归属者
- 同一张表在 5 个以上不同文件中被查询，且投影略有不同
- 切换数据源（例如加一层缓存、换到另一个数据库）需要改动许多文件
- 某个实体没有对应的 `*Repository` 或 `*Store` 类

**检测：**
```
# 在非仓储文件中查找 SQL：
grep -rn "SELECT\|INSERT\|UPDATE\|DELETE\|\.find(\|\.query(" controllers/ services/ views/
# 统计访问同一张表的文件数：
grep -rn "FROM users\|\.findUser\|userModel\." . | grep -v "repository\|repo"
```

**严重级别：** 严重级。

**推荐：** Introduce Repository（catalog-architecture.md）

---

### 缺失领域层（Missing Domain Layer）
**症状：**
- 不存在 `domain/`、`models/` 或 `entities/` 目录——只有 `controllers/` 和 `repositories/`
- 业务逻辑完全住在 API 处理器或工具函数里
- 新增一条业务规则需要同时改动 API 层和数据库层
- 代码库没有"心脏"——没有任何东西表达这个应用究竟在做什么

**检测：**
```
# 检查目录结构中是否存在领域层：
ls src/ app/   # 有 domain/、models/ 或 entities/ 目录吗？
# 检查 controller 是否直接从数据层 import：
grep -rn "^import\|^from\|^require" controllers/ | grep -i "db\|sql\|orm\|repo"
```

**严重级别：** 应用有实质业务逻辑为严重级；只是薄薄一层 CRUD API 为轻微级。

**推荐：** Push Business Logic to Domain（catalog-architecture.md）、Extract Use Case（提炼用例，catalog-architecture.md）

---

## 家族 7：语义漂移（Semantic Drift）

名字与它实际承载的语义已经对不上——或者一个语义在代码里根本没有名字。这类坏味道来自多轮 AI 迭代：每一轮都在旧概念上叠一层新说法，却没人回头统一定词。它们**不一定让代码变长，但持续制造误改**。

**一旦发现其中任何一条：** 先读 `safety.md` §9。语义级改动有硬性前置条件（行为基线 + 术语表），不具备前置条件时**只做取证与记录，不做改动**。

**漂移五分类：**

| 编号 | 分类 | 一句话描述 |
|---|---|---|
| D1 | 同名不同义 | 一个名字指向两件事 |
| D2 | 同义不同名 | 一件事有多个名字 |
| D3 | 名实不符 | 名字描述的与实际行为不一致 |
| D4 | 幽灵概念 | 代码里有完整实现，业务上不存在 |
| D5 | 缺失概念 | 业务上有，代码里没有载体，散落各处 |

**全家族通用术语约定：**

- **命名归一** —— 只改名字（标识符、路径、字段），不改结构、不改行为
- **结构改动** —— 新增/拆分/合并类型、函数、模块；行为保持不变
- **对外契约** —— API 字段名、DB 表列名、事件 payload、错误码、CLI 参数、日志字段名。凡触及即为**契约迁移**，需要四件套（版本提升 / 迁移脚本 / 弃用期 / 回滚步骤），见 `safety.md` §9

---

### 迭代残留命名（Iteration Residue Naming）
**症状：**
- 标识符里带迭代痕迹：`newVersion`、`v2Handler`、`tempLogic`、`finalFinal`、`oldUser`、`userNew`、`userCopy`
- 无意义的数字后缀：`handler2`、`process3`、`UserServiceB`
- 同一个概念同时存在 `oldX` 与 `xNew`（甚至 `xNew2`），且都还有调用点
- 注释里出现"暂时"、"新写法"、"旧逻辑保留"

**为什么有害：** 迭代残留是**临时状态被固化**的证据。它让读者无法判断哪一份是权威（真值来源），也暗示着一个从未被执行的清理任务已经挂了很久。

**检测：**
```
# 迭代残留词根：
grep -rnE "newVersion|v2Handler|tempLogic|finalFinal|old[A-Z]|[a-z]New\b|[a-z]Copy\b" .
# 数字后缀（同一词根出现多个编号）：
grep -rnE "[A-Za-z]+[0-9]+\b" .
# 成对存在（old/new、legacy/current）：
grep -rniE "(old|legacy|deprecated)[A-Z_]|[_-](old|legacy|deprecated)\b" .
```

**严重级别：** 轻微级；`oldX` 与 `xNew` 并存且都有调用点时为严重级。

**推荐（命名归一）：** Rename（函数/类/变量改名，catalog-api.md）。
**契约影响：** 通常无。若残留名已出现在 API 字段、DB 列、事件 payload 或日志字段名中，先按 `safety.md` §9.4 判定契约归属——**保留对外名、只归内部名**，两者不可混为一批。

---

### 迭代编号泄漏进领域层（Iteration Number Leaking into the Domain）
**症状：**
- 领域类型带版本/迭代标记：`OrderV2`、`UserModelNew`、`BillingPhase2`
- 表名或列名带迭代号：`orders_v2`、`user_v2_name`
- API 路径带版本或"迁移中"标记：`/api/v2/orders`、`/migration/orders-new`
- 版本号被当作领域概念参与判断：`if (order.version === 2)`

**为什么有害：** 迭代编号是**过程信息**，不是领域概念。它进了领域层之后会变成永远拆不掉的第二种权威——每次迭代都新增一个平行的领域概念，最终没人知道业务真相落在哪一层。

**检测：**
```
# 领域/模型目录里的版本标记：
grep -rniE "\b(v|version|rev|phase|iter)[0-9]+\b" domain/ models/ entities/ src/domain/
# 表与列的迭代号：
grep -rniE "CREATE TABLE .*(_v[0-9]+|_new|_old)|add_column.*(_v[0-9]+|_new)" migrations/
# 路径里的版本与"迁移中"标记：
grep -rnE "/(v[0-9]+|migration|new|legacy)/" routes/ urls.py controllers/
```

**严重级别：** 严重级；版本号参与业务判断时为阻断级。

**推荐（结构改动：把版本信息移出领域模型；命名归一：领域类型改名）：** 先确认版本段是否属于对外契约，再决定改内部名还是走契约迁移。
**契约影响：** **高**。`/api/v2/`、`orders_v2` 是典型的对外契约。旧路径保持可用（别名 / 重定向 / 视图），新名只在内部使用；确需破坏时四件套齐全再动手。

---

### 一词多义 / 同名不同义（One Name, Many Meanings —— D1）
**症状：**
- 同一个名字在代码里指两件不同的事：`status` 在订单上下文是支付状态、在物流上下文是配送状态
- `Account` 一处是登录账号、另一处是计费主体
- 同名标识符落在**不同的表/字段**上，赋值来源互不相干
- 同一个 DTO 字段名在不同端点含义不同

**为什么有害：** 最危险的一类。任何按名字做的大范围搜索替换都会误伤另一处；新加入的人必然误解；测试通常也只覆盖其中一种含义。

**检测：**
```
# 高危名：先找出现频次最高的公共词根：
grep -rnE "\b(status|state|type|account|user|name|id|amount|total)\b" domain/ models/ | sort | uniq -c | sort -rn
# 同名但定义来源不同：列出全部定义点，再看它们各自读写哪些字段：
grep -rnE "(class|interface|type|struct|CREATE TABLE)\s+\w*(Status|Account|Type)\w*" .
```
判定要点：**看它读写哪些字段、被谁调用**；命名本身不作为结论。

**严重级别：** 阻断级。

**推荐（命名归一）：** 拆成两个概念，其中一个改名（Rename，catalog-api.md）。动手前先各自写出一句话定义与关键不变量——写不出来说明你还没分清两者，此时改名必然改错。
**契约影响：** 取决于哪一个含义位于契约层。若两种含义共用同一个对外字段名，先按真值来源判定（数据形状 > DB schema > 对外 API > 领域类型）——**契约名保留或走契约迁移，只归内部名**。

---

### 同义不同名（Many Names, One Meaning —— D2）
**症状：**
- 同一实体多个名字：`user` / `account` / `customer` / `member` / `client` 混用
- 同一动作多个名字：`get` / `fetch` / `load` / `query` / `find` 混用；`cart` / `basket` / `tray` 混用
- 不同名字指向同一张表或同一批字段，或可以互相赋值
- 改一处功能要在几个同义名字之间来回对照

**为什么有害：** 最普遍的一类，也是"改了 A 忘了 B"的根源。它让搜索不全、评审失真，并且让新代码继续随机选名。

**检测：**
```
# 逐词统计候选同义名的出现次数与所在层：
for n in user account customer member client; do echo "== $n"; grep -rniw "$n" . | wc -l; done
# 动词同义组：
grep -rnE "\b(get|fetch|load|query|find|read)[A-Z]\w*\(" .
# 同义词根同时出现在类型名里：
grep -rnE "(Cart|Basket|Tray)|(User|Account|Customer|Member)" .
```

**严重级别：** 严重级。

**推荐（命名归一）：** 先判定哪个名字是权威定义，其余列为别名，分批改名。判定权威定义的顺序：**数据形状 > DB schema > 对外 API > 领域层类型 > 使用频率最高者**；冲突时以更靠近数据的一层为准，且只用来决定内部代码统一用哪个名字。
**契约影响：** 中。若同义名之一已是对外字段名或列名，内部归一时**不要连带改契约名**；契约改名是另一个批次，需要四件套。

---

### 名实不符（Misleading Name —— D3）
**症状：**
- `validate()` 里有写库副作用（校验与写入混在一起）
- `isValid` 实际判断的是"是否已支付"；`isActive` 实际判断的是"未被软删除"
- `Utils` / `Helper` / `Common` 里装着核心业务规则
- `temp` / `tmp` / `misc` 是长期存在的关键逻辑
- 名字描述的条件与实现里的判断条件不一致

**为什么有害：** 理解成本的主要来源。读者按名字推断行为，而行为不同——评审、测试与后续改动全部建立在错误前提上。

**检测：**
```
# 疑似纯函数/断言式命名：
grep -rnE "function (is|has|can|validate|check)[A-Z]\w*|def (is|has|can|validate|check)_" .
# 再在这些函数体内找写入与外部调用：
grep -rnE "\.(save|insert|update|delete|create|send|publish)\(" .
# 通用名容器里的领域关键词：
grep -rnE "discount|price|tax|eligib|settle|refund" utils/ helpers/ common/ shared/
```

**严重级别：** 严重级；`validate` 类函数带写库副作用时为阻断级（调用方通常以为它只读）。

**推荐（结构改动 + 命名归一，必须拆成两个批次）：** 先把副作用抽出去（Extract Function，catalog-composing.md），批次门禁通过之后再改名（Rename，catalog-api.md）。

> [!WARNING]
> **这是"两类改动混进一个批次"的典型陷阱。** 先结构、后命名；两批之间的基线必须全绿。

**契约影响：** 结构批次通常不动契约。但若副作用本身要换归属（例如从校验迁到领域事件），会触及事件 payload 与错误码——按契约迁移处理。

---

### 幽灵概念（Ghost Concept —— D4）
**症状：**
- 有完整实现（类、表、路由、测试都在），但业务上根本不存在这个概念
- 无调用点：grep 为零，却仍在编译/打包范围内
- 废弃的 feature flag 分支（永远走不到的那一支）
- 早期方案残留、被替换但没删的旧路径、为假想需求做的抽象

**为什么有害：** 占用认知与维护成本，污染真值来源，让审计得出错误结论。幽灵概念一旦被当成真概念写进术语表，会把整个概念的边界一起带偏。

**检测：**
```
# 无调用点（先取符号名再数引用，排除定义处与测试）：
grep -rn "SymbolName" . | grep -v "_test\|\.test\.\|spec\."
# 废弃开关与死分支：
grep -rnE "(featureFlag|FEATURE_|ENABLE_|flag)[A-Za-z_]*\s*[=:]\s*(false|0)" .
# 疑似早期残留路径：
grep -rniE "\b(legacy|deprecated|unused|obsolete|oldPath)\b" .
```
**注意：** 零命中**不等于**无引用。动态引用、反射、路由注册、事件订阅、配置、CI 脚本、文档与前端字符串都可能是唯一调用点——完整清单见 `safety.md` §9.5。

**严重级别：** 严重级，但**风险等级高**（删除是最难回滚的操作之一）。宁可先标记后处理。

**推荐（结构改动：删除）：** 确认全量引用为零后**单独一个批次**删除，并与用户确认数据保留与合规要求。
**契约影响：** **高**。若它出现在 API 路径、DB 表、事件名或错误码里，删除即破坏性变更——四件套齐全再动手。

---

### 缺失概念（Missing Concept —— D5）
**症状：**
- 同一组字段或条件在 3 处以上重复表达同一语义，却没有载体
- "会员等级"靠 `if (total > 1000 && days < 30)` 到处重复判断
- 折扣计算规则散在 5 个文件里，每处略有出入
- 没有类型能承载该语义的不变量，非法状态可以被表达出来

**为什么有害：** 改一处漏一处；无法复用；没有地方承载不变量。缺失概念也让重构无从下手——你想搬移它，但它在代码里还没有可搬移的形状。

**检测：**
```
# 重复的条件组合（取一处有辨识度的子表达式，全库搜）：
grep -rnE "total\s*>\s*[0-9]+.*days\s*<" .
# 重复出现的字段组合（数据泥团是缺失概念的典型信号）：
grep -rnE "(tier|level|grade).*(threshold|limit|min|max)" .
# 以魔法数字表达的领域规则：
grep -rnE "(0\.(1|2|3|15|2)|1000|30)\b" src/ domain/
```

**严重级别：** 严重级；重复范围超过 10 个文件、或涉及定价与权限时为阻断级。

**推荐（结构改动 + 新增抽象）：** 引入值对象 / 类型 / 模块承载该概念（catalog-composing.md、catalog-api.md），把散落的表达收敛进去。风险高于纯改名。
**契约影响：** 低到中。新概念通常内部可见；但若该概念的取值会出现在 API 字段或 DB 列中（例如把散落的等级判断收敛成 `tier` 列），即触及契约。

---

### 伪抽象层 / 防御性重复抽象（Speculative Wrapper / Defensive Abstraction）
**症状：**
- 每个调用点各配一个 wrapper：`getUserForOrder()`、`getUserForInvoice()`——只是转发不同参数
- 只服务单一实现的抽象层（`IUserStore` 只有一个 `SqlUserStore`，且从无第二种实现）
- 为了"将来可能换实现"而加的间接层，当前不添加任何行为
- 抽象层的名字比它包装的实现更含糊（`Processor`、`Engine`、`Handler` 后面没有领域语义）

**为什么有害：** 把语义打散到多层，读者必须一路追到底才知道"实际发生什么"。它同时会掩盖 D1/D2：同一概念在每一层各有一个名字。

**检测：**
```
# 只有一个实现者的抽象：
grep -rnE "^\s*(class|interface|abstract class|trait)\s+I?\w+(Store|Repository|Provider|Adapter|Port)\b" .
# 转发型 wrapper（方法体只有一行转发调用）：
grep -rnA2 "function \w+\(" . | grep -E "return .*->\w+\(|return .*\w+\.\w+\("
# 同词根成组的方法名：
grep -rnE "function (get|fetch)\w+For\w+\(" .
```

**严重级别：** 轻微级；当它掩盖了同名不同义时为严重级。

**推荐（结构改动）：** Inline Class / Remove Middle Man（catalog-organizing.md）、Collapse Hierarchy（折叠继承体系，catalog-inheritance.md）。
**契约影响：** 通常无。若该抽象层本身就是对外接口（例如已发布的 SDK 端口），内联或删除前先确认外部实现方。

---

### 扁平化 handler（Flat Handler）
**症状：**
- 所有逻辑堆在路由处理器/控制器里——校验、取数、计算、写库、发通知一条龙
- handler 超过约 50 行，或内部含领域规则的 `if/else` 分支
- 没有领域层承接：业务规则只存在于 handler 内部
- 逻辑无法脱离 Web 框架做单元测试
- 同一段规则在多个 handler 中复制（常与「霰弹式修改」叠加出现）

**为什么有害：** 缺少承载语义的位置，语义就只能以"某个端点做了什么"的形式存在——概念无法被命名、无法被复用，也就无法被审计。它是「缺失领域层」在语义层面的前兆。

**检测：**
```
# 路由/处理器中的领域关键词：
grep -rnE "discount|eligib|calculate|settle|refund|tax" controllers/ routes/ handlers/ api/
# 单个 handler 的副作用数量：
grep -rnE "\.(save|insert|update|create|send|publish)\(" controllers/ routes/ handlers/
```

**严重级别：** 严重级；业务规则在多个 handler 间重复为阻断级。

> [!NOTE]
> 本条目与家族 6「臃肿的控制器 / 臃肿的路由处理器」是同一现象的两个视角：**家族 6 看架构边界被侵蚀，家族 7 看语义无处安放**。处置前先读 `safety.md` §8 与 §9。

**推荐（结构改动）：** Extract Use Case / Interactor（提炼用例/交互器）、Push Business Logic to Domain（把业务逻辑下沉到领域层），见 catalog-architecture.md；抽取时同步为概念命名（术语表先行）。
**契约影响：** 通常无。若顺带改动端点路径或请求/响应字段名，立即升级为契约迁移——不要与结构批次混合。

---

### 注释与实现矛盾（Comment-Implementation Drift）
**症状：**
- 注释描述的是旧行为（写着"返回未支付订单"，实现已过滤全部活动订单）
- docstring 或参数说明与实际签名不符（参数已删、默认值已改）
- `@deprecated` 标注与代码实际状态矛盾（还标着"新 API"，其实已是唯一入口）
- 中文注释与英文标识符表达两套不同的概念名
- 注释里写"临时"、"先这样"、"后面统一"，已经挂了多轮迭代

**为什么有害：** 注释是审计的**第二真值来源**。矛盾的注释会误导审计结论，让人把已废弃的语义当成现役语义写进术语表——错误会一路传播到目标模型。

**检测：**
```
# TODO / FIXME / 临时标记：
grep -rnE "(TODO|FIXME|HACK|XXX|暂时|临时|先这样|后面统一)" .
# 注释提到旧名或旧行为：
grep -rnE "^\s*(//|#|\*|/\*).*(old|deprecated|legacy|旧|原来的|之前的)" .
# 中文注释单独排查（AI 迭代项目高发）：
grep -rnE "(//|#|\*).*[一-龥]" .
```

**严重级别：** 轻微级；当它导致审计结论错误时为严重级。

**推荐（命名归一 + 文档修正）：** 先按实现取证确定真实语义，再修注释。注释与实现矛盾时**不要反向修改实现**去迎合注释。
**契约影响：** 注释本身不构成契约。但若该注释是对外文档（OpenAPI description、SDK docstring），下游可能按它解析——视为对外契约。

---

### 占位 / Mock 实现混入生产路径（Placeholder Implementation on the Production Path）
**症状：**
- `return mockData` / `return fakeUser` 这类桩数据仍在真实调用路径上
- 硬编码返回值（`return { status: "ok" }`）冒充成功分支
- `TODO` 处返回空实现，而调用方以为它已经生效
- 生产配置指向沙箱地址、测试密钥、`localhost` 桩服务
- 测试替身（stub/mock）被提升进生产模块，而不是留在测试目录

**为什么有害：** **这是行为风险，不只是坏味道。** 它让"行为基线"本身失真——基线记录的是一个从未真正生效的行为，之后所有"基线全绿"的结论都失去意义。

**检测：**
```
# 桩数据与硬编码返回：
grep -rnE "return (mock|fake|dummy|stub|sample)\w*|return \{\s*(status|ok)\s*:" .
# TODO 附近的空实现：
grep -rnB2 -A2 "TODO" . | grep -E "return (null|\[\]|\{\}|''|\"\")"
# 生产代码里的测试替身：
grep -rnE "(mock|stub|fake)[A-Z]\w*" src/ app/ lib/ | grep -v "_test\|\.test\.\|spec\.\|__mocks__"
# 沙箱地址与测试密钥：
grep -rnE "(localhost|127\.0\.0\.1|sandbox|test[-_]?key)" src/ app/ lib/ config/ | grep -vi "test\|dev"
```

**严重级别：** 阻断级。**先确认它是有意为之（灰度/降级）还是残留，再动。**

**推荐（结构改动：换成真实实现或改为显式失败）：** 这属于功能变更而非纯重构——**必须先取得用户批准**，并把它登记为一项**计划内的对外契约破坏**（记录内容、影响的下游调用方、迁移方式）。
**契约影响：** **高**。桩数据的形状常被前端或下游按字段名依赖；替换为真实实现会改变响应结构，按契约迁移处理。

---

### 概念没有权威定义（No Authoritative Definition）
**症状：**
- 同一概念在 5 个地方各有一份定义（领域类型、DB 表、API 字段、前端模型、文档描述），互不一致
- 判定不出哪一份是权威（真值来源），各处"看起来都对"
- 同一字段在不同层类型不同（`id: number` 与 `id: string`）
- 新增字段时只能"照着某处抄一份"

**为什么有害：** 没有权威定义，就无法判定哪一处是"对的"——任何重构都退化成猜测。这也是同名不同义与同义不同名长期潜伏的土壤：从没有人被迫回答"这个概念的唯一定义在哪"。

**检测：**
```
# 同一概念的多份定义点（同名定义在不同层重复出现）：
grep -rnE "(class|interface|type|struct|record|CREATE TABLE)\s+\w*User\w*" .
# 同名定义的数量（>3 即需人工核对）：
grep -rcE "(class|interface|type)\s+\w*(Order|User|Account)\w*" . | grep -v ":0"
# 字段类型不一致：
grep -rnE "\bid\s*[:=]\s*(number|string|int|long|UUID)" .
```

**严重级别：** 严重级；主概念（订单、用户、账户、租户）无权威定义时为阻断级。

**推荐（命名归一）：** 按权威定义判定顺序（数据形状 > DB schema > 对外 API > 领域类型 > 使用频率）定出唯一权威，写进项目文档，再分批把其余名字归一过来。改之前先把"哪个是权威、为什么"讲清楚，否则第三轮又会改回来。

> [!WARNING]
> **冲突不等于错误。** DB 用 `cust_id`、API 用 `customerId` 时，两者都要保留——它们属于不同层的契约。权威定义只用来决定**内部代码**统一叫什么。

**契约影响：** 视判定结果而定。**内部命名归一 ≠ 对外契约改名**，把两者混为一谈是重架构项目炸掉的头号原因。

---

## 坏味道严重级别参考

| 严重级别 | 含义 | 行动 |
|---|---|---|
| **阻断级（Blocker）** | 妨碍安全修改 | 必须在其他工作之前修复 |
| **严重级（Major）** | 显著风险或拖累 | 优先处理；在本次会话中修复 |
| **轻微级（Minor）** | 杂乱或弱耦合 | 顺手修复，或在后续工作中修复 |

**家族 7：语义漂移**的条目按下列默认级别处理（沿用同一列结构；同一坏味道在不同上下文下可能升级，见各条目自身的「严重级别」行）：

| 严重级别 | 含义 | 行动 |
|---|---|---|
| **阻断级（Blocker）** | 【家族 7】同名不同义（D1）；名实不符且带写库副作用（D3）；占位/Mock 实现混入生产路径；主概念无权威定义 | 必须在其他语义改动之前处理；契约处置未判定为「保持不变」或「计划破坏」之前不得进入执行（`safety.md` §9.1） |
| **严重级（Major）** | 【家族 7】同义不同名（D2）；缺失概念（D5，重复 >10 文件或涉及定价/权限）；幽灵概念（D4）；迭代编号泄漏进领域层；扁平化 handler（规则重复时）；伪抽象层掩盖同义不同名时 | 排进语义批次，按影响面从小到大；一个批次只做一类改动（命名归一**或**结构重划） |
| **轻微级（Minor）** | 【家族 7】迭代残留命名；注释与实现矛盾；伪抽象层（仅杂乱、未掩盖语义问题）；扁平化 handler 的纯组织问题 | 顺手修复，或随下一个语义批次一起处理；仍需遵守 §9.1 前置条件 |

---

## 语言无关的 Grep 模式

```
# 过长函数：统计花括号之间的行数（近似）
# 查找内部有许多以空行分隔区块的函数

# 布尔标志参数：
grep -n "function.*true\|false" <file>
grep -n "def .*True\|False" <file>

# 消息链（3 个以上点号）：
grep -n "\.\w\+().*\.\w\+()" <file>

# 类型切换：
grep -n "switch.*type\|switch.*kind\|switch.*role" <file>
grep -n "if.*\.type ===\|if.*\.kind ===" <file>

# 基本类型编码：
grep -n '"premium"\|"standard"\|"admin"\|"user"' <file>

# 被注释掉的代码：
grep -n "^[[:space:]]*\/\/.*[;{}]" <file>    # JS/TS/Java
grep -n "^[[:space:]]*#.*[=:]" <file>         # Python/Ruby
```
