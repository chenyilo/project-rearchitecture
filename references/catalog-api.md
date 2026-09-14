# 目录：API 与方法签名重构

用于改善函数与模块调用方式的操作。它们会影响代码单元之间的契约 —— 在对公共 API 应用其中任何一项之前，请先阅读 safety.md §3。

每条目的结构：**意图 → 手法 → 示例 → 反向手法 → 注意事项**

---

## 重命名函数 / 方法 / 变量 / 类（Rename Function / Method / Variable / Class）

**意图：** 修改名称，使其更好地传达用途，并与领域语言保持一致。

**适用场景：**
- 名称具有误导性、过于缩写，或使用了错误的领域术语
- 函数的行为已经演进到超出其名称所暗示的范围
- 保持代码库中的命名一致（例如所有查找函数都应叫 `findX`，不要与 `getX` 混用）

**手法（安全，适用于内部/私有符号）：**
1. 用新名称创建一个新函数，函数体相同
2. 修改旧函数，让它调用新函数（委托垫片）
3. 逐步更新所有调用方以使用新名称（使用 Grep）
4. 当所有调用方都更新完毕后，删除旧函数
5. 运行测试

**手法（快速，适用于真正的局部作用域）：**
1. 使用编辑器的重命名工具或 Grep+Edit 就地重命名
2. 运行测试

**注意事项：**
- **红线：** 重命名导出的/公共符号 —— 见 safety.md §3
- 重命名前总是用 Grep 统计调用点：`grep -rn "oldName" .`
- 调用点超过 20 处时为黄线 —— 继续之前先警告用户
- 在动态类型语言（JS、Python、Ruby）中，可能存在基于字符串的引用（`getattr(obj, "methodName")`），Grep 无法捕获 —— 也要搜索字符串出现的位置

---

## 添加参数（Add Parameter）

**意图：** 为需要额外上下文才能计算出结果的函数添加一个新参数。

**适用场景：**
- 某个函数目前从全局状态或外部依赖中获取本应由调用方提供的值
- 你正在通过注入依赖来引入可测试性

**手法：**
1. 如果可能，为该参数添加一个合理的默认值（以保持向后兼容）
2. 使用 Grep 找出所有调用点
3. 更新所有调用点以传入新实参
4. 当所有调用点都更新完毕后移除默认值（如果默认值只是迁移期的辅助手段）
5. 运行测试

**注意事项：**
- 向公共 API 添加必填参数是破坏性变更 —— 红线（safety.md §3）
- 如果新增参数超过 1–2 个，考虑改用引入参数对象

---

## 移除参数（Remove Parameter）

**意图：** 移除函数从未使用过的参数。

**适用场景：**
- 某个参数总是被传入，但从未在函数体中被真正使用
- 某次重构使该参数变得不必要

**手法：**
1. 确认该参数确实未被使用（也要检查子类中的所有重写）
2. 从函数签名中移除该参数
3. 使用 Grep 更新所有调用点，不再传入该实参
4. 运行测试

**注意事项：**
- **红线：** 从公共 API 移除参数 —— 见 safety.md §3
- 在动态类型语言中，未使用的参数可能是约定签名的一部分（回调、事件处理器）—— 移除前先确认

---

## 函数参数化（Parameterize Function）

**意图：** 通过添加参数，把两个或多个执行相似逻辑但取值不同的函数统一起来。

**适用场景：**
- 两个函数除了一个字面量取值外做的事完全相同
- 添加一个参数后，可以用一个函数取代这两者

**手法：**
1. 找出两个函数之间变化的那个值
2. 创建统一的函数，并为该值添加一个参数
3. 把对两个原函数的调用替换为对新统一函数的调用
4. 删除两个原函数
5. 运行测试

**示例：**
```
// BEFORE
function tenPercentRaise(person) { person.salary *= 1.10; }
function fivePercentRaise(person) { person.salary *= 1.05; }

// AFTER
function raise(person, factor) { person.salary *= (1 + factor); }
```

**反向手法：** 如果统一后的函数因为代码路径过多而变得过于复杂，则拆分它

**注意事项：**
- 如果两个函数的差异不止一个字面量取值（逻辑路径不同），参数化就是错误的工具 —— 应先提取，然后使用多态
- 向公共 API 添加参数是红线

---

## 移除标记参数（Remove Flag Argument）

**意图：** 用一个布尔参数来选择行为的做法，替换为两个分别显式命名的函数。

