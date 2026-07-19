# 技术｜Go Web 与 RPC 工程实践面试题

> 面向 9 年以上 Golang Web 后端业务研发岗位。本篇围绕“一次请求的一生”组织：进入服务、协议解析、业务调用、访问依赖、返回响应，以及整个链路的超时、幂等、过载、安全和生命周期治理。

---

## 一、Go Web/RPC 学习地图与回答框架

### 1. Go Web 服务的知识主线是什么？

```text
Listener / Connection / TLS
  -> net/http Server / Router / Middleware
  -> Authentication / Validation / Idempotency
  -> Usecase / DB / Redis / MQ / ES
  -> HTTP Client / gRPC / Third-party API
  -> Deadline / Retry / Limit / Circuit / Backpressure
  -> Metrics / Trace / Security / Lifecycle
```

### 2. 资深后端如何回答 Web 工程题？

采用“请求链路 -> 资源边界 -> 失败语义 -> 生产参数 -> 观测验证”。例如连接池不能只列参数，要说明单实例并发、下游容量、等待时间、超时和扩容连接风暴。

### 3. Go Web 服务为什么不应只关注 Handler？

Handler 只是业务入口。生产问题往往来自 Listener、TLS、Header、Body、Context、连接池、下游慢、重试、响应写入和优雅停机。必须端到端理解请求生命周期。

### 4. 同步请求和异步任务如何划分？

用户必须立即知道的校验与受理同步完成；耗时、可重试、可削峰的工作持久化为任务并异步执行。异步接口返回 task_id 和明确状态，不用内存 Goroutine 假装可靠任务系统。

---

## 二、net/http Server、Router 与 Middleware

### 5. 一条 HTTP 请求在 Go 服务中如何执行？

**结论先行：一条 HTTP 请求会依次经过连接建立、协议解析、路由与中间件、业务与依赖调用、响应写回，并在整个生命周期中受 Context、超时和资源上限约束。**

```text
Accept/TLS
  -> 读取并解析 Header/Body
  -> Router 匹配 Method + Path
  -> Middleware：Trace/Recovery/限流/认证
  -> Handler：DTO 校验与协议转换
  -> Usecase：业务编排、DB/Redis/RPC
  -> WriteHeader/Body
  -> 连接复用或关闭
```

`net/http` 的 Server 接受连接并组织请求处理。HTTP/1.1 通常在同一连接上顺序处理请求，不同连接可并发；HTTP/2 在一个连接中使用多个 Stream 并受流控约束。Handler 返回不代表所有副作用都自动取消，依赖调用和后台工作必须显式使用请求 Context。

观测上应拆出总耗时、Handler、连接池等待和各依赖 Span；资源上要限制 Header、Body、并发和 Deadline。这样出现 P99 上升时，才能区分入口排队、本地计算还是下游等待。

**面试一句话：**完整请求不是“进 Handler 再返回”，而是连接、协议、Middleware、Usecase、依赖和响应写出的端到端链路；每一段都要有时间预算、资源上限和可观测性。

### 6. `http.Server` 哪些超时必须设置？

- ReadHeaderTimeout：限制读取 Header，防 Slowloris。
- ReadTimeout：限制读取整个请求，流式/大上传需谨慎。
- WriteTimeout：限制响应写出，但流式响应语义特殊。
- IdleTimeout：限制 Keep-Alive 空闲。
- Handler 内再用 Context Deadline 控制业务链路。

| 超时 | 约束阶段 | 配置风险 |
|---|---|---|
| `ReadHeaderTimeout` | 读取请求头 | 太大易受 Slowloris，太小会误伤慢网络 |
| `ReadTimeout` | 通常覆盖读取整个请求 | 大上传/流式请求需单独设计 |
| `WriteTimeout` | 响应写出窗口 | Streaming 的语义与普通响应不同 |
| `IdleTimeout` | Keep-Alive 空闲连接 | 影响连接复用和 FD 占用 |
| Handler Context | 业务、排队和依赖调用 | 应从端到端预算向下传播 |

