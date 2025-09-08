+++
title = 'Mermaid序列图语法'
date = '2025-09-09T00:26:24+08:00'
author =  "XHao"  # 文章作者
# weight = 1  # 文章权重（数值越大排序越靠前）
# aliases = ["/draft-example"]  # 文章别名（可用于重定向）
tags = ["language"]  # 文章标签（数组形式）
# categories = [""] # 文章分类
description = "Mermaid 序列图语法介绍。"  # 文章描述（用于摘要显示）
draft = false  # 是否为草稿（true表示草稿，不会构建发布）
# mermaid = true # 无需声明，所有文档默认启用 mermaid 支持。
+++

# Mermaid 序列图语法

## 摘要
序列图（Sequence Diagram）是统一建模语言（UML）中一种重要的交互图，它按时间顺序描述了对象之间传递消息的过程，直观地展现了多个参与者为实现某个目标而进行的交互序列。本文档将详细介绍如何使用 Mermaid 语法绘制清晰、规范的序列图。

## 1. 基本结构

一个最基本的 Mermaid 序列图由 `sequenceDiagram` 关键字声明，后跟一系列参与者定义和消息语句。

````markdown
```mermaid
sequenceDiagram
    participant A as 客户端
    participant B as 服务器

    A->>B: 同步请求
    B-->>A: 异步响应
```
````

此代码将生成如下图表，展示了客户端与服务器之间一次简单的请求-响应交互：

```mermaid
sequenceDiagram
    participant A as 客户端
    participant B as 服务器

    A->>B: 同步请求
    B-->>A: 异步响应
```

## 2. 参与者（Participants）

参与者代表交互中涉及的对象、组件或系统。

### 2.1 定义参与者
使用 `participant` 关键字定义，并可选择使用 `as` 设置别名。
```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    participant DB[Database]
```
- **`participant S as Server`**: 定义一个参与者 `S`，在图中显示别名为 "Server"。
- **`participant DB[Database]`**: 另一种语法，效果同上，`[ ]` 内为显示名称。

### 2.2 参与者类型
Mermaid 支持多种参与者类型，通过不同关键字生成不同图标（取决于渲染器支持）：
- `actor`: 角色（人形图标）
- `participant`: 普通参与者（默认矩形）
- `database`: 数据库（圆柱形图标）

## 3. 消息（Messages）

消息是参与者之间传递的通信，由带箭头的线表示。

### 3.1 消息类型
| 语法 | 箭头样式 | 描述 | 示例 |
| :--- | :--- | :--- | :--- |
| `->>` | 实线箭头 → | **同步消息**（通常表示函数调用） | `A->>B: Login()` |
| `-->>` | 虚线箭头 --> | **异步消息**（通常表示返回或响应） | `B-->>A: Success` |
| `->` | 实线 —> | 同步消息（无箭头） | `A->B: Message` |
| `-->` | 虚线 --> | 异步消息（无箭头） | `B-->A: Callback` |
| `-x` | 实线末端打叉 —→✕ | **失败/终止消息** | `B-xA: Error` |
| `--x` | 虚线末端打叉 -->✕ | 异步失败消息 | `A--xB: Timeout` |

```mermaid
sequenceDiagram
    participant A
    participant B

    A->>B: 同步调用
    B-->>A: 异步响应
    A-xB: 抛出异常
```

## 4. 激活条（Activation）

激活条（生命线上的矩形框）表示参与者正在执行操作或处理任务的时间段。

- **`activate [参与者]`**: 激活指定参与者的生命线。
- **`deactivate [参与者]`**: 取消激活指定参与者的生命线。

激活条清晰地标识了控制焦点的位置和操作的执行范围。

```mermaid
sequenceDiagram
    participant U as User
    participant S as Server

    U->>S: 提交订单
    activate S
    S->>S: 处理订单
    S-->>U: 订单确认
    deactivate S
```

## 5. 注释（Notes）

注释用于为图表添加说明文本，可以放在参与者相对位置或跨越多个参与者。

