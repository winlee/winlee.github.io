---
title: Go 的数据结构 Channel
date: 2022-12-10
categories:
  - go
tags:
  - Go
  - Channel
  - 并发
---

> 本文按 Go 语言规范解释 Channel 的可观察行为，并结合 Go 1.25.12 的运行时源码说明一种实现方式。运行时布局属于实现细节，不应作为应用程序依赖的 API。

## 1. Channel 是什么

Channel 是 Go 内建的并发通信机制：它让一个 goroutine 发送指定类型的值，另一个 goroutine 接收值。声明时可以使用双向或单向类型：

```go
var ch chan int       // 双向
var send chan<- int   // 只发送
var recv <-chan int   // 只接收
```

通常通过 `make` 创建可用的 Channel：

```go
jobs := make(chan Job)       // 无缓冲
results := make(chan Result, 8) // 缓冲容量为 8
```

Channel 值像 map 和 slice 一样引用运行时的数据结构。复制 Channel 变量只是复制引用；多个 goroutine 使用同一个 Channel，不会因此得到多个独立队列。

## 2. 无缓冲与有缓冲

无缓冲 Channel 没有可暂存的元素。发送和接收必须在时间上相遇：

```go
ch := make(chan string)

go func() {
    ch <- "ready" // 等待接收方
}()

message := <-ch // 与发送配对后完成
```

无缓冲通信除了传值，也形成同步点。发送操作与对应接收操作具有 happens-before 关系，因此接收方可以看到发送前完成的写入。它不是“某个 goroutine 先执行到哪一行”的保证，而是语言内存模型定义的同步关系。

有缓冲 Channel 允许在缓冲区有空位时先发送，接收方在缓冲区非空时即可读取：

```go
queue := make(chan Job, 2)
queue <- job1
queue <- job2
// 此时缓冲已满；继续发送会阻塞，直到有接收腾出空间。
```

容量是背压边界，不是提高吞吐的万能开关。容量太大可能把内存消耗和积压延迟隐藏起来；容量太小则可能频繁阻塞。选择容量时应结合生产速度、消费速度、可接受排队量和任务对象大小。

## 3. 发送、接收与阻塞

一般情况下，发送表达式与发送值先求值，然后 Channel 通信才开始：

```go
ch <- buildMessage()
```

发送会等待直到操作可以完成。无缓冲 Channel 需要接收方准备好；有缓冲 Channel 需要存在空位。对 nil Channel 发送会永久阻塞；对已关闭 Channel 发送会 panic。

接收表达式在没有可接收值时等待。双返回值形式用于判断关闭状态：

```go
value, ok := <-ch
if !ok {
    // Channel 已关闭且缓存值已全部读完
}
```

关闭并不会丢弃缓冲区内已发送的值。接收方先读完这些值，之后才会立即得到元素类型的零值，并且 `ok == false`。若零值本身有业务含义，例如 `0`、空字符串或 nil 指针，应使用 `ok` 区分“真正发送的零值”和“关闭后的零值”。

也可以使用 `for range` 持续接收，直到 Channel 关闭且缓冲数据耗尽：

```go
for job := range jobs {
    process(job)
}
```

## 4. 关闭规则与所有权

`close(ch)` 表示发送方不再产生新值，它不是“销毁 Channel”，也不是取消接收方正在执行的工作。关闭操作有明确约束：

- 关闭 nil Channel 会 panic。
- 重复关闭会 panic。
- 向已关闭 Channel 发送会 panic。
- 从已关闭 Channel 接收是合法的，缓存被读完后返回零值和 `ok == false`。
- 只有发送方向的代码通常应拥有关闭权限；只接收方不应负责关闭。

多生产者场景尤其要明确谁负责最终关闭：必须确保所有发送方都已停止发送，再由唯一的协调者关闭 Channel。对“发送后再检查是否关闭”的写法很容易产生竞态；需要退出传播时，通常使用 `context.Context`、额外的 done Channel 或显式协调机制，而不是让多个发送方争抢关闭。

## 5. select：等待多个通信事件

`select` 同时等待多个发送或接收操作：

```go
select {
case result := <-results:
    handle(result)
case <-ctx.Done():
    return ctx.Err()
case <-time.After(time.Second):
    return errors.New("timeout")
}
```

若多个通信都已就绪，规范规定从可执行分支中伪随机选择一个；不要把 `select` 当作严格优先级队列。`default` 会使 select 不等待，可用于非阻塞尝试，但忙循环中的 `default` 也可能白白消耗 CPU。

nil Channel 永远不能通信，因此可以在循环中将已结束的输入 Channel 设为 nil，让相应分支暂时退出可选集合：

```go
for left != nil || right != nil {
    select {
    case v, ok := <-left:
        if !ok {
            left = nil
            continue
        }
        consume(v)
    case v, ok := <-right:
        if !ok {
            right = nil
            continue
        }
        consume(v)
    }
}
```

