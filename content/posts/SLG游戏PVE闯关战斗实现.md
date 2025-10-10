---
date: '2025-10-10T22:30:22+08:00'
draft: false
title: 'SLG游戏PVE闯关战斗实现'
tags: []
categories: ['项目经验']
---

# SLG游戏PVE闯关战斗实现

由行为树和ECS系统两部分组成。

## 行为树

行为树是静态静态行为树，由 JSON 文件定义。

特殊行为树节点：
- input 节点：将输入数据转换为对应的 Control 数据（由 System 消费）。
- output 节点：通常是行为树的末端 Action 节点（如释放技能），作用为添加一个对应的 input 。

静态行为树有多个子树组成，通过组合节点和修饰节点进行控制。每个节点启动后都会注册到一个 behaviorManager 中。
节点的逻辑主要有三个部分：init，exit 以及额外的 timeout （通常在 init 时添加定时器，收到消息时检查）。

BevMgr 每帧处理到达的消息，每个节点都会有 handler 函数，处理消息时调用 handler 函数。

## ECS 

index：提供存储各种实体的数据结构。

manager：操作实体的逻辑代码，主要为创建、删除函数，以及其他业务逻辑。

component：为实体提供数据，以及每个数据字段的 get 和 set 函数。

system：每帧处理数据和输入操作。
- system_effect: 将 ControlEffect 转换为 InputEffect（由 cast_effect 行为树节点处理，会被转换成 ControlSettle 输入）。 
- system_settle: 处理 ControlSettle 输入，计算技能效果，统计每个英雄受到的伤害，并添加 EventEffect 数据（如果技能有附加 Buff ，则添加 Buff 实体）。
- system_direction: 调整 Hero 和 Bullet 的方向。
- system_steer: 更新 Hero 和 Bullet 的碰撞和位置。
- system_buff: 清理过期 Buff 。
- system_clear: 清理 Effect 和 Settle，隐藏死亡的英雄单位。
- system_room: 判断战斗是否结束。

## 战斗流程

创建房间：使用参数（英雄单位列表，双方玩家ID等数据）创建房间，设置玩家信息，设置双方英雄单位信息，为每个英雄创建技能行为树。

战斗开始：推进帧。每帧会将内部时钟走一步，处理超时行为，BevMgr处理超时消息或者其他消息，处理系统更新（System 批量处理消息和数据）。

每帧处理：
1. 推进内部时钟。
2. 处理定时Manager，对超时的行为发送超时消息。
3. BevMgr 统一处理所有触发的行为消息，如超时消息 或者 Input 消息。这些消息可能启动新的行为树，创建新的实体，也可能转换为 Control 消息。
4. System 处理数据和消息。
5. 收集所有组件中更新的数据字段，转换为更新消息。
6. 发送更新消息给客户端。

