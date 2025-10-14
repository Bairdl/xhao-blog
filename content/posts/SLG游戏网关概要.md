---
date: '2025-10-06T00:56:07+08:00'
draft: false
title: 'SLG游戏网关设计'
categories: ['项目经验']
---

编程语言：Go 。

网关的主要功能为：
1. 接受来自客户端的连接，并管理连接的生命周期。
2. 接受来自客户端的请求，并路由到相应的后端服务器。
3. 接受来自后端服务器的响应，并转发给客户端。
4. 接受来自后端服务器的推送消息，并转发给客户端或者缓存起来。
5. 认证成功后，将Player与会话关联起来。
6. 允许离线后快速重连（基于TOKEN）。
7. 新客户端登录后踢掉旧客户端。
8. 维护序列号，维持发往客户端消息的有序。
9. 维护Session与player的状态。


---

## 模块划分

### PlayerManager
维护PlayerId与对应Player服务入口的映射关系。提供启动和关闭Player服务的接口。
维护Player的活跃状态（超过指定时间未活跃的Player将被关闭）。
接收来自 HttpServer 的后端推送消息，将其转发给对应的Player。

### Player
维护Player 与 Session 之间的映射关系。
提供发往客户端消息的序列号。
缓存离线期间发送对应Player的消息。
将后端推送的消息发送给对应的Session 服务，如果无Session服务存在，则缓存起来。

### SessionManager
维护SessionId与对应Session服务入口的映射关系。提供创建和关闭Session服务的接口。
维护Session 的活跃状态。（超过指定时间不活跃的Session会被关闭）

### Session
持有Conn，接收来自连接的请求，并向后传递。将对应的响应转发给连接。
多个Session 之间并行。

### 路由模块
根据MessageId或者其他唯一标识符， 将消息转发给指定的后端服务器。

### Session 与 路由模块之间的中间模块
负责路由消息到后端服务前，以及转发后端响应到客户端前 的相关逻辑处理。
比如发送登录消息前，将当前会话状态设置为认证中，接收到登录响应后，根据响应结果更新会话状态。

### TcpServer
监听指定端口，接受来自客户端的建立连接请求。

### HttpServer
启动一个Http服务，监听指定端口。后端服务器通过对该端口发起Post请求来推送数据。

---

## 并发关系

### 并行模块

PlayerManager、多个Player、SessionManager、多个Session、与Session一一对应的Conn、TcpServer、HttpServer 这些服务之间并行执行。

串行的执行的服务为多个服务内部。

可以将这些服务视为Actor，每个Actor 服务内部单线程执行，多个Actor 服务之间并行执行。

服务之间通过信箱通信，属于异步。

---

## 依赖关系

Session 和 Player 彼此持有对应的服务信箱。
多数服务都持有 SessionManager 和 PlayerManager 服务信箱。

---

## 问题

1. Session 、中间模块、路由模块，这三个模块在一个线程中运行，这样设计有问题吗？这样设计的原因是中间模块和路由模块实际上是处理请求和获取响应的一部分。
2. Session在转发响应的时候，需要附加序列号，但是序列号是由player所维护的，这样就需要发送消息到Player的信箱并等待响应。这样的设计是否合理？
3. 后端推送消息的流程是以Post方式访问HTTP服务的指定端口，HTTP服务处理时将数据转发给Playermanager的信箱，Playermanager解析出PlayerIds将消息转发给对应Player的信箱。Player根据Session是否存在，来选择缓存消息还是发送给Session的消息。Session 再转发给客户端。
4. 维护方便的处理Player与Session的逻辑处理，比如获取某些字段，所以多数服务比如Playermanager、SessionManager、Session、Player 、路由模块、中间模块、TcpServer和HttpServer 等等都需要持有 PlayerManager和Sessionmanager来获取对应的Player和Session服务的信箱。这样的设计合理吗？
5. 由于每个服务的内部都是单线程的，且都是依次处理信箱消息，所以好像有死锁的风险，怎么避免？
6. 怎么设计路由模块。给出一个简要的设计思路。
7. 对应Session和Player的关闭消息，我是应该发送给SessionManager和PlayerManager还是直接发送给对应的Session和Player？
8. Session和Player是应该持有对方的ID还是持有对方服务的信箱？
9. 怎么处理服务关闭的情况，比如Session或者Player关闭？
10. 怎么平滑的关闭服务？比如关闭Session，怎么确保其关闭，关闭后发送回调吗？

1. 状态一致性
Session和Player状态如何保证一致性？

当Player被踢时，如何确保旧Session正确关闭？

重连时状态如何平滑转移？

2. 容错机制
某个Player服务崩溃是否影响其他Player？

SessionManager宕机如何恢复？

路由失败时的降级策略？

