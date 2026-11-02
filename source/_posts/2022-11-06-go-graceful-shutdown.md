---
title: Go 的优雅退出实现与探讨
date: 2022-11-06
categories:
  - go
tags:
  - Go
  - 优雅退出
  - signal
---

服务进程收到的终止信号通常来自人工 `Ctrl+C`、系统的 `kill`、容器编排平台或进程管理器。如果直接退出，正在处理的请求会被切断，缓冲中的数据、未提交的事务、未确认的消息都可能丢失。优雅退出要解决的问题是：**停止接收新工作、处理完手头工作、按正确顺序释放资源，最后退出**。

本文先给出可直接使用的实现，再分析标准库的机制，最后讨论实践中容易踩的坑。

## 1. 什么是优雅退出

一个可用的优雅退出至少包含四件事：

1. **停止接收新请求**：关闭监听端口，不再 accept 新连接；
2. **等待存量工作完成**：让进行中的 HTTP 请求、后台任务、消息处理跑完；
3. **按依赖顺序释放资源**：先停入口，再停消费者，最后关数据源；
4. **有兜底**：超过设定时间仍未结束，就强制退出，避免进程永久挂住。

“优雅”不等于“无限等待”。没有超时的优雅退出，可能把发布流程卡死，最终由平台强制杀掉，效果还不如直接退出。

## 2. 基础实现：信号 + context

Go 标准库从 1.16 起提供 `signal.NotifyContext`，把信号转换为 context 取消，是最简洁的入口方式：

```go
package main

import (
    "context"
    "log"
    "os"
    "os/signal"
    "syscall"
)

func main() {
    ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer stop()

    go runServer()

    <-ctx.Done()
    log.Println("收到退出信号，开始关闭")
    stop() // 恢复信号默认行为，此后的信号会直接终止进程
}
```

`os.Interrupt` 对应 `Ctrl+C`（`SIGINT`），`syscall.SIGTERM` 是 `kill` 与容器平台停止容器时发送的信号。两者都应该监听。

`stop()` 除了释放资源，还有一个实际作用：调用后信号处理恢复默认行为，用户在关闭卡住时再按一次 `Ctrl+C` 就能直接终止进程，不必另写强制退出逻辑。

## 3. HTTP 服务的优雅关闭

`http.Server.Shutdown` 会关闭监听器并等待活跃请求结束：

```go
func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        time.Sleep(2 * time.Second)
        w.Write([]byte("done"))
    })

    srv := &http.Server{Addr: ":8080", Handler: mux}

    go func() {
        if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            log.Fatalf("listen: %v", err)
        }
    }()

    ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer stop()
    <-ctx.Done()

    shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    if err := srv.Shutdown(shutdownCtx); err != nil {
        log.Printf("优雅关闭超时: %v", err)
        _ = srv.Close() // 兜底：直接断开剩余连接
    }
    log.Println("已退出")
}
```

几个需要理解的行为：

- `Serve`/`ListenAndServe` 在调用 `Shutdown` 后会返回 `http.ErrServerClosed`，属于正常情况，不能当成错误；
- `Shutdown` 只处理普通连接，**不会关闭也不会等待被 hijack 的连接**（例如 WebSocket）。这类连接需要自己通过 `RegisterOnShutdown` 或业务逻辑通知关闭；
- 超时后 `Shutdown` 返回 `ctx.Err()`，此时进程里可能还有请求在跑，必须用 `Close()` 或直接退出兜底；
- `Shutdown` 与 `Close` 的区别是“等”与“断”，二者不可混用为同一个语义。

## 4. 关闭顺序与多组件编排

真实服务通常不止一个 HTTP Server，还有后台 worker、消息消费者、定时任务和数据库连接。合理的顺序是：

```text
1. 关闭入口监听（HTTP / gRPC）
2. 停止接收新任务（消费者停止拉取、任务队列停止投递）
3. 等待存量任务完成（WaitGroup / 业务信号）
4. 关闭下游连接（MQ、DB、缓存、HTTP 客户端）
5. 退出
```

一个可复用的写法是给每个组件定义统一的接口，再按顺序调用：

```go
type Component interface {
    Name() string
    Shutdown(ctx context.Context) error
}

func shutdownAll(ctx context.Context, components []Component) {
    for _, c := range components {
        if err := c.Shutdown(ctx); err != nil {
            log.Printf("关闭 %s 失败: %v", c.Name(), err)
        }
    }
}
```

后台任务用 `WaitGroup` 等待：

```go
func startWorker(ctx context.Context, wg *sync.WaitGroup) {
    wg.Add(1)
    go func() {
        defer wg.Done()
        for {
            select {
            case <-ctx.Done():
                return
            case job := <-jobs:
                process(job)
            }
        }
    }()
}
```