不能只设置一个大超时覆盖所有阶段，也不能把 Server 超时当成业务 Deadline。入口超时后，DB/RPC 若未监听 Context 仍可能继续消耗资源。配置需要结合代理/LB 超时，避免外层先断开而内层仍执行。

**面试一句话：**Server 超时分别治理读 Header、读 Body、写响应和空闲连接，业务 Deadline 再约束 Handler 与下游；每层职责不同，不能用一个大超时包办。

### 7. 如何限制请求资源？

限制 Header、URL、Body 字节、Multipart、JSON 深度/字段、分页和批量条数；先鉴权和基本校验再执行昂贵解析；使用 `MaxBytesReader` 等能力防止恶意请求消耗内存和磁盘。

### 8. Router 应该承担什么？

路径和方法匹配、参数提取、Middleware 组合。业务规则、数据库事务和复杂协议转换不放 Router。路由冲突、404/405 和尾斜杠语义要有测试。

### 9. Middleware 顺序为什么重要？

推荐外到内大致为 Panic Recovery、Request ID/Trace、访问日志与 Metrics、限流、认证授权、业务 Handler。Metrics 要看到最终状态，Recovery 要覆盖后续层；Body 读取和响应包装要避免破坏流式接口。

### 10. Panic Recovery 怎样才正确？

在每个请求边界 Recover，记录 Panic 值、Stack、Trace 和版本，若尚未写响应则返回通用 500。已部分写出时不能伪装完整成功；Panic 还要计入告警。

Recovery 只保护进程，不修复可能已发生的数据副作用。

### 11. ResponseWriter 有哪些陷阱？

第一次 `WriteHeader/Write` 后状态和 Header 基本确定；默认 Write 会发送 200。包装 ResponseWriter 统计状态时要兼容 Flusher、Hijacker、Pusher 等可选接口，否则 WebSocket/Streaming 可能失效。

### 12. 如何处理大文件上传和下载？

流式读取、总大小限制、内容类型校验、临时文件配额、病毒/安全扫描和 Context 取消；不要 `io.ReadAll` 全部入内存。下载使用流式、Range、对象存储签名 URL 或反向代理卸载。

### 13. WebSocket/SSE 与普通 HTTP 有什么治理差异？

长连接占 FD、内存和 Goroutine，需要心跳、空闲超时、写队列上限、慢客户端剔除、重连游标和发布停机迁移。普通请求级负载均衡不能完全代表连接级分布。

---

## 三、HTTP Client、Transport、DNS 与连接池

### 14. 为什么应该复用 `http.Client` 和 `Transport`？

Transport 管理连接池、DNS/TLS 和 Keep-Alive。每请求创建会失去复用、增加握手和 TIME_WAIT；默认 Client 又没有整体 Timeout。通常按依赖复用一组受治理 Client。

### 15. HTTP Client 的超时层次有哪些？

- 请求 Context：端到端 Deadline，首选。
- Client Timeout：包含重定向和读取 Body 的总上限。
- Dial Timeout、TLS Handshake Timeout、ResponseHeaderTimeout。
- Idle Connection Timeout。

一次调用的时间可以拆为 Pool Wait、DNS、Dial、TLS、写请求、等待响应头和读取 Body。请求 Context 是首选的端到端上限；`Client.Timeout` 是包含重定向与读取 Body 的总时间保护；Transport 中的超时则用于定位和限制具体网络阶段。

子调用不能无视父 Context 重新创建更长超时，应使用父请求剩余预算，并为响应处理预留时间。流式下载、SSE 等长响应不适合简单设置较短的 `Client.Timeout`，更适合 Context、连接阶段超时和空闲/心跳策略组合。

