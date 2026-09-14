# 目录：组合方法（Composing Methods）

用于在函数内部以及函数之间重构代码的手法。这些是最常被应用的重构手法——请优先掌握它们。

每一条目的结构：**意图 → 手法 → 示例 → 反向手法 → 注意事项**

---

## 提取方法 / 提取函数（Extract Method / Extract Function）

**意图：** 把一段代码变成函数，让函数名说明它的用途。

**适用场景：**
- 需要靠注释才能说明其作用的代码块
- 在不止一处被使用的代码
- 长方法内部的某个“段落”（尤其是带注释标题的那种）
- 可以用一个动词短语命名的代码

**手法：**
1. 确定要提取的代码片段
2. 创建一个具有描述性名称的新函数
3. 把该片段复制进新函数
4. 找出该片段用到的局部变量——它们会成为参数
5. 找出该片段修改、且在之后仍被使用的变量——它们会成为返回值
6. 用对新函数的调用替换原片段
7. 运行测试

**示例：**
```
// 之前
function printOwing(invoice) {
  let outstanding = 0;
  for (const o of invoice.orders) {
    outstanding += o.amount;
  }
  // 打印明细
  console.log(`name: ${invoice.customer}`);
  console.log(`amount: ${outstanding}`);
}

// 之后
function printOwing(invoice) {
  const outstanding = calculateOutstanding(invoice);
  printDetails(invoice, outstanding);
}

function calculateOutstanding(invoice) {
  return invoice.orders.reduce((sum, o) => sum + o.amount, 0);
}

function printDetails(invoice, outstanding) {
  console.log(`name: ${invoice.customer}`);
  console.log(`amount: ${outstanding}`);
}
```

**反向手法：** 内联方法（Inline Method）

**注意事项：**
- 如果该片段修改了多个局部变量，可能无法干净地提取——可先考虑提取变量（Extract Variable）来降低复杂度
- 如果某个临时变量在片段内被赋值、而在片段之外被使用，就需要把它作为返回值返回
- 新函数的参数列表过长是一种坏味道——它可能自身需要提取变量（Extract Variable）或引入参数对象（Introduce Parameter Object）

---

## 内联方法 / 内联函数（Inline Method / Inline Function）

**意图：** 移除那些函数体与函数名同样清晰、或者单独存在并不带来价值的函数。

**适用场景：**
- 函数体与函数名一样显而易见（例如 `isValid() { return this.score >= 0; }`）
- 该函数只被调用一次，且把函数体内联后调用处同样易读
- 间接层太多——那些仅仅做转发的委托方法

**手法：**
1. 确认该方法不是多态的（未被子类覆写）——如果是，就不要内联
2. 用 Grep 找出所有调用方
3. 用方法体替换每一处调用（把形参替换为实参）
4. 删除该方法
5. 运行测试

**示例：**
```
// 之前
function getRating(driver) {
  return moreThanFiveLateDeliveries(driver) ? 2 : 1;
}
function moreThanFiveLateDeliveries(driver) {
  return driver.numberOfLateDeliveries > 5;
}

// 之后
function getRating(driver) {
  return driver.numberOfLateDeliveries > 5 ? 2 : 1;
}
```

**反向手法：** 提取方法（Extract Method）

**注意事项：**
- 不要内联多态方法
- 如果该方法在超过 5 处被使用，就不要内联——那种情况下通常应朝提取的方向走
- 注意参数的副作用：如果某个实参表达式带有副作用，内联会改变求值顺序

---

## 提取变量 / 引入解释性变量（Extract Variable / Introduce Explaining Variable）

**意图：** 为复杂表达式命名，使代码能够自我说明。

**适用场景：**
- 需要注释才能理解的复杂条件表达式
- 被多次使用的计算值
- 含义不直观的子表达式

**手法：**
1. 确定要命名的表达式
2. 创建一个具有描述性名称的不可变变量，并赋值为该表达式
3. 用该变量替换原表达式（或其后续使用处）
4. 运行测试

**示例：**
```
// 之前
if (platform.toUpperCase().indexOf("MAC") > -1 &&
    browser.toUpperCase().indexOf("IE") > -1 &&
    wasInitialized() && resize > 0) { ... }

// 之后
const isMacOs = platform.toUpperCase().indexOf("MAC") > -1;
const isIEBrowser = browser.toUpperCase().indexOf("IE") > -1;
const wasResized = resize > 0;
if (isMacOs && isIEBrowser && wasInitialized() && wasResized) { ... }
```

