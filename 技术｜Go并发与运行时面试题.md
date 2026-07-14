# 技术｜Go并发与运行时面试题

> 面向 9 年+ Golang Web 后端业务岗。
> 本文重点覆盖 goroutine、channel、GMP、GC、内存逃逸、锁、pprof 和线上并发问题排查。

## 一、Go 并发与运行时 面试思想方法论篇：先建立结构化回答框架

> 本模块回答“怎么学、怎么答、怎么临场组织”。先建立资深后端视角，再进入能力、原理、实战和项目表达。

### 方法 0：复习地图

```text
Go 并发当前能力 -> 运行时核心原理 -> 线上排障 -> 项目表达 -> 面试方法论
```

### 方法 1：资深后端如何回答 Go 并发与运行时问题？

**结论先行：**
Go 并发题要从“并发模型、调度机制、同步方式、资源治理、故障排查”五层回答，不能只说 goroutine 很轻量。

**分层展开：**

**并发模型**
- goroutine 是用户态轻量线程。
- channel 用于通信和同步。
- context 控制生命周期。

**运行时调度**
- GMP 模型。
- work stealing。
- sysmon 监控和抢占。

**同步手段**
- channel。
- mutex/rwmutex。
- atomic。
- waitgroup/once/cond。

**资源治理**
- goroutine 数量要有上限。
- 外部调用要有超时。
- 防止泄漏、死锁和重试风暴。

**排查工具**
- race detector。
- pprof goroutine/block/mutex。
- runtime trace。

**面试追问：Go 的并发优势是什么？**
Go 把并发作为语言和运行时的一等能力，创建 goroutine 成本低，channel 和 context 让并发协作更标准化，适合 IO 密集型 Web 后端。

### 方法 2：Go 并发与运行时 题目通用决策框架

```text
题目问语言机制？
    先说用途 -> 底层结构 -> 边界坑点 -> 工程建议

题目问并发稳定性？
    控制并发 -> 传递取消 -> 处理错误 -> 防泄漏 -> 可观测

题目问项目实践？
    场景 -> 约束 -> 方案 -> 监控 -> 复盘
```

## 二、Go 并发与运行时 当前能力全景：核心语法、工程能力与边界

> 本模块回答“能做什么、怎么正确使用”。先掌握能力边界，再进入底层原理。

### 1. goroutine 为什么比线程轻？

**结论先行：**
goroutine 由 Go runtime 管理，初始栈小、按需增长、调度在用户态完成，因此比操作系统线程更轻。

**分层展开：**

**栈空间**
- goroutine 初始栈很小。
- 栈可以按需增长和收缩。
- 线程栈通常更大。

**调度成本**
- goroutine 调度由 runtime 管理。
- 减少频繁陷入内核。
- M:N 模型把大量 goroutine 映射到少量 OS 线程。

**创建成本**
- 创建 goroutine 成本低。
- 适合大量并发 IO。

**工程边界**
- goroutine 不是免费资源。
- 数量过多仍会增加内存、调度和 GC 压力。
- 必须设计退出条件。

**面试追问：goroutine 可以无限创建吗？**
不能。高并发服务要用 worker pool、信号量、限流和 context 控制并发，否则会导致内存上涨、调度开销变大和下游被打爆。

### 2. channel 的底层原理是什么？

**结论先行：**
channel 是 Go runtime 提供的并发同步队列，内部包含环形缓冲区、发送等待队列、接收等待队列和锁。

**分层展开：**

**无缓冲 channel**
- 发送和接收必须配对。
- 用于同步交接。
- 发送方和接收方直接传递数据。

**有缓冲 channel**
- 内部有环形队列。
- 缓冲未满时发送不阻塞。
- 缓冲非空时接收不阻塞。

**等待队列**
- 发送阻塞的 goroutine 进入 sendq。
- 接收阻塞的 goroutine 进入 recvq。
- 匹配后 runtime 唤醒对方。

**工程风险**
- 关闭已关闭 channel 会 panic。
- 向已关闭 channel 发送会 panic。
- 从关闭 channel 接收会得到零值和 ok=false。

