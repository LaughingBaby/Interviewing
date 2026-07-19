# 技术｜Linux 操作系统与系统排障面试题

> 面向 9 年以上 Golang Web 后端业务研发岗位。本篇回答应用之下的系统问题：进程如何运行、CPU 为什么排队、内存去了哪里、IO 为什么变慢、容器为何被限制，以及怎样用证据完成生产排障。

---

## 一、Linux 学习地图与系统排障框架

### 1. Linux 后端知识图谱是什么？

```text
Process / Thread / Syscall
  -> Scheduler / CPU / Load / Context Switch
  -> Virtual Memory / Page / Cache / OOM
  -> VFS / Filesystem / Block IO
  -> FD / Pipe / Signal / IPC
  -> Namespace / Cgroup / Container
  -> Metrics / Commands / Evidence
  -> Incident Response / Capacity / Recovery
```

### 2. 资深后端如何回答系统排障题？

采用“业务现象 -> 资源饱和 -> 进程证据 -> 内核机制 -> 止血 -> 根治”。先看成功率、P99、QPS 和影响范围，再判断 CPU、内存、IO、网络或依赖，不从背命令开始。

### 3. Linux 排障的标准六步是什么？

1. 确认时间、实例、主机、容器和业务影响。
2. 保存发布、配置、日志、指标、进程和内核证据。
3. 限流、隔离、回滚或扩容控制影响。
4. 从系统全局到进程、线程、调用栈逐层收敛。
5. 修复后渐进恢复并校验业务。
6. 补容量、告警、运行手册和演练。

### 4. USE 方法如何使用？

对每类资源看 Utilization、Saturation、Errors：CPU 看使用和运行队列，内存看使用/OOM/回收压力，磁盘看利用率/队列/错误，网络看带宽/丢包/重传。

利用率不高也可能饱和，例如单核热点、磁盘队列和连接池等待。

### 5. 常用第一现场命令是什么？

```bash
uptime
top -H -p <pid>
vmstat 1
mpstat -P ALL 1
pidstat -p <pid> 1
iostat -xz 1
free -h
ss -s
dmesg -T | tail
```

命令输出必须与同一时间窗口的业务指标、版本和容器限制关联。

---

## 二、进程、线程、系统调用、信号与生命周期

### 6. 进程和线程有什么区别？

进程拥有独立虚拟地址空间和资源表；同进程线程共享地址空间、文件描述符等，但各有寄存器、栈和调度实体。Linux 内核用 Task 表示可调度实体，创建时通过 Clone Flags 决定共享范围。

### 7. 用户态和内核态如何切换？

系统调用、中断和异常使 CPU 进入内核态。切换本身及数据复制、调度会产生成本。高频小 IO、日志和锁竞争可能表现为 System CPU 高。

### 8. `fork`、`exec` 和 Copy-on-Write 如何工作？

Fork 创建子进程并共享物理页，页表标记只读；任一方写入时复制页面。Exec 用新程序映像替换当前进程。大内存进程 Fork 虽不立即复制全部数据，但页表、COW 写放大和内存峰值仍需评估。

### 9. Zombie 和 Orphan Process 是什么？

子进程退出后父进程未 Wait，保留退出状态成为 Zombie；父进程先退出，子进程由系统指定的 Reaper 接管。大量 Zombie 消耗 PID 表项，说明父进程生命周期处理错误。

### 10. Linux Signal 如何使用？

Signal 是异步通知。SIGTERM 用于可处理终止，SIGKILL 无法捕获，SIGHUP 常用于重载，SIGCHLD 通知子进程变化。Handler 中只能执行安全操作，应用通常转为设置状态或唤醒主循环。

### 11. 为什么 SIGKILL 不适合正常发布？

进程没有机会停止接流量、完成请求、提交 Offset 和 Flush 数据。正常流程先 SIGTERM 与宽限期，超时才 SIGKILL；业务写仍需幂等抵抗非正常退出。

### 12. System Call 慢如何定位？

用 `strace -ttT -p` 受控观察调用耗时和错误，用 `pidstat -w` 看切换，结合应用 Profile。Strace 有开销，生产短时采集；高频 Futex、read/write、fsync、connect 分别指向不同问题。

### 13. 文件描述符是什么？

进程 FD Table 的整数索引指向内核 Open File Description，再关联文件、Socket、Pipe 等。Dup 后多个 FD 可共享 Offset/状态；Fork 也会继承。泄漏会导致 `EMFILE`。

