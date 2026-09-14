# 语言档案（Language Profiles）

用于快速查阅各语言特有的测试命令、惯用法、常见坏味道和格式化工具。重构时，先查本文件，以产出符合语言习惯的代码。

如果项目所用语言未列出，请询问用户：**"你们用什么命令运行测试？"**——然后继续。

---

## TypeScript / JavaScript

| | |
|---|---|
| **测试命令** | `npm test` · `npx jest` · `npx vitest run` · `npx mocha` |
| **构建/类型检查** | `npx tsc --noEmit` · `npx tsc --build` |
| **格式化工具** | `npx prettier --write .` · `npx eslint --fix .` |
| **项目识别** | 存在 `package.json` |

**惯用风格：**
- 默认用 `const`；只有需要重新赋值时才用 `let`；绝不用 `var`
- 回调与简短表达式用箭头函数；顶层具名函数用 `function` 声明
- 用解构来拆包对象/数组：`const { id, name } = user`
- 优先命名导出而非默认导出（更利于工具链和重构）
- 优先模板字符串而非字符串拼接
- 优先可选链 `?.` 与空值合并 `??`，而非显式 null 检查
- 优先 `async/await`，而非裸 Promise 或回调
- 类型：优先 `unknown` 而非 `any`；用类型守卫做收窄

**JS/TS 特有的常见坏味道：**
- `any` 类型抹掉类型安全
- 缺失可选属性导致的隐式 `undefined`（补上 `?` 或默认值）
- `== null` 与 `=== null`——始终用 `===`
- 修改函数参数（尤其是数组/对象）
- 不 await Promise（静默的 fire-and-forget）
- `var` 提升问题掩盖作用域 bug

**模块约定：**
- 使用 ES modules（`import`/`export`）；CommonJS（`require`）已是遗留方式
- 用 barrel 文件（`index.ts`）暴露模块的公共 API；不要把所有东西都 barrel 导出

---

## Python

| | |
|---|---|
| **测试命令** | `pytest` · `python -m pytest` · `python -m unittest` |
| **构建/类型检查** | `mypy .` · `pyright` |
| **格式化工具** | `black .` · `ruff check --fix .` |
| **项目识别** | `pyproject.toml` · `setup.py` · `requirements.txt` |

**惯用风格：**
- 构建集合时优先用列表/字典/集合推导式，而非循环
- 数据容器优先用 `dataclasses.dataclass` 或 `typing.NamedTuple`，而非普通类
- 处处标注类型提示（`def fn(x: int) -> str`）
- 优先 f-string，而非 `.format()` 或 `%` 插值
- 优先 `pathlib.Path`，而非 `os.path`
- 用上下文管理器（`with`）管理资源
- 计算属性用 `@property`，而非 `getX()` 方法
- 不要用可变默认参数：`def fn(items=None): if items is None: items = []`
- 除类名（`PascalCase`）之外，一切都用 `snake_case`

**Python 特有的常见坏味道：**
- 可变默认参数：`def fn(x=[])`——这个列表会在所有调用之间共享
- 裸 `except:` 子句会捕获一切，包括 `KeyboardInterrupt`
- 缺少类型提示，使重构不安全
- 在 `dataclass` 更清晰的地方使用 `dict`
- `global` 关键字——几乎总是坏味道
- 冗长的 `isinstance` 检查链——改用 `match`（3.10+）或多态

---

## Go

| | |
|---|---|
| **测试命令** | `go test ./...` · `go test -v ./pkg/...` |
| **构建/类型检查** | `go build ./...` · `go vet ./...` |
| **格式化工具** | `gofmt -w .` · `golangci-lint run` |
| **项目识别** | 存在 `go.mod` |

**惯用风格：**
- 接口是隐式的——类型只要拥有这些方法就满足该接口；没有 `implements` 关键字
- 错误是值：始终返回 `(T, error)` 并检查 `if err != nil`
- 局部作用域用简短的小写变量名（`i`、`n`、`buf`）——这是惯用写法
- 接受接口，返回具体类型（由调用方决定，而不是被调用方）
- 组合优先用 `struct` 嵌入，而非深层继承（Go 没有继承）
- 表驱动测试：`for _, tc := range testCases { ... }`
- 做 I/O 的函数把 `context.Context` 作为第一个参数
- 错误包装：用 `fmt.Errorf("doing X: %w", err)` 保留堆栈信息