注意顺序不能反：如果先关数据库再停 worker，worker 可能在关闭过程中继续访问已经关闭的连接，产生大量错误日志。入口先停、数据源最后关，是基本纪律。

## 5. 标准库里的机制

### 5.1 `signal.NotifyContext` 做了什么

源码中的实现很直接：

```go
func NotifyContext(parent context.Context, signals ...os.Signal) (ctx context.Context, stop context.CancelFunc) {
    ctx, cancel := context.WithCancel(parent)
    c := &signalCtx{Context: ctx, cancel: cancel, signals: signals}
    c.ch = make(chan os.Signal, 1)
    Notify(c.ch, c.signals...)
    if ctx.Err() == nil {
        go func() {
            select {
            case <-c.ch:
                c.cancel()
            case <-c.Done():
            }
        }()
    }
    return c, c.stop
}
```

关键点：

- 内部 channel **缓冲区为 1**。`os/signal` 向 channel 发送时不会阻塞，如果调用方不消费且缓冲已满，后续同型号信号会被丢弃；
- 起一个 goroutine 等待信号并调用 `cancel()`；
- 返回的 `stop` 会 cancel 上下文并调用 `signal.Stop`，这正是“恢复默认信号行为”的来源。

这也解释了为什么自己用 `signal.Notify` 时 channel 通常声明为 `make(chan os.Signal, 1)`：避免在还没开始接收时丢掉第一个信号。

### 5.2 `http.Server.Shutdown` 做了什么

以 Go 1.25 的实现为例，`Shutdown` 的流程是：

1. 把 `inShutdown` 置为 true，此后新连接会被拒绝；
2. 关闭所有监听器，停止 accept，并等待监听相关的 goroutine 退出；
3. 以 goroutine 方式执行 `RegisterOnShutdown` 注册的回调（用于通知 hijack 连接）；
4. 循环调用 `closeIdleConns()` 关闭空闲连接，直到服务器进入静止状态；
5. 轮询间隔从 1ms 开始指数增长，带 10% 抖动，上限 500ms；
6. 一旦 `ctx.Done()` 触发，立即返回 `ctx.Err()`。

轮询而不是事件通知，换来的是实现简单：每次检查都重新统计活跃连接，因此关闭期间的连接变化不需要额外同步。代价是关闭完成有一定延迟（最多约一个轮询周期），这也是 `Shutdown` 返回值不宜用来精确判断“何时完全停止”的原因。

## 6. Kubernetes 终止流程与信号语义

容器平台上的下线不是"发一个信号"这么简单，而是"摘流 + 等待 + 信号 + 兜底"的组合。理解信号语义，才能把 `preStop`、宽限期和进程内收尾安排明白。

### SIGTERM 与 SIGKILL

| 信号 | 编号 | 能否捕获 | 语义 |
|---|---|---|---|
| `SIGTERM` | 15 | 可以捕获 | "请求你退出"，进程可以自行决定何时退出，因此可以等待 |
| `SIGKILL` | 9 | 不可捕获、不可忽略 | 内核直接终止进程，没有任何清理机会，不会等待 |

### 完整终止顺序

1. Pod 状态变为 `Terminating`，同时从 Service endpoints 中摘除（异步进行，可能比信号更晚完成）；
2. 执行 `preStop` hook（如果配置了），耗时计入宽限期；
3. 向容器内的 **PID 1** 发送 `SIGTERM`；
4. 应用捕获 `SIGTERM`，开始优雅关闭：停监听、等存量请求、关连接；
5. 宽限期（`terminationGracePeriodSeconds`）到期仍未退出，kubelet 发送 `SIGKILL`，进程立即死亡。

`preStop` 与 `SIGTERM` 是**串行**的：`preStop` 执行完毕才会发信号。宽限期从 Pod 开始终止时计时，`preStop` 里的 sleep 同样占用这段时间。

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 5"]
terminationGracePeriodSeconds: 30
```

上面这个配置意味着：应用真正可用于收尾的时间约为 25 秒，而不是 30 秒。

### 信号是发给 PID 1 的

如果 entrypoint 写成 `sh -c "./app"` 而没有使用 `exec`，应用就不是 PID 1，`SIGTERM` 不会被转发，应用永远收不到信号，只能等 `SIGKILL`。这类问题的表现是"优雅退出代码写了但从不执行"。

```dockerfile
# 错误：shell 成为 PID 1，信号不会转发给 app
CMD ./app