---

## 三、CPU 调度、负载、上下文切换与性能

### 14. Load Average 表示什么？

近 1/5/15 分钟可运行和不可中断等待任务的平均数量，不等于 CPU 使用率。Load 高可能是 CPU Run Queue，也可能是大量 D 状态 IO 等待。

与 CPU 核数、`vmstat r/b`、`mpstat` 和任务状态一起判断。

### 15. User、System、Iowait、Steal 分别说明什么？

- user：用户态执行。
- system：内核态工作。
- iowait：CPU 空闲但有 IO 等待，解释需谨慎。
- steal：虚拟化环境被 Hypervisor 占用。

还需看 IRQ/SoftIRQ、Guest 和 Cgroup Throttling。

### 16. CPU 100% 如何分层排查？

确认全核还是单核，找进程与线程，区分 User/System/SoftIRQ/Steal，再用语言 Profile 或 Perf 定位调用栈；对照流量、发布和后台任务。先限流/回滚，扩容前检查下游。

### 17. CPU 不高但接口很慢有哪些原因？

锁/Channel/连接池排队、下游 IO、磁盘、DNS、CPU Throttling、单线程热点、Goroutine/线程睡眠。低使用率不代表没有饱和，必须看 Queue 和等待时间。

### 18. Context Switch 为什么昂贵？

切换需保存状态、调度、可能破坏 Cache/TLB 局部性。自愿切换多通常是阻塞等待，非自愿切换多可能是时间片或竞争。大量线程、锁和小任务会放大。

### 19. 进程状态 R、S、D、T、Z 如何理解？

R 可运行，S 可中断睡眠，D 不可中断等待常见于 IO，T 停止/跟踪，Z 僵尸。大量 D 状态结合磁盘/NFS 指标排查，不能 Kill 立即解决正在内核等待的任务。

### 20. CPU Affinity 和 NUMA 何时重要？

高吞吐低延迟、跨 NUMA 内存访问和中断不均时重要。普通 Web 服务优先让调度器工作；绑核前用 Perf/NUMA 指标证明，容器 CPU Set 也要一致。

### 21. Perf 能解决什么问题？

采样 User/Kernel 调用栈、硬件事件、Cache Miss、分支等。使用 `perf top/record/report` 定位语言 Profile 看不到的 Syscall、内核和 Native 热点；符号、权限和采样开销需准备。

### 22. Run Queue 长如何治理？

降低入口和并发、关闭非核心 CPU 工作、扩 CPU/实例、消除热点锁和忙循环；检查 Cgroup Quota 和单核倾斜。恢复时渐进放量，避免积压瞬间重新排队。

---

## 四、虚拟内存、页、回收、Page Cache 与 OOM

### 23. 虚拟内存解决什么问题？

每进程看到独立连续地址空间，由页表映射物理页或文件；提供隔离、按需分配、共享和换页。VSS 大不等于实际物理占用，RSS/PSS 更接近实际驻留。

### 24. Page Fault 是什么？

访问页表未建立当前映射时触发。Minor Fault 不需磁盘，可能分配零页或建立已有页映射；Major Fault 需从存储读取，延迟高。高 Fault 要结合 mmap、启动预热和内存压力判断。

### 25. RSS、PSS、VSS 和 Working Set 有什么区别？

VSS 是虚拟地址范围；RSS 是驻留物理页，重复计共享页；PSS 按共享比例分摊；容器 Working Set 常近似当前不可轻易回收部分。不同工具口径不同，排障先确认定义。

### 26. Page Cache 为什么既好又会“吃内存”？

文件读写使用空闲内存缓存页面，提升 IO；内存压力时可回收。因此 `free` 低不等于不足，要看 available、回收、Swap、Fault 和应用延迟。

### 27. Anonymous Memory 和 File-backed Memory 有什么区别？

堆、栈等匿名页无文件副本，压力下写 Swap 或 OOM；文件映射页可从原文件重读，脏页需回写。Go Heap 主要是匿名内存，mmap/共享库是文件映射或特殊映射。

### 28. Swap 的收益和风险是什么？

Swap 可避免短时 OOM、容纳冷匿名页，但大量换入换出会产生抖动。数据库/低延迟服务通常严格控制；不能简单把 Swap=0 当绝对规则，要结合工作负载和容器策略。

### 29. 内核如何回收内存？

