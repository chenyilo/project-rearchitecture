# 目录：现代与 Fowler 之后的模式

面向使用现代语言与范式编写的代码的重构手法。这些模式晚于 Fowler 原始目录，针对函数式编程、async/await、响应式编程、依赖注入以及当代语言惯用法。

---

## 函数式编程模式

---

### 以 reduce 取代可变累加器（Replace Mutable Accumulator with Reduce）

**意图：** 用函数式的 fold/reduce 取代命令式的累加器循环。

**适用场景：**
- 循环在开始前初始化一个变量，并在循环内部向其累加
- 累加逻辑是纯转换（循环体中没有副作用）

**示例：**
```
// 之前
let total = 0;
for (const item of items) {
  total += item.price * item.quantity;
}

// 之后
const total = items.reduce((sum, item) => sum + item.price * item.quantity, 0);
```

**注意事项：**
- 当循环体有副作用时不要使用 reduce —— reduce 意味着无副作用
- 复杂的多步归约可能比循环更难读 —— 使用具名的归约函数而不是内联 lambda

---

### 引入管道组合（Introduce Pipeline Composition）

**意图：** 将一系列纯函数组合成管道，使数据转换显式化，并让每一步都可独立测试。

**适用场景：**
- 一个函数对数据施加一系列转换
- 每个转换步骤都可以命名并独立测试
- 这个序列本身就是逻辑（"做什么"），而不是实现细节（"怎么做"）

**示例：**
```
// 之前
function processOrders(orders) {
  const result = [];
  for (const o of orders) {
    if (o.status === "paid") {
      const withTax = { ...o, total: o.subtotal * 1.1 };
      if (withTax.total > 100) {
        result.push({ id: withTax.id, total: withTax.total });
      }
    }
  }
  return result;
}

// 之后
const processOrders = (orders) =>
  orders
    .filter(isPaid)
    .map(applyTax)
    .filter(isSignificant)
    .map(toSummary);

const isPaid = (o) => o.status === "paid";
const applyTax = (o) => ({ ...o, total: o.subtotal * 1.1 });
const isSignificant = (o) => o.total > 100;
const toSummary = (o) => ({ id: o.id, total: o.total });
```

**注意事项：**
- 对非常大的集合使用管道会产生中间数组 —— 在热点路径中，考虑使用 transducer 或显式循环的单遍方案
- 为保持可读性，管道阶段应 ≤4；超过时应为中间结果命名

---

### 以不可变数据 + 纯函数取代共享可变状态（Replace Shared Mutable State with Immutable Data + Pure Functions）

**意图：** 通过让数据流经函数并返回新值，消除共享可变状态。

**适用场景：**
- 多个函数共享一个可变对象，且变更顺序很重要
- 代码难以测试，因为它依赖变更的顺序
- Bug 来自共享对象的意外变更

**手法：**
1. 找出可变的共享状态
2. 修改函数，让其接受状态作为输入并返回修改后的状态作为输出
3. 用构造新对象（`{...old, field: newValue}`）取代原地修改
4. 在函数调用之间显式传递状态
5. 运行测试

**注意事项：**
- 这会让调用签名更复杂 —— 考虑把状态归组为一个带类型的 record
- 在性能关键的代码中，可能需要结构共享（不可变数据结构）以避免过多的拷贝

---

## 异步与并发模式

---

### 以 async/await 取代回调金字塔（Replace Callback Pyramid with async/await）

**意图：** 将深层嵌套的回调（"回调地狱"）展平为顺序的 async/await 代码。

**适用场景：**
- 多个嵌套回调，每个都依赖前一个的结果
- 错误处理在每个嵌套层级上重复
- 代码的意图被嵌套掩盖，难以看清

**示例：**
```
// 之前 — 回调金字塔
readFile(path, (err, data) => {
  if (err) return handleError(err);
  parseJSON(data, (err, parsed) => {
    if (err) return handleError(err);
    fetchUser(parsed.userId, (err, user) => {
      if (err) return handleError(err);
      respond(user);
    });
  });
});

// 之后 — async/await
async function process(path) {
  const data = await readFile(path);
  const parsed = await parseJSON(data);
  const user = await fetchUser(parsed.userId);
  respond(user);
}
```

**注意事项：**
- 对整个 async 函数使用单个 `try/catch` 块；如果不同的错误需要不同的处理，则对每一步使用各自的 `try/catch`
- 当操作相互独立时，不要在循环中 `await` —— 改用 `Promise.all` 并行化

