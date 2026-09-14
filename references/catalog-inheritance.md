# 目录：继承与层次结构重构

用于处理类层次结构的操作 —— 把共享行为上移、把特化行为下移，以及在继承已成为负担时用组合取代继承。

每条目的结构：**意图 → 手法 → 示例 → 反向手法 → 注意事项**

---

## 上移方法（Pull Up Method）

**意图：** 把多个子类中完全相同（或近乎相同）的方法上移到超类。

**适用场景：**
- 两个或多个子类有做同样事情的方法
- 将该方法集中化可以消除重复

**手法：**
1. 确认这些方法确实完全相同（或者经过小幅重构后可以变得相同）
2. 如果它们引用了仅存在于子类中的字段，先上移字段
3. 在超类中创建该方法
4. 从子类中删除重复的方法
5. 运行测试

**注意事项：**
- 如果这些方法“几乎相同”，改用塑造模板方法（见下文）
- 检查超类能否访问被上移方法所需的所有字段/方法
- 如果该方法只在部分子类中才有意义，上移可能是错误的方向 —— 考虑提取超类

---

## 上移字段（Pull Up Field）

**意图：** 把出现在多个子类中的字段移入超类。

**适用场景：**
- 两个或多个子类有同名且用法相同的字段
- 消除重复，并使上移方法成为可能

**手法：**
1. 在超类中声明该字段
2. 从所有子类中移除该字段
3. 运行测试

---

## 下移方法（Push Down Method）

**意图：** 当超类中的某个方法只与特定子类相关时，把它从超类移到该子类。

**适用场景：**
- 超类中的某个方法只会在某一个特定子类上被调用
- 把它下移可减少基类中的噪声，并使意图更清晰

**手法：**
1. 把该方法复制到相关的子类
2. 从超类中移除该方法
3. 更新所有依赖超类类型进行的调用，改用子类类型
4. 运行测试

---

## 下移字段（Push Down Field）

**意图：** 把字段从超类移到实际使用它的子类。

**适用场景：**
- 超类中的某个字段只被一个子类使用
- 其他子类让它保持为 null/未使用

**手法：**
1. 在相关的子类中声明该字段
2. 从超类中移除该字段
3. 更新所有引用
4. 运行测试

---

## 提取超类（Extract Superclass）

**意图：** 创建一个新的超类，并把两个或多个相关类中的共享行为移入其中。

**适用场景：**
- 两个类做类似的事情，但没有共同的父类
- 你想在它们之间共享实现（而不只是接口）

**手法：**
1. 创建一个新的（抽象）超类
2. 让这两个类成为该新超类的子类
3. 使用上移字段和上移方法来移动共享行为
4. 运行测试

**注意事项：**
- 如果你只想共享接口（而不是实现），提取接口更合适
- 一个类只能有一个超类（在大多数语言中）—— 如果该类已经有超类，组合可能更好

---

## 提取接口 / 提取协议（Extract Interface / Extract Protocol）

**意图：** 定义一组类所实现的契约（接口或协议），从而无需继承即可实现多态使用。

**适用场景：**
- 多个类有签名相同但实现不同的方法
- 你想接受任何能响应某些消息的对象，而不要求有共同的祖先
- 你想为测试定义一个接缝（注入一个满足该接口的 mock）

**手法：**
1. 找出构成该契约的方法
2. 用这些方法签名创建一个接口/协议
3. 声明相关的类实现该接口
4. 把引用具体类的代码改为（在合适处）引用该接口
5. 运行测试

**示例：**
```
// BEFORE — two separate classes with no shared type
class Timesheet { getBillableWeeks(employee, start, end) { ... } }
class Employee { getRate() { ... } }

// AFTER — interface defines the contract used by billing
interface Billable {
  getBillableWeeks(start: Date, end: Date): number;
  getRate(): number;
}
```

**注意事项：**
- 不要为只有一个实现者的接口做提取 —— 它只增加噪声而没有收益（夸夸其谈通用性）
- 在 Go 中，接口是隐式的 —— 你无需声明某个类型实现了它们；直接使用该接口类型即可

---

## 折叠继承体系（Collapse Hierarchy）