**面试一句话：**Context 管端到端预算，Client Timeout 提供总上限，Dial/TLS/ResponseHeader 等参数约束具体阶段；流式调用要单独设计，且任何子超时都不能超过父请求剩余时间。

### 16. `MaxIdleConns`、`MaxIdleConnsPerHost` 和 `MaxConnsPerHost` 怎么配？

前两者控制可复用空闲连接，总连接上限控制对单 Host 的在途和拨号。根据实例峰值并发、协议、下游容量和实例数设置；过低导致排队/握手，过高在扩容时形成连接风暴。

必须监控池等待、建立连接、复用率和下游活跃连接。

### 17. 为什么必须关闭并读取 Response Body？

**结论先行：关闭 Body 负责释放响应资源，而在 HTTP/1.1 中将可控大小的 Body 读到 EOF，通常还有助于 Transport 复用底层连接。**两者相关但不是同一动作。

```go
resp, err := client.Do(req)
if err != nil {
	return err
}
defer resp.Body.Close()

body, err := io.ReadAll(io.LimitReader(resp.Body, maxResponseBytes+1))
if err != nil {
	return err
}
if int64(len(body)) > maxResponseBytes {
	return ErrResponseTooLarge
}
```

所有状态码路径都要关闭，不能只在 2xx 分支 `defer`。对未知或巨大响应，不应为了复用连接无界 `io.Copy`；应设置响应上限，超过后关闭并接受该连接可能无法复用。HTTP/2 的复用机制不同，但正确关闭 Body 仍是调用方责任。

**面试一句话：**Body 必须在成功获得 Response 后立即安排关闭；可控响应读到 EOF以利于复用，未知大响应先限字节，不能为了连接复用把恶意 Body 全部读完。

### 18. DNS 问题如何影响 Go Client？

解析延迟、缓存、TTL、搜索域和失效 IP 会表现为拨号慢或连接失败。长连接可能长期不重新解析；服务发现更新后要考虑连接轮换。记录 DNS、Connect、TLS、TTFB 分阶段耗时。

### 19. 如何避免 HTTP Client 重试放大？

只对幂等请求和明确瞬时错误重试，使用总 Deadline、有限次数、指数退避与抖动；尊重 429/Retry-After；避免网关、SDK、业务层同时重试。

POST 可通过 Idempotency-Key 和服务端状态查询变得可安全重试。

### 20. 如何处理第三方 API 限流？

按平台、账号、Endpoint 做令牌桶和并发限制，解析平台 Rate-Limit Header；任务队列吸收短期波动；429 按建议时间退避；配额耗尽时暂停而不是快速重试。

### 21. 如何记录外部调用日志？

记录 dependency、operation、status/error_code、attempt、duration、request_id、quota，不打印 Token 和完整敏感 Body。Trace Span 标注 DNS/Connect/TTFB 可帮助定位网络阶段。

---

## 四、REST API、契约、错误、分页与幂等

### 22. REST API 如何表达资源和动作？

稳定资源使用名词路径和 HTTP 方法；复杂业务动作可用子资源或命令 Endpoint，不必为形式强行 CRUD。关键是语义、幂等、错误和状态转换清楚。

### 23. API DTO 为什么不能直接复用数据库 Model？

外部协议、领域规则和存储结构变化周期不同。直接复用会泄露字段、标签和零值，数据库迁移被迫成为 API Breaking Change。使用显式 DTO 和映射。

### 24. API 错误响应应包含什么？

稳定 code、用户可理解 message、trace_id、可选 field violations 和 retry hint。HTTP Status 表达类别；内部 Cause 和 Stack 不返回客户端。

### 25. 创建接口如何实现幂等？

**结论先行：创建接口的幂等性需要服务端持久化“幂等键—请求语义—处理结果”的绑定，不能只靠客户端避免重复点击。**

