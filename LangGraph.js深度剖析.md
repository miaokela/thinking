# LangGraph.js 核心功能深度剖析

> 分析对象：`@langchain/langgraph` 核心库（`libs/langgraph-core`）
> 源码仓库：[langchain-ai/langgraphjs](https://github.com/langchain-ai/langgraphjs)
> 分析基于源码主干分支，核心位于 `libs/langgraph-core/src/`

---

## 目录

1. [架构总览：四层模型](#1-架构总览四层模型)
2. [Graph 层：工作流构建器](#2-graph-层工作流构建器)
   - [2.1 Graph 基类](#21-graph-基类)
   - [2.2 StateGraph](#22-stategraph)
   - [2.3 Annotation 状态定义系统](#23-annotation-状态定义系统)
   - [2.4 compile()：从构建器到执行引擎](#24-compile从构建器到执行引擎)
3. [Pregel 层：核心执行引擎](#3-pregel-层核心执行引擎)
   - [3.1 Pregel 类概述](#31-pregel-类概述)
   - [3.2 PregelNode：计算单元](#32-pregelnode计算单元)
   - [3.3 PregelLoop：超级步编排器](#33-pregelloop超级步编排器)
   - [3.4 PregelRunner：并发任务执行器](#34-pregelrunner并发任务执行器)
   - [3.5 Algorithm Functions：核心算法函数](#35-algorithm-functions核心算法函数)
4. [Channels 层：状态管理](#4-channels-层状态管理)
5. [Checkpointer 层：持久化](#5-checkpointer-层持久化)
6. [完整调用链分析](#6-完整调用链分析)
   - [6.1 构建阶段](#61-构建阶段)
   - [6.2 执行阶段](#62-执行阶段)
   - [6.3 中断与恢复](#63-中断与恢复)
7. [关键设计解析](#7-关键设计解析)
   - [7.1 为什么是 BSP（Bulk Synchronous Parallel）？](#71-为什么是-bspbulk-synchronous-parallel)
   - [7.2 节点间通信机制](#72-节点间通信机制)
   - [7.3 版本控制系统](#73-版本控制系统)
   - [7.4 流式输出机制](#74-流式输出机制)
8. [总结](#8-总结)

---

## 1. 架构总览：四层模型

LangGraph.js 的架构分为四个清晰的层次，下层为上层提供基础能力：

```
┌─────────────────────────────────────────────────────────┐
│  Graph Layer（工作流定义层）                              │
│  Graph / StateGraph / CompiledStateGraph / Annotation    │
│  职责：节点/边定义、状态类型声明、编译为可执行图          │
├─────────────────────────────────────────────────────────┤
│  Pregel Layer（核心执行引擎）                             │
│  Pregel / PregelNode / PregelLoop / PregelRunner         │
│  职责：超级步调度、并行任务执行、消息传递、流式输出       │
├─────────────────────────────────────────────────────────┤
│  Checkpointer Layer（持久化层）                           │
│  BaseCheckpointSaver / BaseStore / MemorySaver           │
│  职责：状态快照、恢复、时间旅行、断点续传                 │
├─────────────────────────────────────────────────────────┤
│  Channels Layer（状态管理基础）                           │
│  BaseChannel / EphemeralValue / LastValue / Topic        │
│  职责：状态存储、更新协议、序列化                         │
└─────────────────────────────────────────────────────────┘
```

**核心思想**：四层各司其职，上层依赖下层。用户接触最多的是 Graph 层，但理解 Pregel 层才是掌握 LangGraph 的关键。

---

## 2. Graph 层：工作流构建器

### 2.1 Graph 基类

**文件**：`src/graph/graph.ts`

`Graph` 是所有图构建器的基类，维护了图结构的最基本要素：

- `nodes: Record<N, NodeSpecType>` —— 节点映射
- `edges: Set<[N | typeof START, N | typeof END]>` —— 边集合
- `branches: Record<string, Record<string, Branch>>` —— 条件分支

核心方法：

```typescript
addNode(key, action, options?)  // 添加节点
addEdge(startKey, endKey)       // 添加边
addConditionalEdges(source, condition, pathMap?)  // 添加条件边
compile()                       // 编译为 CompiledGraph 返回
```

`CompiledGraph` 继承自 `Pregel`，是编译后的可执行运行时：

```typescript
class CompiledGraph<N, State, ...> extends Pregel<...> {
    builder: Graph;  // 持有对原始构建器的引用
    attachNode(key, node): void;
    attachEdge(start, end): void;
    attachBranch(start, name, branch): void;
}
```

### 2.2 StateGraph

**文件**：`src/graph/state.ts`

`StateGraph` 是 LangGraph 的主力构建器，继承自 `Graph`，增加了**共享状态管理**能力。

**类型参数**（共 11 个，非常复杂）：

```typescript
class StateGraph<
    SD,     // 状态定义（StateDefinition）
    S,      // 状态类型
    U,      // 更新类型
    N,      // 节点名称联合类型
    I,      // 输入定义
    O,      // 输出定义
    C,      // 上下文定义
    NodeReturnType,
    InterruptType,
    WriterType,
> extends Graph<N, S, U, StateGraphNodeSpec<S, U>, ToStateDefinition<C>>
```

**核心实例字段**：

```typescript
channels: Record<string, BaseChannel>;    // 通道映射
waitingEdges: Set<[N[], N]>;               // 等待边（多对一）
_schemaDefinition: StateDefinition;         // 状态定义
_schemaDefinitions: Map<...>;               // 多种状态定义（state/input/output/context）
```

**关键方法链**：

```typescript
// 1. 构造
new StateGraph(StateAnnotation, { input, output, context })
// 或遗留方式
new StateGraph({ channels: {...} })

// 2. 添加节点
.addNode("nodeName", action, {
    retryPolicy?: RetryPolicy,
    cachePolicy?: CachePolicy,
    timeout?: number,
    metadata?: Record<string, unknown>,
    subgraphs?: Pregel[],
    ends?: string[],          // 自动创建边
    errorHandler?: (state, error) => Update,
    input?: Schema,           // 节点输入模式
})

// 3. 添加边
.addEdge(START, "nodeA")      // 单向边
.addEdge(["nodeA", "nodeB"], "nodeC")  // 多对一

// 4. 条件边
.addConditionalEdges("nodeA", (state) => {
    return state.nextStep;  // 返回目标节点名
})

// 5. 批量添加节点（自动连线）
.addSequence([
    ["step1", fn1],
    ["step2", fn2],
    ["step3", fn3],
])

// 6. 设置全局节点默认策略
.setNodeDefaults({
    retryPolicy: { maxAttempts: 3 },
    cachePolicy: { ttl: 60 },
    timeout: 60_000,
    errorHandler: (state, { node, error }) => ({ lastError: error.message }),
})

// 7. 编译
.compile({ checkpointer, store, cache, interruptBefore, interruptAfter, ... })
```

**构造函数输入规范化的流程** (`_normalizeToStateGraphInit`)：

```
StateGraphInit 格式 → 直接使用
Annotation.Root / StateSchema / Zod 格式 → 提取 StateDefinition
遗留 { channels } 格式 → 转换
```

### 2.3 Annotation 状态定义系统

**文件**：`src/graph/annotation.ts`

`Annotation` 是定义图状态的声明式 API：

```typescript
const StateAnnotation = Annotation.Root({
    messages: Annotation<BaseMessage[]>({
        reducer: (x, y) => x.concat(y),  // 状态合并函数
        default: () => [],                 // 默认值
    }),
    step: Annotation<number>({
        reducer: (a, b) => a + b,
        default: () => 0,
    }),
});

// 使用
const graph = new StateGraph(StateAnnotation);
```

每个 `Annotation` 定义了一个**通道工厂**。编译时，`Annotation` 被转换为 `BaseChannel` 实例，通道的 `reducer` 函数决定如何合并来自不同节点的写入。

### 2.4 compile()：从构建器到执行引擎

`compile()` 是 StateGraph 的**核心转换方法**，它将构建器模式下的图定义转换为可执行的 Pregel 实例。

**compile 内部流程**：

```
1. 验证图结构（validate()）
   - 检查所有节点是否可达
   - 检查中断点是否有效

2. 准备输出通道
   - 根据输出定义确定 outputChannels
   - 确定 streamChannels

3. 创建 Pregel 实例
   - 创建 CompiledStateGraph（继承 CompiledGraph → Pregel）
   - 设置 channels / inputChannels / outputChannels / streamChannels
   - 设置 interruptBefore / interruptAfter
   - 设置 checkpointer / store / cache

4. 附着节点
   - 遍历所有节点，转换为 PregelNode
   - 每个 PregelNode 包含：ChannelRead + 用户逻辑 + ChannelWrite

5. 附着边
   - 将边注册为触发关系
   - 将条件边注册为 Branch

6. 解析全局节点默认策略
   - 合并 setNodeDefaults 到每个节点
   - 每个编译调用独立复制，保证稳定性

7. 返回 CompiledStateGraph
```

---

## 3. Pregel 层：核心执行引擎

### 3.1 Pregel 类概述

**文件**：`src/pregel/index.ts`

`Pregel` 是 LangGraph 的**核心运行时引擎**，实现了基于 Google Pregel 论文的**消息传递图计算模型**（Bulk Synchronous Parallel）。

```typescript
class Pregel<Nodes, Channels, ContextType, InputType, OutputType, ...>
    extends PartialRunnable<...>  // Runnable 的子类（通过中间层解决类型覆盖问题）
    implements PregelInterface<...>
```

**关键特性**：
- 消息传递：节点通过 Channel 通信，从不直接通信
- 超级步：离散的执行阶段，节点在超级步内并行执行
- 持久化：通过 Checkpointer 实现状态快照和恢复
- 流式输出：支持 values / updates / messages / debug 多种模式
- 并发执行：超级步内的节点可并行运行
- Human-in-the-loop：通过中断机制允许人工介入

**核心字段**：

```typescript
nodes: Nodes;                                        // 节点映射
channels: Channels;                                  // 通道映射
inputChannels: keyof Channels | Array<keyof Channels>;  // 输入通道
outputChannels: keyof Channels | Array<keyof Channels>; // 输出通道
streamChannels: string | string[];                   // 流通道
streamMode: StreamMode;                              // 流模式
checkpointer?: BaseCheckpointSaver;                  // 检查点器
store?: BaseStore;                                   // 持久化存储
```

**invoke() 内部流程**：

```
1. 验证输入
2. 创建流（duplex stream）
3. 初始化 PregelLoop
4. 创建 PregelRunner
5. 主循环：
   while (await loop.tick({ inputKeys })) {
       await runner.tick({ timeout, retryPolicy, signal });
   }
6. 收集最终结果
7. 处理中断
8. 返回输出
```

### 3.2 PregelNode：计算单元

**文件**：`src/pregel/read.ts`

`PregelNode` 是 Pregel 中的计算单元，继承自 `RunnableBinding`：

```typescript
class PregelNode<RunInput, RunOutput> extends RunnableBinding<...> {
    channels: Record<string, string> | string[];  // 订阅的通道
    triggers: string[];                           // 触发条件
    mapper?: (args) => any;                       // 输入映射
    writers: Runnable[];                          // 写入器
    bound: Runnable;                              // 绑定的可运行对象
    retryPolicy?: RetryPolicy;
    cachePolicy?: CachePolicy;
    timeout?: TimeoutPolicy;
    subgraphs?: Runnable[];                       // 子图
    ends?: string[];                              // 自动结束边
    isErrorHandler?: boolean;
    errorHandlerNode?: string;
}
```

**PregelNode 的组成**：

```
ChannelRead  →  Mapper  →  [Bound Runnable 或 用户逻辑]  →  ChannelWrite
```

每个节点执行时：
1. 从订阅的通道读取数据（`ChannelRead`）
2. 通过 `mapper` 映射输入
3. 执行用户定义的逻辑（可能是函数、Runnable、或子图）
4. 通过 `ChannelWrite` 将结果写入目标通道

### 3.3 PregelLoop：超级步编排器

**文件**：`src/pregel/loop.ts`

`PregelLoop` 管理整个图执行的生命周期，负责超级步的编排。

**构造时初始化**：

```typescript
class PregelLoop {
    input: symbol;           // INPUT_DONE / INPUT_RESUMING
    checkpointer?: BaseCheckpointSaver;
    checkpoint: Checkpoint;
    channels: Record<string, BaseChannel>;
    step: number;            // 当前超级步
    stop: number;            // 最大步数限制
    tasks: Record<string, PregelExecutableTask>;  // 当前步的任务
    nodes: Record<string, PregelNode>;
    stream: [...];           // 流输出
    interruptBefore: string[];
    interruptAfter: string[];
    // ...
}
```

**tick() 方法**：执行一个超级步

```typescript
async tick(params: { inputKeys?: string | string[] }): Promise<boolean> {
    // 1. 首次执行则初始化通道（_first）
    if (this.input 不是 INPUT_DONE/INPUT_RESUMING) {
        await this._first(inputKeys);
    }
    // 2. 检查是否有中断要触发
    else if (this.toInterrupt.length > 0) {
        this.status = "interrupt_before";
        throw new GraphInterrupt();
    }
    // 3. 准备下一批任务（_prepareNextTasks）
    else {
        this.tasks = _prepareNextTasks(
            this.checkpointPendingWrites,
            this.nodes, this.channels,
            this.config, true, { ... }
        );
    }
    // 4. 创建检查点
    await this._putCheckpoint();
    // 5. 返回是否有更多任务
    return Object.keys(this.tasks).length > 0;
}
```

### 3.4 PregelRunner：并发任务执行器

**文件**：`src/pregel/runner.ts`

`PregelRunner` 负责执行 `PregelLoop` 当前步的所有任务。

```typescript
class PregelRunner {
    loop: PregelLoop;
    handledExceptions: WeakSet<Error>;

    async tick(options: TickOptions = {}): Promise<void> {
        // 1. 初始化中止信号链
        const { signals, disposeCombinedSignal } = this._initializeAbortSignals({...});

        // 2. 并发执行任务（带重试）
        const taskStream = this._executeTasksWithRetry(pendingTasks, {
            signals, retryPolicy, maxConcurrency,
        });

        // 3. 处理每个任务的结果
        for await (const { task, error, signalAborted } of taskStream) {
            this._commit(task, error);  // 提交写入
            if (error !== undefined && this.handledExceptions.has(error)) {
                // 错误已由错误处理器处理，继续
            }
        }
    }
}
```

**执行细节**：

- **`_executeTasksWithRetry`**：使用 `Promise.allSettled` 或自定义并发控制来并行执行任务
- **`_commit`**：将任务结果转换为通道写入，通过 `loop.putWrites()` 保存
- **重试机制**：支持可配置的 `RetryPolicy`（最大重试次数、退避策略等）
- **错误处理**：节点级错误处理器（`errorHandler`）在重试耗尽后运行
- **超时控制**：`AbortSignal.timeout` 实现节点级超时

### 3.5 Algorithm Functions：核心算法函数

**文件**：`src/pregel/algo.ts`

这些纯函数实现了 Pregel 算法的核心逻辑：

```typescript
// _prepareNextTasks：根据通道更新准备下一批任务
function _prepareNextTasks(
    pendingWrites, nodes, channels, config, forExecution, extra
): Record<string, PregelExecutableTask> {
    // 遍历所有节点，检查哪些节点应该执行
    // 通过版本号比较决定（节点上次看到版本 vs 通道当前版本）
}

// _prepareSingleTask：准备单个任务
function _prepareSingleTask(
    taskPath, checkpoint, pendingWrites, processes, channels, config, ...
): PregelExecutableTask | undefined {
    // 创建可执行任务描述，包含输入、处理器、重试策略等
}

// _applyWrites：将任务写入应用到通道
function _applyWrites(
    channels, pendingWrites, ... 
): void {
    // 遍历待写入，更新通道值
    // 更新通道版本号
    // 处理 Send / Command 等特殊写入
}
```

---

## 4. Channels 层：状态管理

**文件**：`src/channels/base.ts`

`BaseChannel` 是所有通道的抽象基类：

```typescript
abstract class BaseChannel<ValueType, UpdateType, CheckpointType> {
    abstract fromCheckpoint(checkpoint?: CheckpointType): this;
    abstract update(values: UpdateType[]): boolean;  // 应用更新
    abstract get(): ValueType;                        // 读取当前值
    checkpoint(): CheckpointType | undefined;          // 创建快照
    consume(): boolean;                                // 消费更新
    finish(): boolean;                                 // 结束通知
    isAvailable(): boolean;                            // 是否可用
}
```

**通道类型**：

| 通道 | 行为 | 用途 |
|------|------|------|
| `EphemeralValue` | 非持久化，每次步后重置 | 临时中间结果 |
| `LastValue` | 只保留最新值 | 默认状态通道 |
| `BinaryOperatorAggregate` | 用二元操作符合并值 | 累加器（如消息列表追加） |
| `Topic` | 保留消息历史 | 消息队列场景 |
| `DynamicBarrierValue` | 条件满足时释放 | 等待所有前置节点 |
| `NamedBarrierValue` | 收集指定源的值 | 有条件的等待 |
| `LastValueAfterFinish` | 完成后保留最后值 | 输出通道 |

**通道通信模型**：

```
节点 A  ──write──→  Channel X  ──trigger──→ 节点 B
                    Channel X  ──trigger──→ 节点 C
```

节点**从不直接通信**，所有数据通过通道中介。这实现了松耦合和并行安全。

---

## 5. Checkpointer 层：持久化

**接口**：

```typescript
interface BaseCheckpointSaver {
    put(config, checkpoint, metadata): Promise<string>;  // 保存检查点
    get(config): Promise<Checkpoint | undefined>;         // 获取检查点
    getTuple(config): Promise<CheckpointTuple | undefined>;
    list(config, options?): AsyncIterable<CheckpointTuple>;
}

interface BaseStore {
    start(): Promise<void>;     // 批量存储运行
    put(namespace, key, value): Promise<void>;
    get(namespace, key): Promise<Value | undefined>;
    search(namespace, options?): AsyncIterable<Value>;
}
```

**实现**：
- `MemorySaver`：内存实现（默认）
- 插件：`SQLiteStore`、`PostgresStore`、`MongoDBStore`

**检查点创建时机**：每个超级步结束后自动创建，包含所有非临时通道的快照。

---

## 6. 完整调用链分析

### 6.1 构建阶段

```
用户代码
  │
  ├─ new StateGraph(Annotation)
  │    └─ _normalizeToStateGraphInit() → 统一为 StateGraphInit 格式
  │    └─ 解析通道定义 → 创建 channel 映射
  │
  ├─ .addNode("name", action, options?)
  │    └─ 创建 StateGraphNodeSpec { runnable, retryPolicy, cachePolicy, input, ... }
  │    └─ 注册到 this.nodes
  │
  ├─ .addEdge(START, "nodeA")
  │    └─ 验证节点存在
  │    └─ 添加到 this.edges
  │
  ├─ .addConditionalEdges("nodeA", router)
  │    └─ 创建 Branch 对象
  │    └─ 注册到 this.branches
  │
  └─ .compile({ checkpointer, ... })
       └─ validate() → 检查图结构完整性
       └─ 创建 CompiledStateGraph（extends Pregel）
       └─ 附着节点 → 转换为 PregelNode（ChannelRead + Mapper + Bound + ChannelWrite）
       └─ 附着边 → 注册触发关系
       └─ 附着条件边 → 注册 Branch
       └─ 合并全局默认策略
       └─ 返回 CompiledStateGraph
```

### 6.2 执行阶段

```
compiledGraph.invoke(input)
  │
  └─ Pregel.invoke()
       │
       ├─ 验证输入
       ├─ 创建 IterableReadableWritableStream（双工流）
       │
       ├─ PregelLoop.initialize()
       │    ├─ 从检查点恢复（如果存在）
       │    ├─ 初始化所有通道
       │    └─ 创建 PregelLoop 实例
       │
       ├─ new PregelRunner({ loop })
       │
       └─ 主循环：while (await loop.tick({ inputKeys }))
            │
            ├─ PregelLoop.tick()
            │    ├─ 首次 → _first()：初始化输入通道
            │    ├─ 检查中断 → throw GraphInterrupt
            │    └─ 正常 → _prepareNextTasks()
            │         ├─ 遍历所有节点
            │         ├─ 比较版本号（节点上次看到 vs 通道当前）
            │         ├─ 创建 PregelExecutableTask 列表
            │         └─ 更新检查点
            │
            ├─ PregelRunner.tick({ timeout, retryPolicy })
            │    ├─ _initializeAbortSignals()
            │    ├─ _executeTasksWithRetry(tasks)
            │    │    ├─ 并行执行任务（Promise.allSettled）
            │    │    ├─ 每个任务：
            │    │    │   ├─ 读取通道输入
            │    │    │   ├─ 执行用户逻辑
            │    │    │   └─ 写入通道
            │    │    └─ 重试逻辑（失败时重试）
            │    └─ _commit(task, error)
            │         ├─ 将结果转换为通道写入
            │         └─ loop.putWrites(taskId, writes)
            │
            └─ 处理流输出
                 ├─ values: 每次步后输出完整状态
                 ├─ updates: 只输出状态变化
                 ├─ messages: 内部节点通信
                 └─ debug: 详细执行事件

       └─ 收集最终结果
            ├─ 提取输出通道值
            └─ 返回输出
```

### 6.3 中断与恢复

```
中断流程：
  ┌─ PregelLoop.tick() 检测到中断点
  ├─ throw GraphInterrupt
  ├─ 检查点保存当前状态
  └─ 返回给调用方

恢复流程：
  ├─ 调用方从检查点恢复
  ├─ PregelLoop.initialize() 从检查点恢复通道
  ├─ loop.tick() 检测到是恢复执行
  └─ 继续正常执行循环
```

---

## 7. 关键设计解析

### 7.1 为什么是 BSP（Bulk Synchronous Parallel）？

Pregel 采用**整体同步并行**模型，原因：

1. **确定性**：每个超级步内所有节点完成后再同步，执行顺序可预测
2. **容错性**：超级步边界是天然检查点，故障时回退到上一个超级步
3. **简化并发**：超级步内无竞争条件，节点间通过通道通信而非共享内存
4. **可调试性**：每个超级步的状态可快照，支持时间旅行调试

### 7.2 节点间通信机制

```
节点从不直接通信

节点 A 返回 { messages: [newMsg] }
    ↓
ChannelWrite 将 { messages: [newMsg] } 写入 "messages" 通道
    ↓
通道的 reducer 将新消息追加到现有消息列表
    ↓
通道版本号更新
    ↓
下一个超级步，PRelLoop 检测到版本变化
    ↓
订阅了 "messages" 通道的节点 B 被调度执行
    ↓
节点 B 读取到更新后的消息列表
```

### 7.3 版本控制系统

Pregel 使用版本号来跟踪通道变化，决定哪些节点需要执行：

```
每个通道维护一个版本号（整数）
每个节点记录它上次看到的每个通道的版本号

当通道版本号 > 节点上次看到的版本号 → 节点应执行

版本号在每次通道更新时递增
```

**`_prepareNextTasks` 的核心逻辑**就是通过版本比较找出所有需要执行的节点，然后创建对应的任务。

### 7.4 流式输出机制

LangGraph 支持多种流模式：

| 模式 | 输出内容 | 用途 |
|------|---------|------|
| `values` | 每次超级步后的完整状态 | 查看完整的执行轨迹 |
| `updates` | 每次超级步后的状态变化 | 高效获取增量 |
| `messages` | 内部节点消息 | 监控 LLM 调用 |
| `debug` | 详细执行事件 | 调试和诊断 |
| `custom` | 通过转换器自定义 | 高级场景 |

**实现**：`PregelLoop` 内部维护一个 `stream` 数组，任务执行过程中通过 `_emit()` 方法将事件推入流，`Pregel.invoke()` 通过 `IterableReadableStream` 将流暴露给外部。

---

## 8. 总结

### 核心架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                       用户代码层                                     │
│  new StateGraph(Ann).addNode("a", fn).addEdge(...).compile()         │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ compile()
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   CompiledStateGraph (extends Pregel)                │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────────────┐  │
│  │   Nodes     │  │  Channels    │  │  Checkpointer / Store      │  │
│  │  PregelNode │  │  BaseChannel │  │  MemorySaver / SQLiteStore │  │
│  └─────────────┘  └──────────────┘  └────────────────────────────┘  │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ invoke(input)
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      执行循环（BSP）                                  │
│                                                                     │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────────────┐ │
│  │ PregelLoop   │────▶│ PregelRunner │────▶│ PregelLoop           │ │
│  │ .tick()      │     │ .tick()      │     │ .tick()              │ │
│  │ 准备任务      │     │ 并行执行任务   │     │ 准备下一批任务        │ │
│  └──────────────┘     └──────────────┘     └──────────────────────┘ │
│         │                    │                      │               │
│         ▼                    ▼                      ▼               │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────────────┐ │
│  │ 创建检查点    │     │ 提交写入      │     │ 流式输出              │ │
│  │ _putCheckpoint│    │ _commit      │     │ _emit()              │ │
│  └──────────────┘     └──────────────┘     └──────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

### 关键认知

1. **StateGraph 只是构建器**，`compile()` 才是真正的转换点，将声明式定义转换为命令式执行引擎
2. **Pregel 是核心**，理解 BSP 模型、Channel 通信、版本控制是理解 LangGraph 的关键
3. **节点从不直接通信**，所有数据通过 Channel 中介，这是并发安全的基础
4. **每个超级步都是确定性的**，超级步边界是天然检查点，支持容错和时间旅行
5. **中断机制**使 Human-in-the-loop 成为可能，是构建可控 Agent 工作流的基础