---
date: '2025-10-28T00:03:10+08:00'
draft: false
title: 'Go语言组成要素清单'
tags: []
categories: ["语言基础"]
---

## Go 语言组成要素清单

### 1. **词法元素（Lexical Elements）**
- 标识符（identifier）：由字母、数字、下划线组成，首字符不能为数字
- 关键字（25 个）：`break`, `case`, `chan`, `const`, `continue`, `default`, `defer`, `else`, `fallthrough`, `for`, `func`, `go`, `goto`, `if`, `import`, `interface`, `map`, `package`, `range`, `return`, `select`, `struct`, `switch`, `type`, `var`
- 运算符与标点：`+`, `-`, `*`, `/`, `%`, `&`, `|`, `^`, `<<`, `>>`, `&&`, `||`, `!`, `==`, `!=`, `<`, `<=`, `>`, `>=`, `:=`, `...`, `;`（自动插入）
- 字面量：
  - 整数字面量（十进制、八进制、十六进制、二进制）
  - 浮点数字面量
  - 虚数字面量（如 `3i`）
  - 字符字面量（`'a'`, `'\n'`, `'\u1234'`）
  - 字符串字面量（解释型 ``"..."`` 和原始型 `` `...` ``）
  - 布尔字面量：`true`, `false`
  - 复合字面量：结构体、数组、切片、map 的字面量初始化语法

---

### 2. **基本类型系统**
- 布尔类型：`bool`
- 数值类型：
  - 整数：`int`, `int8`, `int16`, `int32`, `int64`, `uint`, `uint8`, `uint16`, `uint32`, `uint64`, `uintptr`
  - 浮点数：`float32`, `float64`
  - 复数：`complex64`, `complex128`
- 字符与字符串：
  - `byte`（`uint8` 的别名）
  - `rune`（`int32` 的别名，表示 Unicode 码点）
  - `string`（不可变 UTF-8 字节序列）
- 零值语义：每种类型有默认零值（如 `0`, `""`, `nil` 等）

---

### 3. **复合类型**
- 数组（array）：固定长度，`[N]T`
- 切片（slice）：动态长度，`[]T`，包含指针、长度、容量三元组
- 映射（map）：哈希表，`map[K]V`
- 结构体（struct）：字段集合，支持嵌入（匿名字段）
- 指针（pointer）：`*T`，指向变量的内存地址
- 函数类型（function type）：可作为值传递
- 接口（interface）：方法集的抽象，支持空接口 `interface{}`（Go 1.18+ 推荐用 `any`）
- 通道（channel）：`chan T`，用于 goroutine 通信，分带缓冲与无缓冲

---

### 4. **类型系统特性**
- 静态类型：编译期类型检查
- 类型别名：`type NewName = OldType`
- 类型定义：`type MyInt int`（创建新类型，不继承方法）
- 底层类型（underlying type）与类型转换规则
- 方法集（method set）：决定类型实现哪些接口
- 接口的隐式实现：无需显式声明 `implements`
- 类型断言（type assertion）：`x.(T)`
- 类型开关（type switch）：`switch x.(type)`

---

### 5. **函数与方法**
- 函数声明语法：`func name(params) (results) { ... }`
- 多返回值
- 命名返回值（named return values）
- 空标识符 `_` 用于丢弃返回值
- 方法：绑定到类型（非指针或指针）的函数
- 函数值（first-class function）：可赋值、传参、返回
- 闭包（closure）：函数捕获外部变量
- `defer` 语句：延迟执行，遵循 LIFO 顺序

---

### 6. **控制流结构**
- `if` / `else`：条件分支
- `for`：唯一循环语句，支持三种形式：
  - `for {}`（无限循环）
  - `for init; cond; post {}`
  - `for key, val := range iterable {}`
- `switch`：多路分支，支持表达式和类型
- `select`：用于 channel 操作的多路复用
- `goto`：带标签跳转（极少使用）
- `break` / `continue`：可带标签用于跳出多层循环

---

### 7. **并发模型**
- Goroutine：轻量级并发执行单元，由 `go` 关键字启动
- Channel：`chan T`，用于 goroutine 间通信与同步
  - 无缓冲 channel（同步）
  - 有缓冲 channel（异步）
  - 单向 channel：`<-chan T`（只读），`chan<- T`（只写）
- `select` 语句：监听多个 channel 操作
- 内存模型：happens-before 保证
- `sync` 包提供的原语：
  - `Mutex`, `RWMutex`
  - `WaitGroup`
  - `Once`
  - `Cond`
  - `Pool`
- `context` 包：用于传递取消信号、超时、截止时间、请求范围的值

---

### 8. **错误处理机制**
- 错误为值：`error` 是内置接口类型
- 显式错误检查：通过返回值传递错误
- 错误包装（Go 1.13+）：`fmt.Errorf("... %w", err)`
- 错误解包：`errors.Is(err, target)`, `errors.As(err, &target)`
- `panic` / `recover`：运行时异常机制（非错误处理首选）

---

### 9. **包与模块系统**
- 包（package）：代码组织单位，每个文件属于一个包
- 可见性规则：首字母大写为导出（public），小写为包内私有
- 模块（module）：Go 1.11+ 引入的依赖管理单位
  - `go.mod` 文件定义模块路径与依赖
  - 语义化版本控制（SemVer）
  - 伪版本（pseudo-version）用于未打标签的提交
- 导入路径：基于 URL 的包路径（如 `github.com/user/repo`）

---

### 10. **标准库核心组件**
- `builtin`：内置函数（`len`, `cap`, `make`, `new`, `close`, `panic`, `recover` 等）
- `runtime`：运行时控制（Goroutine、GC、内存等）
- `unsafe`：绕过类型安全（含 `Pointer`, `Sizeof`, `Alignof`, `Offsetof`）
- 常用包：
  - `fmt`, `strconv`, `strings`, `bytes`
  - `io`, `os`, `path/filepath`
  - `net/http`, `net`, `time`, `encoding/json`, `encoding/xml`
  - `reflect`：运行时类型反射
  - `testing`：单元测试与基准测试支持

---

### 11. **编译与工具链**
- 编译器：`gc`（官方默认），也可用 `gccgo`
- 构建命令：`go build`, `go run`, `go install`
- 交叉编译：通过 `GOOS` / `GOARCH` 环境变量
- 格式化：`gofmt`
- 依赖管理：`go mod` 系列命令
- 性能分析：`go tool pprof`, `go tool trace`
- 文档生成：`go doc`, `godoc`

---

### 12. **泛型（Go 1.18+）**
- 类型参数（type parameters）：在函数或类型定义中使用 `[T any]`
- 类型约束（type constraints）：通过接口定义允许的类型集
- 泛型函数与泛型类型
- 预定义约束：`comparable`, `any`（即 `interface{}`）