**Go 特有的常见坏味道：**
- 忽略错误：`result, _ := fn()`——几乎总是错的
- 包级全局变量（难以测试）
- 在有具体类型可用时使用 `interface{}` / `any`
- 不关闭 `io.ReadCloser` 资源
- Goroutine 泄漏（启动了 goroutine 却从未通知它停止）
- 大结构体按值传递而非按指针传递

**移动机制：**
- 在包之间移动函数会改变其导入路径——要更新所有 import
- 未导出的标识符（小写）无法在其包外使用——移动时可能需要导出

---

## Rust

| | |
|---|---|
| **测试命令** | `cargo test` · `cargo test -- --nocapture` |
| **构建/类型检查** | `cargo check` · `cargo build` |
| **格式化工具** | `cargo fmt` · `cargo clippy --fix` |
| **项目识别** | 存在 `Cargo.toml` |

**惯用风格：**
- 所有移动都必须遵守所有权——转移所有权与借用是不同的重构
- 错误传播优先用 `?` 运算符，而非 `match`
- 为灵活性优先使用 `impl Trait` 返回类型，而非具体类型
- 优先迭代器与组合子，而非手写循环（`iter().map().filter().collect()`）
- 数据类型使用 `derive(Debug, Clone, PartialEq)`
- `Option` 与 `Result`——库代码中绝不使用 `unwrap()`；至少用 `expect("reason")`
- match 的穷尽性由编译器强制——新增一个枚举变体会迫使所有地方都更新

**Rust 特有的常见坏味道：**
- 不必要的 `.clone()`——往往暗示所有权设计有问题
- 非测试代码中的 `.unwrap()`——改用 `?` 或显式错误处理
- 到处存 `Rc<RefCell<T>>`——往往暗示存在应当重新设计的可变共享状态
- 没有清晰理由的 `unsafe` 块

**移动机制：**
- 在模块之间移动代码会改变可见性规则（`pub`、`pub(crate)`）——相应更新
- trait 实现必须与 trait 或类型位于同一个 crate 中

---

## Java

| | |
|---|---|
| **测试命令** | `./gradlew test` · `mvn test` · `./mvnw test` |
| **构建/类型检查** | `./gradlew build` · `mvn compile` |
| **格式化工具** | Spotless（`./gradlew spotlessApply`）· google-java-format |
| **项目识别** | `pom.xml` · `build.gradle` · `build.gradle.kts` |

**惯用风格：**
- 多态优先用接口而非抽象类
- 优先 Stream 而非命令式循环：`list.stream().filter().map().collect()`
- 参数超过 3 个的构造函数使用建造者模式
- 用 `Optional<T>` 代替返回 `null`
- 不可变数据容器用 `record`（Java 16+）
- 局部变量类型推断用 `var`（Java 10+）
- 优先组合而非继承

**Java 特有的常见坏味道：**
- 滥用 null——改用 `Optional` 或空对象
- 上帝类（数千行——Java 文化历来容忍这种做法）
- 受检异常到处传播——有时包成非受检异常更好
- 逻辑中间出现 `instanceof` 检查——改用多态

---

## Kotlin

| | |
|---|---|
| **测试命令** | `./gradlew test` · `./gradlew :module:test` |
| **构建/类型检查** | `./gradlew build` · `./gradlew compileKotlin` |
| **格式化工具** | ktlint（`./gradlew ktlintFormat`） |
| **项目识别** | 带 Kotlin 插件的 `build.gradle.kts` · `.kt` 文件 |

**惯用风格：**
- 值对象用 `data class`（自动生成 `equals`、`hashCode`、`copy`）
- 穷尽式类型体系用密封类
- 用扩展函数给既有类型添加行为，而无需继承
- 空安全：用 `?.`（安全调用）、`?:`（Elvis），生产代码中绝不用 `!!`
- 穷尽匹配用 `when` 表达式（密封类 + `when` = 安全的类型检查）
- 默认用 `val`；只在需要可变时才用 `var`
- 作用域函数：`let`、`run`、`apply`、`also`、`with`——选择最贴合意图的那个

**Kotlin 特有的常见坏味道：**
- Java 味儿：沿用 Java 的 null 处理模式，而不是 Kotlin 的空安全
- `!!` 运算符（空断言）——几乎总是说明漏了 null 处理
- 返回类型是 `Unit`，却没有把副作用命名清楚

---

## C# / .NET

