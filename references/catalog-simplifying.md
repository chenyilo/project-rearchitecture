# 目录：简化条件与控制流（Simplifying Conditionals & Control Flow）

用于理清复杂条件逻辑的操作。在组合方法之后，这是第二常见的重构需求。

每个条目：**意图 → 手法 → 示例 → 反向手法 → 注意事项**

---

## 分解条件表达式（Decompose Conditional）

**意图：** 把复杂的条件与分支体提取为具名函数，让条件表达式读起来像散文。

**适用场景：**
- 某个 `if` 条件复杂到读者必须停下来仔细解析
- 分支体长到掩盖了整体结构
- 同一条件在多处出现

**手法：**
1. 把条件提取为具名函数（例如 `isSummerRate()`）
2. 把 then 分支提取为具名函数（例如 `summerCharge()`）
3. 把 else 分支提取为具名函数（例如 `regularCharge()`）
4. 用对这些函数的调用替换原来的代码
5. 运行测试

**示例：**
```
// 重构前
if (!date.isBefore(SUMMER_START) && !date.isAfter(SUMMER_END)) {
  charge = quantity * summerRate;
  if (quantity > SUMMER_VOLUME_THRESHOLD) charge -= quantity * summerDiscount;
} else {
  charge = quantity * regularRate + regularServiceCharge;
}

// 重构后
if (isSummer(date)) {
  charge = summerCharge(quantity);
} else {
  charge = regularCharge(quantity);
}
```

**反向手法：** 如果提取出的函数只被调用一次，且函数名并没有增加清晰度，就把它们内联回去

**注意事项：**
- 不要分解本身已是单个单词布尔值的条件 —— `if (isValid)` 无需继续提取
- 分解之后，每个提取出的函数可能暴露出更多坏味道（过长函数、基本类型偏执）

---

## 合并条件表达式（Consolidate Conditional Expression）

**意图：** 把结果相同的一串条件合并为单个表达式。

**适用场景：**
- 多个 `if` 检查全部返回（或执行）相同的内容
- 这串检查读起来是一个单一的逻辑意图

**手法：**
1. 确认所有条件都没有副作用
2. 用 `||`（或相应语言中的 `and`/`or`）合并这些检查
3. 如果合并后的条件能提升清晰度，就把它提取为具名函数（分解条件表达式）
4. 运行测试

**示例：**
```
// 重构前
function disabilityAmount(employee) {
  if (employee.seniority < 2) return 0;
  if (employee.monthsDisabled > 12) return 0;
  if (employee.isPartTime) return 0;
  // ... 实际计算
}

// 重构后
function disabilityAmount(employee) {
  if (isNotEligibleForDisability(employee)) return 0;
  // ... 实际计算
}
function isNotEligibleForDisability(employee) {
  return employee.seniority < 2
    || employee.monthsDisabled > 12
    || employee.isPartTime;
}
```

**反向手法：** 如果每个条件各自具有不同的解释价值，就把它们分解回独立的检查

**注意事项：**
- 如果这些条件是彼此独立的检查、只是恰好共享同一结果，就不要合并 —— 读者会失去「每个检查为何存在」的信息
- 如果任一条件带有副作用，就不要合并

---

## 移除控制标记（Remove Control Flag）

**意图：** 用 `break`、`return` 或 `throw` 取代用于控制循环退出的标记变量。

**适用场景：**
- 某个布尔变量在循环内被赋值，并用作循环退出条件
- 该变量除了控制流程之外没有其它用途

**手法：**
1. 找到为退出循环而给标记赋值的语句
2. 把该赋值替换为 `break`（若在循环中）或 `return`（若函数应当退出）
3. 删除标记变量以及对它的所有引用
4. 运行测试

**示例：**
```
// 重构前
let found = false;
for (const p of people) {
  if (!found) {
    if (p === "Don") { sendAlert(); found = true; }
    if (p === "John") { sendAlert(); found = true; }
  }
}

// 重构后
for (const p of people) {
  if (p === "Don" || p === "John") {
    sendAlert();
    break;
  }
}
```

**注意事项：**
- 有些循环需要在跟踪状态的同时完整跑完 —— 不要移除充当累加器的标记，只移除纯粹控制循环终止的标记
- 在特定结构中不支持 `break` 的语言里，可能需要借助提取方法配合 `return`

---

## 以卫语句取代嵌套条件表达式（Replace Nested Conditional with Guard Clauses）

**意图：** 在函数开头用提前返回处理特殊/边界情况，让主逻辑不再缩进。

**适用场景：**
- 函数因为每条路径都包在条件里而层层嵌套
- 「正常」路径被特殊情况的处理埋住
- 函数只有一条正常路径，却有多条错误/边界情况