**面试追问：channel 是无锁的吗？**
不是。channel 内部有锁来保护队列和状态，但 runtime 做了大量优化。channel 更重要的价值是表达同步语义。

### 3. select 有哪些特性？

**结论先行：**
select 用于同时等待多个 channel 操作，多个 case 同时就绪时会伪随机选择一个，default 会让 select 非阻塞。

**分层展开：**

**基本行为**
- 没有就绪 case 且无 default 时阻塞。
- 有 default 时立即执行 default。
- 多个 case 就绪时随机选择。

**典型用法**
- 监听 `ctx.Done()`。
- 实现超时。
- 多路复用 channel。
- 非阻塞发送或接收。

**风险**
- default 滥用可能导致 busy loop。
- nil channel 会永久阻塞，可用于动态关闭 case。
- select 不保证公平调度业务语义。

**面试追问：如何用 nil channel 控制 select？**
把某个 channel 设为 nil 后，对它的发送和接收永远阻塞，相当于动态禁用对应 case。

### 4. Mutex 和 RWMutex 怎么选？

**结论先行：**
临界区短、写入频繁时用 Mutex；读多写少且读操作确实能并发收益明显时用 RWMutex。

**分层展开：**

**Mutex**
- 简单直接。
- 写多或临界区短时开销小。
- 避免读写锁管理复杂度。

**RWMutex**
- 多个读锁可以并发。
- 写锁会阻塞读写。
- 读多写少才有收益。

**工程判断**
- 不要凭感觉使用 RWMutex。
- 用 mutex profile 或压测比较。
- 临界区内不要做 IO 或慢操作。

**面试追问：RWMutex 会不会饿死写者？**
Go 的 RWMutex 有写者优先倾向，写者等待后会阻止新读者持续进入，降低写饥饿风险。

#### 10.1 slice 和 map 的并发安全边界是什么？

**结论先行：**
slice 和普通 map 都不是“自带并发安全”的容器。多个 goroutine 只读且对象不再变化通常安全；只要读写或写写并发访问同一份状态，就需要同步。

**slice 的关键不是 header，而是底层数组：**

```go
a := make([]int, 10)
b := a[:5]

go func() { a[0] = 1 }()
go func() { _ = b[0] }()
```

`a` 和 `b` 是不同的 slice header，但共享底层数组，因此存在 data race。

并发 append 同一个 slice 更危险，因为 append 既可能复用原数组，也可能扩容并替换 data、len、cap。常用方案：

- 每个 goroutine 写独立结果，结束后由单 goroutine 合并。
- 预先确定长度，让不同 goroutine 写互不重叠的下标；必须确保没有并发 append，且每个内存位置只有一个写者。
- 共享 append 时使用 Mutex 保护整个“检查、append、更新 header”过程。
- 通过 channel 将结果交给单一聚合 goroutine。

**map 的边界：**

```text
并发只读，且此后没有任何写：可以
并发读写：不可以
并发写写：不可以
遍历与写入并发：不可以
```

map 的写操作包括新增、更新、`delete` 和 `clear`。问题可能表现为 data race，也可能触发：

```text
fatal error: concurrent map read and map write
fatal error: concurrent map writes
```

不能把“没有 panic”当作线程安全证明，应该用：

```bash
go test -race ./...
go test -race -run TestName -count=100 ./path/to/pkg
```

#### 10.2 并发 map 的四种治理方案怎么选？

##### 方案一：map + RWMutex

这是业务代码的通用默认方案，类型安全、语义清晰，并且可以维护跨多个字段的不变量。

```go
type UserCache struct {
	mu   sync.RWMutex
	data map[int64]User
}

func (c *UserCache) Get(id int64) (User, bool) {
	c.mu.RLock()
	defer c.mu.RUnlock()
	user, ok := c.data[id]
	return user, ok
}

func (c *UserCache) Set(id int64, user User) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.data[id] = user
}
```

注意事项：

- 锁应和被保护的数据放在同一结构体中。
- 不要返回内部 map，让调用方绕过锁修改。
- 临界区内避免网络、磁盘和不受控回调。
- “先读后写”的复合操作必须在同一次写锁中完成。