1. 客户端为一次业务意图生成稳定的 `Idempotency-Key`。
2. 服务端把租户、接口和 Key 组成唯一范围，同时保存规范化请求摘要。
3. 首次请求在事务中创建幂等记录与业务资源，状态从 Processing 推进到 Succeeded/Failed。
4. 相同 Key、相同摘要的重复请求返回原资源或原响应。
5. 相同 Key、不同摘要返回冲突，防止错误复用。
6. 遇到 Processing 返回明确状态或查询地址，而不是再次执行。

记录需设置与业务重试窗口匹配的保留期。若创建还调用外部平台，应把本地状态机、Outbox 和平台幂等号结合；客户端超时只代表结果未知，不代表服务端未执行。

**面试一句话：**幂等键必须由服务端持久化并绑定请求摘要和最终资源；重复同请求返回原结果，复用 Key 改参数报冲突，超时后先查询状态而不是盲目再创建。

### 26. PUT、PATCH 和 POST 的幂等性如何理解？

HTTP 方法语义不自动保证业务实现。PUT 目标状态相同通常幂等；PATCH 取决于操作；POST 可借助业务 ID 幂等。重试决策必须依据服务端实际语义。

### 27. Cursor Pagination 如何设计？

按稳定索引排序，Cursor 包含最后一条排序值、唯一 Tie-breaker、查询版本和签名。限制 page_size；条件变化后旧 Cursor 失效。不要暴露可篡改原始内部状态。

### 28. API 版本如何演进？

优先向后兼容地新增字段；破坏性语义才引入版本。监控旧版本消费者，提供废弃周期。URL/Header/媒体类型只是载体，兼容流程更重要。

### 29. 批量 API 如何设计？

限制条数和总字节，定义全成全败还是逐项结果；每项有业务 ID 和错误；服务端控制并发并支持部分重试。不要把一万个请求简单包装成一个超大请求。

### 30. 如何防止重复提交和并发更新覆盖？

创建用唯一业务键，更新用 ETag/Version 条件，状态迁移带当前状态，客户端超时后按业务 ID 查询。前端按钮禁用只是体验优化，不是正确性。

---

## 五、gRPC、Protobuf、Interceptor 与流式调用

### 31. gRPC 的请求链路是什么？

客户端 Stub 序列化 Protobuf，经 HTTP/2 Stream 发送；Resolver 提供地址、LB 选择 Subchannel；服务端解码，Interceptor 执行横切逻辑，Handler 返回 Status 和 Message；Deadline/Metadata 沿链路传播。

### 32. gRPC 相比 HTTP/JSON 的核心优势是什么？

强 IDL、代码生成、紧凑二进制、HTTP/2 多路复用和原生流式。代价是浏览器/调试、网关、Schema 管理和跨团队生成工具。内部高频调用常合适，外部开放 API 未必。

### 33. Protobuf 字段如何兼容演进？

**结论先行：Protobuf 兼容的核心是保护 Wire Contract，而不是只保证 `.proto` 能编译。**字段号标识线上数据含义，一旦发布就不能分配给另一语义。

- 新增字段通常向后兼容，但消费者必须把“字段缺失/默认值”解释清楚。
- 删除字段后同时 `reserved` 字段号和名称，防止未来误复用。
- 不随意改变字段类型、Repeated/Optional 语义或消息结构；少数 Wire 兼容转换也可能破坏业务语义。
- Enum 消费者必须能处理未知数值，不能默认落入危险业务状态。
- 先发布能读取新旧格式的消费者，再让生产者开始写新字段。

CI 可做 Breaking Change 检查，跨语言还要生成并运行契约测试。数据库回填、事件保留期和离线消费者会让兼容窗口远长于一次滚动发布。

**面试一句话：**字段号发布后永不复用，删除就 reserved；演进采用消费者先兼容、生产者后写入，并同时验证默认值、未知 Enum、历史消息和跨语言生成代码。

### 34. Unary 和 Streaming 如何选择？

普通短请求用 Unary；连续大结果、实时事件或双向交互才用 Streaming。流式需要消息上限、背压、心跳、半关闭、断线续传和长连接负载均衡。