---

### 合并独立的异步操作（Consolidate Independent Async Operations）

**意图：** 并行运行相互独立的异步操作，而不是顺序执行。

**适用场景：**
- 顺序出现多个 `await` 调用，其中第二个不依赖第一个
- 本可并发运行的操作被顺序执行，带来不必要的延迟

**示例：**
```
// 之前 — 顺序执行（浪费时间）
const user = await fetchUser(userId);
const settings = await fetchSettings(userId);
const posts = await fetchPosts(userId);

// 之后 — 并行
const [user, settings, posts] = await Promise.all([
  fetchUser(userId),
  fetchSettings(userId),
  fetchPosts(userId),
]);
```

**各语言对应写法：**
- JS/TS：`Promise.all`
- Python：`asyncio.gather`
- Go：goroutine 配合 WaitGroup 或 errgroup
- C#：`Task.WhenAll`
- Rust：`tokio::join!` 或 `futures::join!`

**注意事项：**
- `Promise.all` 会在任意一个 promise 被拒绝时立即拒绝 —— 如果需要所有结果（包括失败），请使用 `Promise.allSettled`
- 并行化大量操作时，注意限流与资源竞争

---

### 避免 async void / 无错误处理的即发即弃（Avoid Async Void / Fire-and-Forget Without Error Handling）

**意图：** 确保异步操作要么其结果是 await 的，要么有显式的错误处理。

**适用场景：**
- 函数返回 Promise/Task，但调用方忽略了它（没有 `await`，也没有 `.catch`）
- async 函数的返回类型是 `void`，且调用时没有错误处理

**手法：**
1. 找出即发即弃的调用
2. 如果结果不重要但错误重要：添加 `.catch(handleError)`，或在 async 函数内部使用 `try/catch`
3. 如果结果重要：await 它，或把 Promise 返回给调用方
4. 运行测试

**注意事项：**
- 在某些框架中，即发即弃是刻意为之（后台任务、分析埋点）—— 即便如此也要加上显式的错误日志
- 在 Go 中，goroutine 默认就是即发即弃的 —— 始终从错误 channel 读取，或使用 wait group

---

## 依赖注入与模块化

---

### 为可测试性提取接口（Extract Interface for Testability，引入接缝）

**意图：** 引入接口/协议，使依赖可以被测试替身替换。

**适用场景：**
- 类直接实例化一个难以测试的依赖（数据库、网络、时间）
- 你希望在测试中注入 mock、stub 或 fake

**手法：**
1. 从具体依赖中提取接口（Extract Interface）
2. 修改该类，使其依赖接口而非具体类型
3. 通过构造函数或工厂注入依赖
4. 在测试中注入测试替身；在生产中注入真实实现
5. 运行测试

**示例：**
```
// 之前 — 硬依赖真实时钟
class SubscriptionRenewal {
  isExpired() { return new Date() > this.expiryDate; }
}

// 之后 — 时钟由外部注入
interface Clock { now(): Date; }
class SubscriptionRenewal {
  constructor(private clock: Clock) {}
  isExpired() { return this.clock.now() > this.expiryDate; }
}
```

---

### 以注入的依赖取代静态调用（Replace Static Call with Injected Dependency）

**意图：** 移除对静态方法或单例的调用，改为注入依赖。

**适用场景：**
- 代码以静态方式调用 `Logger.log()`、`Config.get()`、`Database.query()`
- 静态调用使测试和替换实现变得不可能

**手法：**
1. 从静态类中提取接口
2. 添加一个接口类型的构造函数参数
3. 用注入依赖上的实例调用取代静态调用
4. 更新所有构造函数与工厂，传入真实实现
5. 运行测试

**注意事项：**
- 不要什么都注入 —— 只注入有多个实现（测试 vs. 生产）或需要被 mock 的依赖

---

### 将配置合并为带类型的配置对象（Consolidate Configuration into Typed Config Object）

**意图：** 用启动时加载的单个经过校验、带类型的配置对象，取代散落的 `process.env` / `os.environ` / `config.get()` 调用。

**适用场景：**
- 配置值在整个代码库中从环境读取
- 缺失或错误的配置在运行时才于调用栈深处被发现
- 无法清晰了解应用需要哪些配置

