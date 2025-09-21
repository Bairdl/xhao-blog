+++
date = '2025-09-15T22:32:30+08:00'
draft = false
title = '游戏网关设计 GO'
# mermaid = true
+++

# 游戏网关设计

## 目标

负责处理游戏客户端与服务端之间传递的消息。

1. 消息转发：负责客户端请求和服务端响应的转发。
2. 向客户端推送服务端广播的消息。
3. 维护当前网关活跃的玩家列表，缓存一定时间内的离线消息。
4. 要有高扩展性，可以方便地在网关上添加部分业务逻辑。（例如将地图隶属的逻辑在网关部分处理。）
5. 与当前Go服务兼容，使用当前Go服务端的服务结构，以Go服务端的一个服务运行。
6. 统计所需数据。

## 设计

### 当前设计

- 使用 Player 维护玩家信息，包括离线消息等。使用 PlayerManager 维护当前网关活跃玩家列表，定期清理不活跃的玩家信息。  
- 使用 Session 维护游戏客户端与网关之间的 Tcp 连接，负责接受来自游戏客户端的消息，并将其封装后以 HTTP 请求形式发送给服务端，接收到 HTTP 响应后，转换数据使用 Tcp 连接发送回游戏客户端，并更新 Player 状态（最新活跃时间等）。如果是检查登录等消息，还需要更新 Player 与 Session 的映射关系。  
- 使用 SessionManager 维护当前网关活跃的 Session 列表。  
- 使用 Tcp 监听服务，接受新的客户端连接请求，并通知 SessionManager 创建新的 Session 进行管理。  
- 使用 HTTP 监听服务，处理指定路由并使用Post格式的请求。服务端会通过使用指定方式向指定路由发送 HTTP 请求的形式主动要求网关向游戏客户端推送数据。  

心跳机制

Session的心跳机制，避免在客户端的静默期被错误清理。当前仅仅使用下行数据包来检测存活状态是不靠谱的。

## 数据统计

### 一、 指标分类与具体指标

我们将指标分为四大黄金信号，并额外增加一个资源类。

#### 1. 流量指标 (Traffic - 了解负载和规模)
*   `gateway_requests_total` (Counter): 接收到的客户端请求总数。
    *   **标签**: `cmd` (指令号), `player_id`, `route_target` (转发目标服务, 如 "map_service"), `result` ("success", "error", "drop")
    *   **用途**: 了解API调用量、热门指令、异常玩家（`player_id`过滤）。
*   `gateway_bytes_in_total` (Counter): 接收到的网络字节总数。
*   `gateway_bytes_out_total` (Counter): 发送出的网络字节总数。
    *   **用途**: 评估带宽成本和使用情况。

#### 2. 错误指标 (Errors - 发现问题)
*   `gateway_errors_total` (Counter): 网关内部错误总数。
    *   **标签**: `type` ("auth_failed", "decode_failed", "route_not_found", "backend_unavailable", "session_invalid")
    *   **用途**: 快速定位错误来源，区分是客户端问题、网络问题还是后端问题。
*   `gateway_backend_errors_total` (Counter): 调用后端服务失败次数。
    *   **标签**: `target_service`, `status_code` (5xx, 408等), `error_type` ("timeout", "connect_error")
    *   **用途**: 评估后端服务健康度，快速定位故障服务。

#### 3. 延迟指标 (Latency - 评估性能)
*   `gateway_request_duration_seconds` (Histogram): 处理每个请求的完整耗时（从收到包到发送响应）。
    *   **标签**: `cmd`, `route_target`
    *   **用途**: 分析性能瓶颈。95th/99th分位数至关重要。
*   `gateway_backend_duration_seconds` (Histogram): 调用后端服务的耗时（从发起到收到响应）。
    *   **标签**: `target_service`
    *   **用途**: 精确评估每个后端服务的响应性能。

#### 4. 饱和度量指标 (Saturation - 评估容量)
*   `gateway_sessions_current` (Gauge): 当前活跃的会话数量。
    *   **标签**: `status` ("connected", "authed")
    *   **用途**: 监控在线玩家数，评估网关负载。
*   `gateway_requests_in_flight` (Gauge): 当前正在处理的请求数（尚未返回）。
    *   **用途**: 判断网关是否过载，如果此值持续很高，说明处理能力不足。

#### 5. 资源指标 (Resources - 监控网关本身)
*   `gateway_goroutines_total` (Gauge): 当前Goroutine数量。
*   `process_resident_memory_bytes` (Gauge): 进程占用内存大小。（通常由官方库自动提供）
*   `process_cpu_seconds_total` (Counter): 进程占用CPU时间。（通常由官方库自动提供）
    *   **用途**: 监控网关自身健康度，防止资源泄漏。

---

### 二、 Prometheus如何统计：定义与埋点