```go
func (c *UserCache) Increment(id int64) {
	c.mu.Lock()
	defer c.mu.Unlock()

	user := c.data[id]
	user.Score++
	c.data[id] = user
}
```

##### 方案二：sync.Map

`sync.Map` 主要针对两类场景优化：

- 某个 key 写入一次、读取很多次，例如只增长的缓存。
- 多个 goroutine 操作彼此不相交的 key。

```go
var cache sync.Map

cache.Store("campaign:1", campaign)
value, ok := cache.Load("campaign:1")
cache.Delete("campaign:1")
```

它不一定适合频繁覆盖、强类型要求、复杂复合操作或需要跨 key 一致性的业务。Go 1.20+ 支持 `CompareAndSwap`、`CompareAndDelete`，但不能因此把复杂状态机拆成一组缺少整体原子性的单步操作。

##### 方案三：分片 map

将 key 映射到固定 shard，每个 shard 独立加锁，可以降低单锁竞争：

```go
const shardCount = 64

type shard struct {
	mu   sync.RWMutex
	data map[string]Value
}

type ShardedMap struct {
	shards [shardCount]shard
}

func (m *ShardedMap) shardFor(key string) *shard {
	index := fnv32(key) % shardCount
	return &m.shards[index]
}
```

它适合经过压测确认存在锁竞争的高并发场景。代价是全量遍历、统计、扩缩容和跨 shard 原子操作更复杂，不能只因为“并发高”就提前引入。

##### 方案四：单 goroutine + channel

由一个 goroutine 独占 map，其他 goroutine 通过消息请求读写：

```text
调用方 -> command channel -> owner goroutine -> map
调用方 <- response channel <-------------------┘
```

适合状态变更天然需要串行化、有明确事件顺序的场景，例如任务状态机和房间状态。它不是无成本方案：所有操作会经过同一个 owner，慢处理会形成排队，查询也需要设计响应通道和退出协议。

**选型口诀：**

```text
通用业务状态              -> map + Mutex/RWMutex
写少读多或 key 基本不重叠 -> sync.Map
锁竞争已成为瓶颈          -> 分片 map
状态变更必须严格串行       -> 单 goroutine + channel
```

**面试追问：为什么不能只用 `len(m)` 判断后再写？**
“检查后执行”是两个步骤。即使单次读取和写入分别加锁，中间释放锁仍可能被其他 goroutine 修改；复合不变量必须放在同一个临界区内。

### 5. atomic 适合什么场景？

**结论先行：**
atomic 适合简单数值和指针状态的无锁读写，例如计数器、开关、状态标记；复杂不变量仍应使用锁。

**分层展开：**

**适合场景**
- QPS 计数。
- 配置开关。
- 熔断状态。
- 原子替换只读快照指针。

**不适合场景**
- 多字段一致更新。
- 复杂 map 读写。
- 需要条件判断和多步修改的业务状态。

**工程风险**
- 可读性差。
- 容易破坏内存可见性假设。
- ABA 问题。

**面试追问：atomic 一定比 mutex 快吗？**
不一定。低竞争下 mutex 很快，atomic 复杂逻辑容易出错。性能要靠 benchmark 证明。

### 6. WaitGroup 使用有哪些坑？

**结论先行：**
WaitGroup 用于等待一组 goroutine 结束，核心坑是 Add 时机错误、Done 次数不匹配、复用不当。

**分层展开：**

**正确用法**
- 启动 goroutine 前 Add。
- goroutine 内 defer Done。
- 主协程 Wait。

**常见错误**
- goroutine 内部 Add，可能 Wait 先返回。
- Done 多调用导致计数为负 panic。
- WaitGroup 复制后使用。
- 复用时上一轮 Wait 未结束就 Add。

**工程建议**
- Add 和启动 goroutine 放在一起。
- 封装启动函数降低误用。
- 对错误返回用 errgroup 更方便。

**面试追问：errgroup 比 WaitGroup 多了什么？**
errgroup 支持收集第一个错误，并可结合 context 在任一任务失败时取消其他任务。

### 7. sync.Once 有什么使用场景？