---

## 扩展

### 路由模块简单设计

``` go
type Router struct {
    // 消息ID到后端服务的映射
    routes map[int32]RouteInfo
    
    // 负载均衡器
    balancers map[string]LoadBalancer
    
    // 请求超时管理
    timeout time.Duration
}

type RouteInfo struct {
    ServiceName string    // 后端服务名
    MsgType     RouteType // 广播、单播、任意播
    NeedAuth    bool      // 需要认证
}

// 路由策略
func (r *Router) Route(msg *Message) []BackendInstance {
    routeInfo := r.routes[msg.MsgID]
    instances := r.balancers[routeInfo.ServiceName].Pick()
    return instances
}
```

### 服务关闭

需要双向确认。

``` go
// 关闭流程：
// 1. Manager发送关闭请求到服务信箱
// 2. 服务开始关闭流程，处理剩余消息
// 3. 服务关闭完成后通知Manager
// 4. Manager从注册表中移除

type ShutdownAck struct {
    ServiceID string
    Completed bool
}

func (m *SessionManager) shutdownSession(sessionID string) {
    session := m.sessions[sessionID]
    if session == nil {
        return
    }
    
    // 发送关闭请求
    session.mailbox <- ShutdownRequest{}
    
    // 等待确认或超时
    select {
    case ack := <-m.shutdownAckChan:
        if ack.Completed {
            delete(m.sessions, sessionID)
        }
    case <-time.After(5 * time.Second):
        // 强制清理
        delete(m.sessions, sessionID)
    }
}
```

关闭状态机
``` go
const (
    StateRunning = iota
    StateClosing  // 不再接受新消息，但处理存量消息
    StateClosed   // 完全关闭
)
```

消息处理策略：
- 运行态：正常处理所有消息
- 关闭中：只处理特定消息（如关闭确认、响应消息），拒绝新请求
- 已关闭：所有消息丢弃或返回错误

### 服务迁移的思路和基础要求

- 状态可序列化：Player所有重要状态都要能导出/导入
- 依赖倒置：通过接口访问其他服务，不要直接引用
- 生命周期明确：每个服务都有清晰的准备→迁移→提交流程
- 幂等操作：迁移可能重试，操作要支持幂等

``` go
// 迁移流程：
// 1. 源网关：暂停Player接受新消息，序列化状态
// 2. 协调器：选择目标网关，传输状态数据  
// 3. 目标网关：创建新Player服务，恢复状态
// 4. 更新路由信息，客户端重连到新网关

type MigrationCoordinator struct {
    // 管理整个迁移过程
    // 确保状态一致性
    // 处理迁移失败回滚
}
```

``` go
// PlayerManager维护PlayerID到信箱的映射
type PlayerManager struct {
    players map[int64]chan<- Message  // PlayerID -> 信箱
    
    // 迁移时：临时存储玩家状态
    pendingTransfers map[int64]*PlayerState
}

// 迁移流程：
// 1. 标记Player为"迁移中"，停止接受新消息
// 2. 序列化Player状态到临时存储
// 3. 创建新的Player服务，加载状态
// 4. 更新映射表，指向新信箱
// 5. 清理旧Player服务
```

---

## 服务关闭流程

### TcpServer关闭流程
1. **停止接受新连接**：关闭监听socket，不再accept新连接
2. **通知所有Session**：向SessionManager发送关闭信号，触发所有Session的关闭流程
3. **等待处理中请求**：等待已建立连接完成当前请求处理
4. **强制关闭残留连接**：超时后强制关闭所有剩余TCP连接
5. **释放资源**：清理socket资源，确认关闭完成

### HttpServer关闭流程  
1. **停止监听端口**：关闭HTTP监听器
2. **完成处理中请求**：等待正在处理的POST请求完成
3. **拒绝新请求**：返回503状态码给新到达的推送请求
4. **清理连接池**：关闭所有空闲HTTP连接
5. **资源释放**：清理HTTP处理器相关资源

### SessionManager关闭流程
1. **停止创建新Session**：拒绝新的Session创建请求
2. **广播关闭信号**：向所有活跃Session发送关闭指令
3. **等待Session关闭确认**：收集每个Session的关闭确认
4. **处理超时Session**：对未及时响应的Session强制清理
5. **清理注册表**：清空SessionID映射表，通知Player解除关联

### Session关闭流程
1. **状态标记为关闭中**：设置状态位，拒绝新消息入队
2. **关闭网络连接**：断开与客户端的TCP连接
3. **处理剩余消息**：继续处理信箱中已存在的消息直到空
4. **通知关联Player**：发送解除关联消息给对应Player
5. **确认关闭**：向SessionManager发送关闭完成确认
6. **资源清理**：释放所有缓冲区、定时器等资源

