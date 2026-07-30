# Go Context 完全指南

## 一、Context 是什么

**一句话：Context 是一个"遥控器"，用来控制其他 goroutine 什么时候该停。**

```
你正在厨房做饭（goroutine 在干活）
    │
    ▼
你妈打电话说："别做了，外卖到了"（context 发出取消信号）
    │
    ▼
你关火，收拾，去吃饭（goroutine 收到信号，退出）
```

### 为什么需要 Context

想象一个场景：你写了一个 HTTP 服务，用户请求来了，你需要：
1. 查数据库
2. 调用外部 API
3. 处理数据

如果用户等不及了，点了"取消"或关闭浏览器，你的后台还在傻傻地查数据库、调 API，浪费资源。

**Context 就是解决这个问题的：让后台任务知道"甲方跑了，你也别干了"。**

---

## 二、Context 的四种创建方式

### 1. `context.Background()` 和 `context.TODO()`

```go
// Background：程序最顶层，永远不会被取消
ctx := context.Background()

// TODO：还不确定用哪个 context 时，先占个位
ctx := context.TODO()
```

**类比：**
- `Background()` = 电源插座，一直有电
- `TODO()` = 临时用个充电宝，以后再换

### 2. `WithCancel` —— 手动取消

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())

    go doWork(ctx)

    time.Sleep(2 * time.Second)
    cancel() // 手动按下"停止"按钮
}

func doWork(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            fmt.Println("收到取消信号，停工！")
            fmt.Println("原因：", ctx.Err()) // context canceled
            return
        default:
            fmt.Println("工作中...")
            time.Sleep(500 * time.Millisecond)
        }
    }
}
```

**类比：** 你有多个员工在干活，你可以随时说"停"，所有人同时停工。

### 3. `WithTimeout` —— 自动超时

```go
func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel() // ⭐ 必须调用，防止内存泄漏

    select {
    case <-ctx.Done():
        fmt.Println("超时了！")
        fmt.Println("原因：", ctx.Err()) // context deadline exceeded
    }
}
```

**类比：** 给员工一个任务："给你 3 小时，做不完就别做了。" 到时间自动停工。

### 4. `WithValue` —— 传递信息

```go
type contextKey string
const traceIDKey contextKey = "traceID"

func main() {
    // 放入信息
    ctx := context.WithValue(context.Background(), traceIDKey, "abc-123")

    go doWork(ctx)
}

func doWork(ctx context.Context) {
    // 取出信息
    if traceID, ok := ctx.Value(traceIDKey).(string); ok {
        fmt.Println("追踪ID:", traceID) // abc-123
    }
}
```

**类比：** 在工单上写备注："这个订单是 VIP 客户"，下游的人能看到。

---

## 三、Context 的内部原理

### 核心：就是定时器 + 关闭 Channel

```
WithTimeout(ctx, 3s) 做了什么？
    │
    ├─ 1. 创建一个 channel（通知渠道）
    ├─ 2. 启动一个定时器（time.AfterFunc）
    └─ 3. 定时器到期 → close(channel)
            │
            ▼
    所有监听这个 channel 的 goroutine 立即收到信号
```

### 用代码描述

```go
// 简化版原理
func WithTimeout(parent context.Context, timeout time.Duration) {
    ch := make(chan struct{})

    // 启动定时器
    time.AfterFunc(timeout, func() {
        close(ch) // ⭐ 关键：关闭 channel
    })

    // 返回一个 Done() 方法返回这个 channel 的 context
    return &myContext{done: ch}
}
```

### 为什么用 Channel

```
goroutine A:  select { case <-ch: ... }    等待中...
                                          │
                              time.AfterFunc 触发
                                          │
                                          ▼
                              close(ch)
                                          │
                                          ▼
goroutine A:  ← 立即被唤醒，执行后续逻辑
```

**Channel 被 close 后，所有阻塞在 `<-ch` 上的 goroutine 都会立即收到零值。** 这就是信号传递的底层机制。

---

## 四、Context 的工作流程

### 完整流程图

```
┌─────────────────────────────────────────────────────────────┐
│                    Context 生命周期                          │
│                                                             │
│  1. 创建                                                    │
│     ctx, cancel := context.WithTimeout(parent, 3s)         │
│                                                             │
│  2. 传递                                                    │
│     go doWork(ctx)  // 传给子 goroutine                      │
│     db.QueryContext(ctx, ...) // 传给数据库                   │
│                                                             │
│  3. 检查                                                    │
│     select {                                                │
│     case <-ctx.Done():  // 收到信号了吗？                     │
│     case result := <-ch: // 任务完成了吗？                    │
│     }                                                       │
│                                                             │
│  4. 触发（二选一）                                           │
│     a) 超时到期 → time.AfterFunc 触发 → close(done)          │
│     b) 手动调用 cancel() → close(done)                      │
│                                                             │
│  5. 响应                                                    │
│     ctx.Err() 返回：                                         │
│       - context.Canceled     → 手动取消                      │
│       - context.DeadlineExceeded → 超时                      │
└─────────────────────────────────────────────────────────────┘
```

### 信号传递链路

```
Background()
    │
    ├── WithTimeout(3s) ──→ ctx1
    │       │
    │       ├── WithValue(key, val) ──→ ctx2
    │       │       │
    │       │       └── 子 goroutine 使用 ctx2
    │       │           ctx2.Done() = ctx1.Done() = ctx.Done()
    │       │           （同一条通知链路）
    │       │
    │       └── 子 goroutine 使用 ctx1
    │           3秒后全部收到取消信号
    │
    └── WithCancel() ──→ ctx3
            │
            └── 手动调用 cancel() → ctx3 的子 goroutine 收到信号