**结论先行：**
sync.Once 用于确保初始化逻辑只执行一次，适合单例、配置加载、延迟初始化等场景。

**分层展开：**

**适合场景**
- 单例客户端。
- 懒加载配置。
- 初始化全局资源。

**注意事项**
- Once 执行过后，即使函数内部失败，也不会自动重试。
- 初始化函数中 panic 后，再次调用行为要谨慎理解。
- 不要把业务可重试动作放进 Once。

**工程建议**
- 初始化失败需要显式保存 error。
- 需要重试时不要使用 Once 或自行设计状态机。

**面试追问：Once 能不能重置？**
标准 sync.Once 没有重置语义。需要重置说明场景可能不是 Once 应该解决的问题。

## 三、核心能力原理分析：为什么这样设计、代价是什么

> 本模块回答“为什么能做到、哪里会失效”。复杂原理要结合案例和线上问题理解。

### 8. GMP 模型是什么？

**结论先行：**
GMP 是 Go runtime 的调度模型：G 表示 goroutine，M 表示 OS 线程，P 表示处理器上下文，P 负责维护本地运行队列并连接 G 和 M。

**分层展开：**

**G**
- 表示一个 goroutine。
- 保存栈、状态、执行函数等信息。

**M**
- 表示操作系统线程。
- 真正执行代码的是 M。
- M 必须绑定 P 才能执行 Go 代码。

**P**
- 表示调度上下文。
- 数量由 `GOMAXPROCS` 控制。
- 持有本地 run queue、缓存等资源。

**调度流程**
- P 从本地队列取 G 给 M 执行。
- 本地队列为空时从全局队列或其他 P 偷取。
- G 阻塞系统调用时，P 可解绑给其他 M 继续执行。

**面试追问：GOMAXPROCS 控制什么？**
它控制同时执行 Go 代码的 P 的数量，通常近似决定 CPU 并行度，不等于 goroutine 数量。

### 9. Go 调度什么时候发生？

**结论先行：**
Go 调度会在 goroutine 阻塞、系统调用、channel 操作、函数调用安全点、GC 协作抢占等时机发生。

**分层展开：**

**主动让出**
- channel 阻塞。
- mutex 阻塞。
- sleep。
- 网络 IO 等待。

**系统调用**
- G 进入 syscall 后，M 可能阻塞。
- P 会被 runtime 解绑，交给其他 M。

**抢占调度**
- runtime 会避免某个 G 长时间霸占 CPU。
- 新版本 Go 支持更强的异步抢占。

**工程影响**
- 长时间纯 CPU 循环仍可能影响延迟。
- cgo 或阻塞 syscall 要谨慎。
- 热路径需要关注 P99。

**面试追问：一个 goroutine 死循环会不会卡死程序？**
现代 Go 有抢占调度，通常不会完全卡死，但 CPU 密集死循环仍会消耗核心，影响整体延迟和吞吐。

### 10. channel 适合替代锁吗？

**结论先行：**
channel 和锁解决的问题不同。channel 适合任务传递、事件通知和所有权转移；锁适合保护共享状态。

**分层展开：**

**适合 channel**
- producer-consumer。
- 超时和取消。
- fan-in/fan-out。
- 串行化状态变更。

**适合锁**
- 共享 map/cache。
- 短临界区。
- 读多写少状态。
- 性能敏感计数。

**判断原则**
- 如果关注“传递数据”，优先 channel。
- 如果关注“保护数据”，优先 mutex。
- 不要为了口号强行用 channel。

**面试追问：为什么说不要通过共享内存来通信？**
这是 Go 的并发哲学，强调通过 channel 传递所有权，减少共享状态。但工程上锁仍是必要工具，关键是选择合适模型。

### 11. channel 关闭原则是什么？

**结论先行：**
channel 应由发送方关闭，而且通常是唯一发送方关闭；接收方不应随意关闭 channel。

**分层展开：**

**原因**
- 关闭 channel 表示不会再发送数据。
- 接收方无法知道是否还有其他发送方。
- 多发送方关闭容易 panic。