### PlayerManager关闭流程
1. **停止Player创建**：拒绝创建新Player的请求
2. **发起Player关闭**：向所有活跃Player发送关闭指令
3. **等待关闭确认**：收集Player关闭完成响应
4. **处理迁移中Player**：等待迁移操作完成或取消迁移
5. **清理全局状态**：清空Player注册表，持久化必要状态

### Player关闭流程
1. **状态标记为关闭中**：设置状态，停止接受新业务消息
2. **处理缓存消息**：将缓存中的离线消息持久化到存储
3. **解除Session关联**：清理与Session的绑定关系
4. **持久化状态**：保存Player当前状态到数据库
5. **确认关闭**：向PlayerManager发送关闭完成确认
6. **资源释放**：清理消息缓存、定时器等资源

---

## 服务迁移实现

### 迁移前提条件
- Player状态必须完整序列化
- 网关集群共享配置和路由信息
- 客户端支持网关重定向机制
- 存在共享存储用于迁移状态同步

### 迁移流程详细步骤

#### 阶段一：迁移准备
1. **迁移触发**
   - 源网关检测到负载过高或维护需求
   - 迁移协调器选择目标网关实例
   - 验证目标网关资源充足性

2. **Player状态锁定**
   ```go
   // 伪代码
   player.SetState(StateMigrating)     // 标记为迁移中
   player.StopAcceptingNewMessages()   // 停止接受新消息
   ```
   - 拒绝新的客户端请求
   - 继续处理已接收的消息直到完成

3. **状态序列化**
   - 序列化Player完整状态：会话数据、缓存消息、序列号等
   - 生成状态校验和用于完整性验证
   - 将序列化数据写入共享存储

#### 阶段二：迁移执行
4. **目标网关准备**
   - 目标网关从共享存储加载序列化状态
   - 验证状态完整性和校验和
   - 创建新的Player服务实例

5. **状态恢复**
   ```go
   // 伪代码  
   newPlayer := CreatePlayer(playerID)
   newPlayer.RestoreState(serializedData)
   newPlayer.ResumeSequence()  // 恢复消息序列号
   ```
   - 反序列化并恢复所有状态数据
   - 重建消息序列号生成器状态
   - 恢复离线消息缓存队列

6. **路由表更新**
   - 全局路由表更新：PlayerID → 目标网关地址
   - 通知所有相关服务路由变更
   - 等待路由传播完成

#### 阶段三：切换与清理
7. **客户端重定向**
   - 源网关向客户端发送重连指令（包含目标网关地址）
   - 客户端断开当前连接，连接到新网关
   - 新网关验证客户端Token，关联到迁移后的Player

8. **源网关清理**
   - 确认客户端已成功切换到新网关
   - 清理源Player服务实例
   - 释放所有相关资源

9. **迁移完成确认**
   - 迁移协调器标记迁移任务完成
   - 清理共享存储中的临时迁移状态
   - 更新监控和统计信息

### 迁移实现要点

#### 状态一致性保证
- 使用分布式锁确保同一Player不会同时迁移
- 迁移过程中所有状态变更通过共享存储同步
- 采用状态快照+操作日志的混合模式

#### 容错处理
- **超时机制**：每个迁移步骤设置超时，超时则回滚
- **回滚流程**：迁移失败时恢复源网关状态
- **幂等操作**：支持迁移操作重试，避免重复执行副作用

#### 性能优化
- 增量迁移：对大型Player状态支持增量传输
- 并行迁移：支持多个Player同时迁移到不同网关
- 流量控制：避免迁移过程影响正常服务性能

### 关键注意事项

1. **消息不丢失**：迁移过程中产生的消息必须可靠传递
2. **顺序保证**：客户端消息的顺序性在迁移后必须保持
3. **原子性**：迁移操作要么完全成功，要么完全回滚
4. **客户端透明**：尽量减少对客户端的感知和影响
5. **监控完备**：每个迁移步骤都有详细日志和监控指标

---

## Session服务关闭时的消息处理

### 关闭过程中的消息状态管理

#### 1. 接收但未处理的消息
- **识别来源**：区分来自客户端的请求 vs 来自后端的响应
- **请求消息**：直接拒绝，返回连接关闭错误给客户端
- **响应消息**：尝试转发，如失败则丢弃并记录日志

#### 2. 已处理但未转发的响应
```go
// 伪代码 - Session关闭时的消息清理
func (s *Session) shutdown() {
    s.state = StateClosing
    
    // 1. 停止接收新消息
    s.conn.SetReadDeadline(now)
    
    // 2. 处理已接收但未处理的消息
    for msg := range s.receiveBuffer {
        if msg.IsRequest() {
            s.sendErrorResponse(msg, "ServiceUnavailable")
        } else {
            // 响应消息，尝试发送
            s.trySendMessage(msg)
        }
    }
    
    // 3. 处理已处理但未发送的响应
    for msg := range s.sendBuffer {
        s.trySendMessage(msg)
    }
}
```

