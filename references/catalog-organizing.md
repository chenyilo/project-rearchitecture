# 目录：组织数据与搬移特性（Organizing Data & Moving Features）

用于把代码移动到它该在的地方的操作 —— 正确的类、正确的模块、正确的抽象层级。

每个条目：**意图 → 手法 → 示例 → 反向手法 → 注意事项**

---

## 移动方法 / 移动函数（Move Method / Move Function）

**意图：** 把方法移到它实际使用最多的类或模块中。

**适用场景：**
- 一个方法使用另一个类的数据或方法比使用自己的还多（依恋情结坏味道）
- 该方法在逻辑上属于另一个抽象
- 移动它能降低类之间的耦合

**手法：**
1. 用 Grep 找出该方法的所有调用方
2. 在目标类上创建该方法（复制函数体，调整对 `this`/`self` 的引用）
3. 调整原方法，让它委托到新位置（用于一步式迁移）
4. 更新所有调用点，改为调用新位置
5. 删除原方法（或委托垫片）
6. 运行测试

**示例：**
```
// 重构前 —— Account 有一个其实属于 AccountType 的方法
class Account {
  overdraftCharge() {
    if (this.type.isPremium) {
      const baseCharge = 10;
      if (this.daysOverdrawn <= 7) return baseCharge;
      return baseCharge + (this.daysOverdrawn - 7) * 0.85;
    }
    return this.daysOverdrawn * 1.75;
  }
}

// 重构后
class AccountType {
  overdraftCharge(daysOverdrawn) {
    if (this.isPremium) {
      const baseCharge = 10;
      if (daysOverdrawn <= 7) return baseCharge;
      return baseCharge + (daysOverdrawn - 7) * 0.85;
    }
    return daysOverdrawn * 1.75;
  }
}
class Account {
  overdraftCharge() { return this.type.overdraftCharge(this.daysOverdrawn); }
}
```

**注意事项：**
- 如果该方法引用了原类的许多字段，它可能并不属于别处 —— 请重新考虑
- 红线：如果该方法属于公共 API，参见 safety.md 第 3 节
- 移动前检查是否有子类覆写了该方法

---

## 移动字段（Move Field）

**意图：** 把字段移到使用它最多的类。

**适用场景：**
- 一个字段被另一个类使用的次数超过拥有它的类
- 在执行提取类时 —— 字段需要移到新类中

**手法：**
1. 如果尚未这样做，先用访问器封装该字段（封装变量）
2. 把字段添加到目标类
3. 运行测试，确认这次搬移目前还没有破坏任何东西
4. 更新源类中的访问器，让它委托到目标类
5. 更新该字段的所有使用者，让它们直接使用目标类（如果这样更利于封装）
6. 删除原字段
7. 运行测试

**注意事项：**
- 如果两个类对该字段的使用程度相同，它可能是数据泥团 —— 考虑改用引入参数对象
- 在并发代码中，移动字段会改变保护它的锁 —— 参见 safety.md 第 3 节

---

## 提取类（Extract Class）

**意图：** 把一个身兼两职的类拆成两个类，各自拥有清晰的职责。

**适用场景：**
- 一个类的方法和字段过多（过大的类坏味道）
- 类中的一部分字段与方法构成一个内聚的组
- 两项清晰的「职责」可以被分别命名

**手法：**
1. 给新类命名，并确定哪些字段/方法属于它
2. 创建新类
3. 把确定的字段移到新类（移动字段）
4. 把确定的方法移到新类（移动方法）
5. 决定两者的关系：原类是否持有对新类的引用？
6. 更新所有调用点
7. 运行测试

**示例：**
```
// 重构前 —— Person 同时承担个人信息与联系信息两种职责
class Person {
  name; officeAreaCode; officeNumber;
  telephoneNumber() { return `(${this.officeAreaCode}) ${this.officeNumber}`; }
}

// 重构后
class TelephoneNumber {
  areaCode; number;
  toString() { return `(${this.areaCode}) ${this.number}`; }
}
class Person {
  name;
  officeTelephone = new TelephoneNumber();
  telephoneNumber() { return this.officeTelephone.toString(); }
}
```

**反向手法：** 内联类（Inline Class）

**注意事项：**
- 这是一次规模较大的操作 —— 增量进行（一次移动一个字段，一次移动一个方法）
- 新类可能需要自己的测试
- 检查新类应当是值对象（不可变）还是实体

---

## 内联类（Inline Class）

**意图：** 把一个存在价值不足的类折叠进另一个类。

**适用场景：**
- 经过其它重构后，该类剩下的职责非常少
- 它基本只是一个没有真实行为的数据容器
- 它提供的抽象不足以抵偿额外一个类带来的认知负担

**手法：**
1. 在被吸收的类上声明该类待内联的公共接口
2. 更新对内联类方法的所有引用，改用被吸收的类
3. 把所有字段和方法移入被吸收的类
4. 删除现已为空的类
5. 运行测试

**反向手法：** 提取类（Extract Class）

**注意事项：**
- 常作为朝另一个方向提取类的前奏（「先弄糟，再弄好」）
- 不要内联一个在多处使用的类 —— 这强烈说明它有价值

---

## 隐藏委托关系（Hide Delegate）

