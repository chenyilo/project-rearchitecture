# 目录：架构重构

用于在架构分层之间移动代码的操作 —— 安全、增量，且不改变行为。

**使用这里的任何操作之前：** 完整阅读 `safety.md` §8。架构重构总会触发 §8 与大型代码库协议（§6）。在动手之前，你必须先摸清当前架构。

每个条目的结构：**意图 → 适用场景 → 前置条件 → 手法 → 示例 → 注意事项**

---

## 移动不变量（The Moving Invariant）

**永远不要在同一步中同时改变逻辑和位置。**

每一次架构层面的移动都遵循这个三步序列：

```
步骤 A —— 引入：创建新位置。旧位置委托给新位置。测试通过。
步骤 B —— 重定向：所有调用方改用新位置。旧位置此时已无人使用。测试通过。
步骤 C —— 移除：删除旧位置。测试通过。
```

如果任何一步测试失败，只回退那一步。不要继续下一步。

---

## 提取 ViewModel（Extract ViewModel）

**意图：** 把状态派生、格式化与显示逻辑从 View/Activity/Fragment/Component 移入专门的 ViewModel，让视图成为被动的渲染器。

**适用场景：**
- View/Activity/Fragment 计算派生状态（例如格式化货币、过滤列表、计算可见性）
- 业务或展示逻辑存在于事件处理器 / 生命周期方法中
- View 难以做单元测试，因为它混在一起了 UI 与逻辑
- 模式目标：MVVM

**前置条件（开始前必须全部成立）：**
- [ ] 已完整阅读当前视图文件
- [ ] 将要移动的所有逻辑都已识别并列出（移动中途不出现意外）
- [ ] 现有测试覆盖了将要移动的逻辑
- [ ] 已确定目标平台的 ViewModel 机制（Android：`ViewModel` + `StateFlow`/`LiveData`；iOS：`ObservableObject`；Web：组件状态 hook / store）

**手法：**
1. 在视图旁边创建新的 `[FeatureName]ViewModel` 类/文件
2. 为视图当前计算的每一份状态添加一个字段/属性
3. **步骤 A（引入）：** 把第一个逻辑单元移入 ViewModel。视图委托：调用 ViewModel，渲染结果。运行测试。
4. **步骤 B（重定向）：** 更新视图以观察 ViewModel 的状态（绑定、StateFlow、响应式订阅）。运行测试。
5. **步骤 C（移除）：** 从视图中删除该逻辑。运行测试。
6. 对每个额外的逻辑单元重复步骤 A–C —— 一次一个。

**示例：**
```kotlin
// 之前 — Activity 承担了太多职责
class OrderActivity : AppCompatActivity() {
    override fun onCreate(...) {
        val subtotal = order.items.sumOf { it.price * it.quantity }
        val tax = subtotal * 0.1
        val total = subtotal + tax
        val label = if (total > 100) "Free shipping!" else "Add ${100 - total} for free shipping"
        totalTextView.text = "$${"%.2f".format(total)}"
        shippingLabel.text = label
    }
}

// 之后 — 逻辑由 ViewModel 持有
class OrderViewModel(private val order: Order) : ViewModel() {
    val total: Double get() = order.items.sumOf { it.price * it.quantity } * 1.1
    val shippingLabel: String get() =
        if (total > 100) "Free shipping!" else "Add ${"%.2f".format(100 - total)} for free shipping"
    val formattedTotal: String get() = "$${"%.2f".format(total)}"
}

class OrderActivity : AppCompatActivity() {
    private val viewModel: OrderViewModel by viewModels()
    override fun onCreate(...) {
        totalTextView.text = viewModel.formattedTotal
        shippingLabel.text = viewModel.shippingLabel
    }
}
```

**注意事项：**
- ViewModel 不得持有 View 或 Activity context 的引用 —— 这会导致内存泄漏
- 如果 ViewModel 需要异步数据，请使用作用域限定在 ViewModel（而非 View）上的 coroutine / Combine / RxJava
- 平台生命周期很重要：Android 的 ViewModel 能在旋转后存活；SwiftUI 的 StateObject 默认跨导航不保留
- 不要把导航逻辑移入 ViewModel —— 在大多数模式中导航属于 View 的职责

---

## 引入 Presenter（Introduce Presenter）

**意图：** 从 Controller/Activity/ViewController 中提取出 Presenter，使所有展示逻辑都能在没有 UI 框架的情况下做单元测试。

