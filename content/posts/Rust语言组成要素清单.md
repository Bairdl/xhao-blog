---
date: '2025-10-28T00:04:11+08:00'
draft: false
title: 'Rust语言组成要素清单'
tags: []
categories: ["语言基础"]
---

## Rust 语言组成要素清单

### 1. **词法与语法基础**
- 标识符：由 Unicode 字母、数字、下划线组成，不能以数字开头，支持原始标识符（`r#keyword`）
- 关键字（约 50+ 个）：
  - 严格关键字：`as`, `break`, `const`, `continue`, `crate`, `else`, `enum`, `extern`, `false`, `fn`, `for`, `if`, `impl`, `in`, `let`, `loop`, `match`, `mod`, `move`, `mut`, `pub`, `ref`, `return`, `self`, `Self`, `static`, `struct`, `super`, `trait`, `true`, `type`, `unsafe`, `use`, `where`, `while`
  - 保留关键字（未来可能使用）：`abstract`, `become`, `box`, `do`, `final`, `macro`, `override`, `priv`, `typeof`, `unsized`, `virtual`, `yield`
- 字面量：
  - 整数（十进制、十六进制 `0xff`、八进制 `0o755`、二进制 `0b1010`）
  - 浮点数（`3.14`, `2.5e10`）
  - 布尔：`true`, `false`
  - 字符：`'a'`, `'\n'`, `'\u{1F600}'`
  - 字符串：`"hello"`（UTF-8），支持转义和原始字符串 `r#"..."#`
  - 字节字面量：`b'A'`, `b"hello"`（`[u8; N]` 或 `&[u8]`）
- 运算符：算术、比较、逻辑、位运算、解引用 `*`、取地址 `&`、所有权转移 `move`（隐式）、模式匹配 `@`、范围 `..`, `..=`
- 宏调用语法：以 `!` 结尾，如 `println!`, `vec!`

---

### 2. **基本类型系统**
- 标量类型：
  - 整数：`i8`, `i16`, `i32`, `i64`, `i128`, `isize`；`u8` 到 `u128`, `usize`
  - 浮点数：`f32`, `f64`
  - 布尔：`bool`
  - 字符：`char`（Unicode 标量值，4 字节）
- 复合类型：
  - 元组（tuple）：`(i32, bool)`，固定长度，异构
  - 数组：`[i32; 5]`，固定长度，栈上分配
- 单元类型：`()`，无值，常用于无返回函数
- Never 类型：`!`，表示永不返回（如 `panic!`, `loop {}`）

---

### 3. **指针与引用类型**
- 引用（reference）：
  - 不可变引用：`&T`
  - 可变引用：`&mut T`
- 裸指针（raw pointer）：
  - `*const T`（只读）
  - `*mut T`（可变）
  - 仅在 `unsafe` 块中解引用
- 智能指针（标准库提供）：
  - `Box<T>`：堆分配，独占所有权
  - `Rc<T>`：引用计数，单线程共享所有权
  - `Arc<T>`：原子引用计数，多线程共享所有权
  - `Cell<T>`, `RefCell<T>`：内部可变性（运行时借用检查）
  - `Mutex<T>`, `RwLock<T>`：线程安全的内部可变性

---

### 4. **所有权与借用系统**
- 所有权规则：
  - 每个值有且仅有一个所有者
  - 值离开作用域时自动调用 `drop`
  - 所有权可转移（move），不可隐式复制（除非实现 `Copy`）
- 借用规则（编译期检查）：
  - 任意时刻，要么一个可变引用，要么任意数量不可变引用
  - 引用必须始终有效（不能悬垂）
- 生命周期（lifetime）：
  - 用 `'a` 表示，标注引用的有效范围
  - 函数签名中可显式声明生命周期参数
  - 编译器支持生命周期省略规则（lifetime elision）

---

### 5. **用户自定义类型**
- 结构体（`struct`）：
  - 元组结构体：`struct Point(i32, i32)`
  - 普通结构体：`struct User { name: String, age: u32 }`
  - 单元结构体：`struct Unit;`
- 枚举（`enum`）：
  - 可携带数据：`enum Option<T> { Some(T), None }`
  - 支持多种变体（variant），每个可含不同字段
- 联合体（`union`）：仅在 `unsafe` 中使用，手动管理内存布局
- 类型别名：`type Name = String;`
- 新类型模式：通过单字段元组结构体创建新类型（如 `struct Wrapper(i32);`）

---

### 6. **泛型系统**
- 泛型参数：`fn foo<T>(x: T) -> T`
- 泛型结构体/枚举：`struct Vec<T> { ... }`
- trait 约束：`fn foo<T: Display>(x: T)`
- where 子句：`fn foo<T>(x: T) where T: Debug + Clone`
- 关联类型（associated types）：在 trait 中定义占位类型
- 默认泛型参数：`struct Vec<T = i32>`
- 高阶 trait 约束（HRTB）：`for<'a> Fn(&'a T)`

---

### 7. **Trait 系统**
- trait 定义：`trait Display { fn fmt(&self, f: &mut Formatter) -> Result; }`
- trait 实现：`impl Display for MyType { ... }`
- 自动派生（`#[derive]`）：`Clone`, `Debug`, `PartialEq`, `Eq`, `PartialOrd`, `Ord`, `Hash`, `Default`, `Copy` 等
- trait 对象（动态分发）：`Box<dyn Trait>`, `&dyn Trait`
- blanket implementation（泛实现）：如 `impl<T> ToString for T where T: Display`
- 内建 trait（marker traits）：
  - `Sized`：编译期已知大小
  - `Copy`：可按位复制
  - `Send`：可安全跨线程传递
  - `Sync`：可安全跨线程共享
  - `Unpin`, `Unsize`, `Drop` 等