**手法：**
1. 创建一个 `Config` 类/record，在一处读取所有环境变量
2. 在构造时校验并赋予类型（必填值缺失时在启动阶段抛出异常）
3. 沿依赖图注入 `Config` 对象
4. 用 `config.X` 取代所有散落的 `process.env.X` 读取
5. 运行测试

---

## 响应式与事件驱动

---

### 以 Observable/事件取代轮询（Replace Polling with Observable/Event）

**意图：** 用状态变化时触发的 事件/observable，取代检查状态变化的轮询循环。

**适用场景：**
- 代码在循环或定时器中反复检查 `if (someCondition)`
- 该条件由已经存在或可添加的事件控制

**示例：**
```
// 之前 — 轮询
setInterval(() => {
  if (queue.hasMessages()) { processNext(queue.dequeue()); }
}, 100);

// 之后 — 事件驱动
queue.on("message", (msg) => { processNext(msg); });
```

**注意事项：**
- 事件要求事件源知晓监听者 —— 如果事件源与消费者不应互相知晓，考虑通过消息总线解耦

---

### 以状态机取代命令式状态变更（Replace Imperative State Mutation with State Machine）

**意图：** 用显式状态机取代散落的 `if (status === X) { status = Y }` 逻辑。

**适用场景：**
- 对象有一个在多个值之间迁移的 `status` 或 `state` 字段
- 由于缺乏约束，出现非法的状态迁移
- 迁移逻辑分散在多个方法中

**手法：**
1. 列出所有合法状态与所有合法迁移
2. 创建状态机（可以是简单表：用 `Map<State, Set<State>>` 表示允许的迁移）
3. 通过单个 `transition(event)` 方法集中所有状态迁移逻辑
4. 用对 `transition` 的调用取代散落的状态变更
5. 运行测试 —— 包括针对非法迁移的测试

---

## 现代语言惯用法

---

### 优先使用穷尽式模式匹配（Prefer Exhaustive Pattern Matching）

**意图：** 当情形集合已知且有限时，使用语言原生的模式匹配而不是 if/else 链。

**语言：** Rust（`match`）、Kotlin（`when`）、C#（`switch expression`）、Python 3.10+（`match`）、Swift（`switch`）、Scala、Haskell

**示例（Kotlin）：**
```
// 之前
fun describe(obj: Any): String {
  if (obj is Int) return "Int: $obj"
  else if (obj is String) return "String of length ${obj.length}"
  else return "Unknown"
}

// 之后
fun describe(obj: Any) = when (obj) {
  is Int -> "Int: $obj"
  is String -> "String of length ${obj.length}"
  else -> "Unknown"
}
```

**核心收益：** 穷尽性检查 —— 如果向 sealed class/enum 添加新情形，编译器会就未处理的匹配发出警告。

---

### 在调用点使用具名参数 / 关键字参数（Use Named Arguments / Keyword Arguments at Call Sites）

**意图：** 通过使用具名参数让调用点自解释，尤其是布尔与数字字面量。

**示例：**
```
// 之前 — 这里的 `true` 是什么意思？
createUser("Alice", true, false, 30);

// 之后 — 清晰
createUser(name: "Alice", isAdmin: true, isBanned: false, age: 30);
// 或者用 Python：
create_user("Alice", is_admin=True, is_banned=False, age=30)
```

**注意事项：**
- 这关乎调用点的清晰度 —— 如果函数只有一个参数，具名参数就没有必要
- 在没有原生具名参数的语言中（Java、解构语法之前的 JS），改用参数对象

---

### 对不可变数据优先使用值类型 / Record / Data Class（Prefer Value Types / Records / Data Classes for Immutable Data）

**意图：** 在不需要变更时，使用语言原生的不可变数据类型而不是可变对象。

| 语言 | 不可变数据类型 |
|---|---|
| Kotlin | `data class` + `val` 字段 |
| C# | `record` |
| Python | `@dataclass(frozen=True)` 或 `NamedTuple` |
| Swift | `struct` |
| Rust | struct（默认不可变） |
| Java | `record`（Java 16+） |
| Scala | `case class` |

**适用场景：**
- 类只持有数据，除相等性与展示外没有行为
- 对象代表某个时间点的快照（事件、测量值、配置）
- 你希望具有结构相等性（`==` 比较字段，而非身份）

**注意事项：**
- 值类型是拷贝而非共享 —— 注意大结构的性能影响
- 如果类型需要频繁演进（增删字段），可变类可能更易使用