```

**核心原则：父 context 取消 → 所有子 context 一起取消。**

---

## 五、实战应用场景

### 场景 1：HTTP 请求超时

```go
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context() // 客户端断开时自动取消

    // 数据库查询，最多等 2 秒
    dbCtx, cancel := context.WithTimeout(ctx, 2*time.Second)
    defer cancel()

    user, err := db.QueryContext(dbCtx, "SELECT * FROM users WHERE id = ?", 1)
    if err != nil {
        if dbCtx.Err() == context.DeadlineExceeded {
            http.Error(w, "查询超时", 504)
        }
        return
    }

    json.NewEncoder(w).Encode(user)
}
```

### 场景 2：并发任务统一取消

```go
func fetchAll(urls []string) []Result {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    results := make([]Result, len(urls))

    var wg sync.WaitGroup
    for i, url := range urls {
        wg.Add(1)
        go func(i int, url string) {
            defer wg.Done()

            req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
            resp, err := http.DefaultClient.Do(req)

            // 如果 context 已取消，直接退出
            if ctx.Err() != nil {
                return
            }

            results[i] = Result{Resp: resp, Err: err}
        }(i, url)
    }

    wg.Wait()
    return results
}
```

### 场景 3：优雅退出服务

```go
func main() {
    // 监听 Ctrl+C
    ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt)
    defer stop()

    server := &http.Server{Addr: ":8080"}

    go server.ListenAndServe()

    <-ctx.Done() // 等待中断信号

    // 给服务器 10 秒时间处理完剩余请求
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    server.Shutdown(shutdownCtx)
    fmt.Println("服务器已关闭")
}
```

### 场景 4：限制并发数

```go
func processItems(ctx context.Context, items []Item) {
    // 最多同时 5 个 goroutine
    sem := make(chan struct{}, 5)

    for _, item := range items {
        select {
        case sem <- struct{}{}: // 获取信号量
        case <-ctx.Done():
            return // 超时或取消，不再处理
        }

        go func(item Item) {
            defer func() { <-sem }() // 释放信号量

            // 处理逻辑
            process(item)
        }(item)
    }
}
```

### 场景 5：取消传播（级联取消）

```go
func handleRequest(ctx context.Context) error {
    ctx, cancel := context.WithTimeout(ctx, 10*time.Second)
    defer cancel()

    // 三个子任务共享同一个超时
    eg, ctx := errgroup.WithContext(ctx)

    eg.Go(func() error {
        return step1(ctx) // 任一失败，其他全部取消
    })
    eg.Go(func() error {
        return step2(ctx)
    })
    eg.Go(func() error {
        return step3(ctx)
    })

    return eg.Wait()
}
```

---

## 六、常见陷阱

### 陷阱 1：忘记调用 cancel()

```go
// ❌ 内存泄漏！timer 和 channel 永远不被回收
func bad() {
    ctx, _ := context.WithTimeout(context.Background(), time.Second)
    doWork(ctx)
}

// ✅ 始终 defer cancel()
func good() {
    ctx, cancel := context.WithTimeout(context.Background(), time.Second)
    defer cancel()
    doWork(ctx)
}
```

### 陷阱 2：用 Value 传业务参数

```go
// ❌ 类型不安全，依赖 key 字符串不冲突
ctx = context.WithValue(ctx, "userID", 42)
ctx = context.WithValue(ctx, "userName", "张三")

// ✅ 业务参数用函数参数
func handleOrder(ctx context.Context, userID int, userName string) {
    // ...
}

// ✅ Value 只传请求级元数据
ctx = context.WithValue(ctx, traceIDKey, "abc-123")
```

### 陷阱 3：goroutine 泄漏

```go
// ❌ goroutine 永远阻塞
func leak() {
    ch := make(chan int)
    go func() {
        val := <-ch // 永远没人写入
        fmt.Println(val)
    }()
}

// ✅ 用 context 保护
func noLeak(ctx context.Context) {
    ch := make(chan int)
    go func() {
        select {
        case val := <-ch:
            fmt.Println(val)
        case <-ctx.Done():
            return // 可以退出
        }
    }()
}
```

---

## 七、总结

| 要点 | 说明 |
|---|---|
| 本质 | 定时器 + 关闭 Channel = 信号通知 |
| 核心方法 | `Done()` 返回 channel，`Err()` 返回取消原因 |
| 嵌套规则 | 父取消 → 子全部取消 |
| 最佳实践 | 始终 `defer cancel()` |
| 适用场景 | HTTP 请求、数据库查询、并发控制、优雅退出 |
| 不适用 | 纯计算、不需要取消的操作 |

> **Context 不做任何业务逻辑，它只是一个"信号传递员"。**
> 
> 你设置超时，它到时间告诉你；你手动取消，它通知所有人。
> 
> 简单、轻量、可靠。