**多发送方场景**
- 使用额外 done channel 通知退出。
- 使用 WaitGroup 等待所有发送方结束后由协调者关闭。
- 用 context 传播取消。

**工程规范**
- 明确 channel owner。
- 不用 close 表达所有事件。
- 关闭只用于广播结束或数据流结束。

**面试追问：如何判断 channel 已关闭？**
接收时使用 `v, ok := <-ch`，ok=false 表示 channel 已关闭且缓冲数据已读完。

### 12. Go GC 是怎么工作的？

**结论先行：**
Go GC 是并发三色标记清扫，目标是在控制停顿时间的同时回收不可达对象，核心成本来自堆大小、分配速度和对象图复杂度。

**分层展开：**

**基本流程**
- 标记准备阶段有短暂停顿。
- 并发标记可达对象。
- 标记终止阶段短暂停顿。
- 清扫不可达对象。

**三色标记**
- 白色：未标记对象。
- 灰色：已发现但子对象未扫描。
- 黑色：对象和子对象已扫描。

**写屏障**
- 并发标记期间程序仍在修改引用。
- 写屏障保证标记正确性。

**性能影响**
- 分配越快，GC 越频繁。
- 活对象越多，标记成本越高。
- 指针越多，对象图扫描越重。

**面试追问：如何降低 GC 压力？**
减少临时对象分配、复用 buffer、控制缓存对象数量、避免不必要指针、优化序列化和批量处理。

### 13. 什么是逃逸分析？

**结论先行：**
逃逸分析是编译器判断变量能否分配在栈上的过程；如果变量生命周期超出当前函数或无法证明安全，就会逃逸到堆上。

**分层展开：**

**常见逃逸原因**
- 返回局部变量指针。
- interface 装箱。
- 闭包捕获变量。
- 变量太大或动态大小。
- 编译器无法证明生命周期。

**影响**
- 堆分配增加。
- GC 压力增加。
- 热路径性能下降。

**排查方式**
- 使用 `go build -gcflags=-m`。
- benchmark 查看 allocs/op。
- pprof heap 分析热点。

**面试追问：返回局部变量指针安全吗？**
安全。Go 编译器会让该变量逃逸到堆上，保证生命周期正确。问题不是安全，而是可能增加堆分配。

### 14. sync.Pool 适合什么场景？

**结论先行：**
sync.Pool 适合复用临时对象、降低 GC 压力，例如 buffer、编码器、临时结构；不适合作为业务缓存。

**分层展开：**

**适合场景**
- 大量短生命周期对象。
- 临时 byte buffer。
- JSON 编解码辅助对象。
- 高 QPS 热路径。

**特性**
- Pool 中对象可能在 GC 后被清理。
- 不保证放进去就一定能取出来。
- 每个 P 有本地缓存，减少竞争。

**风险**
- 放入带状态对象前要 reset。
- 不要存连接、事务、业务状态。
- 复用大对象可能导致内存水位过高。

**面试追问：sync.Pool 为什么不能做连接池？**
因为 Pool 对对象生命周期没有强保证，GC 可清理对象，也不支持最大连接数、健康检查和关闭语义。

### Go 内存分配器 mcache、mcentral、mheap 怎么理解？

**结论先行：**
Go 内存分配器的核心目标是降低锁竞争和系统调用成本。小对象优先从当前 P 绑定的 `mcache` 分配，`mcache` 不够再找 `mcentral`，更大粒度由 `mheap` 向操作系统申请。

```text
goroutine
   |
   v
P 本地 mcache
   |
   | 不够
   v
mcentral 按 size class 管理 span
   |
   | 不够
   v
mheap 管理全局 heap / span
   |
   | 不够
   v
操作系统 mmap
```

**核心概念：**

- `mcache`：每个 P 的本地缓存，分配小对象时通常不需要加全局锁。
- `mcentral`：按 size class 管理 span，给各个 mcache 补货。
- `mheap`：全局堆，负责更大粒度 span 管理和向 OS 申请内存。
- size class：小对象会按大小规格归类，减少碎片。
- tiny allocator：很小且不含指针的对象可能被合并分配，降低分配成本。