在 Watermark 压力下回收 Page Cache、写回脏页、交换匿名页；Direct Reclaim 会让申请线程参与，影响延迟。观察 pgscan/pgsteal、kswapd、Major Fault 和 PSI Memory。

### 30. OOM Killer 如何选择进程？

内核根据内存占用、`oom_score_adj` 等选择牺牲者；Cgroup 内存超限可在组内 OOM。查看 Dmesg、Cgroup Event、退出码和容器事件，应用往往没有机会日志。

### 31. Linux 内存高如何排查？

先主机/容器总量，再按进程 RSS/PSS，拆匿名、文件缓存、Slab、共享内存和 Huge Page；结合应用 Heap、线程栈、mmap/CGO。止血前保留映射和 Profile。

### 32. 内存碎片是什么？

物理页虽有空闲但难以找到所需连续块，或应用分配器 Size Class 内部浪费。看 Buddy Info、Compact、THP 与应用 Heap；扩内存不是唯一方案，需修对象尺寸和生命周期。

### 33. Transparent Huge Pages 有什么影响？

大页减少 TLB Miss，但分配/合并和回收可能带来延迟抖动。数据库等工作负载常有明确建议；生产按组件和实测配置，不全局照抄。

---

## 五、VFS、文件系统、磁盘与 Block IO

### 34. 一次文件读取经过哪些层？

应用 read -> VFS -> 文件系统 -> Page Cache；命中直接复制，未命中提交 Block IO 到设备，完成后唤醒任务。Direct IO 会绕过部分 Page Cache，但有对齐和复杂度。

### 35. Inode 和 Dentry 分别是什么？

Inode 保存文件元数据和数据块映射，不含文件名；Dentry 表示路径组件到 Inode 的关联并缓存路径解析。FD 指向打开文件状态。磁盘空间有但无法建文件可能是 Inode 耗尽。

### 36. `df` 和 `du` 为什么可能不一致？

已删除但仍被进程打开的文件仍占空间，du 路径不可见而 df 统计；还可能有挂载点、稀疏文件、Reserved Block。用 `lsof +L1` 查 Deleted Open File。

### 37. Buffering、Flush、fsync 有什么区别？

应用 Buffer、语言 Runtime、Page Cache 和设备 Cache 是不同层。Flush 常只把用户缓冲写入内核；fsync 请求文件数据/元数据达到持久化边界，实际保证还取决于文件系统和硬件。

### 38. 顺序 IO 和随机 IO 为什么差异大？

顺序访问利于预读、合并和设备吞吐；随机 IO 增加寻址、队列和放大，SSD 虽改善仍有 IOPS/写放大限制。数据库/MQ 通过顺序日志和批量降低成本。

### 39. `iostat` 重点看什么？

设备吞吐、IOPS、平均等待、队列和利用率。不同版本字段定义有差异；高 `%util` 在并行设备不必然饱和，应结合 await、queue、应用延迟和设备能力。

### 40. 磁盘 IO 高如何排查？

确认设备和挂载，找进程/线程，区分读写、吞吐/IOPS、同步写和后台回写；对照日志、备份、Compaction、数据库和发布。先暂停非核心批任务，保护数据库和日志。

### 41. IO Wait 高一定是磁盘慢吗？

不一定，可能 NFS、Block Device、存储网络或任务结构；Iowait 统计本身受 CPU 是否有其他工作影响。用 D 状态、设备指标、文件系统和远端存储证据确认。

### 42. 日志为什么可能拖垮服务？

同步格式化/写盘、日志暴增、磁盘满、采集 Agent 阻塞和大量小写。使用级别/采样/异步有界队列、轮转和磁盘配额；错误日志也不能无限重试打印。

### 43. 磁盘满的正确处理顺序是什么？

确认占用和增长源，停止非核心写与日志风暴，扩容或安全清理可再生数据；检查 Deleted Open File；恢复后验证数据库和文件系统。禁止未经确认直接删除数据目录。

---

## 六、FD、Pipe、Socket、IPC、epoll 与资源边界

### 44. FD 耗尽有什么表现？

`too many open files`、无法 Accept/打开日志/连接依赖。查进程 FD Limit、`/proc/<pid>/fd` 类型和增长；常见原因是 Body、Rows、文件和 Socket 未关闭，或连接无上限。

### 45. Soft/Hard Limit 如何理解？

Soft 是进程当前限制，可在权限范围调整；Hard 是上限。Systemd、容器和 Shell 启动链可能有不同 Limit，不能只看交互式 `ulimit`。