### 35. gRPC Deadline 为什么必须显式设置？

默认可能无限等待。客户端根据端到端预算设置，服务端观察 Context 取消并停止工作；跨语言传递时注意 Deadline 转换和时钟偏差。

### 36. gRPC 重试有哪些边界？

Service Config 可按方法配置尝试、退避和可重试状态；但应用必须保证幂等，并理解透明重试和负载放大。`INVALID_ARGUMENT` 等永久错误不重试，`UNAVAILABLE` 也需受 Deadline/预算约束。

### 37. Unary/Stream Interceptor 适合做什么？

认证授权、Trace、Metrics、日志、Recovery、限流和错误规范化。顺序需明确；业务补偿、复杂重试和方法特有规则放 Usecase/Client Wrapper。

### 38. gRPC Health Check 如何使用？

服务注册标准 Health Service，启动完成置 SERVING，摘流量/停机先置 NOT_SERVING。可按 Service 粒度；健康不等于所有下游完美，避免深度探测引发级联。

### 39. gRPC 长连接为什么可能负载不均？

HTTP/2 多路复用让大量 RPC 固定在少量连接，扩容后旧 Client 未建立新 Subchannel。使用正确 Resolver/LB Policy、连接生命周期和慢启动，观察每实例请求分布。

### 40. gRPC 消息大小和压缩怎么治理？

限制收发大小，避免大对象撑爆内存；批量按字节而非条数；压缩节省带宽但消耗 CPU，按消息类型和压测开启。超大文件走对象存储/流式，不塞单个 Protobuf。

---

## 六、数据库、Redis、Kafka、ES 客户端工程实践

### 41. Go 数据客户端的通用治理框架是什么？

单例复用、连接池、Context Deadline、并发上限、错误分类、幂等、Metrics/Trace、优雅关闭。每个依赖按 SLA 和容量独立配置，不能共享一个万能超时。

### 42. `database/sql` 连接池如何配置？

**结论先行：`database/sql` 的连接池既是复用器也是数据库并发闸门，参数必须从数据库总预算反推，而不是按服务 QPS随意调大。**

| 参数 | 作用 | 过小/过大的典型后果 |
|---|---|---|
| `MaxOpenConns` | 打开连接总上限 | 过小产生等待；过大压垮 DB |
| `MaxIdleConns` | 可保留的空闲连接 | 过小频繁建连；过大长期占用 DB 连接 |
| `ConnMaxLifetime` | 连接最长寿命 | 用于轮换、适配 LB/服务端策略，过短增加握手 |
| `ConnMaxIdleTime` | 连接最大空闲时间 | 回收低峰冗余连接 |

先从数据库可承受连接数扣除运维、迁移和其他服务预算，再除以最大实例数，并考虑扩容期间新旧实例重叠。应用并发最好不显著超过池容量，否则大量 Goroutine 只是在池前排队。监控 `DB.Stats()` 的 `InUse`、`Idle`、`WaitCount`、`WaitDuration`、`MaxIdleClosed` 和 `MaxLifetimeClosed`，结合慢 SQL 与事务时长调优。

**面试一句话：**连接池上限按“数据库总预算 ÷ 峰值实例数”反推并留运维余量；用池等待、在用连接、慢 SQL 和事务时长判断瓶颈，不能靠无限加连接解决排队。

### 43. 数据库事务在 Go 中有哪些坑？

使用 `BeginTx(ctx)`，所有语句通过 Tx；错误后 Rollback，Commit Error 也要处理；事务短小，不在内部调用外部 HTTP；不要混用 DB 和 Tx 导致部分操作跑出事务。

### 44. Cursor/Rows 为什么必须关闭？

未关闭会长期占连接；遍历后检查 `rows.Err/cursor.Err`；限制返回和 Batch。`defer Close` 放在合适函数作用域，循环中不要积累。