**大对象分配：**
超过一定大小的大对象通常绕过小对象 size class，直接从 heap 分配，成本更高，也更容易增加 GC 压力。

**面试追问：Go 为什么分配快？**
不是因为没有成本，而是通过 P 本地缓存、分级分配、size class、批量申请 span 减少锁竞争和系统调用。高频路径仍要关注 `allocs/op`，因为分配快不等于可以无节制分配。

**工程落点：**

- 热路径减少临时对象。
- byte buffer、临时结构可以用 `sync.Pool`，但要 reset。
- 避免把大对象挂在全局 map、长生命周期 goroutine 或小 slice 上。
- 用 benchmark 的 `allocs/op` 和 heap profile 验证优化。

### GC 调优、GOGC、GOMEMLIMIT 与线上指标

**结论先行：**
Go GC 调优不是手动回收对象，而是控制分配速度、活对象规模和内存上限。面试里要能把 GC 原理、指标和排障路径串起来。

**关键参数：**

- `GOGC`：控制下一次 GC 的堆增长目标，默认 100，表示活对象翻倍附近触发下一轮 GC。
- `GOMEMLIMIT`：软内存上限，适合容器环境，帮助 runtime 在接近限制前更积极 GC。
- `SetGCPercent`：运行时调整 GOGC。
- `SetMemoryLimit`：运行时调整内存软上限。

**关键指标：**

- heap alloc：当前已分配且仍在使用的堆。
- heap inuse：runtime 向 OS 申请并正在使用的页。
- heap objects：堆对象数量。
- GC pause：STW 停顿时间。
- GC CPU fraction：GC 消耗的 CPU 比例。
- next GC / heap goal：下一轮 GC 目标。

**排查路径：**

1. 看 RSS、heap alloc、heap inuse 是否一起涨。
2. 看 goroutine 数是否上涨，排除 goroutine 泄漏。
3. 抓 heap profile，对比两个时间点。
4. 看 alloc_space 找分配热点，看 inuse_space 找存活对象。
5. 看 GC 次数、pause、CPU fraction，判断是分配太快还是活对象太多。

**面试追问：调低 GOGC 一定能解决内存问题吗？**
不一定。调低 GOGC 会更频繁 GC，可能降低堆水位，但会增加 CPU 成本。如果是业务缓存无限增长、goroutine 泄漏或全局引用持有大对象，调 GC 参数只能缓解，不能根治。

**项目表达：**
线上内存问题我会先区分“真实泄漏、缓存增长、短期分配峰值、GC 尚未归还 OS、goroutine 泄漏”。不会一上来就改 GC 参数，而是先用指标和 pprof 定位对象来源。

## 四、Go 并发与运行时 实战设计：会用、用好、知道边界

> 本模块回答“怎么落到工程设计里”。重点是有界并发、可取消任务、错误传播和优雅退出。

### 实战总览：Go 并发设计怎么落地？

**结论先行：**
Go 并发不是 goroutine 开得越多越好，而是要控制并发数量、传递取消信号、汇总错误、避免泄漏，并通过 pprof 和日志指标验证运行状态。

**常用组合：**
- `errgroup + SetLimit`：批量任务并发控制和错误传播。
- `context`：取消信号和超时预算传递。
- `channel/semaphore`：限制同时执行数量。
- `sync.WaitGroup`：等待固定任务集合结束。
- `pprof`：排查泄漏、CPU、内存和阻塞。

## 五、线上治理与故障排查：发现问题、定位问题、解决问题

> 本模块回答“线上出问题怎么办”。重点是 pprof、泄漏、CPU、内存、优雅停机和 race。

### 15. goroutine 泄漏是什么？怎么排查？

**结论先行：**
goroutine 泄漏是 goroutine 因阻塞或缺少退出条件长期不退出，导致内存、调度和资源持续增长。

**分层展开：**

**常见原因**
- channel 读写无人接收。
- select 没有监听 context。
- 定时器或 ticker 未停止。
- 下游请求无超时。
- worker 等待任务但任务源已退出。

**排查方式**
- 观察 goroutine 数量趋势。
- pprof goroutine 查看阻塞栈。
- trace 分析调度和阻塞。
- 压测后看资源是否回落。