---

### 8. **模式匹配**
- `match` 表达式：穷尽性检查，支持守卫（`if` 条件）
- 解构语法：
  - 元组：`let (x, y) = point;`
  - 结构体：`let User { name, age } = user;`
  - 枚举：`match opt { Some(x) => ..., None => ... }`
- `if let` / `while let`：简化单分支匹配
- 范围匹配：`1..=10`
- 绑定：`@` 模式（如 `x @ 0..=10`）

---

### 9. **函数与闭包**
- 函数定义：`fn name(params) -> ReturnType { ... }`
- 函数指针：`fn(i32) -> bool`
- 闭包（closure）：
  - 三种 trait：`Fn`, `FnMut`, `FnOnce`
  - 自动捕获环境（by reference, by mutable reference, or by move）
  - 类型匿名，只能通过泛型或 trait 对象使用
- 高阶函数：函数可接收/返回闭包或函数
- `main` 函数：程序入口，可返回 `()` 或 `Result<(), E>`

---

### 10. **控制流**
- 条件：`if`, `if let`
- 循环：`loop`, `while`, `while let`, `for ... in`
- `break` / `continue`：支持带标签（`'label: loop { break 'label; }`）
- 表达式导向：大多数结构是表达式（如 `if` 可返回值）
- `return`：显式返回，也可省略（最后一行表达式作为返回值）

---

### 11. **模块系统与可见性**
- 模块（`mod`）：代码组织单元，可嵌套
- 文件结构：`mod.rs` 或同名 `.rs` 文件
- 路径：绝对路径（`crate::...`）、相对路径（`self::`, `super::`）
- 可见性修饰符：
  - `pub`：公开
  - `pub(crate)`：包内公开
  - `pub(super)`, `pub(in path)`：限定范围公开
- `use` 声明：引入路径，支持通配符 `*` 和重命名 `as`
- `extern crate`（旧版） / `use`（现代）引入外部 crate

---

### 12. **错误处理**
- `Result<T, E>`：表示可能失败的操作
- `Option<T>`：表示可能存在或不存在的值
- `?` 操作符：自动传播 `Err` 或 `None`
- `panic!`：运行时异常，触发栈展开（unwind）或中止（abort）
- `unwrap()`, `expect()`：用于原型或确定不会失败的场景
- 自定义错误类型：通常实现 `std::error::Error` trait

---

### 13. **并发与并行**
- 线程：`std::thread::spawn`
- 消息传递：`std::sync::mpsc`（多生产者单消费者通道）
- 共享状态：
  - `Arc<T>` + `Mutex<T>` / `RwLock<T>`
  - `atomic` 类型（`AtomicBool`, `AtomicUsize` 等）
- 无畏并发（fearless concurrency）：由类型系统保证数据竞争自由
- async/await（通过 `async-std` 或 `tokio` 等运行时）：
  - `async fn` 返回 `impl Future`
  - `await` 暂停执行
  - 异步运行时提供任务调度、I/O 多路复用

---

### 14. **宏系统**
- 声明宏（`macro_rules!`）：基于模式匹配的语法扩展
- 过程宏（procedural macros）：
  - 属性宏（`#[my_macro]`）
  - 函数式宏（`my_macro!()`，类似函数）
  - 派生宏（`#[derive(MyTrait)]`）
- 宏在编译期展开，不产生运行时开销

---

### 15. **标准库核心组件**
- `std` crate（标准库）包含：
  - `vec!`, `String`, `HashMap`, `HashSet`
  - `io`, `fs`, `env`, `process`
  - `net`（TCP/UDP）
  - `time`, `sync`, `thread`
  - `collections`, `boxed`, `rc`, `cell`
  - `fmt`, `error`, `result`, `option`
- `core` crate：无 std 环境（如嵌入式）可用的子集
- `alloc` crate：提供堆分配相关类型（`Vec`, `String` 等），用于 no_std + allocator 场景

---

### 16. **编译与工具链**
- 编译器：`rustc`
- 包管理器与构建工具：`cargo`
  - 依赖管理（`Cargo.toml`）
  - 构建、测试、运行、文档生成
- 版本渠道：`stable`, `beta`, `nightly`
- 特性门控（feature gates）：实验性功能需在 nightly 启用
- 交叉编译：通过 `--target` 指定目标平台
- 格式化：`rustfmt`
- 静态分析：`clippy`
- 文档生成：`cargo doc`
- 内存安全分析：由 borrow checker 在编译期完成，无 GC

---

### 17. **内存模型与运行时**
- 无垃圾回收（GC）
- 无运行时（zero-cost abstractions）：标准库外无强制运行时
- 栈与堆分配：`Box`, `Vec`, `String` 等使用堆
- 析构函数：`Drop` trait，RAII 风格资源管理
- 内存布局控制：`#[repr(C)]`, `#[repr(packed)]`, `#[repr(transparent)]`
- 未定义行为（UB）：仅在 `unsafe` 块中可能触发

---

### 18. **Unsafe Rust**
- `unsafe` 块：允许执行以下操作：
  - 解引用裸指针
  - 调用 unsafe 函数（如 C FFI）
  - 实现 unsafe trait（如 `Send`, `Sync`）
  - 访问或修改可变静态变量
  - 读写联合体字段
- Unsafe 不等于“不安全”，而是“编译器无法验证安全性，由程序员保证”