### 45. Redis Client 如何治理？

复用连接池，设置 Dial/Read/Write/Pool Timeout；Pipeline 控制批次；大 Key/响应有限制；只对幂等命令重试；监控 Pool Wait、Timeout、命中率和 Hot Key。

### 46. Kafka Consumer 的并发边界是什么？

Partition 决定消费组并行上限；处理要幂等；按 Key 顺序时同一分区不可随意并行；Rebalance 期间收敛在途；提交 Offset 与业务结果顺序正确；队列有界并保护下游数据库。

### 47. ES Bulk Worker 如何设计？

按字节/时间组成有限 Batch，有限并发，逐 Item 检查错误；429 退避，Mapping 错误隔离；Context 取消时停止接收并 Flush；监控 Queue、Bulk P99、拒绝和同步延迟。

### 48. 如何防止客户端连接风暴？

启动抖动、慢启动、限制并发建连、复用 Client、每实例连接上限和全局数据库预算。扩容平台应考虑下游容量；Readiness 在连接和预热完成后开启。

### 49. 依赖错误怎样统一分类？

Adapter 将驱动错误转换为 Timeout、Unavailable、Conflict、NotFound、RateLimited、Permanent 等内部类别，同时保留 Cause。Usecase 根据类别决定重试/补偿，Transport 映射协议。

---

## 七、Deadline、重试、限流、熔断、降级与背压

### 50. 端到端 Deadline 如何分配？

**结论先行：Deadline 是整条请求的时间预算，必须从入口向下游递减传播；每层只能消费剩余时间，不能重新获得一个完整超时。**

```text
入口总预算
  - 网关与入口排队
  - 本地校验/业务计算
  - 串行依赖 A
  - 串行依赖 B（含有限重试）
  - 序列化与响应预留
```

串行阶段的预算之和要小于总预算；并行调用虽然共享墙钟时间，但仍受连接池和并发上限约束。发起子调用前检查剩余时间，若已不足以完成最小有意义操作，应快速失败。重试必须消耗同一个总预算，并为最后一次响应处理留余量。

服务端收到取消后，DB、HTTP/gRPC、队列等待和循环都应主动退出。仅设置客户端超时而下游继续工作，会形成“调用方已经放弃、服务端仍占资源”的幽灵请求。

**面试一句话：**我从入口 SLO 中扣除排队、本地处理、下游和响应预留，子调用只使用剩余预算；重试也在同一 Deadline 内，取消必须真正传到所有阻塞操作。

### 51. 如何实现 Retry Budget？

限制重试流量占正常请求的比例或令牌数量；只有初次调用成功率允许时补充预算。这样依赖异常时不会所有请求都乘倍。按依赖和方法监控 attempt 与 retry success。

### 52. 本地限流和全局限流如何组合？

Gateway 做租户/用户全局配额，实例做本地 QPS/并发保护，资源层做账号/数据库等细粒度限制。本地近似换高可用，全局精确承担额外依赖。

### 53. 熔断和限流有什么区别？

限流控制进入速率/并发，熔断根据下游失败和慢调用暂时停止请求。二者与 Deadline、重试和降级共同构成韧性，不能互相替代。

### 54. 如何做 Bulkhead 隔离？

不同媒体、租户或优先级使用独立 Semaphore、Queue、Client/Pool 或实例组。一个慢平台不能占满全部 Goroutine 和连接。隔离份额要允许动态调整和闲时借用。

### 55. 服务降级如何避免假成功？

读请求可返回缓存/简化数据并标识陈旧；非核心字段可缺失；写请求若仅受理异步任务就返回 Accepted/Processing，不能把未执行伪装成功。降级后补偿和恢复条件明确。

### 56. 请求合并解决什么问题？

同一进程内多个相同 Key 的缓存 Miss 可共享一次下游查询，降低热点击穿。要限制等待人数、传播 Context 和错误，不能让一个短 Deadline 取消所有其他调用者。