**治理方式**
- 所有后台 goroutine 都有退出路径。
- 使用 context 取消。
- 关闭 channel 要有明确 owner。
- 定时器及时 stop。

**面试追问：如何设计一个不会泄漏的 worker pool？**
任务队列有关闭机制，worker select 同时监听任务和 context，WaitGroup 等待退出，提交方在关闭后不能继续写入任务。

### 16. pprof 常用哪些 profile？

**结论先行：**
pprof 常用 CPU、heap、goroutine、block、mutex，用来定位 CPU 热点、内存分配、goroutine 泄漏、阻塞和锁竞争。

**分层展开：**

**CPU profile**
- 看函数 CPU 消耗。
- 定位计算热点、序列化热点。

**heap profile**
- 看内存分配和存活对象。
- 定位大对象和频繁分配。

**goroutine profile**
- 看 goroutine 数量和阻塞栈。
- 排查泄漏。

**block profile**
- 看 channel、select、锁等阻塞。

**mutex profile**
- 看锁竞争热点。

**面试追问：线上能直接开 pprof 吗？**
可以但要做好权限、采样、限流和隔离，避免暴露敏感信息或采样本身影响服务。

### 17. Go 服务 CPU 飙高怎么排查？

**结论先行：**
CPU 飙高先判断是业务流量上涨、计算热点、GC、锁竞争、序列化、日志过量还是死循环，再用 pprof 和指标定位。

**分层展开：**

**快速止血**
- 限流降级。
- 扩容实例。
- 关闭非核心耗 CPU 功能。

**定位路径**
- 看 QPS、P99、错误率。
- 抓 CPU profile。
- 看 GC 指标。
- 看 goroutine 数量。
- 查最近发布变更。

**常见原因**
- JSON/Proto 序列化过重。
- 正则或复杂算法。
- 热点锁自旋。
- 日志同步写。
- 重试风暴。

**面试追问：CPU 高但 QPS 没涨可能是什么？**
可能是死循环、锁竞争、GC 压力、下游异常导致重试、日志暴增或新版本引入了计算热点。

### 18. Go 服务内存持续上涨怎么排查？

**结论先行：**
内存上涨要区分真实泄漏、缓存增长、goroutine 泄漏、对象分配变多和 GC 未及时回收。

**分层展开：**

**观察指标**
- RSS。
- heap inuse。
- heap alloc。
- goroutine 数量。
- GC 次数和暂停。

**定位方式**
- heap profile 对比两个时间点。
- goroutine profile 看泄漏。
- 检查本地缓存、全局 map、slice 持有大数组。
- 查是否有请求堆积。

**治理**
- 控制缓存容量。
- 修复 goroutine 泄漏。
- 大对象及时释放引用。
- 优化分配热点。

**面试追问：Go 内存下降慢一定是泄漏吗？**
不一定。Go runtime 不一定立即把内存还给操作系统，要结合 heap 存活对象、GC 后水位和业务缓存判断。

### 19. Go 服务如何做优雅停机？

## 六、项目实战表达：把 Go 并发与运行时讲成项目能力

> 本模块用于项目深挖，把语言和运行时能力落到业务复杂度、稳定性和工程收益上。

### 20. Go 并发在你的项目里有哪些真实场景？

**推荐回答：**

我项目里的 Go 并发主要不是为了“炫 goroutine”，而是服务于几个具体场景：

- 广告素材批量同步：控制并发调用媒体 API，避免触发平台限流。
- ES fallback 补偿：分页扫描未同步数据，批量 upsert/delete。
- 360 设备事件消费：消费者并发处理消息，但要保护下游 DB 和推送服务。
- 知乎统一发布服务：发布前、发布中、发布后有部分非阻塞插件异步执行。
- 定时任务：Cron 多实例部署下通过 Redis 锁保证单实例执行，任务内部再用 worker pool 控制并发。

**总结句：**
我会优先考虑可控并发，包括 context 取消、worker 数量、错误收敛、超时、限流和监控，而不是无限制启动 goroutine。

