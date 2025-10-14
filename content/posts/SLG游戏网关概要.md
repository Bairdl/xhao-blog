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