**适用场景：**
- 某个函数有一个布尔参数，它会从根本上改变函数的行为
- 调用处读起来像这样：`process(order, true)` —— 布尔值在调用点含义不明
- 函数体顶层就是一个 `if (flag)`

**手法：**
1. 创建两个新函数，分别对应标记所控制的两条路径
2. 把相应的函数体移入各自函数中
3. 更新所有调用方，调用正确的具名函数
4. 删除带标记参数的原始函数
5. 运行测试

**示例：**
```
// BEFORE
function setDimension(name, value) {
  if (name === "height") { this.height = value; return; }
  if (name === "width") { this.width = value; return; }
}
// Called as: setDimension("height", 10)  ← 含义不明

// AFTER
function setHeight(value) { this.height = value; }
function setWidth(value) { this.width = value; }
```

**注意事项：**
- 如果该布尔标记是合理的领域参数（例如查询用的 `includeArchived: boolean`），不要应用此手法
- 如果函数有很多标记，考虑使用参数对象或建造者（Builder）模式

---

## 保持对象完整（Preserve Whole Object）

**意图：** 传入整个对象，而不是从中提取多个字段再传入。

**适用场景：**
- 某个函数接收的若干取值都来自同一个源对象
- 同一组字段总是被一起传入（数据泥团）
- 被调方通过参数伸手获取调用方的局部状态

**手法：**
1. 修改函数，让它接收源对象而不是各个单独的取值
2. 更新函数体，改用该对象的字段/方法
3. 更新所有调用点以传入该对象
4. 运行测试

**示例：**
```
// BEFORE
const low = room.daysTempRange.low;
const high = room.daysTempRange.high;
if (plan.withinRange(low, high)) { ... }

// AFTER
if (plan.withinRange(room.daysTempRange)) { ... }
```

**反向手法：** 如果函数不应依赖整个对象，则改为以查询取代参数

**注意事项：**
- 这会在函数与对象类型之间建立依赖 —— 只有在这种耦合合理时才值得做
- 如果函数位于不同模块中，传入整个对象可能引入不想要的依赖

---

## 以查询取代参数（Replace Parameter with Query）

**意图：** 移除一个参数，让函数自行推导出该值。

**适用场景：**
- 某个参数值可以从函数中已有的其他信息推导出来
- 调用方总是计算同一个表达式来作为该实参传入

**手法：**
1. 如果推导过程复杂，先用提取方法把推导逻辑提取出来
2. 从函数签名中移除该参数
3. 更新函数体，改为调用该查询
4. 更新所有调用点，不再传入该实参
5. 运行测试

**示例：**
```
// BEFORE
function finalPrice(basePrice, discountLevel) {
  return basePrice - discountFor(discountLevel);
}
// called as: finalPrice(base, discountLevel(quantity))

// AFTER
function finalPrice(basePrice) {
  return basePrice - discountFor(discountLevel());
}
function discountLevel() { return quantity > 100 ? 2 : 1; }
```

**反向手法：** 以参数取代查询

**注意事项：**
- 仅当该查询没有副作用，且在函数上下文中总是确定性的才有效
- 如果函数是纯计算、不应访问外部状态，则保留该参数

---

## 以参数取代查询（Replace Query with Parameter）

**意图：** 添加一个参数，用于传入函数当前在内部推导出的值 —— 以提高纯粹性和可测试性。

**适用场景：**
- 函数查询了你想要注入的全局状态或外部依赖
- 你希望把函数变成纯函数，以便更容易测试
- 将查询与计算分离（与 CQS 相关）

**手法：**
1. 把内部查询提取到一个变量中
2. 为该值添加一个参数
3. 更新函数体以使用该参数
4. 更新所有调用点以传入该值
5. 运行测试

**示例：**
```
// BEFORE — function reaches out to global thermostat
function targetTemperature(plan) {
  const currentTemp = thermostat.currentTemperature;
  return plan.target > currentTemp ? "heat" : "cool";
}

// AFTER — pure, testable
function targetTemperature(plan, currentTemperature) {
  return plan.target > currentTemperature ? "heat" : "cool";
}
```

**反向手法：** 以查询取代参数

---

## 引入参数对象（Introduce Parameter Object）

**意图：** 用单个对象取代一组相关参数。