**适用场景：**
- Controller/Activity 难以测试，因为它直接调用 UI 方法
- 逻辑与生命周期回调纠缠在一起
- 你需要替换 UI 实现（例如不同形态、A/B 测试）
- 模式目标：MVP

**前置条件：**
- [ ] 已完整阅读当前的 controller/activity
- [ ] 已清点所有 UI 交互（UI 需要显示什么？controller 响应什么？）
- [ ] 已确认测试替身机制（通过框架 mock View 接口，或手工 stub）

**手法：**
1. 定义 `[Feature]View` 接口，列出 controller 在 UI 上执行的每个显示动作（例如 `showLoading()`、`showError(message)`、`displayResult(data)`）
2. 让真实 View/Activity 实现该接口
3. 创建 `[Feature]Presenter` 类；通过构造函数注入 View 接口
4. **步骤 A：** 把第一个逻辑单元移入 Presenter。Controller/Activity 委托给 Presenter。运行测试。
5. **步骤 B：** 使用 stub/mock 的 View 为移动后的逻辑编写单元测试。确认测试通过。
6. **步骤 C：** 从 Controller/Activity 中删除该逻辑。运行全部测试。
7. 对每个逻辑单元重复 A–C。

**示例：**
```typescript
// 之前 — controller 把逻辑与 UI 耦合在一起
class LoginController {
  async onSubmit(email: string, password: string) {
    this.view.showLoading();
    try {
      const user = await this.authService.login(email, password);
      this.view.navigateTo('/dashboard');
    } catch (e) {
      this.view.showError(e.message);
    } finally {
      this.view.hideLoading();
    }
  }
}

// 之后
interface LoginView {
  showLoading(): void;
  hideLoading(): void;
  showError(message: string): void;
  navigateTo(path: string): void;
}

class LoginPresenter {
  constructor(private view: LoginView, private authService: AuthService) {}

  async onSubmit(email: string, password: string) {
    this.view.showLoading();
    try {
      await this.authService.login(email, password);
      this.view.navigateTo('/dashboard');
    } catch (e) {
      this.view.showError(e.message);
    } finally {
      this.view.hideLoading();
    }
  }
}

// 真实 View 实现 LoginView；测试中用 stub 替代
```

**注意事项：**
- View 接口只应包含显示命令，绝不包含业务逻辑
- 在 View 可能被销毁的语言/环境中，避免让 Presenter 强引用 View —— 使用弱引用或显式 detach
- 如果 Presenter 需要在 View 销毁后异步回调视图，程序会崩溃 —— 把 coroutine/task 的作用域限定在视图的生命周期上

---

## 把业务逻辑下推到领域层（Push Business Logic to Domain Layer）

**意图：** 把领域规则从 service/controller 类移入拥有这些数据的领域对象，让领域模型变得充血而非贫血。

**适用场景：**
- `*Service` 或 `*Manager` 中包含针对单个领域对象（而非跨聚合）的逻辑
- 领域对象只是纯数据容器（只有 getter/setter）
- 同一条规则在多个 service 中重复
- 检测到的坏味道：贫血领域模型（Anemic Domain Model）

**前置条件：**
- [ ] 该规则作用于单个领域对象所拥有的数据
- [ ] 移动该规则不会引入从领域层到外层（outer layer）的依赖（领域层不得导入仓储、HTTP 客户端或 UI 类型）
- [ ] 该规则在其当前位置已有测试

**手法：**
1. 识别该规则，确认它属于单个领域对象
2. **步骤 A：** 把方法添加到领域对象上。Service 委托：调用领域方法。运行测试。
3. **步骤 B：** 更新所有重复此规则的 service，改为调用领域方法。每次更新一个 service 后运行测试。
4. **步骤 C：** 删除原来的 service 方法。运行测试。

**示例：**
```python
# 之前 — 贫血的 Order、臃肿的 OrderService
class Order:
    def __init__(self, items, customer_tier):
        self.items = items
        self.customer_tier = customer_tier

class OrderService:
    def calculate_discount(self, order: Order) -> float:
        subtotal = sum(i.price * i.qty for i in order.items)
        if order.customer_tier == "premium":
            return subtotal * 0.2
        return subtotal * 0.05

# 之后 — 充血的 Order
class Order:
    def __init__(self, items, customer_tier):
        self.items = items
        self.customer_tier = customer_tier

    def discount(self) -> float:
        subtotal = sum(i.price * i.qty for i in self.items)
        return subtotal * (0.2 if self.customer_tier == "premium" else 0.05)

class OrderService:
    def calculate_discount(self, order: Order) -> float:
        return order.discount()  # 步骤 A：委托；步骤 C：删除此方法
```

