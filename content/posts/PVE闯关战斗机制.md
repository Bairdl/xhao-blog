+++
date = '2025-09-25T22:50:32+08:00'
draft = false
title = 'PVE闯关战斗机制'
categories = ['项目经验']
+++

# SLG 游戏 PVE闯关战斗机制设计

## 项目当前设计  

### lib 部分

lib 部分主要为相关战斗部分代码提供数据结构以及函数支持。

包括以下部分：
1. behavior：行为。
2. behaviorManager：行为管理器。
3. timer：计时器，用来计算每帧的间隔。
4. syncer：同步各个客户端。


### server 部分

server 部分提供具体的战斗机制实现。

包含以下部分：
1. index：index部分包含多个index，实际上是每个实体对应的哈希存储实现，用于维护战斗过程中各个实体的存储。
2. entity：每个entity的实现，如bullet、hero、buff、effect、skill、dire、position。
3. system：实现各个系统的结算，如bullet、hero、buff、effect、skill、dire、position、clearance、room。
4. manager：提供各个实体的管理，如创建、更新、删除等，并负责触发各个实体的行为回调。
5. 整体通过行为树驱动。

### 当前设计分析

帧同步、确定性模拟、决策树、组件式状态存储、批量System处理、全量状态推送。

使用 tick 进行帧驱动，每帧按固定顺序结算系统。

每个单位有单独的行为配表，记录单位ID，行为，目标ID，效果等等。

对客户端推送时，每次只推送 Frame结果，记录创建的Component，删除的Componet，以及每个Component更新的字段。

服务端每帧推进：
1. 收集输入。
2. 执行决策。
3. 按顺序计算：BUff、EFFECT、BULLET、DIRECTION、POSITION、HERO、CLEARANCE、STATIC、ROOM。
4. 判断 ROOM 的状态（战斗是否结束）。
5. 收集状态变更。
6. 推送给客户端。


``` txt
+----------------------------------+
|       Battle Server (战斗服)      |
|                                  |
|  +---------------------------+   |
|  |     Frame Driver          |   | ← 主循环，每 100ms 推进一帧（10 FPS）
|  |     - 支持 1x / 2x 速度    |   |    2x = 每 50ms 一帧
|  +---------------------------+   |
|                                  |
|  +---------------------------+   |
|  |     Input Collector       |   | ← 收集玩家手动技能请求
|  |     - 来自客户端的 cast 指令 |   |
|  |     - 校验合法性（CD、目标） |   |
|  +---------------------------+   |
|                                  |
|  +---------------------------+   |
|  |     AI Engine             |   | ← 自动模式：行为树决定是否放技能
|  |     - 每帧为每个英雄决策    |   |
|  +---------------------------+   |
|                                  |
|  +---------------------------+   |
|  |     Systems Pipeline      |   | ← 每帧按序批量执行
|  |  1. PositionSystem        |   |    更新位置，处理碰撞
|  |  2. SkillSystem           |   |    批量减 CD，标记“可释放”
|  |  3. EffectSystem          |   |    执行技能效果（伤害、Buff）
|  |  4. BuffSystem            |   |    批量更新 Buff 时间
|  |  5. CombatSystem          |   |    计算伤害（攻击 - 防御）
|  |  6. ClearanceSystem       |   |    清理死亡单位、过期 Buff
|  +---------------------------+   |
|                                  |
|  +---------------------------+   |
|  |     Entity Manager        |   | ← 创建/更新/删除单位
|  |     - CreateHero()        |   |
|  |     - DestroyUnit()       |   |
|  +---------------------------+   |
|                                  |
|  +---------------------------+   |
|  |     Index (状态存储层)      |   | ← 所有状态的“组件式”存储
|  |     - heroIndex[101]       |   |    存储每个英雄的 HP、CD、位置等
|  |     - skillIndex[101]      |   |    存储技能 CD 状态
|  |     - buffIndex[101]       |   |    存储 Buff 持续时间
|  +---------------------------+   |
|                                  |
|  +---------------------------+   |
|  |     State Snapshot        |   | ← 每帧生成“完整状态快照”
|  |     - collectAllStates()   |   |    遍历 index，生成全量数据
|  +---------------------------+   |
|                                  |
|  +---------------------------+   |
|  |     Syncer → Client       |   | ← 直接推送“本帧完整状态”
|  |     - WebSocket / TCP     |   |
|  +---------------------------+   |
|                                  |
|  +---------------------------+   |
|  |     Determinism Guard     |   | ← 保证确定性
|  |     - 固定随机种子         |   |
|  |     - 有序遍历（slice）    |   |
|  +---------------------------+   |
+----------------------------------+
```

``` txt
/battle
  ├── driver/           # 帧驱动
  │   └── frame_driver.go
  │
  ├── system/           # 系统模块（批量处理）
  │   ├── position_system.go
  │   ├── skill_system.go
  │   ├── buff_system.go
  │   ├── combat_system.go
  │   └── clearance_system.go
  │
  ├── entity/           # 实体定义
  │   ├── hero.go
  │   ├── enemy.go
  │   ├── skill.go
  │   └── buff.go
  │
  ├── manager/          # 实体管理
  │   └── entity_manager.go
  │
  ├── index/            # 状态存储（核心！）
  │   ├── hero_index.go
  │   ├── skill_index.go
  │   ├── buff_index.go
  │   └── position_index.go
  │
  ├── ai/               # AI 决策
  │   └── behavior_tree.go  # 或简单规则引擎
  │
  ├── input/            # 手动输入处理
  │   └── input_validator.go
  │
  ├── sync/             # 同步模块
  │   ├── snapshot.go   # 生成全量状态
  │   └── syncer.go     # 推送状态给客户端
  │
  └── battle.go         # 主战斗类，串联所有模块
```