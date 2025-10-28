---
date: '2025-10-29T00:04:56+08:00'
draft: false
title: 'Go中Etcd库的使用'
tags: []
categories: []
---


# Go 中 etcd 的使用指南

## 1. 官方客户端库

在 Go 语言中，应使用 **etcd 官方维护的客户端库**：

- **库名称**：`go.etcd.io/etcd/clientv3`
- **GitHub 地址**：https://github.com/etcd-io/etcd/tree/main/client/v3
- **Go Module 路径**：`go.etcd.io/etcd/client/v3`

> 注意：旧版路径 `go.etcd.io/etcd/clientv3` 已重定向，推荐使用新路径 `go.etcd.io/etcd/client/v3`（v3.5+）。

安装命令：
```bash
go get go.etcd.io/etcd/client/v3
```

该库基于 gRPC 实现，完全支持 etcd v3 API 的所有核心功能，包括 KV 操作、Watch、Lease、事务等。

---

## 2. 基础使用：客户端初始化

```go
import (
    "context"
    "time"
    "go.etcd.io/etcd/client/v3"
)

// 创建客户端
cli, err := clientv3.New(clientv3.Config{
    Endpoints:   []string{"localhost:2379"}, // etcd 服务地址
    DialTimeout: 5 * time.Second,           // 连接超时
})
if err != nil {
    // 处理错误
}
defer cli.Close()
```

生产环境中建议配置 TLS、认证、负载均衡等参数。

---

## 3. 核心操作示例

### 3.1 KV 操作（Put / Get / Delete）

```go
// Put
_, err = cli.Put(context.TODO(), "mykey", "myvalue")
if err != nil { /* handle */ }

// Get
resp, err := cli.Get(context.TODO(), "mykey")
if err != nil { /* handle */ }
for _, kv := range resp.Kvs {
    fmt.Printf("Key: %s, Value: %s, Revision: %d\n", kv.Key, kv.Value, kv.ModRevision)
}

// Delete
_, err = cli.Delete(context.TODO(), "mykey")
```

支持范围删除（`Delete(context.TODO(), "prefix/", clientv3.WithPrefix())`）。

---

### 3.2 使用 Lease（租约）

```go
// Grant a 10-second lease
leaseResp, err := cli.Grant(context.TODO(), 10)
if err != nil { /* handle */ }

// Attach key to lease
_, err = cli.Put(context.TODO(), "mykey", "myvalue", clientv3.WithLease(leaseResp.ID))

// KeepAlive (自动续期)
ch, kaErr := cli.KeepAlive(context.TODO(), leaseResp.ID)
if kaErr != nil { /* handle */ }

// 接收续期响应（可选）
go func() {
    for {
        select {
        case _, ok := <-ch:
            if !ok {
                // KeepAlive stopped
                return
            }
            // Successfully renewed
        }
    }
}()
```

Lease 过期后，绑定的 key 自动删除，适用于服务注册场景。

---

### 3.3 Watch 机制

```go
// Watch a key
watchCh := cli.Watch(context.TODO(), "mykey")

// Watch with prefix
// watchCh := cli.Watch(context.TODO(), "service/", clientv3.WithPrefix())

go func() {
    for wresp := range watchCh {
        for _, ev := range wresp.Events {
            switch ev.Type {
            case clientv3.EventTypePut:
                fmt.Printf("PUT: %s = %s\n", ev.Kv.Key, ev.Kv.Value)
            case clientv3.EventTypeDelete:
                fmt.Printf("DELETE: %s\n", ev.Kv.Key)
            }
        }
    }
}()
```

Watch 支持从指定 revision 开始监听（`clientv3.WithRev(rev)`），用于断线重连。

---

### 3.4 事务（Txn）

```go
txnResp, err := cli.Txn(context.TODO()).
    If(clientv3.Compare(clientv3.Value("mykey"), "=", "oldvalue")).
    Then(clientv3.OpPut("mykey", "newvalue")).
    Else(clientv3.OpGet("mykey")).
    Commit()

if err != nil { /* handle */ }

if txnResp.Succeeded {
    // 条件成立，执行了 Put
} else {
    // 条件不成立，执行了 Get
}
```

支持复合条件（`If(...).If(...)`）和多操作（`Then(Op1, Op2)`）。

---

### 3.5 批量操作与范围查询

```go
// 获取所有以 "config/" 开头的 key
resp, err := cli.Get(context.TODO(), "config/", clientv3.WithPrefix())

// 按字典序范围查询
resp, err := cli.Get(context.TODO(), "a", clientv3.WithRange("z"))
```

---

## 4. 生产环境注意事项

### 4.1 错误处理
- 所有操作均可能返回 `context.DeadlineExceeded`、`rpctypes.ErrGRPC*` 等错误；
- 网络分区或 Leader 切换时可能返回 `etcdserver: request timed out`，需重试。

### 4.2 连接复用
- `clientv3.Client` 是线程安全的，应全局复用，避免频繁创建；
- 不要为每次操作新建客户端。

### 4.3 资源管理
- 使用 `defer cli.Close()` 确保连接释放；
- Watch 和 KeepAlive 的 channel 需妥善管理，避免 goroutine 泄漏。

### 4.4 版本兼容性
- 确保客户端与 etcd 服务端版本兼容（建议主版本一致）；
- v3.5+ 客户端默认使用新 module 路径。

---

## 5. 典型应用场景代码结构

### 服务注册与发现
```go
// 注册
lease, _ := cli.Grant(ctx, 10)
cli.Put(ctx, "/services/myapp/"+host, ip, clientv3.WithLease(lease.ID))
cli.KeepAlive(ctx, lease.ID)

// 发现
watchCh := cli.Watch(ctx, "/services/myapp/", clientv3.WithPrefix())
```

### 分布式锁（基于 Lease + Compare）
```go
lockKey := "/locks/mylock"
lease, _ := cli.Grant(ctx, 10)
_, err := cli.Txn(ctx).
    If(clientv3.Compare(clientv3.CreateRevision(lockKey), "=", 0)).
    Then(clientv3.OpPut(lockKey, "", clientv3.WithLease(lease.ID))).
    Commit()
// 成功则获得锁
```

---

## 总结

Go 中使用 etcd 应始终采用官方 `client/v3` 库。其 API 设计清晰，完整支持 KV、Lease、Watch、Txn 等核心原语。在生产环境中，需重点关注连接复用、错误重试、资源泄漏和版本兼容性问题。掌握该库的正确用法，是构建可靠分布式 Go 应用的基础。