**意图：** 在服务类上创建方法以隐藏对另一个对象的委托，从而消除消息链。

**适用场景：**
- 调用方需要穿透链条调用：`person.getDepartment().getManager()`
- 修改 `Department` 类会迫使各处的 `Person` 调用方都要改

**手法：**
1. 对于客户端经由服务类（Person）调用的委托对象（Department）的每个方法，在服务类上创建一个委托方法
2. 更新客户端，改为调用服务类的方法而不是穿透链条
3. 如果没有客户端再直接访问该委托对象，就移除该委托对象的访问器
4. 运行测试

**示例：**
```
// 重构前
manager = person.getDepartment().getManager();

// 重构后
class Person {
  getManager() { return this.department.getManager(); }
}
manager = person.getManager();
```

**反向手法：** 移除中间人（Remove Middle Man，当委托不再简化事情时）

**注意事项：**
- 不要过度应用 —— 如果客户端确实需要与 Department 对象打交道，隐藏它反而增加了偶然复杂度
- 如果你需要隐藏委托对象的许多方法，提取类可能是更好的做法

---

## 移除中间人（Remove Middle Man）

**意图：** 移除只做委托的类，直接暴露被委托对象。

**适用场景：**
- 一个类有过多没有价值的委托方法
- 委托已经增长到客户端直接调用被委托对象反而更简单的程度

**手法：**
1. 在服务类上暴露一个获取被委托对象的访问器
2. 对每个委托方法，更新客户端让它直接调用被委托对象
3. 删除这些委托方法
4. 运行测试

**反向手法：** 隐藏委托关系（Hide Delegate）

**注意事项：**
- 与隐藏委托关系互为反向 —— 选择对具体客户端更能降低耦合的那一个
- 检查移除委托是否会暴露一个本应保持私有的内部类型

---

## 以对象取代数据值 / 值对象（Replace Data Value with Object / Value Object）

**意图：** 用封装领域行为的正式对象取代基本类型或数据记录。

**适用场景：**
- 某个基本类型值承载领域含义（Money、PhoneNumber、Email、DateRange）
- 围绕该基本类型，相同的校验或格式化逻辑出现在多处
- 你将来想为这份数据添加行为

**手法：**
1. 创建一个新类，包含一个存放原基本类型的字段
2. 添加构造函数和 getter
3. 添加所需的校验、格式化或比较行为
4. 把所有对原始基本类型的使用替换为新类
5. 运行测试

**示例：**
```
// 重构前
class Order {
  customerName: string;  // "John Smith" —— 用于展示、排序、比较
}

// 重构后
class CustomerName {
  constructor(private readonly value: string) {
    if (!value.trim()) throw new Error("Name cannot be blank");
  }
  toString() { return this.value; }
  equals(other: CustomerName) { return this.value === other.value; }
}
class Order {
  customerName: CustomerName;
}
```

**注意事项：**
- 值对象应当是不可变的 —— 如果需要「修改」值，就创建新实例
- 不要把每个基本类型都做成值对象 —— 只做那些承载领域含义或行为重复的
- 检查序列化：值对象需要能正确序列化/反序列化（safety.md 第 3 节中的红线）

---

## 以对象取代数组/元组（Replace Array/Tuple with Object）

**意图：** 当位置具有语义含义时，用带具名字段的对象取代按位置取值的数组/元组。

**适用场景：**
- 代码按索引访问数组：`result[0]`、`result[1]`
- 用元组返回多个本身有名字的值

**手法：**
1. 创建一个对象/record/struct，为每个位置设置对应的具名字段
2. 把数组构造替换为对象构造
3. 把索引访问替换为字段访问
4. 运行测试

**示例：**
```
// 重构前
const result = [startDate, endDate, totalDays];
console.log(result[2]);

// 重构后
const result = { startDate, endDate, totalDays };
console.log(result.totalDays);
```

---

## 封装变量 / 封装字段（Encapsulate Variable / Encapsulate Field）

**意图：** 把公共字段私有化，并通过方法提供受控访问。

**适用场景：**
- 某个字段从其类外部被直接访问
- 你希望在访问时添加校验、通知或延迟加载
- 你需要拦截读取或写入，以实现监控或缓存

**手法：**
1. 为该字段创建 get 和 set 访问器
2. 找出对字段的所有引用，替换为访问器调用
3. 把字段设为 private
4. 运行测试

**注意事项：**
- 在支持属性语法（property syntax）的语言（Python、C#、Kotlin）中，这是语言特性 —— 直接用它
- 不要盲目给每个字段都加 getter/setter —— 只在需要访问控制时才加

---

## 重命名字段（Rename Field）

**意图：** 重命名字段，使其更贴合用途或领域的通用语言（ubiquitous language）。

**适用场景：**
- 字段名有误导性、被缩写，或与领域语言不一致
- 字段的含义在命名之后已经演变

**手法：**
1. 如果字段在类外部被使用，先封装它（封装变量）
2. 重命名字段
3. 更新访问器名称以保持一致
4. 更新所有调用点
5. 运行测试

**注意事项：**
- 如果字段名属于某种序列化格式（JSON、DB 列、proto），这是红线 —— 参见 safety.md 第 3 节
- 对广泛使用的字段，重命名前用 Grep 梳理所有使用位置