- `Note right of [参与者]: 注释文本`: 在参与者右侧添加注释。
- `Note left of [参与者]: 注释文本`: 在参与者左侧添加注释。
- `Note over [参与者1],[参与者2]: 注释文本`: 跨越多个参与者添加注释。

```mermaid
sequenceDiagram
    participant A
    participant B

    Note left of A: 这是客户端
    Note right of B: 这是服务器
    A->>B: 请求
    Note over A,B: 这是一个重要的交互过程
    B-->>A: 响应
```

## 6. 逻辑片段（Fragments）

逻辑片段用于表示循环、条件判断等复杂逻辑。

### 6.1 循环片段（Loop）
使用 `loop` 和 `end` 关键字表示一段重复执行的逻辑。
```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server

    loop 每秒查询
        C->>S: 获取状态
        S-->>C: 返回状态
    end
```

### 6.2 条件片段（Alternative）
使用 `alt`/`else` 和 `end` 关键字表示条件判断。
```mermaid
sequenceDiagram
    participant U as User
    participant S as System

    U->>S: 登录
    alt 凭证有效
        S-->>U: 登录成功
    else 凭证无效
        S-->>U: 登录失败
    end
```

### 6.3 可选片段（Optional）
使用 `opt` 和 `end` 关键字表示可选的执行路径。
```mermaid
sequenceDiagram
    participant U as User
    participant S as Service

    U->>S: 提交请求
    opt 数据校验
        S->>S: 验证数据
    end
    S-->>U: 处理结果
```

## 7. 其他实用功能

### 7.1 自动编号
使用 `autonumber` 关键字自动为消息添加序列号。
```mermaid
sequenceDiagram
    autonumber
    participant A
    participant B
    
    A->>B: 第一条消息
    B-->>A: 第二条消息
    A->>B: 第三条消息
```

### 7.2 设置标题
使用 `title: 标题文本` 为图表添加标题。
```mermaid
sequenceDiagram
    title: 用户登录认证流程
    participant U as User
    participant S as Auth Server
    
    U->>S: 发送凭据
    S-->>U: 返回Token
```

## 8. 完整示例

下面是一个综合多种元素的完整序列图示例：

````markdown
```mermaid
sequenceDiagram
    title: 电子商务订单处理流程
    autonumber
    actor C as Customer
    participant W as Web Server
    participant O as Order Service
    participant P as Payment Service
    participant D as Database

    C->>W: 提交订单
    activate W
    W->>O: 创建订单请求
    activate O
    
    alt 库存充足
        O->>D: 锁定库存
        activate D
        D-->>O: 锁定成功
        deactivate D
        
        O->>P: 支付请求
        activate P
        P-->>O: 支付成功
        deactivate P
        
        O->>D: 更新订单状态
        activate D
        D-->>O: 更新成功
        deactivate D
        
        O-->>W: 订单创建成功
    else 库存不足
        O-->>W: 库存不足错误
    end
    
    deactivate O
    W-->>C: 显示结果
    deactivate W
```
````

```mermaid
sequenceDiagram
    title: 电子商务订单处理流程
    autonumber
    actor C as Customer
    participant W as Web Server
    participant O as Order Service
    participant P as Payment Service
    participant D as Database

    C->>W: 提交订单
    activate W
    W->>O: 创建订单请求
    activate O
    
    alt 库存充足
        O->>D: 锁定库存
        activate D
        D-->>O: 锁定成功
        deactivate D
        
        O->>P: 支付请求
        activate P
        P-->>O: 支付成功
        deactivate P
        
        O->>D: 更新订单状态
        activate D
        D-->>O: 更新成功
        deactivate D
        
        O-->>W: 订单创建成功
    else 库存不足
        O-->>W: 库存不足错误
    end
    
    deactivate O
    W-->>C: 显示结果
    deactivate W
```

## 结论

Mermaid 序列图提供了一种基于文本的强大工具，用于可视化对象之间的交互序列。通过简洁的语法，开发者可以创建出表达清晰、结构规范的序列图，有效支持系统设计、架构分析和文档编写工作。其与 Markdown 的良好集成特性，使其成为技术文档编写的理想选择。