### 46. Pipe 的语义是什么？

内核有界字节缓冲，读空阻塞、写满阻塞；所有写端关闭后读到 EOF。Shell Pipeline、子进程 Stdout 和应用通信若不消费会互相阻塞。

### 47. Unix Domain Socket 与 TCP Socket 如何选择？

同机 IPC 可用 Unix Socket，省网络协议部分开销并支持文件权限；跨机必须网络 Socket。容器/代理架构还要考虑挂载、权限和故障边界。

### 48. epoll 的核心模型是什么？

将感兴趣 FD 注册到内核集合，就绪时返回事件，避免每轮线性扫描全部 FD。它通知“可进行 IO”，应用仍需非阻塞读写到 EAGAIN 并维护连接状态。

### 49. Level Trigger 与 Edge Trigger 有什么区别？

LT 只要仍就绪就持续通知，编程简单；ET 状态边缘变化通知，必须一次读/写到 EAGAIN，否则可能漏后续处理。不能只说 ET 一定更快，复杂度和事件量要实测。

### 50. epoll 惊群是什么？

多个等待者被同一事件唤醒但只有少数获得工作，造成无效调度。内核和服务器使用独占唤醒、Accept 分配和 SO_REUSEPORT 等机制缓解；具体实现结合版本。

### 51. PSI 解决什么问题？

Pressure Stall Information 统计任务因 CPU、内存或 IO 资源不足而停顿的时间，比单纯使用率更直接体现资源压力。可用于过载告警和扩缩容，但需结合业务 SLO。

---

## 七、Namespace、Cgroup、容器与 Kubernetes 系统边界

### 52. Container 与 VM 的核心区别是什么？

容器共享宿主机 Kernel，通过 Namespace 隔离视图、Cgroup 限制资源、Filesystem Layer 提供环境；VM 有独立 Guest Kernel。容器更轻但内核故障域共享。

### 53. 常见 Namespace 有哪些？

PID、Network、Mount、UTS、IPC、User、Cgroup、Time 等，分别隔离进程号、网络、挂载、主机名、IPC、身份等。隔离不是安全全部，还需 Capability、Seccomp、LSM 和网络策略。

### 54. Cgroup CPU Limit 如何工作？

Quota/Period 限制时间窗口 CPU，达到后 Throttle；CPU Request/Weight影响竞争份额。应用平均 CPU 未满但 Throttling 高时 P99 仍会抖动。

### 55. Cgroup Memory Limit 与 OOM 如何工作？

统计组内匿名、Page Cache 等口径，达到限制且无法回收时触发 Cgroup OOM。看 `memory.events`、current、stat 和 PSI；Go GOMEMLIMIT 要低于容器 Limit并留非 Go 余量。

### 56. Pod 内看到的 CPU/内存为什么和宿主机不同？

Namespace/Cgroup 视图、指标口径和工具版本不同；`free/top` 未必按限制展示。排障同时看容器 Cgroup、Pod 指标和 Node 压力。

### 57. Kubernetes Pod 被 Evict 与 OOMKilled 有什么区别？

OOMKilled 常是容器/节点内核杀进程；Eviction 是 Kubelet 因节点内存、磁盘、PID 等压力驱逐 Pod。事件、退出码和 Node Condition 可区分，治理路径不同。

### 58. CPU Request 和 Limit 如何影响服务？

Request影响调度和竞争权重，Limit 可能 Throttle。低延迟服务需压测是否使用 Limit、如何设置 GOMAXPROCS 和副本；不能只靠提高 Limit掩盖低效或下游瓶颈。

### 59. 容器磁盘问题有哪些？

Writable Layer、EmptyDir、日志和镜像占用可能触发 Ephemeral Storage Eviction；OverlayFS 小文件/写放大也影响。大数据使用持久卷/对象存储并监控 Inode 与空间。

---

## 八、系统观测、命令选择与证据保存

### 60. `/proc` 能提供什么？

进程状态、Cmdline、FD、Maps、Smaps、Limits、IO、Net 和系统级内存/压力。它是许多命令的数据源；容器中需确认 PID Namespace 和挂载权限。

### 61. `top`、`pidstat`、`vmstat` 如何分工？

Top 快速全局/线程定位；pidstat 连续观察进程 CPU、IO、切换、Fault；vmstat 看运行队列、内存、Swap、IO 和 CPU 全局趋势。不要只截一个瞬时值。