### 57. 有界队列满了怎么办？

根据业务阻塞等待、快速 429/503、降级、丢低优先级或持久化到 MQ。所有等待有 Deadline；返回 Retry-After；监控拒绝和最老任务年龄。无限 Queue 只是推迟 OOM。

---

## 八、认证、授权、输入与数据安全

### 58. Authentication 和 Authorization 如何区分？

认证确认是谁，授权确认能对哪个资源做什么。Gateway 可验证 Token，服务仍需资源级授权和租户过滤，不能只信前端或内网 Header。

### 59. JWT 使用有哪些风险？

验证算法、Issuer、Audience、Expiration、NotBefore 和 Key ID；密钥轮换；Token 不放敏感可变数据；撤销和权限变更需短有效期/状态检查。禁止接受 `none` 或混淆算法。

### 60. mTLS 解决什么？

加密传输并认证双方工作负载身份。它不等于业务授权，服务仍检查主体是否可访问资源；证书签发、轮换、过期和时钟要监控。

### 61. 如何防 SSRF？

**结论先行：防 SSRF 不能只做 URL 字符串黑名单，必须在应用解析、每次实际连接和网络出口三层共同限制目的地。**

1. 优先让用户选择业务允许的目标标识，而不是提交任意 URL。
2. 仅允许 `https` 等 Scheme、明确 Host/Port 白名单，拒绝 URL 中的用户信息和异常编码。
3. 解析 DNS 后检查所有候选 IP，禁止 Loopback、私网、Link-local、组播和云 Metadata 地址。
4. 自定义 Dial 流程校验实际连接 IP，防止校验后 DNS Rebinding。
5. 重定向每一跳重新执行同样校验，并限制跳数和跨域。
6. 通过出网代理、防火墙/网络策略和云权限限制进程能访问的网段。

还要限制响应大小、超时和协议升级，避免 SSRF 同时演化为资源耗尽。解析器应使用标准 URL/IP 库，不手写字符串前缀判断 IPv6、整数 IP 和混合编码。

**面试一句话：**SSRF 防护要对白名单 URL 做规范化，校验 DNS 结果和实际 Dial IP，每次重定向重验，再用出网策略兜底；字符串黑名单挡不住 Rebinding 和地址编码绕过。

### 62. 如何防注入和越权？

SQL 参数化、禁止拼 DSL/Shell；字段白名单；租户 ID 从可信身份注入而非请求自报；对象级授权；批量接口逐项校验。日志和错误不泄露查询与 Secret。

### 63. JSON/Protobuf 反序列化如何保护？

限制 Body/Message 大小、嵌套和重复字段策略；验证必填、范围、Enum 和字符串长度；对外 JSON 可拒绝未知字段或按兼容策略处理；反序列化后仍需业务校验。

### 64. Secret 和凭证如何进入 Client？

从 Secret Manager/工作负载身份获取，内存中最小范围持有，支持轮换；不进入 URL、日志、Trace、错误和镜像。不同依赖/环境使用最小权限账号。

---

## 九、启动、健康检查、优雅停机与发布生命周期

### 65. 服务启动顺序是什么？

读取并校验配置 -> 初始化日志/Trace -> 建立必要客户端 -> 执行轻量连通性和缓存预热 -> 启动 Listener/Consumer -> Readiness=true。失败明确退出，避免半初始化服务接流量。

### 66. Liveness、Readiness 和 Startup Probe 如何区分？

Liveness 判断进程是否需要重启；Readiness 判断是否接流量；Startup 保护慢启动。临时数据库故障通常让 Readiness 降级，不应让所有实例同时 Liveness 失败重启。

### 67. 优雅停机的完整步骤是什么？

收到 SIGTERM -> Readiness=false -> 等负载均衡传播 -> 停止新 HTTP/RPC/MQ 任务 -> Server Shutdown/GracefulStop -> 取消后台 Context -> 等待在途 -> Flush/提交 -> 超时强退。