**注意事项：**
- 不要把需要仓储、外部 API 或副作用的逻辑移入领域对象 —— 那些属于应用层的关注点
- 如果规则横跨两个领域对象（跨聚合），它属于领域服务，而不属于其中任一对象
- 检查调用该 service 方法的所有位置 —— 所有调用方都需要在步骤 B 中更新

---

## 引入仓储（Introduce Repository）

**意图：** 把某个领域实体的所有数据访问集中到单个 Repository 之后，消除散落在各层的查询/变更代码。

**适用场景：**
- SQL 查询、ORM 调用或 HTTP 客户端调用同时出现在 controller、service 和/或 presenter 中
- 给定实体的数据访问没有单一归属者
- 切换数据源（例如缓存、测试数据库）需要改动许多文件
- 检测到的坏味道：数据访问散落（Scattered Data Access）

**前置条件：**
- [ ] 已用 Grep 摸清该实体现有的所有数据访问点
- [ ] 已设计目标接口（仓储需要哪些操作？）
- [ ] 已确认测试替身策略（内存 fake、mock 或测试数据库）

**手法：**
1. 创建 `[Entity]Repository` 接口，为所有需要的操作给出方法签名
2. 创建具体实现（`[Entity]RepositoryImpl`），包装现有的数据访问
3. **步骤 A：** 把仓储注入第一个消费方（controller/service）。消费方调用仓储。仓储委托给原始的数据访问代码。运行测试。
4. **步骤 B：** 把实际的数据访问代码从原位置移入 `RepositoryImpl`。运行测试。
5. **步骤 C：** 从原位置删除数据访问代码。运行测试。
6. 对每个额外的消费方重复 A–C。

**示例：**
```go
// 之前 — 数据访问写在 handler 里
func GetUserHandler(db *sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        id := r.PathValue("id")
        var user User
        db.QueryRow("SELECT id, name, email FROM users WHERE id = ?", id).
            Scan(&user.ID, &user.Name, &user.Email)
        json.NewEncoder(w).Encode(user)
    }
}

// 之后
type UserRepository interface {
    FindByID(ctx context.Context, id string) (User, error)
}

type SQLUserRepository struct{ db *sql.DB }

func (r *SQLUserRepository) FindByID(ctx context.Context, id string) (User, error) {
    var user User
    err := r.db.QueryRowContext(ctx, "SELECT id, name, email FROM users WHERE id = ?", id).
        Scan(&user.ID, &user.Name, &user.Email)
    return user, err
}

func GetUserHandler(repo UserRepository) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        user, err := repo.FindByID(r.Context(), r.PathValue("id"))
        if err != nil { http.Error(w, err.Error(), 500); return }
        json.NewEncoder(w).Encode(user)
    }
}
```

**注意事项：**
- Repository 接口属于领域/应用层，而不是数据层 —— 数据层实现它，而不是拥有它（依赖倒置原则）
- 不要把事务管理放进 Repository —— 事务是应用层的关注点（工作单元）
- 除非该模式已经确立，否则避免使用泛型仓储（`Repository[T]`）—— 它们会把数据层概念（查询对象、规约）泄漏到领域层

---

## 提取用例 / 交互器（Extract Use Case / Interactor）

**意图：** 把在多个 controller 或 presenter 中重复的横切业务流提取为单个、可独立测试的 Use Case 类。

**适用场景：**
- 同一个业务流（例如"下单"、"注册用户"）出现在多个 controller 中
- 一个 controller/presenter 为完成单个动作而顺序调用 4 个以上的 service
- 不 mock 整个 Web/API 框架就无法对该流程做单元测试
- 模式目标：整洁架构（Clean Architecture）、六边形架构（Hexagonal Architecture）

**前置条件：**
- [ ] 已通过 Grep 找出重复流程的所有调用点
- [ ] 该流程的输入与输出已清晰定义
- [ ] 该流程没有直接的 UI 依赖（如果有，先提取它）

**手法：**
1. 定义 Use Case：一个类，一个公开的 `execute(input)` 方法
2. 识别该流程需要的所有依赖（仓储、service、外部网关）
3. **步骤 A：** 创建 `[Action]UseCase` 类。第一个 controller 委托给它。运行测试。
4. **步骤 B：** 用 mock 的依赖为该 Use Case 编写单元测试。确认测试通过。
5. **步骤 C：** 从第一个 controller 中删除重复逻辑。运行测试。
6. 对每个仍重复该流程的 controller 重复步骤 C。