### 62. `sar` 的价值是什么？

保留历史 CPU、内存、IO、网络趋势，适合事故发生后回看；前提是采集已启用、采样频率合适。云/容器环境还需平台历史指标。

### 63. eBPF 适合解决什么？

低侵入观测 Syscall、调度、网络、Block IO 和内核事件，可按 PID/Cgroup 聚合。工具强大但需 Kernel/BTF/权限和开销控制；先明确问题，不因流行直接上生产探针。

### 64. 如何安全保存故障证据？

记录时间、Host/Pod/Container、PID、Version、Cgroup Limit；保存命令完整输出、Profile、Trace 和 Dmesg；避免只截 Top 截图。敏感数据受控存储，采集动作和开销记录。

### 65. 运行手册中的命令应怎样写？

写目标、命令、预期字段、危险性、权限和回滚；禁止只有一串无解释命令。定期演练，避免工具未安装或容器权限不足。

---

## 九、生产故障分支、止血、恢复与复盘

### 66. 接口大量超时时如何分层？

入口流量/LB -> 应用队列/CPU/GC -> 连接池 -> DB/缓存/RPC -> 主机 CPU/内存/IO/网络 -> 最近变更。先 Trace 定位慢层，再选系统工具，避免重复调查。

### 67. 主机 Load 很高但 CPU 空闲怎么办？

看 `vmstat b`、D 状态和 Wchan，检查磁盘/NFS/块设备；Iostat、Mount 和 Dmesg 确认证据。暂停重 IO 作业并保护写链路；重启不一定能中断内核等待。

### 68. 内存压力时如何止血？

限制入口和批任务、回滚泄漏版本、缩小缓存/并发、扩容或迁移；保留 Heap/Smaps/Dmesg。避免手工 Drop Cache 作为常规修复，它可能制造 IO 风暴。

### 69. 磁盘延迟突增如何止血？

暂停备份/Compaction/批量重放等非核心任务，降低日志，限写入并保护数据库；确认设备故障和云盘限额。恢复按优先级渐进，检查数据一致性和 fsync 错误。

### 70. 系统故障复盘应回答什么？

为什么资源达到上限、为何未提前告警、哪一步放大、为何恢复慢。改进落到容量阈值、Cgroup、应用限流、日志配额、发布门禁、备份和演练，而非只写“增加机器”。

---

## 十、业务场景与项目实战

### 71. Go 服务发布后 P99 抖动如何结合 Linux 排查？

按版本比较业务和 Runtime 指标，再看 Cgroup Throttling、Run Queue、Context Switch、Fault 和 IO；Profile 定位代码。若新版本分配增加导致 GC CPU 和 CPU Throttle，先回滚，再减少分配/调整资源并灰度验证。

### 72. 广告批量任务导致整机异常如何处理？

任务无限并发使 CPU、FD 和数据库连接排队。入口暂停任务，保留 Profile/系统指标；改为 Cgroup 独立资源、有界 Worker、连接预算和任务队列；监控队列年龄、拒绝、CPU PSI 和 FD。

### 73. 日志打满磁盘如何表达？

先停止日志风暴和非核心写，扩容/安全清理并处理 Deleted Open File；根因可能重试循环每次打印大响应。修复为错误聚合/采样、有界异步日志、轮转和磁盘水位告警，验证故障时日志不会再放大。

### 74. 容器 OOM 但 Go Heap 不高如何回答？

同时检查 RSS、线程栈、CGO/mmap、Page Cache 和 Sidecar；Cgroup Event确认 OOM；Smaps和进程映射找非 Heap 占用。GOMEMLIMIT 仅控制 Runtime口径，需为非 Go 内存留余量。

### 75. 如何用两分钟回答 Linux 排障？

> 我先从业务成功率、P99、QPS 和时间线确定影响，再按 USE 判断 CPU、内存、磁盘和网络是否饱和。CPU 看核分布、Run Queue、User/System/Steal/Throttle，再到进程线程和 Profile；内存区分 RSS、匿名页、Page Cache、回收、Cgroup 与 OOM；IO 从应用、VFS、Page Cache、文件系统到块设备看队列和延迟；资源泄漏看 FD、进程状态和 /proc。止血时先控制入口、暂停非核心任务或回滚，避免重启丢证据、扩容打爆下游。恢复后渐进放量并校验数据，复盘把问题落到容量、告警、Cgroup、发布门禁和运行手册。