### 68. gRPC `GracefulStop` 有什么风险？

等待现有 RPC 完成，长流可能永不结束。应先停止新流并通知 Stream 关闭，外层设置宽限时间，到期使用 Stop。所有 Handler 必须尊重 Context。

### 69. Consumer 如何优雅退出？

停止 Poll/拉取，等待已分配 Partition 的在途任务完成或安全取消，提交已完成 Offset，关闭 Session/Client。Rebalance 回调不能无限等待，幂等保证重复处理安全。

### 70. 滚动发布为什么会出现连接不均？

旧长连接仍指向少量实例，新实例虽 Ready 但无流量。客户端/Proxy 需要连接年龄、Endpoint 更新和慢启动；发布监控按版本和实例观察，而不只看副本数。

### 71. 如何设计快速回滚？

应用版本向后兼容 Schema/事件，配置和 Feature Flag 可独立回退，镜像不可变；发布前验证旧版本能读取新数据。回滚后检查任务积压和已产生副作用，不止恢复进程。

---

## 十、业务架构与项目实战

### 72. 广告媒体 API 适配层如何设计？

每个平台实现内部统一接口；Adapter 负责协议、鉴权、错误分类、限流和幂等号；账号级 Semaphore 隔离；平台响应映射领域状态；原始 request_id 脱敏保存供排障。业务不直接依赖 SDK 类型。

### 73. 批量素材同步接口如何设计？

API 幂等创建 Batch/Item 任务并返回 task_id；Outbox 发布 Kafka；Worker 分页、有界并发调用媒体和 ES；每项状态落库；可取消、重试和人工处理；指标覆盖排队、成功、429、同步延迟和内存峰值。

### 74. 外部 API 超时结果未知怎么办？

请求携带平台支持的幂等 ID；超时后先调用查询接口确认，不能直接重复创建；本地状态标记 Unknown 并进入查询任务；最终按平台结果推进状态机，无法确认时人工处理。

### 75. 如何保护数据库不被 Go 服务扩容打爆？

数据库总连接预算按服务和实例分配；实例本地并发小于连接池；扩容慢启动并限制 MaxConnecting；入口限流；慢 SQL 治理；池等待告警。扩服务前同时评估数据库和第三方容量。

### 76. 如何回答一次完整 Web 请求？

> 请求经过 Listener/TLS 和 HTTP 解析，Server 在受控超时与大小限制下交给 Router；Middleware 完成 Trace、恢复、限流、认证和指标；Handler 校验 DTO 后调用 Usecase。Context Deadline 传播到 DB、Redis 和 RPC，各客户端复用连接池并有独立并发预算；写请求用业务 ID 幂等，错误按类别映射协议。响应只返回必要字段。线上监控各阶段耗时、池等待和依赖错误，发布时先 Ready 再接流量，停机时先摘流量再等待在途。核心是每个资源都有上限，每个失败都有语义，每个步骤可观测。

### 77. 如何用两分钟回答 Go Web/RPC 工程实践？

> 我从端到端请求而不是单个 Handler 设计 Go 服务。Server 设置 Header、请求体、空闲和业务 Deadline，Middleware 统一 Trace、认证、限流、Recovery 和错误；HTTP/gRPC Client 单例复用连接，按依赖设置连接池、超时、并发和重试预算。API 使用 DTO、稳定错误码、Cursor 和 Idempotency-Key；gRPC 通过 Protobuf 兼容规则、Interceptor、Health 和正确 LB 治理。DB、Redis、Kafka、ES 都传播 Context、限制队列并分类错误。安全上做工作负载身份、资源授权、输入上限和 Secret 轮换。启动完成才 Ready，停机先摘流量再收敛请求和消费者。生产效果用 P99、错误率、池等待、拒绝、依赖可用性和任务积压验证。