| | |
|---|---|
| **测试命令** | `dotnet test` · `dotnet test --filter TestName` |
| **构建/类型检查** | `dotnet build` |
| **格式化工具** | `dotnet format` |
| **项目识别** | `.csproj` · `.sln` 文件 |

**惯用风格：**
- 不可变数据用 `record` 类型（C# 9+）
- 用 `switch` 表达式做模式匹配（C# 8+）
- 集合变换用 LINQ：`.Where().Select().ToList()`
- 全程 `async`/`await`——避免使用可能死锁的 `.Result` 和 `.Wait()`
- 单表达式方法用表达式主体成员：`int Double(int x) => x * 2;`
- 启用可空引用类型——使用 `?` 标注和 null 条件运算符

**C# 特有的常见坏味道：**
- 在 `record` 就够用的地方使用可变类状态
- `.Result`/`.Wait()` 在异步上下文中造成死锁
- 用 `object` 类型代替泛型

---

## Swift

| | |
|---|---|
| **测试命令** | `swift test` · `xcodebuild test -scheme MyScheme` |
| **构建/类型检查** | `swift build` |
| **格式化工具** | `swiftformat .` · `swiftlint --fix` |
| **项目识别** | `Package.swift` · `.xcodeproj` |

**惯用风格：**
- 数据优先用值类型（`struct`、`enum`），而非引用类型（`class`）
- 面向协议编程——优先协议而非类继承
- 用 `guard` 做提前退出/前置条件检查
- 用带关联值的 `enum` 建模和类型
- 只有在崩溃确实是正确行为时才强制解包 `!`（库代码中绝不使用）
- 序列化用 `Codable`，而非自行解析

---

## Ruby

| | |
|---|---|
| **测试命令** | `bundle exec rspec` · `bundle exec rake test` · `ruby -Itest test/test_*.rb` |
| **构建/类型检查** | `bundle exec rubocop` |
| **格式化工具** | `bundle exec rubocop --autocorrect` |
| **项目识别** | `Gemfile` · `.ruby-version` |

**惯用风格：**
- 优先用块和 proc，而非回调和传递函数对象
- 优先 `Enumerable` 方法，而非手写循环
- 鸭子类型——检查方法是否存在，而不是检查类类型
- 实例变量用 `attr_reader`/`attr_writer`/`attr_accessor`
- `method_missing` 是坏味道，不是特性——使用显式委托
- 性能敏感处使用冻结的字符串字面量

---

## C / C++

| | |
|---|---|
| **测试命令** | `ctest` · `make test` · `./build/tests` |
| **构建/类型检查** | `cmake --build . && cmake --build . -- check` |
| **格式化工具** | `clang-format -i **/*.cpp **/*.h` |
| **项目识别** | `CMakeLists.txt` · `Makefile` · `.cpp`/`.c` 文件 |

**惯用风格（现代 C++）：**
- RAII：资源获取即初始化；析构函数释放资源
- 优先 `std::unique_ptr` / `std::shared_ptr`，而非裸的所有权指针
- 参数和成员函数默认加 `const`
- 使用基于范围的 for 循环
- 返回值必须被检查的函数加 `[[nodiscard]]`
- 保持头文件/源文件分离——声明放 `.h`，定义放 `.cpp`

**C/C++ 特有的常见坏味道：**
- 没有 RAII 包装的裸 `new`/`delete`
- 全局可变状态
- 长长的头文件里泄漏了实现——检查 `inline` 是否被滥用
- 本可以用 `std::vector` 或智能指针的手工内存管理

---

## SQL

| | |
|---|---|
| **测试命令** | （人工评审 + `EXPLAIN ANALYZE`） |
| **格式化工具** | `sqlfluff fix .` · pgFormatter |

**惯用风格：**
- 为可读性优先用 CTE（`WITH`），而非嵌套子查询
- 避免 `SELECT *`——显式列出列名
- 两者都可行时优先 `JOIN` 而非子查询
- 索引感知的重写：在索引列上过滤，避免在 WHERE 中对索引列使用函数
- 行排名模式优先用窗口函数（`ROW_NUMBER`、`RANK`、`LAG`），而非自连接
- 命名：列名和表名用 `snake_case`；SQL 关键字用 `UPPER_CASE`

**SQL 特有的常见坏味道：**
- `SELECT *`——对 schema 变更很脆弱
- WHERE/SELECT 中逐行执行的相关子查询
- 连接列或高基数过滤列上缺少索引
- N+1 查询模式（应用层循环，每行一次查询）