注意 select 分支中的 Channel 表达式和发送值会在进入 select 时求值；它们可能产生副作用，即使最终选择的是另一个分支也一样。

## 6. 运行时如何实现 Channel

下面只讨论 Go 1.25.12 的一种实现。`runtime/chan.go` 中，Channel 运行时对象 `hchan` 大致包含：

- `qcount`、`dataqsiz`：当前排队元素数与缓冲容量；
- `buf`、`sendx`、`recvx`：环形缓冲区及发送、接收位置；
- `sendq`、`recvq`：等待发送和等待接收的 goroutine 队列；
- `closed`：关闭状态；
- `lock`：保护状态转换和等待队列的互斥锁。

发送操作在持锁后，会先看是否有等待中的接收者。若有，运行时可以把值直接交给接收者，不必先经过缓冲区；否则若缓冲区有空位，则写入环形缓冲区；若两种条件都不满足，当前 goroutine 会进入发送等待队列并 park。接收路径同理：匹配等待发送者、从缓冲区读取，或者将自身挂入接收队列等待。

这里的 park 不是把操作系统线程永久阻塞，而是让 goroutine 暂停，运行时可调度其他 goroutine。Channel 代码还要处理垃圾回收、栈移动、竞态检测、select 同时等待多个 Channel 等复杂情况。因此，源码结构可帮助理解阻塞和唤醒，但业务代码不应依赖 `hchan` 或假设某种实现一定保持不变。

## 7. Channel 常见设计

### 7.1 Worker Pool

一个有界任务 Channel 可以限制待处理任务数量，从而将压力传回生产者：

```go
func worker(ctx context.Context, jobs <-chan Job, results chan<- Result) {
    for {
        select {
        case <-ctx.Done():
            return
        case job, ok := <-jobs:
            if !ok {
                return
            }
            result := process(job)
            select {
            case results <- result:
            case <-ctx.Done():
                return
            }
        }
    }
}
```

要进一步保证系统可退出，还需要由上层协调取消、停止生产、关闭任务 Channel、等待 worker 结束，并确保结果 Channel 的关闭者不会与仍在发送的 worker 竞争。

### 7.2 Pipeline 与错误传播

流水线中的每个阶段通常从输入接收、处理后向输出发送。阶段必须定义清楚：谁关闭输出、消费者提前退出时上游如何停止、错误如何传回调用方。仅仅创建 Channel 并启动 goroutine，不会自动解决 goroutine 泄漏；如果下游提前返回，上游可能一直阻塞在发送上。将 `context.Context` 纳入发送和接收的 select 通常是实用的取消路径。

## 8. 常见误区

1. **把缓冲区当作同步保障。** 有缓冲 Channel 的发送可能早于接收发生；若需要明确的阶段同步，考虑无缓冲 Channel 或其他同步原语。
2. **忘记关闭。** `for range ch` 会一直等待，除非 Channel 被关闭或接收方被其他机制取消。
3. **从多个 goroutine 同时关闭。** 关闭权应有清晰的单一所有者，必要时使用 `sync.Once` 保护关闭动作，但这仍不能替代“所有发送已经结束”的生命周期设计。
4. **把 `len(ch)` 当作检查后再操作的依据。** 长度只反映瞬间缓冲元素数，不是并发条件判断；检查后状态可能立即改变。
5. **把 Channel 当成通用锁。** Channel 适合表达值传递、所有权转移、任务队列和同步事件；对保护共享状态，`sync.Mutex` 往往更直接。
6. **误解 `select` 的公平性。** 多个就绪分支没有固定优先级，也不承诺业务层面的公平调度。

## 9. 小结

Channel 的核心价值不仅是“队列”，更是通过发送、接收与关闭明确表达并发协议。无缓冲通信提供同步，有缓冲通信提供有界排队和背压，单向类型帮助表达职责，`select` 和取消信号则让生命周期可被组合。理解阻塞条件、关闭所有权和退出路径，比记住运行时结构字段更重要。

## 10. 参考资料

- [The Go Programming Language Specification：Channel types](https://go.dev/ref/spec#Channel_types)、[Send statements](https://go.dev/ref/spec#Send_statements)、[Receive operator](https://go.dev/ref/spec#Receive_operator)、[Select statements](https://go.dev/ref/spec#Select_statements)、[Close](https://go.dev/ref/spec#Close)
- [The Go Memory Model](https://go.dev/ref/mem)
- [Go 1.25.12 runtime/chan.go](https://github.com/golang/go/blob/go1.25.12/src/runtime/chan.go)
- [Go blog: Go Concurrency Patterns: Pipelines and cancellation](https://go.dev/blog/pipelines)