### 21. 如何排查 goroutine 泄漏？

**典型现象：**
- goroutine 数持续上涨不回落。
- 内存持续上涨。
- 服务 P99 抖动。
- 发布后一段时间才出现问题。

**排查步骤：**
1. 看监控里的 goroutine 数量趋势。
2. 通过 pprof 抓 goroutine profile。
3. 看大量 goroutine 卡在哪些栈上：channel send/recv、锁、网络 IO、time.After、context 未取消。
4. 对照最近发布的异步逻辑、定时任务、消费逻辑。
5. 修复后压测或灰度观察 goroutine 是否稳定。

**常见原因：**
- 生产者退出但消费者还在等 channel。
- 消费者退出但生产者阻塞发送。
- context 没有 cancel。
- for 循环里创建 goroutine 但没有并发上限。
- 使用 `time.After` 在高频循环里造成资源堆积。

### 22. 如何设计可取消的批量任务？

**场景：**
广告创建、素材同步、ES 补偿都可能批量处理几百到几千条任务。如果用户取消、服务停机、上游超时，任务应该能尽快停止。

**核心设计：**
- 函数入参必须带 `context.Context`。
- worker 内部每次取任务、调用外部接口、写 DB 前都检查 context。
- 外部 HTTP/RPC 调用设置超时。
- 主协程负责关闭 job channel。
- worker 退出后通过 WaitGroup 收敛。
- 错误通过带 buffer 的 error channel 返回，避免 goroutine 阻塞。

**追问：如果部分任务成功，部分失败怎么办？**
不要用内存状态当唯一依据。每个任务应该有独立状态，成功和失败都落库。批量任务最终状态由子任务聚合，失败任务可以重试或人工处理。

### 23. 如何回答 Go race 检测的真实案例？

**项目案例：**
知乎统一发布服务里，曾经通过 Go race 检测定位过三方 SDK 中 map 并发不安全问题。表现是并发压测或线上高并发下偶发 panic/数据异常，最终发现 SDK 内部共享 map 在多个 goroutine 下读写。

**处理方式：**
- 本地和测试环境用 `go test -race` 或压测方式复现。
- 定位共享变量的读写路径。
- 短期规避：调用侧串行化或加锁保护。
- 长期方案：推动 SDK 修复，或在业务层封装并发安全访问。

**面试亮点：**
这类问题不是靠看代码就一定能发现，race 工具能把“偶现并发问题”变成可定位的问题。

### 24. pprof 如何结合项目讲？

**回答模板：**

如果 Go 服务 CPU 飙高，我会先看是单实例还是全局，是发布后还是流量上涨后。然后抓 CPU profile，看热点函数是否集中在 JSON 序列化、正则、加解密、排序、锁竞争、日志打印或业务循环。  

如果内存上涨，我会看 heap profile 和 goroutine profile，区分是对象分配过多、缓存不释放、切片引用大数组、goroutine 泄漏，还是 GC 压力。  

结合项目，比如 ES fallback 或素材同步任务，如果一次性加载大量素材到内存，就可能导致 heap 突增；如果每条素材都开 goroutine，就可能导致 goroutine 暴涨。优化方向是分页扫描、worker pool、批量写入和对象复用。

**常用指标：**
- goroutine 数量。
- heap inuse/alloc。
- GC 次数和暂停时间。
- CPU profile top 函数。
- mutex/block profile。
- P50/P95/P99。

**结论先行：**
优雅停机要停止接收新流量，等待存量请求完成，取消后台任务，释放资源，并设置最大等待时间。

**分层展开：**

**流程**
- 接收 SIGTERM。
- 标记服务不健康，从负载均衡摘除。
- HTTP server shutdown。
- context 通知后台 goroutine 退出。
- 等待任务完成或超时强制退出。

**注意事项**
- 停机窗口要小于容器编排平台宽限时间。
- 长任务要支持 checkpoint 或补偿。
- MQ 消费者要停止拉取新消息并处理 ack。

**面试追问：为什么不能直接 kill？**
直接 kill 会导致请求中断、消息重复或丢失、锁未释放、连接未关闭，影响用户体验和数据一致性。