# 正确：exec 让 app 替换 shell，成为 PID 1
CMD ["/app"]
```

### 镜像可以覆盖停止信号

Dockerfile 的 `STOPSIGNAL` 会改变运行时发送的终止信号。例如 nginx 官方镜像使用 `SIGQUIT` 做优雅退出：

```dockerfile
STOPSIGNAL SIGQUIT
```

此时 Kubernetes 下发的就不是 `SIGTERM` 了。排查"信号处理没生效"时，先确认镜像声明的停止信号。

### `SIGINT` 的定位

`SIGINT` 是终端里 `Ctrl+C` 触发的信号，Kubernetes **不会**发送它。本地开发时同时监听 `os.Interrupt` 和 `syscall.SIGTERM` 是常见做法，线上真正起作用的是 `SIGTERM`。

### 三者的分工

- `preStop`：解决"信号已发出，但流量还没摘干净"的竞态，通常就是短 sleep 或调用下线接口；
- `SIGTERM`：解决"进程内部如何有序收尾"，业务逻辑写在这里；
- `SIGKILL`：兜底，保证进程最终一定会死，所以收尾必须在宽限期内完成。

需要强调的是：`SIGTERM` 的"等待"不是信号本身在等，而是**进程捕获后主动延长了退出时间**；`SIGKILL` 的"不等待"则是因为内核直接终止进程，用户态没有任何执行机会。

## 7. 实践探讨与常见坑

### 超时设置要与平台对齐

优雅关闭的超时应当**小于**平台的强制终止时间。以 Kubernetes 为例，默认 `terminationGracePeriodSeconds` 是 30 秒，再减去 `preStop` 的耗时；如果 `Shutdown` 也按 30 秒等，进程很可能在收尾阶段被 `SIGKILL`，连日志都写不完。实践上留出余量，比如 10 到 20 秒。

### 二次信号：要么强制退出，要么恢复默认

两种常见策略：

```go
// 策略一：自己接管第二次信号
sigCh := make(chan os.Signal, 1)
signal.Notify(sigCh, os.Interrupt, syscall.SIGTERM)
<-sigCh
go gracefulShutdown()
<-sigCh
log.Println("收到第二次信号，强制退出")
os.Exit(1)
```

```go
// 策略二：NotifyContext + 提前 stop，交给系统默认行为
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
<-ctx.Done()
stop() // 之后再次 Ctrl+C 直接终止
```

不要两种情况混用：既没有 `stop()`，又用 `os.Exit` 之外的路径“等第二次信号”，会出现关闭逻辑无法被打断的僵局。

### `os.Exit` 不会执行 defer

在关闭逻辑中调用 `os.Exit` 会跳过所有 defer，包括 `defer cancel()`、`defer db.Close()`。要么先把清理显式做完再退出，要么只让 `main` 正常返回。

### 长连接与流式响应

`Shutdown` 不管 hijack 的连接，也不管已经升级的协议。WebSocket、gRPC streaming、SSE 都需要业务自己发送关闭通知，并设置读超时，否则这些连接会让关闭一直等到超时。

### 后台任务要能响应取消

只在 `main` 里等待信号是不够的。worker 内部必须通过 `select` 监听 `ctx.Done()`，或者把 ctx 一路传进阻塞调用，否则 `WaitGroup.Wait()` 会无限等待。关不掉的 goroutine 是优雅退出最常见的失败原因。

### Windows 与 Linux 的信号差异

`syscall.SIGTERM` 在 Windows 上不会被外部正常发送，`signal.Notify` 注册它没有实际作用；Windows 下通常只有 `Ctrl+C`（`os.Interrupt`）和 `Ctrl+Break`。因此优雅退出的行为差异最好在 Linux 环境或 WSL 中验证。

### 只关监听器不等于关连接池

进程退出前还应关闭 HTTP 客户端的空闲连接（`http.Transport.CloseIdleConnections`）、消息消费者和数据库连接池。只关闭自己的 Server，不会自动清理这些外部资源。

### 幂等与并发

`Shutdown` 可能被多个路径触发（信号、健康检查失败、管理接口）。用一个 `sync.Once` 或原子标志包住整个关闭流程，避免重复关停导致日志混乱或二次关闭错误。

## 8. 参考文档

- [Go 语言规范之外：os/signal 包文档](https://pkg.go.dev/os/signal)
- [os/signal.NotifyContext 源码（Go 1.25）](https://github.com/golang/go/blob/go1.25.12/src/os/signal/signal.go)
- [net/http.Server.Shutdown 文档](https://pkg.go.dev/net/http#Server.Shutdown)
- [net/http 服务端源码（Go 1.25）](https://github.com/golang/go/blob/go1.25.12/src/net/http/server.go)
- [Go 博客：Contexts and Cancellation](https://go.dev/blog/context)
- [Linux signal(7)：信号语义与默认行为](https://man7.org/linux/man-pages/man7/signal.7.html)
- [Docker 文档：STOPSIGNAL](https://docs.docker.com/reference/dockerfile/#stopsignal)
- [Kubernetes 文档：Pod 生命周期与优雅终止](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination)