**适用场景：**
- 多个参数总是同时出现（数据泥团坏味道）
- 过长的参数列表（超过 3–4 个参数）
- 该参数组代表一个值得命名的领域概念

**手法：**
1. 为该参数组创建一个新的类/记录/结构体
2. 为函数添加一个该新类型的参数
3. 更新函数体以使用该对象的字段
4. 更新所有调用点以构造并传入该对象
5. 移除原先的各个单独参数
6. 运行测试

**示例：**
```
// BEFORE
function amountInvoiced(startDate, endDate) { ... }
function amountReceived(startDate, endDate) { ... }
function amountOverdue(startDate, endDate) { ... }

// AFTER
class DateRange { constructor(startDate, endDate) { ... } }
function amountInvoiced(range) { ... }
function amountReceived(range) { ... }
function amountOverdue(range) { ... }
```

**注意事项：**
- 参数对象是值对象 —— 让它不可变
- 创建对象之后，把行为移入其中（以查询取代临时变量、移动方法），以获得真正的领域价值

---

## 以工厂函数取代构造函数（Replace Constructor with Factory Function）

**意图：** 用具名的工厂函数取代构造函数，以获得清晰性与灵活性。

**适用场景：**
- 存在多个含义不同的构造函数（重载造成混淆）
- 构造函数的签名没有意义或令人困惑
- 你需要根据输入返回不同的子类
- 你想缓存或池化实例

**手法：**
1. 创建一个静态工厂方法（或顶层工厂函数）
2. 在工厂中调用构造函数
3. 更新所有调用点以使用该工厂
4. 可选：把构造函数改为 private/protected
5. 运行测试

**示例：**
```
// BEFORE
const employee = new Employee("full-time", name, salary);

// AFTER
const employee = Employee.createFullTime(name, salary);
```

**注意事项：**
- 工厂函数不太适合被继承 —— 如果很可能会扩展，请考虑这一点
- 红线：如果构造函数属于公共 API 的一部分，把它改成工厂属于破坏性变更

---

## 以异常 / Result 类型取代错误码（Replace Error Code with Exception / Result Type）

**意图：** 使用恰当的错误信号机制，而不是魔法返回值。

**适用场景：**
- 函数返回 `-1`、`null`、`""` 或 `0` 来表示失败
- 调用方忘记检查错误返回值（静默失败）
- 语言支持异常、Result 类型或 Option 类型

**手法：**
1. 创建一个异常类（或者，如果该语言的惯用做法如此，则使用 Result/Option 类型 —— 见 language-profiles.md）
2. 修改函数，抛出/返回异常或错误类型，而不是魔法值
3. 更新所有调用方以处理该异常/Result
4. 运行测试

**示例：**
```
// BEFORE (error code)
function findCustomer(id) {
  const customer = db.lookup(id);
  if (!customer) return -1;  // caller must check for -1
  return customer;
}

// AFTER (exception)
function findCustomer(id) {
  const customer = db.lookup(id);
  if (!customer) throw new CustomerNotFoundError(id);
  return customer;
}
```

**注意事项：**
- **红线：** 改变公共 API 的错误契约 —— 捕获特定错误值或错误类型的调用方会出错
- 在 Go 和 Rust 中，返回错误是惯用做法 —— 不要强行把异常引入这些语言
- 在 TS/JS 中，对于可恢复的错误，考虑返回 `Result<T, E>` 类型而不是异常

---

## 返回修改后的值（Return Modified Value）

**意图：** 不要修改参数，而是返回新值，让调用方重新赋值。

**适用场景：**
- 函数接收一个数据结构、修改它，而调用方期望这次修改
- 你想朝着不可变数据模式演进
- 函数目前会就地修改某个参数（移除对参数的赋值坏味道）

**手法：**
1. 修改函数，返回修改后的值，而不是修改参数
2. 更新所有调用方以接收返回值：`data = transform(data)`
3. 从函数体中移除就地修改
4. 运行测试

**示例：**
```
// BEFORE — mutates the passed-in list
function addItem(list, item) {
  list.push(item);  // mutation
}
addItem(myList, newItem);

// AFTER — returns new value
function addItem(list, item) {
  return [...list, item];
}
myList = addItem(myList, newItem);
```

**注意事项：**
- 在性能关键的代码中，每次调用都创建新对象是有代价的 —— 请注意这一权衡
- 确保所有调用方都重新赋值返回值；丢弃返回值是此重构后常见的缺陷