**反向手法：** 内联变量（Inline Variable）

**注意事项：**
- 选择能解释领域含义的名称，而不只是描述该表达式在算什么
- 如果该表达式出现在多个方法中，为了复用应改为考虑提取方法（Extract Method）

---

## 内联变量（Inline Variable）

**意图：** 移除那些名称所传达的信息并不比表达式本身更多的变量。

**适用场景：**
- 该变量只被赋值一次，且紧接着只被使用一次
- 变量名并不比它所承载的表达式更清晰
- 该变量只是在提取方法（Extract Method）过程中充当一块踏板

**手法：**
1. 确认该变量只被赋值一次
2. 用右侧表达式替换该变量的唯一一次使用
3. 删除该变量声明
4. 运行测试

**示例：**
```
// 之前
const basePrice = order.basePrice;
return basePrice > 1000;

// 之后
return order.basePrice > 1000;
```

**反向手法：** 提取变量（Extract Variable）

**注意事项：**
- 如果表达式带有副作用且被多次使用，就不要内联
- 如果删除该变量会让代码更难调试（例如删掉计算过程中命名清晰的中间量），就不要内联

---

## 以查询取代临时变量（Replace Temp with Query）

**意图：** 用函数调用替换局部变量，使该计算可被其他方法复用。

**适用场景：**
- 某个临时变量保存着同一个类中其他方法也需要的计算结果
- 该变量由该类可获取的数据计算得出（而不是来自本方法独有的参数）
- 该计算并非廉价到可以忽略（值得命名，也可能值得缓存）

**手法：**
1. 把右侧表达式提取为新方法
2. 用对新方法的调用替换该变量赋值
3. 用对该方法的调用替换该变量的后续使用处
4. 删除已不再使用的变量
5. 运行测试

**示例：**
```
// 之前
function getPrice() {
  const basePrice = quantity * itemPrice;
  const discountFactor = basePrice > 1000 ? 0.95 : 0.98;
  return basePrice * discountFactor;
}

// 之后
function getPrice() {
  return basePrice() * discountFactor();
}
function basePrice() { return quantity * itemPrice; }
function discountFactor() { return basePrice() > 1000 ? 0.95 : 0.98; }
```

**反向手法：** 引入本地扩展（Introduce Local Extension）/ 缓存结果

**注意事项：**
- 仅当该值在方法内不变（变量只被赋值一次）时才成立
- 如果担心重复调用带来的性能问题，加一条注释说明取舍——不要过早优化
- 在把提取出的表达式变成函数之前，先检查它是否有副作用

---

## 拆分临时变量（Split Temporary Variable）

**意图：** 当同一个临时变量因两种不同用途而被反复赋值时，为每种用途各给一个变量。

**适用场景：**
- 某个变量被赋值多次，且这些赋值服务于不同的语义目的
- 变量名已无法准确描述它的全部用途
- 常见于在函数中途被改作他用的“累加器”变量

**手法：**
1. 在第一次赋值处重命名该变量，以反映它的第一个用途
2. 在第二次赋值处引入一个新变量，其名称反映第二个用途
3. 把两次赋值之间的所有引用改为使用第一个名称
4. 把第二次赋值之后的所有引用改为使用第二个名称
5. 运行测试

**示例：**
```
// 之前
let temp = 2 * (height + width);
console.log(temp);
temp = height * width;
console.log(temp);

// 之后
const perimeter = 2 * (height + width);
console.log(perimeter);
const area = height * width;
console.log(area);
```

**反向手法：**（无——这个方向总是正确的）

**注意事项：**
- 常见于从过程式风格转换而来的代码——变量在长函数中被反复用于不相关的用途
- 拆分之后，每个新变量都是提取变量（Extract Variable）的候选

---

## 移除对参数的赋值（Remove Assignments to Parameters）

**意图：** 绝不要修改参数——改用局部变量。

**适用场景：**
- 函数修改了它的某个输入参数
- 该修改本意只想影响本地值，而不是调用方的副本（在按引用传递的语言中尤其相关）

**手法：**
1. 创建一个局部变量，并用该参数的值初始化它
2. 把对该参数（在初次读取之后）的所有修改和使用都换成该局部变量
3. 运行测试