**手法：**
1. 找出处理特殊情况的那些条件（null、错误、前置条件失败）
2. 把它们作为提前返回移到函数开头
3. 主逻辑现在位于基础缩进层级
4. 运行测试

**示例：**
```
// 重构前
function getPayAmount(employee) {
  let result;
  if (employee.isSeparated) {
    result = separatedAmount();
  } else {
    if (employee.isRetired) {
      result = retiredAmount();
    } else {
      result = normalPayAmount();
    }
  }
  return result;
}

// 重构后
function getPayAmount(employee) {
  if (employee.isSeparated) return separatedAmount();
  if (employee.isRetired) return retiredAmount();
  return normalPayAmount();
}
```

**反向手法：** 如果逻辑实际上包含权重相同的情况，就移除卫语句（改用合并条件表达式）

**注意事项：**
- 「特殊」情况明显属于例外时（错误状态、前置条件、null 检查）效果最好
- 如果所有条件都是同等有效的业务情况，这样做可能反而降低代码清晰度 —— 改用分解条件表达式
- 某些风格指南偏好单一返回点 —— 大范围应用前先与团队讨论

---

## 以多态取代条件表达式（Replace Conditional with Polymorphism）

**意图：** 把基于类型的条件表达式的每个分支，移入相应子类或策略对象上的方法。

**适用场景：**
- `switch` 或 `if/else if` 链根据对象类型选择行为
- 同一个 switch 在多处出现
- 新增一种类型需要修改多个 switch 语句

**手法：**
1. 如果该条件表达式所在的函数并不属于被 switch 的那个类型，先用移动方法把它放过去
2. 为类型码的每个取值创建一个子类（或策略对象）
3. 把每个分支体移入对应子类，作为共享方法的覆写
4. 基类（或接口）声明该方法；每个子类覆写它
5. 用多态分发（在对象上调用方法）取代条件表达式
6. 运行测试

**示例：**
```
// 重构前
class Bird {
  getSpeed(type) {
    switch(type) {
      case "EUROPEAN": return baseSpeed();
      case "AFRICAN": return baseSpeed() - loadFactor() * numberOfCoconuts;
      case "NORWEGIAN_BLUE": return isNailed ? 0 : baseSpeed(voltage);
    }
  }
}

// 重构后
class EuropeanBird { getSpeed() { return baseSpeed(); } }
class AfricanBird { getSpeed() { return baseSpeed() - loadFactor() * this.numberOfCoconuts; } }
class NorwegianBlueBird { getSpeed() { return this.isNailed ? 0 : baseSpeed(this.voltage); } }
```

**反向手法：** 如果类型种类有限且不太可能增长，改用合并条件表达式

**注意事项：**
- 只有在你预期类型集合会增长时才应用 —— 不要为 2 种类型过度设计
- 在函数式语言中，穷尽式模式匹配往往比多态更清晰 —— 参见 language-profiles.md
- 这是一次规模较大的重构 —— 分步进行：先提取方法，再创建类，然后移动方法

---

## 引入特例 / 空对象（Introduce Special Case / Null Object）

**意图：** 用自身能处理特殊情况的对象，取代反复出现的 null/undefined 检查（或其它特殊情况检查）。

**适用场景：**
- 同一处 null 检查在代码库中多处出现
- 在使用 customer 对象之前，你反复检查 `if (customer === null)`
- 该特殊情况总是导致相同的行为（例如默认值、空操作）

**手法：**
1. 创建一个「特例」类，实现与普通类相同的接口
2. 它的方法针对该特殊情况返回默认行为（例如 `getName()` 返回 "Guest"）
3. 把创建 `null` 的地方（或返回 `null` 的代码）改为创建该特例对象
4. 删除每个调用点的 null 检查
5. 运行测试

**示例：**
```
// 重构前
let customerName;
if (customer === null) {
  customerName = "occupant";
} else {
  customerName = customer.name;
}

// 重构后（使用 NullCustomer 对象）
const customerName = customer.name; // NullCustomer.name 返回 "occupant"
```

**注意事项：**
- 只有当同一个 null 检查出现 3 次以上、且兜底行为相同时才值得做
- 如果不同调用点希望 null 情况有不同的行为，就不要使用 —— 那种场景下条件逻辑更清晰
- 特例对象必须实现真实对象的完整接口 —— 不要创建残缺的桩对象

---

## 将查询与修改分离（Separate Query from Modifier，命令查询分离）

**意图：** 确保一个函数要么返回值（查询），要么改变状态（命令）—— 绝不同时做两件事。