**示例：**
```csharp
// 之前 — PlaceOrder 逻辑在 WebController 与 MobileController 中重复

// 之后
public class PlaceOrderUseCase
{
    private readonly IOrderRepository _orders;
    private readonly IInventoryService _inventory;
    private readonly IPaymentGateway _payments;

    public PlaceOrderUseCase(IOrderRepository orders,
        IInventoryService inventory, IPaymentGateway payments)
    {
        _orders = orders; _inventory = inventory; _payments = payments;
    }

    public async Task<OrderResult> Execute(PlaceOrderInput input)
    {
        await _inventory.Reserve(input.Items);
        var order = Order.Create(input.CustomerId, input.Items);
        var payment = await _payments.Charge(order.Total, input.PaymentToken);
        order.ConfirmPayment(payment.TransactionId);
        await _orders.Save(order);
        return new OrderResult(order.Id, order.Total);
    }
}

// WebController 与 MobileController 现在都注入并调用 PlaceOrderUseCase
```

**注意事项：**
- Use Case 的输入/输出类型（DTO）不得是框架类型（例如 `HttpRequest`、`Intent`）—— 让它们保持为普通数据对象
- 一个 Use Case = 一个业务动作。不要创建用标志参数处理多个流程的"上帝 Use Case"
- 如果 Use Case 超过约 50 行，说明它做得太多 —— 分解它

---

## 修复分层违规（Fix Layer Violation）

**意图：** 移除从外层到超出允许范围的内层的直接依赖，通过引入正确的抽象恢复清晰的分层。

**适用场景：**
- 展示层直接 import 数据/仓储包
- Controller 直接 import 具体的 ORM 实体或数据库模型类型
- 领域对象 import 了 service 或 repository（依赖方向倒置）
- 检测到的坏味道：分层违规（Layer Violation）

**前置条件：**
- [ ] 已找出违规文件中所有跨层 import（对 import 语句使用 Grep）
- [ ] 已与用户确认预期的分层边界

**手法：**
1. 识别被跨边界 import 的类型
2. 创建一个领域层级的接口或 DTO，让外层改为使用它
3. **步骤 A：** 更新外层以使用新类型。内层负责与领域类型之间的转换。运行测试。
4. **步骤 B：** 从外层移除跨边界的 import。运行测试。
5. 如果多个文件存在同样的违规，逐个文件重复。

**注意事项：**
- 红线：如果跨边界的类型是序列化格式（映射到数据库列的 ORM 实体），改动它可能需要数据迁移 —— 停下来与用户确认
- 层间映射会带来样板代码 —— 这是有意为之；替代方案是紧密耦合

---

## 分离读模型与写模型（Separate Read Model from Write Model）

**意图：** 把同时服务复杂查询与变更的单一模型拆分为聚焦的写模型（命令侧）与轻量的读模型（查询侧）。

**适用场景：**
- 领域实体有超过 10 个字段，但大多数查询只用到 2–3 个
- 构建单个实体需要复杂的 join
- 因为写模型过于笨重而难以查询，导致读取性能很差
- 模式目标：CQRS（模型层面的命令/查询分离）

**不适用场景：**
- 没有复杂查询需求的简单 CRUD —— 这是过度设计
- 团队不熟悉 CQRS —— 先从提取用例或下推业务逻辑开始

**手法：**
1. 找出正在受损的查询用例（具体是哪些查询？）
2. 为每种查询模式创建只读的 `[Entity]Summary` / `[Entity]View` DTO
3. 创建直接返回该 DTO 的查询处理器 / 仓储方法（不构造领域对象）
4. **步骤 A：** 添加新的读取路径。既有读取路径仍可工作。运行测试。
5. **步骤 B：** 把调用方迁移到新的读取路径。每迁移一个调用方后运行测试。
6. **步骤 C：** 如果所有调用方都已迁移，移除旧的查询路径。运行测试。

**注意事项：**
- 如果写入侧发生变化，读模型可能变陈旧 —— 请小心维护读模型投影
- 不要向读模型添加写入逻辑（校验、状态迁移）—— 它们必须是只读的
- 此操作无需事件溯源即可完成 —— 模型层面的纯 CQRS 要简单得多