#### 1. 定义指标（项目初始化时）

在`metrics.go`或类似文件中，定义所有指标。

```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    // 1. 流量指标
    requestsTotal = promauto.NewCounterVec(prometheus.CounterOpts{
        Name: "gateway_requests_total",
        Help: "The total number of processed requests",
    }, []string{"cmd", "player_id", "route_target", "result"})

    // 2. 延迟指标
    requestDuration = promauto.NewHistogramVec(prometheus.HistogramOpts{
        Name:    "gateway_request_duration_seconds",
        Help:    "Time spent processing a request",
        Buckets: prometheus.DefBuckets, // 使用默认桶 [.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10]
    }, []string{"cmd", "route_target"})

    // 3. 错误指标
    errorsTotal = promauto.NewCounterVec(prometheus.CounterOpts{
        Name: "gateway_errors_total",
        Help: "The total number of internal errors",
    }, []string{"type"})

    // 4. 饱和度量指标
    sessionsCurrent = promauto.NewGaugeVec(prometheus.GaugeOpts{
        Name: "gateway_sessions_current",
        Help: "The current number of active sessions",
    }, []string{"status"})
)
```

#### 2. 在流程中埋点

以下是在网关核心流程中埋点的伪代码：

```go
// --- 埋点示例：处理一个客户端请求 ---
func (s *Session) HandlePacket(packet *Packet) {
    // A. 开始计时器
    start := time.Now()
    // 预先定义结果标签，默认为成功，若出错则修改
    resultLabel := "success" 

    // B. 在函数退出前记录耗时和请求量
    defer func() {
        duration := time.Since(start).Seconds()
        // 记录耗时
        requestDuration.WithLabelValues(packet.Cmd, s.RouteTarget).Observe(duration)
        // 记录请求总量
        requestsTotal.WithLabelValues(packet.Cmd, s.PlayerID, s.RouteTarget, resultLabel).Inc()
    }()

    // C. 核心处理逻辑
    // 1. 解码/验证
    if err := packet.Decode(); err != nil {
        resultLabel = "error"
        errorsTotal.WithLabelValues("decode_failed").Inc() // <- 特定错误埋点
        return
    }

    // 2. 路由 (决定发往哪个后端服务，如 "map_service")
    target, err := router.Route(s, packet)
    if err != nil {
        resultLabel = "drop"
        errorsTotal.WithLabelValues("route_not_found").Inc() // <- 特定错误埋点
        return
    }
    s.RouteTarget = target

    // 3. 转发请求到后端并获取响应
    resp, err := backendClient.ForwardRequest(target, packet)
    if err != nil {
        resultLabel = "error"
        // 记录后端错误详情
        backendErrorsTotal.WithLabelValues(target, parseErrorType(err)).Inc()
        return
    }

    // 4. 发送响应回客户端
    if err := s.Send(resp); err != nil {
        resultLabel = "error"
        errorsTotal.WithLabelValues("send_failed").Inc()
    }
}
```

```go
// --- 埋点示例：会话管理 ---
func (sm *SessionManager) CreateSession(conn net.Conn) *Session {
    session := NewSession(conn)
    // 增加“已连接”会话数
    sessionsCurrent.WithLabelValues("connected").Inc()
    return session
}

func (sm *SessionManager) DestroySession(s *Session) {
    // 根据会话状态，减少相应的会话数
    if s.Authed {
        sessionsCurrent.WithLabelValues("authed").Dec()
    } else {
        sessionsCurrent.WithLabelValues("connected").Dec()
    }
    s.Conn.Close()
}
```

---

### 三、 数据如何转化为可理解的洞察

1.  **可视化 (Grafana)**
    *   **大盘 (Dashboard)**: 创建多个面板，分别展示：
        *   **总览**: QPS (使用`rate(requests_total[1m])`)、在线人数、错误率、平均延迟。
        *   **深度分析**: 按`cmd`分解的QPS和延迟（用于发现热门或性能差的指令），按`player_id`过滤的请求频率（用于抓“刷子”）。
        *   **后端依赖**: 各个`target_service`的错误率和延迟（一眼找到故障或性能瓶颈服务）。
        *   **资源**: Goroutine数、内存使用量。

2.  **告警 (Prometheus Alertmanager)**
    *   **错误激增**: `rate(gateway_errors_total[5m]) > 10`
    *   **延迟过高**: `histogram_quantile(0.99, rate(gateway_request_duration_seconds_bucket[5m])) > 1.5` (99%线延迟大于1.5秒)
    *   **后端服务宕机**: `rate(gateway_backend_errors_total{status_code="500"}[2m]) > 5`
    *   **玩家异常**: `rate(requests_total{player_id!=""}[5m]) > 1000` (单个玩家请求速率超过1000/分钟)