**适用场景：**
- 一个函数既返回值又有副作用
- 调用方无法只检查取值而不触发副作用
- 测试变得复杂，因为调用查询就会改变状态

**手法：**
1. 创建一个返回值的新函数（查询）—— 复制当前函数体
2. 修改原函数使其没有返回值（命令）—— 它只执行副作用
3. 更新调用方：对需要该值的调用方，先调用查询，再调用命令
4. 运行测试

**示例：**
```
// 重构前
function getTotalOutstandingAndSendBill() {
  const result = customer.invoices.reduce((total, each) => each.amount + total, 0);
  sendBill();
  return result;
}

// 重构后
function totalOutstanding() {
  return customer.invoices.reduce((total, each) => each.amount + total, 0);
}
function sendBill() { /* 仅副作用 */ }

// 调用方：
const amount = totalOutstanding();
sendBill();
```

**注意事项：**
- 在函数式/响应式代码中，CQS 本就是默认做法 —— 这条在 OO 代码中最相关
- 隐式更新读取次数或最后访问字段的数据库查询是常见违规 —— 考虑这些副作用是否真的重要

---

## 引入断言（Introduce Assertion）

**意图：** 在代码中把某个假设显式表达出来，使其被违反时大声失败。

**适用场景：**
- 代码假设某个值在某个范围内、非空或具有特定属性，但从不检查
- 注释里写着「这个值绝不应为负」或「调用方必须保证 X」
- 存在一个从函数签名看不出来的隐藏前置条件

**手法：**
1. 找出该假设
2. 在假设应当成立的位置添加断言（依语言而定：`assert`、`console.assert`、`Debug.Assert`、`invariant(...)`）
3. 断言应在开发环境中抛出或大声崩溃，但在生产环境中可能被禁用（取决于语言）
4. 运行测试 —— 确认断言不会在合法输入上触发

**示例：**
```
// 重构前
function applyDiscount(product, discount) {
  return product.price * (1 - discount);  // 假设 discount 在 0..1 之间
}

// 重构后
function applyDiscount(product, discount) {
  console.assert(discount >= 0 && discount <= 1, `discount must be 0-1, got ${discount}`);
  return product.price * (1 - discount);
}
```

**注意事项：**
- 断言用于程序员错误（不变量），不用于用户输入校验
- 不要把断言当作运行时情况（网络故障、畸形输入）的错误处理
- 在生产系统中，断言可能被构建工具剥离 —— 任何关键不变量也要在测试中写清楚

---

## 移除死代码（Remove Dead Code）

**意图：** 删除永远不可能被执行的代码。

**适用场景：**
- 条件表达式的某个分支可证明不可达
- 函数没有任何调用点（经 Grep 确认）
- 变量被赋值但从未被读取
- 特性开关始终开启或始终关闭

**手法：**
1. 确认代码确实不可达 —— 用 Grep 验证没有调用点，并检查是否存在动态分发或反射
2. 删除该代码
3. 运行测试（包括专门针对死代码的测试 —— 那些也应一并删除）

**注意事项：**
- 不要删除你「相当确定」不可达的代码 —— 用 Grep 和 git 历史确认
- 警惕动态分发（反射、eval、基于字符串的方法调用），它们会让 Grep 不足以判断
- 公共 API 方法可能没有内部调用点但存在外部调用方 —— 删除前先看 `safety.md` 第 3 节

---

## 简化布尔表达式（Simplify Boolean Expression）

**意图：** 移除只增加噪音、不增加信息的冗余布尔判断。

**常见模式：**

```
// 模式 1：与布尔字面量的冗余比较
if (isValid === true)   →   if (isValid)
if (found === false)    →   if (!found)
return x > 0 ? true : false   →   return x > 0

// 模式 2：双重否定
if (!(!condition))   →   if (condition)
!!value   →   Boolean(value)  （或在布尔上下文中直接用 value）

// 模式 3：return 之后多余的 else
if (condition) {
  return x;
} else {       // return 之后 else 不可达
  return y;
}
→
if (condition) return x;
return y;

// 模式 4：返回条件本身的三元表达式
condition ? true : false   →   condition
condition ? false : true   →   !condition
```

**手法：** 识别模式，应用简化，运行测试。

**注意事项：**
- 在具有 truthy/falsy 语义的语言（JS、Python、Ruby）中，`!!value` 的意义不止是简化 —— 它会把值强制转换为严格的布尔值。只有当类型已经是布尔时才能简化。
- `return x > 0 ? true : false` → `return x > 0` 仅在返回类型是布尔时才安全 —— 如果返回类型可能为 `undefined` 或 `null`，原写法可能是有意为之（不过仍值得质疑）