#### 3. 正在路由中的请求
- **维护请求映射表**：记录RequestID与客户端的映射关系
- **关闭时清理**：通知路由模块取消所有未完成请求
- **超时处理**：依赖后端服务的请求超时机制

---

## 服务迁移的完整方案

### 需要迁移的服务

#### 1. Player服务（必须迁移）
- 玩家状态数据
- 消息序列号状态
- 离线消息缓存
- Session关联信息

#### 2. Session服务（可选迁移）
**方案一：Session随Player迁移**
- 优点：客户端无感知，体验平滑
- 缺点：技术实现复杂，网络连接需要保持

**方案二：Session重建**
- 优点：实现简单，各网关独立
- 缺点：客户端需要重连，短暂中断

### 推荐方案：Player迁移 + Session重建

### 迁移详细流程

#### 阶段一：准备阶段
1. **迁移决策**
   - 源网关检测迁移条件（负载、维护等）
   - 选择目标网关，验证资源
   - 锁定Player和关联Session

2. **Session状态处理**
   ```go
   // 伪代码 - 准备迁移
   func prepareMigration(playerID int64) {
       // 1. 暂停Session接受新请求
       for _, session := range player.GetSessions() {
           session.PauseRequestAccepting()
       }
       
       // 2. 等待进行中请求完成
       waitForPendingRequests(playerID, 5*time.Second)
       
       // 3. 记录Session状态（Token、认证信息等）
       sessionStates := snapshotSessionStates(playerID)
   }
   ```

3. **Player状态序列化**
   - 序列化Player核心状态
   - 包含关联Session的元信息
   - 生成完整性校验码

#### 阶段二：迁移执行
4. **目标网关初始化**
   - 创建新Player实例
   - 恢复Player状态
   - 重建消息序列号生成器

5. **Session连接转移**
   - **关键问题**：TCP连接无法直接跨机器迁移
   - **解决方案**：
     ```go
     // 伪代码 - Session重建通知
     func notifySessionTransfer(sessions []*Session, targetGateway string) {
         for _, session := range sessions {
             // 发送重连指令给客户端
             session.SendReconnectCommand(targetGateway, newToken)
             
             // 设置短暂等待期，让客户端有时间接收指令
             session.SetGracefulClose(3 * time.Second)
         }
     }
     ```

6. **未完成请求处理**
   - **源网关**：继续处理已接收但未完成的请求
   - **响应路由**：响应通过全局路由发送到新网关的Player
   - **超时控制**：设置迁移期特殊超时处理

#### 阶段三：切换完成
7. **客户端重连**
   - 客户端收到重连指令，连接到目标网关
   - 使用新Token认证，关联到迁移后的Player
   - 新Session与Player建立关联

8. **源网关清理**
   - 确认所有Session已转移或关闭
   - 清理源Player实例
   - 通知相关服务迁移完成

### 迁移期间的请求处理策略

#### 1. 客户端新请求
- **迁移准备阶段**：接受但排队处理
- **迁移执行阶段**：拒绝，返回重连指令
- **迁移完成后**：由新网关处理

#### 2. 后端推送消息
- **全局路由表**：确保消息路由到正确的网关
- **消息缓冲**：短暂不可达时缓冲，恢复后投递
- **顺序保证**：通过序列号机制保证消息顺序

#### 3. 进行中请求的响应
```go
// 伪代码 - 迁移期响应处理
func handleMigrationResponse(response *Message) {
    if isDuringMigration(response.PlayerID) {
        // 通过全局路由转发到目标网关
        targetGateway := getMigrationTarget(response.PlayerID)
        forwardToGateway(targetGateway, response)
    } else {
        // 正常处理
        processNormalResponse(response)
    }
}
```

### 关键技术要点

#### 1. 状态一致性
- **原子性迁移**：Player状态转移是原子的
- **Session重建**：客户端重连时状态完全恢复
- **消息不丢失**：所有在途消息都有妥善处理

#### 2. 用户体验
- **快速重连**：Token机制确保快速认证
- **状态保持**：Player状态完全迁移，用户无感知
- **优雅降级**：迁移失败时回滚，不影响服务

#### 3. 容错设计
- **超时机制**：每个迁移步骤都有超时控制
- **回滚预案**：迁移失败时恢复原有状态
- **监控告警**：全程监控迁移状态和性能指标

这种方案在技术复杂度和用户体验间取得了平衡，既保证了状态的完整迁移，又通过Session重建避免了TCP连接迁移的技术难题。