**示例：**
```
// 之前
function discount(inputVal, quantity) {
  if (inputVal > 50) inputVal -= 2;     // 修改了参数
  if (quantity > 100) inputVal -= 1;
  return inputVal;
}

// 之后
function discount(inputVal, quantity) {
  let result = inputVal;
  if (result > 50) result -= 2;
  if (quantity > 100) result -= 1;
  return result;
}
```

**注意事项：**
- 在参数按值传递的语言中（JS/Python/Go 中的大多数基本类型），这是风格问题
- 在对象按引用传递的语言中，这是正确性问题——修改参数就是修改调用方的对象
- 如果函数接收一个对象并调用会改变它的方法，那是另一种坏味道（依恋情结（Feature Envy）或不当的亲密关系（Inappropriate Intimacy））

---

## 替换算法（Substitute Algorithm）

**意图：** 用一个更清晰、结果相同的算法替换复杂算法。

**适用场景：**
- 存在更简单的算法（例如使用此前不知道的语言内置能力）
- 当前算法难以理解，而更清晰的版本同样正确
- 该算法写于覆盖此场景的某个库函数出现之前

**手法：**
1. **在动手之前：** 编写穷尽覆盖当前算法各种输出的测试（如果尚不存在）。这就是行为基线。
2. 实现新算法（在临时位置，或以新函数的形式）
3. 对新算法运行这些测试，确认全部通过
4. 用新算法替换旧算法
5. 再次运行全部测试

**示例：**
```
// 之前——手工查找
function foundPerson(people) {
  for (let i = 0; i < people.length; i++) {
    if (people[i] === "Don" || people[i] === "John" || people[i] === "Kent") {
      return people[i];
    }
  }
  return "";
}

// 之后——使用内置方法
function foundPerson(people) {
  return people.find(p => ["Don", "John", "Kent"].includes(p)) ?? "";
}
```

**注意事项：**
- 这比结构性重构风险更高——你是在重写逻辑，而不只是重新组织它
- 测试必须在替换**之前**编写，而不是之后
- 如果测试不存在，未经用户确认不要继续

---

## 拆分循环（Split Loop）

**意图：** 当循环做了两件互不相关的事时，把它拆成两个循环。

**适用场景：**
- 一个循环同时累积两个用途不同的结果
- 拆分后每个循环都可以被独立命名、提取或优化

**手法：**
1. 复制该循环
2. 从第一份副本中移除第二个操作
3. 从第二份副本中移除第一个操作
4. 运行测试

**示例：**
```
// 之前
let youngest = people[0];
let totalSalary = 0;
for (const p of people) {
  if (p.age < youngest.age) youngest = p;
  totalSalary += p.salary;
}

// 之后
let youngest = people[0];
for (const p of people) {
  if (p.age < youngest.age) youngest = p;
}

let totalSalary = 0;
for (const p of people) {
  totalSalary += p.salary;
}
```

**注意事项：**
- 这会暂时增加迭代次数——对于非常大的集合，请注明这一性能取舍（不过现代 CPU 通过缓存能很好地处理）
- 拆分之后，每个循环都是提取方法（Extract Method）→ 以管道取代循环（Replace Loop with Pipeline）的有力候选

---

## 以管道取代循环（Replace Loop with Pipeline）

**意图：** 用函数式管道（map/filter/reduce）替换命令式循环。

**适用场景：**
- 该循环在构建一个集合或单个聚合值
- 其中的操作（过滤、变换、聚合）可以清晰分离
- 语言原生支持管道操作（JS/TS、Python、Ruby、Kotlin、C#、Rust 等）

**手法：**
1. 确定被迭代的集合
2. 确定其中的操作：过滤元素？变换元素？还是累积单个值？
3. 把每个操作对应到它的管道等价物：过滤 → `filter()`，变换 → `map()`，聚合 → `reduce()`
4. 构建该管道，并赋值给最终变量
5. 删除该循环
6. 运行测试

**示例：**
```
// 之前
const names = [];
for (const i of input) {
  if (i.job === "programmer") {
    names.push(i.name);
  }
}

// 之后
const names = input
  .filter(i => i.job === "programmer")
  .map(i => i.name);
```

**反向手法：**（在性能关键路径上，若分配开销重要，则换回循环）

**注意事项：**
- 对于带复杂提前退出逻辑的循环，不要使用管道——带 `break` 的 `for` 循环无法干净地映射为管道
- 在性能关键的内部循环中，管道分配的开销可能很重要——如果你意识到了这一点，就留下注释
- 避免没有中间变量的超过 4 级的管道——可读性会下降