**意图：** 合并相似到不值得维护这种区分程度的超类与子类。

**适用场景：**
- 某个子类几乎没有给其超类增加任何东西
- 只有一个子类，而且不太可能出现其他子类
- 该超类是投机性地创建出来的，从未被充分证明其必要性

**手法：**
1. 选择由哪个类吸收另一个（通常由超类吸收子类）
2. 使用上移方法/上移字段把子类中的所有内容移到超类
3. 把所有对子类的引用改为使用超类
4. 删除现在已为空的子类
5. 运行测试

**反向手法：** 提取超类（当出现更多子类时）

---

## 以委托取代子类（Replace Subclass with Delegate）

**意图：** 把继承关系替换为组合关系（has-a 而不是 is-a）。

**适用场景：**
- 子类关系不再严格成立（“is-a”值得怀疑）
- 你需要在运行时改变对象的“类型”（用继承无法做到）
- 该类还需要另一个超类
- 该子类只用于改变行为的某一个方面

**手法：**
1. 创建一个委托类（原来的子类变成一个独立的类）
2. 在原类中添加一个指向该委托的字段
3. 把子类特有的方法移到该委托中
4. 更新原类，让它在这些行为上调用该委托
5. 删除该子类
6. 运行测试

**示例：**
```
// BEFORE — PremiumBooking extends Booking
class PremiumBooking extends Booking {
  hasTalkback() { return this.show.hasOwnProperty("talkback"); }
}

// AFTER — PremiumBookingDelegate handles the premium behavior
class PremiumBookingDelegate {
  hasTalkback(show) { return show.hasOwnProperty("talkback"); }
}
class Booking {
  constructor(..., isPremium) {
    if (isPremium) this.premiumDelegate = new PremiumBookingDelegate();
  }
  hasTalkback() {
    return this.premiumDelegate
      ? this.premiumDelegate.hasTalkback(this.show)
      : false;
  }
}
```

**反向手法：** 以子类取代委托（如果类层次结构更简单）

**注意事项：**
- 这是一次较大的重构 —— 请增量进行，一次一个方法
- 移动之后，委托类可能会出现依恋情结 —— 它可能需要自己的、来自原类的字段

---

## 以委托取代超类（Replace Superclass with Delegate）

**意图：** 当子类并不真正表示 is-a 关系时，用字段（委托）取代对超类的继承。

**适用场景：**
- 某个类继承另一个类仅仅是为了复用其方法，而不是因为它真的是一个子类型
- 该子类的调用方不应能把它当作超类来使用
- 违反了里氏替换原则（Liskov Substitution Principle）

**经典反模式：**
```
// Stack extends List — but Stack should NOT be substitutable for List
class Stack extends List {
  push(item) { this.add(item); }
  pop() { return this.remove(this.lastElement()); }
}
// Problem: callers can call stack.remove(0) — which breaks stack semantics
```

**手法：**
1. 在子类中创建一个字段，持有超类的一个实例
2. 对于子类使用到的每个超类方法，添加一个委托方法
3. 移除 extends/inherits 声明
4. 运行测试

**注意事项：**
- 这可能会产生许多小的委托方法 —— 请考虑相比一开始就使用组合，这样做是否值得
- 如果超类接口确实有用，先提取接口，这样调用方仍然可以使用该接口类型

---

## 移除子类（Remove Subclass）

**意图：** 把只为改变某个常量值而存在的子类折叠为类型码或工厂。

**适用场景：**
- 某个子类的存在只是为了返回不同的常量值，或持有不同的数据
- 没有任何多态行为 —— 只是靠数据区分

**手法：**
1. 在超类中添加一个类型码字段
2. 把所有常量方法移入超类（根据类型码返回不同结果）
3. 把所有创建子类实例的代码改为创建带类型码的超类实例
4. 删除该子类
5. 运行测试

**示例：**
```
// BEFORE
class Male extends Person { genderCode() { return "M"; } }
class Female extends Person { genderCode() { return "F"; } }

// AFTER
class Person {
  constructor(genderCode) { this.genderCode = genderCode; }
}
const male = new Person("M");
const female = new Person("F");
```
