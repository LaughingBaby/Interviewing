# 技术｜缓存之 Redis

> 面向 BAT、字节、小红书等大厂后端面试的 Redis 题库与结构化答案。
> 当前文档按“三大主线”重排：Redis 当前能力、核心能力原理分析、实战设计与线上治理。
> 答案统一采用“结论先行 -> 分层展开 -> 面试追问”的金字塔表达方式，方便在面试中先给判断，再给原理、场景和权衡。

相关补充文档：

- 网络 IO 深挖：[技术｜Linux网络与线上排障面试题.md](/Users/guoyimeng/Documents/Coding/Interviewing/技术｜Linux网络与线上排障面试题.md)
- 通用数据结构与索引对比：[技术｜核心数据结构与存储引擎索引对比.md](/Users/guoyimeng/Documents/Coding/Interviewing/技术｜核心数据结构与存储引擎索引对比.md)
- Redis 跳表图解：[zskiplist数据结构辅助理解.md](/Users/guoyimeng/Documents/Coding/Interviewing/zskiplist数据结构辅助理解.md)

## 一、Redis 当前能力全景：命令、类型、编码与架构模式

> 这一章回答“Redis 有什么能力”。先知道命令、数据类型、底层编码、持久化、过期淘汰、复制、哨兵、Cluster、Pipeline 和 key 设计。

### 1. Redis 命令作用速查

这一节把 Redis 基础数据类型相关命令集中放在一起。面试时先说“命令做什么”，再讲数据结构、复杂度、业务场景和风险。下面覆盖核心数据类型的常用命令；模块命令如 Bloom Filter 需确认线上是否部署 RedisBloom。

**Key 通用命令**

| 命令 | 作用 | 面试提醒 |
|---|---|---|
| `EXISTS key` | 判断 key 是否存在 | 可用于兜底判断，但不要代替业务状态校验 |
| `TYPE key` | 查看 key 的数据类型 | 排查线上 key 类型误用 |
| `DEL key` | 同步删除 key | 大 key 删除可能阻塞，优先考虑 `UNLINK` |
| `UNLINK key` | 异步删除 key | 适合删除大 key，降低主线程阻塞 |
| `EXPIRE key seconds` / `PEXPIRE key ms` | 设置过期时间 | 缓存必须有 TTL 兜底 |
| `TTL key` / `PTTL key` | 查看剩余过期时间 | 排查缓存是否会集中失效 |
| `PERSIST key` | 移除过期时间 | 小心把缓存变成永久 key |
| `RENAME key newkey` / `RENAMENX key newkey` | 重命名 key | `RENAMENX` 仅在新 key 不存在时成功 |
| `SCAN cursor MATCH pattern COUNT n` | 渐进式扫描 key | 线上替代 `KEYS` |
| `KEYS pattern` | 全量匹配 key | 生产禁用或极慎用，可能阻塞主线程 |

**String 命令**

| 命令 | 作用 | 示例含义 |
|---|---|---|
| `SET key val` | 设置 key 的值为 val；不存在则创建，存在则覆盖 | `SET name tom`：创建/更新 `name=tom` |
| `SET key val NX EX seconds` | key 不存在才设置，并设置秒级 TTL | `SET lock:order:1 requestId NX EX 10`：加 10 秒过期锁 |
| `SETNX key val` | key 不存在才设置 | 旧式锁基础命令，通常更推荐 `SET NX EX` |
| `SETEX key seconds val` / `PSETEX key ms val` | 设置值并同时设置过期时间 | 缓存写入时原子设置 TTL |
| `GET key` | 获取 key 对应 value | `GET user:1`：读取缓存值 |
| `MGET key1 key2` | 批量获取多个 key | 减少网络往返，Cluster 下注意 slot |
| `MSET key val ...` / `MSETNX key val ...` | 批量设置多个 key | `MSETNX` 要求所有 key 不存在 |
| `GETSET key val` | 返回旧值并设置新值 | 老命令，很多场景可用 `GETEX/GETDEL` 替代 |
| `GETEX key EX seconds` | 获取值并修改过期时间 | 读取同时续期 |
| `GETDEL key` | 获取值并删除 key | 一次性 token/验证码场景 |
| `INCR key` / `DECR key` | 整数自增/自减 1 | 计数器、固定窗口限流 |
| `INCRBY key n` / `DECRBY key n` | 整数增加/减少指定值 | 批量计数 |
| `INCRBYFLOAT key f` | 浮点数增加指定值 | 金额类不建议只靠 Redis 做权威数据 |
| `APPEND key val` | 追加字符串 | 日志片段类场景慎用，容易形成大 key |
| `STRLEN key` | 获取字符串长度 | 排查大 value |
| `GETRANGE key start end` | 获取子串 | 字节范围读取 |
| `SETRANGE key offset val` | 从 offset 开始覆写部分内容 | 底层仍是 String，offset 太大可能撑大 value |

**Hash 命令**

| 命令 | 作用 | 示例含义 |
|---|---|---|
| `HSET key field val` | 设置 field 的值 | `HSET user:1 name Tom`：设置 name |
| `HSETNX key field val` | field 不存在才设置 | 防止覆盖已有字段 |
| `HGET key field` | 获取指定 field | `HGET user:1 name`：读取 name |
| `HMGET key field1 field2` | 批量获取多个 field | 只取需要字段，避免全量 |
| `HGETALL key` | 获取全部 field-value | 大 Hash 慎用，可能阻塞和打满网络 |
| `HKEYS key` / `HVALS key` | 获取全部 field / value | 大 Hash 慎用 |
| `HLEN key` | 获取 field 数量 | 判断 Hash 规模 |
| `HEXISTS key field` | 判断 field 是否存在 | 字段级存在性判断 |
| `HDEL key field` | 删除 field | `HDEL cart:user:1 sku:2`：删购物车商品 |
| `HINCRBY key field n` | 整数字段增加 n | `HINCRBY product:1 stock -1`：库存字段 -1 |
| `HINCRBYFLOAT key field f` | 浮点字段增加 f | 统计值场景 |
| `HSTRLEN key field` | 获取 field 对应 value 长度 | 排查字段大 value |
| `HRANDFIELD key count WITHVALUES` | 随机返回 field，可带 value | 抽样检查 Hash |
| `HSCAN key cursor MATCH pattern COUNT n` | 渐进式扫描 field | 大 Hash 遍历替代 `HGETALL` |

**List 命令**

| 命令 | 作用 | 示例含义 |
|---|---|---|
| `LPUSH key val` / `RPUSH key val` | 从左/右侧插入元素 | 队列、栈、最新列表 |
| `LPUSHX key val` / `RPUSHX key val` | key 存在时才插入 | 避免意外创建队列 |
| `LPOP key` / `RPOP key` | 从左/右侧弹出元素 | 消费队列元素 |
| `BLPOP key timeout` / `BRPOP key timeout` | 阻塞式弹出元素 | 简单阻塞队列 |
| `LLEN key` | 获取列表长度 | 判断队列堆积 |
| `LINDEX key index` | 按下标读取元素 | 随机访问不是 List 强项 |
| `LSET key index val` | 按下标设置元素 | 下标越界会失败 |
| `LRANGE key start stop` | 获取范围元素 | `LRANGE list 0 -1` 取全量，大 List 慎用 |
| `LTRIM key start stop` | 只保留指定范围 | 维护最新 N 条 |
| `LINSERT key BEFORE/AFTER pivot val` | 在指定元素前/后插入 | 需要查找 pivot |
| `LREM key count val` | 删除指定值元素 | count 控制方向和数量 |
| `LPOS key val` | 查找元素位置 | Redis 6.0+ 常见 |
| `LMOVE source dest LEFT RIGHT` | 从 source 弹出并推入 dest | 队列迁移/可靠队列雏形 |
| `BLMOVE source dest LEFT RIGHT timeout` | 阻塞版 `LMOVE` | 等待 source 有数据 |
| `RPOPLPUSH source dest` / `BRPOPLPUSH source dest timeout` | 右弹左推 | 旧式可靠队列方案，逐渐被 `LMOVE/BLMOVE` 替代 |

**Set 命令**

| 命令 | 作用 | 示例含义 |
|---|---|---|
| `SADD key member` | 添加元素，天然去重 | 标签、关注、黑名单 |
| `SREM key member` | 删除元素 | 删除标签/取消关注 |
| `SISMEMBER key member` | 判断元素是否存在 | 判断用户是否点赞/关注 |
| `SMISMEMBER key m1 m2` | 批量判断多个元素是否存在 | 批量判断关系 |
| `SMEMBERS key` | 获取全部元素 | 大 Set 慎用 |
| `SCARD key` | 获取元素数量 | 统计集合规模 |
| `SPOP key count` | 随机弹出元素 | 抽奖/随机消费 |
| `SRANDMEMBER key count` | 随机返回元素但不删除 | 随机展示 |
| `SMOVE source dest member` | 将元素从 source 移到 dest | 状态集合迁移 |
| `SINTER key1 key2` | 求交集 | 共同好友、共同标签 |
| `SINTERCARD numkeys key...` | 只返回交集数量 | 不返回大结果，适合计数 |
| `SUNION key1 key2` | 求并集 | 合并集合 |
| `SDIFF key1 key2` | 求差集 | A 有 B 没有 |
| `SINTERSTORE dest key...` / `SUNIONSTORE dest key...` / `SDIFFSTORE dest key...` | 结果写入目标 Set | 离线/缓存集合结果 |
| `SSCAN key cursor MATCH pattern COUNT n` | 渐进式扫描 Set | 大 Set 遍历替代 `SMEMBERS` |

**ZSet 命令**

| 命令 | 作用 | 示例含义 |
|---|---|---|
| `ZADD key score member` | 添加/更新 member 分数 | `ZADD rank 100 user:1` |
| `ZINCRBY key increment member` | 增加 member 分数 | 排行榜分数累加 |
| `ZSCORE key member` / `ZMSCORE key m1 m2` | 查询一个/多个 member 分数 | 查个人分数 |
| `ZCARD key` | 获取元素数量 | 排行榜规模 |
| `ZCOUNT key min max` | 统计 score 范围内数量 | 区间计数 |
| `ZLEXCOUNT key min max` | 按字典序范围计数 | 相同 score 下字典序场景 |
| `ZRANGE key start stop WITHSCORES` | 按排名/score/字典序范围查询 | 新版 `ZRANGE` 可覆盖多类范围查询 |
| `ZREVRANGE key start stop WITHSCORES` | 按分数倒序查询排名范围 | TopN |
| `ZRANGEBYSCORE key min max` | 按 score 范围查询 | 延迟队列取到期任务 |
| `ZRANGEBYLEX key min max` | 按字典序范围查询 | 特定字典序场景 |
| `ZRANK key member` / `ZREVRANK key member` | 查询正序/倒序排名 | 个人排名 |
| `ZREM key member` | 删除 member | 删除已处理任务 |
| `ZREMRANGEBYSCORE key min max` | 删除 score 范围 | 滑动窗口清理过期请求 |
| `ZREMRANGEBYRANK key start stop` | 删除排名范围 | 保留 TopN |
| `ZREMRANGEBYLEX key min max` | 删除字典序范围 | 字典序清理 |
| `ZPOPMIN key count` / `ZPOPMAX key count` | 弹出最小/最大分数元素 | 任务/排行榜裁剪 |
| `BZPOPMIN key timeout` / `BZPOPMAX key timeout` | 阻塞弹出最小/最大元素 | 阻塞任务队列 |
| `ZMPOP` / `BZMPOP` | 从多个 ZSet 弹出元素 | 多队列场景 |
| `ZUNION` / `ZINTER` / `ZDIFF` | 多个 ZSet 并/交/差计算 | 多榜单合并 |
| `ZUNIONSTORE` / `ZINTERSTORE` / `ZDIFFSTORE` | 结果写入目标 ZSet | 缓存榜单结果 |
| `ZRANGESTORE dest src ...` | 范围结果写入目标 ZSet | 保存范围查询结果 |
| `ZRANDMEMBER key count WITHSCORES` | 随机返回 member | 抽样 |
| `ZSCAN key cursor MATCH pattern COUNT n` | 渐进式扫描 ZSet | 大 ZSet 遍历 |

**Bitmap 命令**

| 命令 | 作用 | 示例含义 |
|---|---|---|
| `SETBIT key offset value` | 设置某个 bit 位为 0/1 | 签到、是否已读 |
| `GETBIT key offset` | 获取某个 bit 位 | 查某天是否签到 |
| `BITCOUNT key` | 统计 bit 为 1 的数量 | 统计签到天数 |
| `BITPOS key bit` | 查找第一个 0/1 的位置 | 找第一个未完成位置 |
| `BITOP AND/OR/XOR/NOT dest key...` | 多个 bitmap 做位运算 | 多日活跃交并集 |
| `BITFIELD key ...` | 按位段读写整数 | 多 bit 状态压缩 |
| `BITFIELD_RO key ...` | 只读 bit field | 只读场景更安全 |

**HyperLogLog 命令**

| 命令 | 作用 | 示例含义 |
|---|---|---|
| `PFADD key element` | 添加元素参与基数估算 | 记录 UV |
| `PFCOUNT key` | 估算基数 | 估算独立用户数 |
| `PFMERGE dest key...` | 合并多个 HLL | 合并多天 UV |

**Geo 命令**

| 命令 | 作用 | 示例含义 |
|---|---|---|
| `GEOADD key lon lat member` | 添加地理位置 | 添加门店坐标 |
| `GEOPOS key member` | 查询成员坐标 | 查门店位置 |
| `GEODIST key m1 m2 unit` | 计算两点距离 | 计算用户到门店距离 |
| `GEOHASH key member` | 返回 geohash 字符串 | 地理编码调试 |
| `GEOSEARCH key ...` | 按半径/矩形搜索附近成员 | 附近的人/店 |
| `GEOSEARCHSTORE dest key ...` | 搜索结果写入目标 key | 缓存附近结果 |
| `GEORADIUS` / `GEORADIUSBYMEMBER` | 旧版附近搜索命令 | 新项目优先使用 `GEOSEARCH` |

**Stream 命令**

| 命令 | 作用 | 示例含义 |
|---|---|---|
| `XADD stream * field val` | 追加消息 | 写入事件流 |
| `XLEN stream` | 查看消息数量 | 判断 Stream 规模 |
| `XRANGE stream start end` / `XREVRANGE stream end start` | 正序/倒序范围读取 | 消息回溯 |
| `XTRIM stream MAXLEN n` | 裁剪 Stream 长度 | 控制消息保留量 |
| `XDEL stream id` | 删除消息 | 删除指定消息 |
| `XREAD STREAMS stream id` | 普通读取 Stream | 非消费者组读取 |
| `XGROUP CREATE stream group id MKSTREAM` | 创建消费者组 | 建立消费组 |
| `XGROUP SETID stream group id` | 修改消费者组游标 | 调整消费起点 |
| `XGROUP DESTROY stream group` | 删除消费者组 | 清理消费组 |
| `XGROUP DELCONSUMER stream group consumer` | 删除消费者 | 清理下线消费者 |
| `XREADGROUP GROUP group consumer STREAMS stream >` | 消费者组读取新消息 | 组内分摊消费 |
| `XACK stream group id` | 确认消息已处理 | 从 PEL 移除 |
| `XPENDING stream group` | 查看未确认消息 | 排查堆积/失败消费 |
| `XCLAIM stream group consumer min-idle id` | 转移 pending 消息给其他消费者 | 消费者故障接管 |
| `XAUTOCLAIM stream group consumer min-idle start` | 自动转移 pending 消息 | 批量接管超时消息 |
| `XINFO STREAM/GROUPS/CONSUMERS` | 查看 Stream/组/消费者信息 | 排障和监控 |

**Bloom Filter 命令，RedisBloom 模块**

| 命令 | 作用 | 示例含义 |
|---|---|---|
| `BF.RESERVE key error_rate capacity` | 创建 Bloom Filter 并设置误判率/容量 | 提前规划容量 |
| `BF.ADD key item` | 添加元素 | 写入合法商品 ID |
| `BF.MADD key item...` | 批量添加元素 | 初始化布隆 |
| `BF.EXISTS key item` | 判断元素是否可能存在 | 防缓存穿透 |
| `BF.MEXISTS key item...` | 批量判断 | 批量防穿透 |
| `BF.INSERT key ...` | 带参数批量插入 | 可指定容量/误判策略 |
| `BF.INFO key` | 查看 Bloom 信息 | 排查容量和误判配置 |

### 2. Redis 底层数据结构速记

这张表用来背“类型 -> 编码 -> 为什么这么设计”。面试时不需要一上来报源码细节，但要能讲出结构和取舍。

| Redis 类型 | 常见底层编码/结构 | 速记 | 适合场景 | 主要风险 |
|---|---|---|---|---|
| String | SDS；对象编码可能是 `int`、`embstr`、`raw` | `len + alloc + buf`，二进制安全，O(1) 取长度 | 缓存对象、计数、锁、限流计数 | 大 value、整体序列化成本 |
| Hash | 小 Hash 用 listpack；大 Hash 用 hashtable | 小对象紧凑，大对象查找快 | 用户字段、商品属性、购物车 | 字段太多形成大 key，无 field TTL |
| List | quicklist，节点内部是 listpack | 双向链表 + 紧凑块，兼顾两端操作和内存 | 简单队列、最新列表 | 大 List、缺 ACK |
| Set | intset / listpack / hashtable | 小整数省内存，普通集合靠哈希去重 | 标签、关注、黑名单、共同好友 | 大集合全量/交并差阻塞 |
| ZSet | 小 ZSet 用 listpack；大 ZSet 用 dict + skiplist | dict 查 member，skiplist 做排序和范围 | 排行榜、延迟队列、滑动窗口 | 大 ZSet、范围操作成本 |
| Bitmap | 本质是 String/SDS 的 bit 操作 | 一个 bit 表示一个布尔状态 | 签到、在线、是否完成动作 | offset 稀疏会浪费空间 |
| HyperLogLog | 稀疏/密集概率结构，Redis 以字符串对象承载 | 用误差换极低内存 | UV、独立 IP、独立设备 | 有误差，不能取具体元素 |
| Geo | 底层使用 ZSet，score 存 geohash 编码 | 地理位置转成可排序 score | 附近的人/店 | 精度、半径查询成本 |
| Stream | radix tree/rax + listpack；消费者组维护 PEL | append-only log + 消费组确认 | 轻量消息流、异步任务 | 不等同 Kafka，需治理 PEL |
| Bloom Filter | bitmap + 多个 hash 函数，RedisBloom 模块提供 | 说不存在一定不存在，说存在只是可能存在 | 缓存穿透、黑名单 | 有误判、删除困难、需容量规划 |

**底层结构一句话背法：**

```text
String 是 SDS；
Hash 小用 listpack，大用 hashtable；
List 是 quicklist；
Set 小整数用 intset，普通用 hashtable；
ZSet 小用 listpack，大用 dict + skiplist；
Bitmap 是 String 的 bit；
HLL 是概率计数；
Geo 是 ZSet 的地理编码；
Stream 是 rax + listpack + PEL；
Bloom 是 bitmap + 多 hash。
```

### 3. Redis 常见数据结构如何总览？

**结论先行：**
Redis 更准确地说是一个内存数据结构服务器，不只是简单 key-value。面试时不要只背命令，要讲清楚“结构能力、底层编码、适用场景、风险边界”。

| 数据结构 | 核心能力 | 典型场景 | 主要风险 |
| --- | --- | --- | --- |
| String | 二进制安全 value、原子计数 | 缓存对象、计数器、锁、限流 | 大 value、整体序列化成本 |
| Hash | field-value 对象 | 用户信息、商品属性、购物车 | 字段过多形成大 key、无 field TTL |
| List | 有序、可重复、两端操作 | 简单队列、栈、最新列表 | 缺少 ACK、大 List 风险 |
| Set | 无序、唯一、集合运算 | 标签、关注、共同好友、去重 | 大集合交并差阻塞 |
| ZSet | 唯一、score 排序 | 排行榜、延迟队列、滑动限流 | 大 ZSet、范围操作成本 |
| Bitmap | bit 状态 | 签到、在线状态、是否完成动作 | 只支持 0/1、稀疏 offset 浪费 |
| HyperLogLog | 近似基数统计 | UV、独立 IP、独立设备数 | 有误差、不能取元素 |
| Bloom Filter | 判断可能存在 | 缓存穿透、黑名单、已读判断 | 有误判、删除困难、需容量规划 |
| Stream | 消息流、消费组、ACK | 轻量消息队列、事件流 | 不能替代专业 MQ 的所有能力 |

一句话记忆：

```text
String 适合整体值；
Hash 适合局部字段；
List 适合简单队列；
Set 适合去重集合；
ZSet 适合排序范围；
Bitmap 适合海量布尔状态；
HyperLogLog 适合近似去重计数；
Bloom Filter 适合防穿透；
Stream 适合轻量可靠消息流。
```

### 4. Redis 的 String 底层是什么？适合哪些场景？

**结论先行：**
String 是 Redis 最基础的数据类型，底层不是直接使用 C 字符串，而是使用 SDS，也就是 Simple Dynamic String。它适合缓存对象、计数器、分布式锁、限流计数、Session、验证码、Token、简单状态值和二进制数据。

**为什么不用 C 字符串？**

C 字符串有几个问题：

```text
1. 获取长度需要 O(N)，因为要从头扫到 '\0'
2. 不二进制安全，遇到 '\0' 会被认为字符串结束
3. 拼接时容易缓冲区溢出
4. 扩容和内存管理不方便
```

SDS 可以理解成：

```text
SDS = header + buf[]

┌────────────┬────────────┬──────────────────────┐
│ len        │ alloc/free │ buf[]                │
│ 已用长度   │ 可用容量    │ 实际字节内容          │
└────────────┴────────────┴──────────────────────┘
```

**SDS 的核心优点**

- O(1) 获取长度：直接读 header 里的 len，不需要像 `strlen` 一样扫描。
- 二进制安全：不依赖 `\0` 判断结束，可以存 JSON、ProtoBuf、MessagePack、图片字节、压缩后的字节。
- 减少频繁扩容：内部保存容量信息，空间够就直接追加，空间不够再扩容，思路类似 Go slice 的 `len/cap`。

**String 的对象编码**

```text
String 对象
├── int      -> 适合整数值，比如 "100"
├── embstr   -> 适合短字符串，Redis object 和 SDS 连续分配，减少内存分配次数
└── raw      -> 普通长字符串
```

**常见命令和复杂度**

```redis
SET user:1 '{"id":1,"name":"Tom"}'
GET user:1

INCR article:100:views
DECR stock:sku:1

SET lock:order:1 requestId NX EX 10
```

- `GET` / `SET` 通常是 O(1)。
- `INCR` / `DECR` 是原子自增自减，适合计数器。
- `SET key value NX EX seconds` 是分布式锁的基础命令。

**典型场景**

缓存对象：

```redis
SET user:1001 '{"id":1001,"name":"Tom","age":18}' EX 3600
GET user:1001
```

适合对象整体读写、字段不需要频繁单独修改、跨语言序列化方便的场景。

计数器：

```redis
INCR article:1001:views
INCR user:1001:likes
```

`INCR` 是 Redis 原子操作，多个客户端并发执行也不会丢更新。

分布式锁：

```redis
SET lock:order:1001 requestId NX EX 10
```

`NX` 表示 key 不存在才设置，`EX` 表示过期时间，`requestId` 是锁持有者唯一标识。释放锁时不能简单 `DEL`，要先判断 value 是否是自己的 requestId，再删除，否则可能误删别人的锁。

固定窗口限流：

```redis
INCR rate:user:1001:2026-07-10-10:00
EXPIRE rate:user:1001:2026-07-10-10:00 60
```

固定窗口实现简单，但有边界突刺问题，比如 10:00:59 和 10:01:00 各请求很多次，短时间内可能突破真实限流。

**风险**

- value 太大形成大 key。
- 整体更新需要反序列化和重新序列化。
- 局部字段更新成本高。
- 热点 key 访问压力大。
- 分布式锁实现不当会有误删风险。

**面试追问：String 的 SDS 和 C 字符串有什么区别？**
C 字符串以 `\0` 结尾，获取长度需要遍历，不是二进制安全的。SDS 自己保存长度和容量，所以获取长度是 O(1)，并且二进制安全，可以存任意字节内容。SDS 还可以通过容量信息减少频繁扩容。

### 5. Redis 的 Hash 适合存对象吗？

**结论先行：**
Hash 适合存储字段较少、结构扁平、需要局部读写的对象，例如用户基础信息、商品基础属性、购物车、配置项集合、计数字段集合。

**Hash 的结构**

```text
key = user:1001

field      value
------------------
name       Tom
age        18
city       Beijing
level      3
```

命令示例：

```redis
HSET user:1001 name Tom age 18 city Beijing
HGET user:1001 name
HGETALL user:1001
HINCRBY user:1001 level 1
```

图示：

```text
Redis key: user:1001
        |
        v
      Hash
┌──────────────┬──────────────┐
│ field        │ value        │
├──────────────┼──────────────┤
│ name         │ Tom          │
│ age          │ 18           │
│ city         │ Beijing      │
└──────────────┴──────────────┘
```

**Hash 底层编码**

```text
Hash
├── 小对象：listpack，紧凑、省内存
└── 大对象：hashtable，查询和更新效率更好
```

listpack 可以理解成一块连续紧凑内存：

```text
listpack:

┌────────┬────────┬────────┬────────┬────────┬────────┐
│ name   │ Tom    │ age    │ 18     │ city   │ BJ     │
└────────┴────────┴────────┴────────┴────────┴────────┘
```

优点是省内存、指针少、缓存友好；缺点是字段多时查找可能需要遍历，中间插入/删除成本相对更高。

当 Hash 变大后，会转成 hashtable：

```text
field hash -> bucket -> field/value
```

hashtable 字段查找快、字段更新快，适合字段数量较多的对象，但比 listpack 更耗内存，也有哈希表扩容成本。

**典型场景**

用户基础信息：

```redis
HSET user:1001 name Tom age 18 city Beijing
HGET user:1001 name
HSET user:1001 age 19
```

商品库存或属性：

```redis
HSET product:1001 name phone price 3999 stock 100
HINCRBY product:1001 stock -1
```

购物车：

```redis
HSET cart:user:1001 sku:1 2
HINCRBY cart:user:1001 sku:1 1
HDEL cart:user:1001 sku:2
```

结构：

```text
cart:user:1001
├── sku:1 -> 3
├── sku:2 -> 1
└── sku:3 -> 5
```

**String 和 Hash 怎么选？**

```text
String + JSON / ProtoBuf:
    适合整体读写、结构复杂、跨语言序列化。

Hash:
    适合字段扁平、字段级读写频繁、字段数量可控。
```

流程图：

```text
缓存一个对象
    |
    ├── 经常整体读写？
    |       └── String + JSON / ProtoBuf
    |
    ├── 经常单独读写某几个字段？
    |       └── Hash
    |
    ├── 对象字段很多？
    |       └── 谨慎 Hash，避免大 key
    |
    └── 对象结构复杂、有嵌套？
            └── String + JSON / ProtoBuf 更直接
```

**风险**

- 字段过多会形成大 key。
- `HGETALL` 大 Hash 会阻塞主线程。
- 删除、过期、迁移、复制大 Hash 都会变重。
- Redis 不支持 field 级 TTL，只能给整个 key 设置过期。
- 嵌套复杂对象不适合直接拆成 Hash。

**面试追问：Hash 扩容会不会阻塞？**
Redis 的字典扩容通常采用渐进式 rehash，不会一次性迁移全部数据，而是在后续操作中逐步迁移，减少单次阻塞风险。但如果 Hash 本身是超大 key，集中访问、删除、迁移、`HGETALL` 仍然可能造成延迟抖动。

### 6. List、Set、ZSet 分别适合什么场景？

**结论先行：**
List 是有序可重复列表，适合队列和最新列表；Set 是无序去重集合，适合标签、关注和集合运算；ZSet 是带 score 的有序集合，适合排行榜、延迟队列、滑动窗口限流和范围查询。

**总体对比**

```text
List：有序，可重复，适合队列和最新列表
Set：无序，唯一，适合去重和集合运算
ZSet：唯一，有 score，适合排序、排行榜、范围查询
```

**List：有序、可重复、两端操作**

List 支持两端 push/pop：

```redis
LPUSH queue task1
RPUSH queue task2

LPOP queue
RPOP queue
```

图示：

```text
left/head                         right/tail
   |                                  |
   v                                  v
┌──────┬──────┬──────┬──────┬──────┐
│  a   │  b   │  c   │  d   │  e   │
└──────┴──────┴──────┴──────┴──────┘

LPUSH 从左边插入
LPOP  从左边弹出
RPUSH 从右边插入
RPOP  从右边弹出
```

现代 Redis List 底层常见是 quicklist：

```text
quicklist = 双向链表 + listpack

┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ listpack    │ -> │ listpack    │ -> │ listpack    │
│ a b c d     │    │ e f g h     │    │ i j k l     │
└─────────────┘    └─────────────┘    └─────────────┘
```

如果只用链表，两端插入删除方便，但每个节点都有指针开销；如果只用连续数组，省内存但中间插入删除成本高。quicklist 的折中是：每个节点内部用 listpack 紧凑存储，节点之间用双向链表连接，兼顾内存和两端操作效率。

List 做简单队列：

```redis
LPUSH queue task1
LPUSH queue task2
BRPOP queue 0
```

风险：

- 缺少完善 ACK 机制。
- 消费者取出后宕机，消息可能丢。
- 不适合复杂消息追踪。
- 大 List 做 `LRANGE 0 -1` 可能很重。
- 队列堆积会形成大 key。

**Set：无序、唯一、集合运算**

Set 适合标签集合、关注集合、粉丝集合、抽奖去重、共同好友、黑名单、权限集合。

```redis
SADD user:1:tags go redis backend
SISMEMBER user:1:tags redis
SREM user:1:tags go
SMEMBERS user:1:tags

SINTER user:1:follows user:2:follows
SUNION set1 set2
SDIFF set1 set2
```

Set 底层编码常见有：

```text
intset      小整数集合
listpack    Redis 7.2+ 小集合可能使用
hashtable   普通集合
```

共同好友示例：

```redis
SADD user:1:follows u2 u3 u4
SADD user:2:follows u3 u4 u5
SINTER user:1:follows user:2:follows
```

结果：

```text
user:1:follows = {u2, u3, u4}
user:2:follows = {u3, u4, u5}

交集 = {u3, u4}
```

风险：

- 大 Set 做 `SMEMBERS` 会很重。
- 大 Set 做 `SINTER` / `SUNION` / `SDIFF` 可能阻塞。
- 集合元素太多会形成大 key。
- Set 无序，不适合需要排序的场景。

**ZSet：唯一、score 排序、范围查询**

ZSet 特点：

```text
member 唯一
每个 member 有一个 score
按 score 排序
支持排名和范围查询
```

适合排行榜、延迟队列、按时间排序的 Feed、滑动窗口限流、权重排序、范围检索、优先级队列。

常见命令：

```redis
ZADD rank 100 user:1
ZADD rank 200 user:2
ZADD rank 150 user:3

ZRANGE rank 0 -1 WITHSCORES
ZREVRANGE rank 0 9 WITHSCORES
ZSCORE rank user:2
ZREVRANK rank user:2
ZREM rank user:1
```

ZSet 常见两种编码：

```text
小 ZSet     -> listpack
普通 ZSet   -> skiplist + dict
```

普通 ZSet 可以理解成两套结构：

```text
ZSet
├── dict
│     └── member -> score
│
└── skiplist
      └── 按 score 排序的 member
```

图示：

```text
dict:
user:1 -> 100
user:2 -> 200
user:3 -> 150

skiplist:
score 100 -> user:1
score 150 -> user:3
score 200 -> user:2
```

为什么要 dict + skiplist 两套结构？

- `dict` 负责快速按 member 查 score，例如 `ZSCORE rank user:2`。
- `skiplist` 负责按 score 排序和范围扫描，例如 `ZRANGE`、`ZRANGEBYSCORE`。
- 这是典型的空间换时间。如果只有 dict，不能高效按 score 排序；如果只有 skiplist，根据 member 查 score 不够直接。

排行榜：

```redis
ZINCRBY game:rank 10 user:1001
ZREVRANGE game:rank 0 9 WITHSCORES
ZREVRANK game:rank user:1001
```

延迟队列：

```redis
ZADD delay:queue 1720000000 task:1
ZADD delay:queue 1720000010 task:2
ZRANGEBYSCORE delay:queue -inf now LIMIT 0 10
ZREM delay:queue task:1
```

这条查询命令可以读成：从 `delay:queue` 里取出 score 小于等于当前时间的前 10 个任务。

| 片段 | 是否动态 | 含义 |
|---|---|---|
| `ZRANGEBYSCORE` | 固定 | 按 ZSet score 范围查询 |
| `delay:queue` | 动态 | 延迟队列 key，可以按业务拆成 `delay:queue:{biz}` |
| `-inf` | 通常固定 | score 下界，表示从最早任务开始查 |
| `now` | 动态 | 当前时间戳；实际执行时要替换成具体时间，例如 `1720000050` |
| `LIMIT` | 固定 | 限制返回数量 |
| `0` | 通常固定 | 从结果第 0 条开始取 |
| `10` | 动态 | 本批最多取 10 条，避免一次拉太多 |

多消费者并发时，要避免重复消费，可以用 Lua 脚本保证“取出 + 删除”的原子性。

滑动窗口限流：

ZSet 做滑动窗口限流的核心是：

```text
key    = 限流维度
score  = 请求发生时间戳
member = 请求唯一 ID
```

推荐的核心命令顺序：

```redis
ZREMRANGEBYSCORE rate:user:1001 -inf now-60
ZCARD rate:user:1001
ZADD rate:user:1001 now requestId
EXPIRE rate:user:1001 60
```

逐条理解：

| 命令 | 作用 | 动态部分 |
|---|---|---|
| `ZREMRANGEBYSCORE rate:user:1001 -inf now-60` | 删除 60 秒窗口外的旧请求 | `rate:user:1001`、`now-60` |
| `ZCARD rate:user:1001` | 统计当前窗口内请求数 | `rate:user:1001` |
| `ZADD rate:user:1001 now requestId` | 未超限时写入当前请求 | `rate:user:1001`、`now`、`requestId` |
| `EXPIRE rate:user:1001 60` | 给限流 key 设置过期时间，避免冷 key 常驻 | `rate:user:1001`、`60` |

实际工程中，判断逻辑通常是：

```text
1. 删除窗口外请求
2. 统计窗口内请求数
3. 如果 count >= limit，拒绝
4. 如果 count < limit，写入当前请求并放行
5. 设置 TTL
```

这几步要用 Lua 包起来，保证“清理 + 统计 + 判断 + 写入 + 设置过期”原子执行，否则并发请求可能同时看到未超限并一起放行。

工程示例：调用 Facebook 获取 campaign 列表接口，多层限流：

```text
接口：GET /facebook/campaigns
限制：
    每秒最多 60 次
    每 5 分钟最多 3000 次
    每天最多 10000 次
```

可以按平台、账号、接口、窗口拆 key：

```text
rate:facebook:{account_id}:campaign_list:1s
rate:facebook:{account_id}:campaign_list:5m
rate:facebook:{account_id}:campaign_list:1d
```

每次请求同时检查三个窗口：

```text
窗口 1：1 秒，limit = 60
窗口 2：300 秒，limit = 3000
窗口 3：86400 秒，limit = 10000
```

伪流程：

```text
for each rule in [1s/60, 5m/3000, 1d/10000]:
    key = rate:facebook:{account_id}:campaign_list:{window}
    ZREMRANGEBYSCORE key -inf now-window
    count = ZCARD key
    if count >= limit:
        reject

for each rule:
    ZADD key now requestId
    EXPIRE key window + buffer
```

面试表达：

> ZSet 滑动窗口限流用 score 存请求时间，用 member 存请求唯一 ID。每次请求先清理窗口外数据，再统计当前窗口请求数，未超阈值才写入当前请求。像 Facebook campaign 列表这种外部 API，可以按平台、账号、接口维度做多层窗口，例如 1 秒 60 次、5 分钟 3000 次、1 天 1 万次。多个窗口必须同时通过才放行，并且用 Lua 保证原子性。

它比 String 固定窗口更平滑、更精确，但存储和计算成本更高，因此要控制 key 粒度、窗口数量、TTL 和单次清理规模。

**面试追问：排行榜为什么用 ZSet？**
因为 ZSet 同时支持 member 唯一、score 排序、按排名范围查询、按分数范围查询、更新分数、删除元素等操作。排行榜需要频繁更新分数、查 TopN、查个人排名，这些都和 ZSet 的能力高度匹配。

### 7. Bitmap、HyperLogLog、Bloom Filter 分别解决什么问题？

**结论先行：**
Bitmap 解决海量布尔状态，HyperLogLog 解决近似去重计数，Bloom Filter 解决高效判断“可能存在或一定不存在”。

```text
Bitmap:
    海量布尔状态

HyperLogLog:
    近似去重计数

Bloom Filter:
    判断元素是否可能存在
```

一句话区别：

```text
Bitmap 关心某个位置是 0 还是 1；
HyperLogLog 关心大概有多少个不同元素；
Bloom Filter 关心某个元素是否可能存在。
```

**Bitmap：海量布尔状态**

Bitmap 本质上不是一种独立的底层对象，而是基于 String 的 bit 操作。一个 bit 表示一个状态：

```text
0 = 否
1 = 是
```

用户签到：

```redis
SETBIT sign:user:1001 0 1
SETBIT sign:user:1001 1 0
GETBIT sign:user:1001 0
BITCOUNT sign:user:1001
```

图示：

```text
user:1001 本月签到情况：

day:  1 2 3 4 5 6 7 8
bit:  1 1 0 1 0 0 1 1

sign:user:1001
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ 1 │ 1 │ 0 │ 1 │ 0 │ 0 │ 1 │ 1 │
└───┴───┴───┴───┴───┴───┴───┴───┘
 1日 2日 3日 4日 5日 6日 7日 8日
```

优点是极省内存，适合海量用户布尔状态；缺点是只能表达 0/1，offset 过大可能导致底层 String 被撑大，不适合非常稀疏且 ID 极大的场景，通常需要对 ID 做映射或分片。

**HyperLogLog：近似去重计数**

HyperLogLog 适合做 UV、独立 IP、独立设备数、独立搜索关键词数。

```redis
PFADD uv:2026-07-10 user1
PFADD uv:2026-07-10 user2
PFADD uv:2026-07-10 user1

PFCOUNT uv:2026-07-10
```

即使 user1 添加两次，估算基数也会按去重后计算。它的优势是内存占用很小，适合允许少量误差的统计；限制是只能估算数量，不能列出元素，有误差，不能判断某个元素是否存在。

**Bloom Filter：判断是否可能存在**

Bloom Filter 用很少内存判断一个元素是否可能存在。它的结论只有两种：

```text
说不存在：
    一定不存在

说存在：
    可能存在
```

原理是多个 hash 函数把一个元素映射到多个 bit 位：

```text
添加 item:
item -> hash1 -> bit 3 = 1
     -> hash2 -> bit 8 = 1
     -> hash3 -> bit 15 = 1

查询 item:
如果对应 bit 位有一个是 0：
    一定不存在

如果对应 bit 位全是 1：
    可能存在
```

缓存穿透流程：

```text
请求 productId
    |
    v
Bloom Filter
    |
    ├── 不存在：直接返回空
    |
    └── 可能存在
            |
            v
          查 Redis 缓存
            |
            ├── 命中：返回
            └── 未命中：查 DB
```

风险：

- 有误判，少量不存在请求仍可能打到后端。
- 普通 Bloom Filter 删除困难，删除可能影响其他元素。
- 容量和误判率要提前规划，否则误判率会升高。
- 数据初始化和增量更新要做好。
- Bloom Filter 本身也要考虑可用性。

**面试追问：Bloom Filter 为什么说不存在一定不存在，存在只是可能存在？**
查询时，如果对应 bit 位有任何一个是 0，说明这个元素一定没被添加过，所以一定不存在。如果对应 bit 位全是 1，可能是这个元素设置的，也可能是其他元素碰撞设置的，所以只是可能存在。

### 8. Redis Stream 和 List 做消息队列有什么区别？

**结论先行：**
List 可以做简单队列，Stream 更像轻量消息队列，支持消息 ID、消费者组、ACK、Pending Entries List 和消息回溯。

**List 做队列**

```redis
LPUSH queue task1
BRPOP queue 0
```

流程：

```text
生产者
  |
  | LPUSH
  v
List 队列
  |
  | BRPOP
  v
消费者
```

优点是简单、高效、容易理解，适合轻量异步任务。缺点是没有内置消息 ID、没有消费者组、没有天然 ACK，消费者取出后宕机可能丢消息，也不方便追踪消息状态。可以用 `BRPOPLPUSH` 或 `BLMOVE` 做可靠队列增强，但复杂度会上升。

**Stream 做消息队列**

```redis
XADD mystream * user 1001 action login
XGROUP CREATE mystream group1 0 MKSTREAM
XREADGROUP GROUP group1 consumer1 COUNT 10 STREAMS mystream >
XACK mystream group1 messageId
```

Stream 消息结构：

```text
stream key: mystream

messageId                 fields
-----------------------------------------
1720000000000-0           user=1001 action=login
1720000000001-0           user=1002 action=pay
```

消费者组可以让多个消费者分摊消息：

```text
Stream
  |
  v
Consumer Group: group1
  ├── consumer1
  ├── consumer2
  └── consumer3
```

消费者读到消息后，如果还没 ACK，这条消息会进入 PEL：

```text
PEL = Pending Entries List

XREADGROUP 读取消息
    |
    v
消息进入 PEL
    |
    ├── 处理成功：XACK，从 PEL 删除
    └── 消费者宕机：消息仍在 PEL，可重试或转移
```

**选型建议**

```text
本地异步、小规模缓冲：
    List

需要 ACK、消费者组、消息回溯：
    Stream

强一致、高吞吐、复杂消息治理：
    Kafka / RocketMQ / Pulsar
```

**面试追问：Stream 能替代 Kafka 吗？**
不能简单替代。Stream 适合 Redis 生态内的轻量异步任务和中小规模消息流。Kafka 更适合大吞吐、分区扩展、持久日志、跨系统数据管道和复杂消息治理。选型上，如果只是业务内部轻量队列，Stream 可以；如果是核心消息中间件、大规模日志流或跨系统事件总线，应该选 Kafka、RocketMQ、Pulsar。

### 9. RDB 和 AOF 有什么区别？

**结论先行：**
RDB 是某个时间点的内存快照，恢复快、文件小，但可能丢数据；AOF 是写命令日志，数据安全性更高，但文件更大、恢复相对慢。

**分层展开：**

**RDB**
- 定期生成快照文件。
- 适合备份、灾备、快速恢复。
- 如果宕机，可能丢失上次快照后的数据。

**AOF**
- 追加记录写命令。
- `appendfsync always/everysec/no` 控制刷盘策略。
- `everysec` 常用，通常最多丢 1 秒数据。

**混合持久化**
- AOF rewrite 后文件前半部分可以是 RDB 格式，后半部分是增量 AOF。
- 兼顾恢复速度和数据安全。

**选择建议**
- 缓存场景可只开 RDB 或不开持久化。
- 重要数据建议 AOF everysec + RDB。
- 金融级数据不应只依赖 Redis 持久化。

**面试追问：AOF 重写会不会阻塞主线程？**
AOF rewrite 由子进程完成，主线程继续处理请求。但 fork、写时复制、磁盘 IO 和 rewrite buffer 仍可能带来延迟和内存压力。

### 10. Redis 过期删除策略是什么？

**结论先行：**
Redis 采用惰性删除 + 定期删除结合的策略，兼顾 CPU 开销和内存回收效率。

**分层展开：**

**惰性删除**
- 访问 key 时检查是否过期。
- 如果过期就删除。
- 优点是省 CPU，缺点是没人访问的过期 key 可能滞留内存。

**定期删除**
- Redis 周期性抽样检查带过期时间的 key。
- 删除其中已过期的 key。
- 避免过期 key 长期占用内存。

**内存淘汰**
- 如果内存达到 `maxmemory`，触发淘汰策略。
- 淘汰和过期不是一回事：过期是 TTL 到期，淘汰是内存不够。

**面试追问：为什么不定时精确删除？**
给每个 key 设置定时器会消耗大量 CPU 和内存，海量 key 场景下不可接受。

### 11. Redis 有哪些内存淘汰策略？

**结论先行：**
Redis 淘汰策略围绕“从哪些 key 中淘汰”和“按什么规则淘汰”两条线展开，生产缓存场景常用 `allkeys-lru`、`volatile-lru` 或 LFU 类策略。

**分层展开：**

**不淘汰**
- `noeviction`：内存满后写入报错。
- 适合不能丢数据的场景。

**在所有 key 中淘汰**
- `allkeys-lru`：淘汰最近最少使用。
- `allkeys-lfu`：淘汰使用频率低的。
- `allkeys-random`：随机淘汰。

**只在设置 TTL 的 key 中淘汰**
- `volatile-lru`。
- `volatile-lfu`。
- `volatile-random`。
- `volatile-ttl`：优先淘汰快过期的。

**选择建议**
- 纯缓存系统常用 `allkeys-lru` 或 `allkeys-lfu`。
- 混合存储重要数据和缓存时要谨慎，避免重要 key 被淘汰。
- 最好不同用途拆实例。

**面试追问：Redis 的 LRU 是精确的吗？**
不是完全精确 LRU，而是近似 LRU。Redis 通过采样若干 key 后淘汰其中最久未访问的，平衡性能和效果。

### 12. Redis 主从复制是怎么工作的？

**结论先行：**
Redis 主从复制分为全量复制和增量复制，从节点通过复制主节点数据实现读扩展和高可用基础，但复制是异步的，存在延迟和数据丢失风险。

**分层展开：**

**全量复制**
- 从节点第一次连接主节点。
- 主节点生成 RDB 快照并发送给从节点。
- 同步期间新增写命令写入复制缓冲区，再发送给从节点。

**增量复制**
- 主从短暂断开后，从节点携带复制偏移量重新连接。
- 如果主节点复制积压缓冲区还保留缺失数据，就只补发增量命令。

**异步复制风险**
- 主节点写成功后不等待从节点确认。
- 主节点宕机时，未复制的数据可能丢失。

**工程建议**
- 读从库要接受短暂旧数据。
- 关键写读一致场景读主库。
- 配置合理的复制 backlog，减少全量复制。

**面试追问：主从复制能提高写性能吗？**
不能直接提高单 key 写性能。写仍走主节点，从节点主要用于读扩展、备份和高可用切换。

### 13. Redis Sentinel 哨兵解决什么问题？

**结论先行：**
Sentinel 主要解决主从架构下的自动故障发现、主节点选举、故障转移和客户端发现新主节点的问题。

**分层展开：**

**监控**
- 哨兵持续 ping 主从节点。
- 判断主观下线和客观下线。

**选主**
- 主节点客观下线后，哨兵从从节点中选一个新主。
- 选择依据包括优先级、复制偏移量、runid 等。

**故障转移**
- 将选出的从节点提升为主。
- 让其他从节点复制新主。
- 通知客户端新的主节点地址。

**局限**
- 不解决水平扩容。
- 异步复制仍可能丢数据。
- 故障切换期间存在短暂不可用。

**面试追问：哨兵至少几个节点？**
生产通常至少 3 个哨兵节点，避免单点和误判，并通过多数派完成客观下线判断和 leader 选举。

### 14. Redis Cluster 如何实现分片？

**结论先行：**
Redis Cluster 通过 16384 个 hash slot 实现数据分片，每个 key 根据 CRC16 计算所属 slot，再映射到具体 master 节点。

**分层展开：**

**slot 分片**
- 全集群共有 16384 个 slot。
- 每个 master 负责一部分 slot。
- key 通过 CRC16(key) % 16384 定位 slot。

**请求路由**
- 客户端访问错误节点时，Redis 返回 MOVED 或 ASK。
- 智能客户端会缓存 slot 到节点的映射。

**高可用**
- 每个 master 可以有 replica。
- master 故障后，replica 被提升为新 master。

**扩容缩容**
- 本质是迁移 slot。
- 迁移期间客户端可能遇到 ASK 重定向。

**面试追问：Cluster 支持多 key 操作吗？**
只支持同一个 slot 内的多 key 操作。可以用 hash tag，例如 `{user:1}:profile` 和 `{user:1}:orders` 会落到同一 slot，但不要滥用，否则会导致数据倾斜。

### 15. Redis Pipeline 是什么？和事务有什么区别？

**结论先行：**
Pipeline 是客户端批量发送命令以减少网络往返，不能保证原子性；事务是 Redis 服务端按顺序执行命令队列，强调隔离执行。

**分层展开：**

**Pipeline**
- 多个命令一次发送。
- Redis 依次执行后批量返回。
- 主要优化网络 RTT 和客户端、服务端的 read/write 系统调用次数。
- 不保证原子性，也不会让 Redis 并行执行命令。

**普通请求/响应**

```text
write cmd1 -> read resp1
write cmd2 -> read resp2
write cmd3 -> read resp3
```

每条命令都要等待一次网络往返，请求量大时大量时间消耗在 RTT 和系统调用上。

**Pipeline 请求/响应**

```text
write cmd1+cmd2+cmd3 -> read resp1+resp2+resp3
```

客户端不用等上一条响应就继续发送后续命令，最后统一读取响应。严格来说，不一定永远只有一次 `read/write`，因为还受 TCP 缓冲区、命令大小、客户端库实现影响，但它会把多次网络往返和系统调用压缩成少数几次。

**事务**
- `MULTI/EXEC`。
- 保证事务队列执行期间不被插入其他命令。
- 不等于数据库事务。

**使用建议**
- 批量读写用 Pipeline。
- 多步状态变更需要原子性用 Lua 或事务。
- Pipeline 批量不能无限大，否则会占用内存和网络缓冲。

**面试追问：Pipeline 在 Cluster 下有什么注意点？**
不同 key 可能落到不同节点，客户端需要按 slot 拆分 pipeline 并发送到对应节点。多 key 原子操作仍要求同 slot。

**面试追问：Pipeline 和 IO 多路复用是什么关系？**
IO 多路复用是 Redis 服务端高效管理大量 socket 连接的网络模型；Pipeline 是客户端批量发送命令、批量读取响应的使用方式。前者解决“一个线程怎么管理很多连接”，后者解决“一个连接上多条命令怎么减少 RTT 和系统调用”。

### 16. Redis 为什么不建议使用 KEYS？

**结论先行：**
`KEYS` 会遍历整个 keyspace，是 O(N) 命令，在生产大实例上可能阻塞主线程，导致请求延迟飙升。

**分层展开：**

**问题**
- 全量扫描所有 key。
- 单线程执行期间其他请求排队。
- 大实例上耗时不可控。

**替代方案**
- 用 `SCAN` 渐进式扫描。
- 业务侧维护索引集合。
- 按前缀和业务维度规划 key。

**SCAN 注意点**
- `SCAN` 不是一次完整快照。
- 可能返回重复 key。
- 扫描期间新增或删除 key 会影响结果。

**面试追问：SCAN 会阻塞吗？**
单次 SCAN 只扫描一小部分，阻塞风险远小于 KEYS，但如果 count 过大、频率过高或 value 后续操作很重，仍会影响线上。

### 17. Redis key 如何设计？

**结论先行：**
好的 key 设计要做到可读、可控、可分片、可治理，避免过长、过大、无 TTL 和热点集中。

**分层展开：**

**命名规范**
- 使用业务前缀：`user:profile:{uid}`。
- 层次清晰：`biz:module:type:id`。
- 避免含糊和冲突。

**长度控制**
- key 太长会浪费内存和网络。
- key 太短又不利于排查。
- 在可读性和成本之间平衡。

**过期策略**
- 缓存 key 必须有 TTL。
- TTL 加随机值。
- 永不过期 key 要有明确理由。

**Cluster 友好**
- 需要同 slot 的 key 使用 hash tag。
- 避免滥用 hash tag 造成倾斜。

**治理能力**
- 可按前缀统计。
- 可定位 owner。
- 可批量清理。

**面试追问：key 越短越好吗？**
不是。key 太短排查困难、容易冲突。大厂更看重可治理性，通常会牺牲少量内存换取可读性和可维护性。

## 二、核心能力原理分析：为什么这样设计

> 这一章回答“Redis 为什么这样做”。面试时从机制、复杂度、风险边界讲，避免只背结论。

### 18. Redis 为什么这么快？

**结论先行：**
Redis 快的核心原因是“纯内存 + 单线程命令执行 + IO 多路复用 + 高效数据结构 + 工程级优化”。

**分层展开：**

**纯内存访问**
- 大部分操作在内存中完成，避免磁盘随机 IO。
- 常见命令复杂度为 O(1) 或 O(logN)。

**单线程执行命令**
- Redis 主要用单线程执行用户命令，避免多线程锁竞争、上下文切换和复杂并发控制。
- 单线程不是没有并发，而是网络连接很多，命令执行串行。

**IO 多路复用**
- 通过 epoll/kqueue/select 等机制，一个线程可以监听大量 socket。
- 连接就绪后再读取和处理，避免一个连接阻塞整个服务。

**高效数据结构**
- SDS、dict、ziplist/listpack、quicklist、skiplist、intset 等针对场景做了内存和性能优化。

**后台线程辅助**
- Redis 6 之后引入多线程 IO，主要用于网络读写，命令执行仍以单线程为主。
- lazy free、AOF fsync、关闭文件等任务可后台处理，减少主线程阻塞。

**面试追问：Redis 单线程为什么还能支持高并发？**
因为 Redis 的瓶颈通常不是 CPU，而是网络和内存访问。单线程执行命令很快，配合 IO 多路复用可以高效处理大量连接。但如果命令本身很重，例如大 key、复杂 Lua、全量扫描，仍会阻塞主线程。

### 19. Redis 网络 IO 与 Pipeline 应该怎么讲？

**结论先行：**
Redis 的网络模型可以简化成：主线程通过 IO 多路复用监听大量 socket，就绪后读取请求、执行命令、写回响应；Pipeline 通过批量发送命令和批量读取响应减少 RTT 和系统调用次数，但不会让 Redis 并行执行命令。

**面试回答主线：**

1. Redis 命令执行主要是单线程，避免锁竞争。
2. 网络连接很多，但不是一个连接一个线程，而是用 epoll/kqueue/select 这类 IO 多路复用机制管理。
3. `GET/SET` 的读写是 Redis 数据语义；socket 可读/可写是网络事件；`read/write` 是系统调用，不要混在一起。
4. Pipeline 不是事务，也不是并行，只是把多条命令批量发送，减少网络往返。
5. Pipeline 批次不能无限大，否则会造成客户端/服务端缓冲区压力和延迟抖动。

**详细拆解位置：**
网络 IO、socket 可读/可写、`read/write` 系统调用、Pipeline 完整链路已经拆到：
[技术｜Linux网络与线上排障面试题.md](/Users/guoyimeng/Documents/Coding/Interviewing/技术｜Linux网络与线上排障面试题.md)

**面试追问：Pipeline 是否保证原子性？**
不保证。Pipeline 只减少网络往返，Redis 仍按命令顺序逐条执行。如果中间有其他客户端命令插入，Pipeline 本身不提供事务隔离；需要原子性时用 Lua 或事务。

### 20. ZSet 为什么用跳表而不是红黑树？

**结论先行：**
Redis ZSet 选择跳表，是因为跳表实现简单、范围查询自然、性能足够稳定，配合 dict 可以很好满足 ZSet 的需求。

**跳表是什么？**

跳表可以理解成多层有序链表，上层是索引，底层保存完整数据：

```text
Level 3:  1 ----------------------> 9
Level 2:  1 ----------> 5 --------> 9
Level 1:  1 -> 3 -> 5 -> 7 -> 9
```

查找 `7` 时，从高层开始跳，跳不过去就下降一层，最后在底层找到目标。

平均复杂度：

```text
查找 O(logN)
插入 O(logN)
删除 O(logN)
范围遍历 O(logN + M)
```

其中 M 是返回元素数量。

**为什么不用红黑树？**

范围查询更自然：

```redis
ZRANGE rank 0 9
ZRANGEBYSCORE rank 100 200
ZREVRANGE rank 0 9
```

跳表找到起点后，可以沿底层链表顺序往后扫：

```text
score: 100 -> 120 -> 150 -> 180 -> 220
              ↑----------------↑
                 范围查询结果
```

红黑树也可以做范围查询，但实现上通常要做中序遍历，节点关系维护更复杂。

实现更简单：

- 红黑树需要旋转、染色和复杂平衡逻辑。
- 跳表通过随机层高维持概率平衡，代码更容易维护。

性能足够稳定：

- 跳表理论上有退化概率，但概率极低。
- Redis 通过随机层数、最大层数限制和合理概率参数，让跳表在工程上保持接近 O(logN)。

**面试追问：ZSet 为什么还要配 dict？**
因为跳表不擅长通过 member 直接定位 score。比如 `ZSCORE rank user:1001`，用 dict 做 `member -> score` 很快；skiplist 负责 score 排序和范围查询。两者配合是空间换时间。

### 21. 缓存和数据库如何保证一致性？

**结论先行：**
缓存与数据库无法在常规架构下做到绝对强一致，工程上通常采用 Cache Aside，通过“先更新数据库，再删除缓存”，配合重试、延迟双删和过期兜底实现最终一致。

**分层展开：**

**读路径**
- 先读缓存。
- 缓存命中直接返回。
- 缓存未命中读数据库，再写入缓存。

**写路径**
- 先更新数据库。
- 再删除缓存，而不是直接更新缓存。
- 删除失败需要重试或异步补偿。

**为什么不是先删缓存再写数据库**
- 先删缓存后，另一个读请求可能读到旧数据库值并回填缓存。
- 后续数据库更新完成后，缓存里反而留下旧值。

**为什么不是更新数据库后更新缓存**
- 并发写时可能乱序覆盖。
- 缓存结构可能是复杂聚合，更新成本高。
- 删除缓存让下次读取重新加载，逻辑更简单。

**兜底手段**
- TTL 过期。
- 删除失败重试。
- binlog/MQ 异步删除。
- 热点数据短暂互斥重建。

**面试追问：先更新数据库再删除缓存，删除失败怎么办？**
要把删除缓存变成可重试动作，例如写入 MQ、订阅 binlog、记录重试表，或者由异步任务补偿。TTL 是最后兜底，但不能只依赖 TTL，否则不一致窗口过长。

### 22. 延迟双删是什么？什么时候用？

**结论先行：**
延迟双删是在写数据库前后分别删除一次缓存，并在第二次删除前等待一小段时间，用来降低并发读回填旧值的概率。

**分层展开：**

**基本流程**
- 删除缓存。
- 更新数据库。
- sleep 一小段时间。
- 再删除缓存。

**解决的问题**
- 第一次删除后，如果并发读从旧数据库读出数据并写回缓存，第二次删除可以清理这份旧缓存。

**关键难点**
- sleep 时间不好设置。
- 时间太短，读请求可能还没完成。
- 时间太长，会增加写链路耗时。

**工程建议**
- 高并发核心链路不建议同步 sleep 太久。
- 更推荐异步延迟消息、binlog 订阅、缓存删除重试。

**面试追问：延迟双删能保证强一致吗？**
不能，只是降低不一致概率。真正强一致要引入事务、锁、串行化或读写都走同一一致性系统，但成本通常很高。

### 23. 缓存穿透、击穿、雪崩分别是什么？怎么解决？

**结论先行：**
穿透是查不存在数据，击穿是热点 key 过期，雪崩是大量 key 同时失效或 Redis 故障。三者本质都是请求绕过缓存打到后端，需要分别用空值缓存、互斥重建、过期随机化和降级限流解决。

**分层展开：**

**缓存穿透**
- 现象：请求的数据缓存没有，数据库也没有。
- 风险：恶意或异常请求持续打 DB。
- 方案：缓存空值、Bloom Filter、参数校验、黑名单限流。

**缓存击穿**
- 现象：一个热点 key 过期，大量请求同时打 DB。
- 风险：瞬时 DB 压力暴涨。
- 方案：互斥锁重建、逻辑过期、热点 key 不过期、后台刷新。

**缓存雪崩**
- 现象：大量 key 同时失效，或者 Redis 集群不可用。
- 风险：后端服务和数据库被流量压垮。
- 方案：TTL 加随机值、多级缓存、限流降级、熔断、Redis 高可用。

**面试追问：热点 key 你会用互斥锁还是逻辑过期？**
强一致要求更高、可接受少量等待时用互斥锁；更关注可用性和低延迟时用逻辑过期，先返回旧值，后台异步刷新。

### 24. Redis 如何实现分布式锁？

**结论先行：**
Redis 分布式锁的基础实现是 `SET lockKey requestId NX EX seconds`，释放时用 Lua 脚本校验 requestId 后删除，保证“加锁原子、解锁不误删”。

**分层展开：**

**加锁**
- `SET key value NX EX ttl`。
- `NX` 表示不存在才设置。
- `EX/PX` 设置过期时间，防止客户端宕机后死锁。
- value 必须是唯一标识，例如 UUID + 线程 ID。

**解锁**
- 不能直接 `DEL key`。
- 要先判断 value 是否属于自己，再删除。
- 判断和删除必须用 Lua 脚本保证原子性。

**锁超时**
- TTL 太短会导致业务没执行完锁就过期。
- TTL 太长会导致异常时阻塞时间过长。
- 可使用看门狗续期，但要防止无限续期。

**适用边界**
- 适合防重复提交、热点资源互斥、缓存重建。
- 不适合金融级强一致事务。

**面试追问：Redis 锁一定安全吗？**
不是绝对安全。主从异步复制下，主节点加锁成功但还没复制到从节点就宕机，故障切换后可能出现锁丢失。强一致锁要考虑 ZooKeeper、etcd 或数据库事务锁。

### 25. Redlock 是什么？你会怎么评价？

**结论先行：**
Redlock 是 Redis 官方提出的多节点分布式锁算法，通过在多个独立 Redis 节点上加锁并获得多数成功来提升可靠性，但业界对其强一致性仍有争议。

**分层展开：**

**基本思想**
- 客户端向多个 Redis 实例尝试加锁。
- 获得多数节点成功才认为加锁成功。
- 锁有效时间要扣除网络耗时。

**优点**
- 比单 Redis 节点锁更能抵抗单点故障。
- 不依赖 Redis 主从复制。

**争议**
- 网络分区、时钟漂移、GC 停顿可能导致安全问题。
- 如果业务要求严格互斥，Redlock 也不是形式化强一致方案。

**工程建议**
- 普通互联网业务互斥可用 Redis 锁。
- 强一致场景优先 ZooKeeper/etcd。
- 关键业务要配合 fencing token，防止旧持锁者继续写。

**面试追问：什么是 fencing token？**
它是单调递增的令牌。每次成功拿锁获得更大的 token，下游资源只接受更大的 token，拒绝旧 token，避免锁过期后旧客户端继续写入造成覆盖。

### 26. Redis 锁的过期时间怎么设置？

**结论先行：**
锁过期时间要覆盖 P99 业务执行时间，并预留网络和 GC 抖动，同时不能长到影响故障恢复；复杂任务需要续期机制或任务拆分。

**分层展开：**

**评估业务耗时**
- 用真实监控看 P95/P99，而不是拍脑袋。
- TTL 至少大于大部分正常执行时间。

**考虑异常抖动**
- GC、网络超时、慢 SQL、下游重试都会拉长耗时。
- 高峰期和低峰期耗时可能完全不同。

**控制锁粒度**
- 锁粒度过大会降低并发。
- 锁粒度过细会增加管理成本和死锁风险。

**续期与兜底**
- Redisson 看门狗会自动续期。
- 续期要绑定持锁线程，业务结束后及时释放。
- 任务太长时考虑拆分或改为状态机。

**面试追问：锁过期业务还没执行完怎么办？**
会出现并发执行风险。解决思路是合理 TTL、续期机制、幂等写入、版本号或 fencing token。不能只靠锁本身兜住所有一致性问题。

### 27. Redis 事务和数据库事务有什么区别？

**结论先行：**
Redis 事务通过 `MULTI/EXEC` 保证命令顺序执行，但不支持数据库那种完整 ACID 回滚语义。

**分层展开：**

**Redis 事务特性**
- `MULTI` 开启事务。
- 命令进入队列。
- `EXEC` 按顺序执行。
- 执行期间不会被其他客户端命令插入。

**不支持完整回滚**
- 命令运行时报错，不会回滚前面已经执行的命令。
- 编译期错误会导致事务无法执行。

**乐观锁**
- `WATCH` 监控 key。
- 如果 key 被其他客户端修改，`EXEC` 失败。

**使用建议**
- 简单原子组合可用事务。
- 复杂逻辑优先 Lua。
- 跨系统一致性不要依赖 Redis 事务。

**面试追问：为什么 Redis 事务不支持回滚？**
Redis 命令执行失败通常是编程错误或类型错误，官方设计上更强调简单和性能。如果需要复杂事务语义，应该由业务或数据库保证。

### 28. Lua 脚本有什么优势和风险？

**结论先行：**
Lua 脚本的优势是把多步操作放到 Redis 内原子执行，减少网络往返；风险是脚本执行时间过长会阻塞主线程。

**分层展开：**

**优势**
- 原子执行。
- 减少多次网络请求。
- 适合限流、锁释放、库存扣减、状态机转换。

**风险**
- 长脚本阻塞 Redis。
- 访问大 key 或循环过多会造成延迟。
- 脚本调试和版本管理更复杂。

**工程规范**
- 控制脚本复杂度。
- 入参显式传入，不在脚本中扫大量 key。
- 使用 SHA 缓存脚本。
- 设置脚本超时治理预案。

**面试追问：Lua 脚本执行中能被打断吗？**
Redis 保证脚本原子执行。长时间脚本会阻塞其他命令，必要时可通过 `SCRIPT KILL` 处理未写入脚本；如果脚本已执行写操作，处理会更复杂。

### 29. Redis fork 子进程时有什么风险？

**结论先行：**
Redis fork 子进程用于 RDB、AOF rewrite 等操作，风险主要是 fork 耗时、写时复制导致内存放大、磁盘 IO 影响性能。

**分层展开：**

**fork 耗时**
- Redis 内存越大，页表越大，fork 越慢。
- fork 期间主线程可能短暂阻塞。

**写时复制**
- 子进程生成快照时，父进程继续写数据。
- 被修改的内存页会复制一份，导致内存短时上涨。

**磁盘 IO**
- RDB 保存和 AOF rewrite 都会产生大量磁盘写。
- 如果磁盘慢，会影响 fsync 和整体延迟。

**优化建议**
- 控制单实例内存大小。
- 避免高峰期做重写和全量备份。
- 监控 `latest_fork_usec`、内存使用和磁盘 IO。

**面试追问：为什么 Redis 实例不建议特别大？**
实例太大会让 fork、迁移、恢复、主从全量同步、故障切换都变慢。大厂通常会做分片和容量上限治理。

### 30. Redis Cluster 脑裂是什么？怎么避免？

**结论先行：**
脑裂是旧主节点与新主节点同时对外提供写服务，导致数据分叉。避免脑裂要依赖多数派、合理超时、从节点复制限制和客户端写入保护。

**分层展开：**

**产生原因**
- 网络分区导致主节点与集群多数节点失联。
- 集群另一侧提升从节点为新主。
- 旧主如果仍被客户端访问并接受写入，就产生脑裂。

**Redis 的保护**
- Cluster 依赖多数 master 判断故障。
- 主节点失去多数派后应停止服务。
- 配置 `min-replicas-to-write` 和 `min-replicas-max-lag` 可以降低主从复制异常时继续写的风险。

**业务侧治理**
- 客户端及时刷新拓扑。
- 写请求走集群客户端，不绕过路由。
- 对关键写入加版本号或幂等校验。

**面试追问：脑裂一定能完全避免吗？**
分布式系统里不能只靠某个配置绝对避免所有异常。工程上要降低概率，并让业务写入具备幂等、版本保护和可补偿能力。

## 三、Redis 实战设计：会用、用好、知道边界

> 这一章回答“怎么用 Redis 解决业务问题”。重点是数据结构选择、key 设计、TTL、一致性、原子性、失败兜底。

### 31. 如何设计一个高并发商品详情缓存？

**结论先行：**
商品详情缓存要按“多级缓存 + 热点保护 + 最终一致 + 降级兜底”设计，不能只写一个 `get/set`。

**分层展开：**

**读链路**
- 本地缓存 Caffeine/Guava。
- Redis 分布式缓存。
- 数据库或商品服务。
- 本地缓存 TTL 短，Redis TTL 稍长。

**写链路**
- 商品变更先落库。
- 发 MQ 或 binlog 事件。
- 异步删除 Redis 和本地缓存。
- 删除失败重试。

**热点保护**
- 热点商品预热。
- 逻辑过期或互斥锁重建。
- 大促前固定热点 key 不设置短 TTL。
- 对 DB 回源做并发限制。

**降级策略**
- Redis 故障时返回本地旧值。
- 非核心字段可降级为空。
- 保护数据库连接池和线程池。

**监控指标**
- 缓存命中率。
- 回源 QPS。
- 热 key 分布。
- Redis 延迟和慢查询。
- DB 查询耗时。

**面试追问：商品价格库存能不能和详情一起缓存？**
一般不建议混在一个大缓存里。价格、库存变化频率和一致性要求更高，应该拆分缓存或直接走专门服务，避免一个字段频繁变化导致整个详情缓存频繁失效。

### 32. 如何用 Redis 设计排行榜？

**结论先行：**
排行榜优先用 ZSet，score 表示排序分数，member 表示用户或对象 ID，配合定时归档和冷热拆分解决容量问题。

**分层展开：**

**基础操作**
- 更新分数：`ZADD rank score userId`。
- 增加分数：`ZINCRBY rank delta userId`。
- 查前 N 名：`ZREVRANGE rank 0 N WITHSCORES`。
- 查个人排名：`ZREVRANK rank userId`。

**时间维度**
- 日榜：`rank:daily:2026-07-08`。
- 周榜：`rank:weekly:2026-W28`。
- 总榜：`rank:total`。

**容量治理**
- 只保留 top N 或活跃用户。
- 历史榜单落库归档。
- 大榜单分业务、地区、时间拆分。

**一致性**
- 分数变更可以先写 Redis，再异步落库。
- 或写业务事件，由消费者更新 Redis 和 DB。
- 要保证事件幂等。

**面试追问：排行榜同分怎么排序？**
Redis ZSet score 相同时按 member 字典序排序。如果需要按时间先后排序，可以把 score 设计成复合分数，或在 member/额外结构中记录时间并由业务层处理。

### 33. 如何用 Redis 做限流？

**结论先行：**
Redis 限流可以实现固定窗口、滑动窗口、令牌桶和漏桶；简单场景用 `INCR + EXPIRE`，精确滑动窗口用 ZSet + Lua。

**分层展开：**

**固定窗口**
- key 按用户和时间窗口命名。
- `INCR` 后设置过期时间。
- 简单高效，但窗口边界可能突刺。

**滑动窗口**
- 用 ZSet 存请求时间戳。
- 删除窗口外数据。
- 统计窗口内请求数。
- 判断是否放行。
- 外部 API 场景可以做多层窗口，例如 Facebook campaign 列表接口同时检查 `1s/60`、`5m/3000`、`1d/10000`，任一窗口超限就拒绝。

**令牌桶**
- 按固定速率生成令牌。
- 请求消耗令牌。
- 允许一定突发流量。

**工程建议**
- 限流逻辑用 Lua 保证原子。
- key 设置 TTL，避免无限增长。
- 高 QPS 全局限流不要让 Redis 成为瓶颈，可结合本地限流。
- key 建议包含平台、账号、接口和窗口，例如 `rate:facebook:{account_id}:campaign_list:1s`，方便隔离不同媒体账号和接口。

**面试追问：为什么限流要用 Lua？**
因为限流通常包含读计数、判断、写计数、设置过期等多个步骤。Lua 能保证这些步骤在 Redis 中原子执行，避免并发穿透。

### 34. 如何用 Redis 实现延迟队列？

**结论先行：**
延迟队列可以用 ZSet 实现，score 存执行时间戳，消费者轮询取出到期任务，再用原子操作删除并处理。

**分层展开：**

**写入任务**
- `ZADD delay_queue executeAt taskId`。
- task 详情可以存 String/Hash 或数据库。

**消费任务**
- 查询 score <= 当前时间的任务。
- 获取少量任务后尝试删除。
- 删除成功才执行业务，防止多个消费者重复处理。

**可靠性**
- 任务处理要幂等。
- 失败任务要重试或进入死信。
- 消费和删除最好用 Lua 或事务控制。

**适用边界**
- 适合中小规模延迟任务。
- 大规模、高可靠延迟消息建议 RocketMQ、Kafka 延迟方案或专门调度系统。

**面试追问：消费者宕机导致任务丢失怎么办？**
不要先删除再处理，或者需要处理中的队列和超时回收机制。更常见是删除成功后业务处理失败，通过幂等和重试表补偿。

### 35. 如何用 Redis 实现点赞系统？

**结论先行：**
点赞系统要拆成“是否点赞、点赞数、点赞列表”三个问题，分别用 Set/String/ZSet 或数据库事件流组合实现。

**分层展开：**

**是否点赞**
- `SADD like:post:{id} userId`。
- `SISMEMBER` 判断用户是否点赞。

**点赞数**
- 小规模可用 `SCARD`。
- 高并发建议独立计数器 `INCR/DECR`。

**点赞列表**
- 需要按时间排序时用 ZSet，score 为时间戳。
- 只需要去重集合时用 Set。

**落库策略**
- Redis 承接高频写。
- 异步批量落库。
- 用户重复点击要做幂等。

**面试追问：点赞取消和重复点击如何保证正确？**
使用 `SADD`/`SREM` 的返回值判断状态是否真的变化，再决定是否增减计数。计数更新和集合更新可用 Lua 保证原子。

### 36. 如何设计 Redis 缓存预热？

**结论先行：**
缓存预热是把预计会被高频访问的数据提前加载到 Redis，核心是识别热点、控制节奏、校验结果和提供回滚。

**分层展开：**

**热点识别**
- 历史访问 TopN。
- 大促活动商品。
- 推荐位、首页、榜单。

**加载方式**
- 发布前批量加载。
- 定时任务刷新。
- MQ 事件触发。
- 灰度预热。

**节奏控制**
- 分批写入，避免 Redis 和 DB 被打满。
- 控制并发和 QPS。
- 设置合理 TTL 和随机过期。

**验证和回滚**
- 校验 key 数量、命中率、数据版本。
- 预热失败时保留旧缓存或降级。

**面试追问：预热会不会造成缓存污染？**
会。如果预热了大量实际不访问的数据，会浪费内存并挤掉真正热点。要根据访问数据动态调整预热范围。

### 37. Redis 和本地缓存怎么配合？

**结论先行：**
本地缓存解决极致低延迟和热 key 压力，Redis 解决分布式共享缓存，两者结合是多级缓存，但要额外处理本地缓存一致性。

**分层展开：**

**架构**
- 请求先查本地缓存。
- 未命中查 Redis。
- Redis 未命中查 DB。

**优点**
- 降低 Redis QPS。
- 降低网络开销。
- 提升热点数据访问性能。

**一致性问题**
- 每个应用实例都有本地副本。
- 数据变更时需要广播失效或设置短 TTL。
- 本地缓存不适合强一致数据。

**典型场景**
- 配置项。
- 商品详情中的低频变化字段。
- 热点榜单。
- 字典数据。

**面试追问：本地缓存怎么失效？**
常见做法是短 TTL、MQ 广播、配置中心推送、Redis Pub/Sub 或版本号校验。核心是接受短暂不一致，并把窗口控制在业务可接受范围内。

### 38. 缓存数据要不要存 null？

**结论先行：**
可以缓存 null，但要设置较短 TTL，用于防止缓存穿透；不能长期缓存 null，否则会掩盖后续新增数据。

**分层展开：**

**适用场景**
- 查询不存在的用户、商品、订单。
- 防止相同不存在 key 反复打 DB。

**注意事项**
- TTL 要短，例如几十秒到几分钟。
- 数据新增时要删除 null 缓存。
- 区分“空值”和“系统异常”。

**替代方案**
- Bloom Filter。
- 参数校验。
- 黑名单和风控。

**面试追问：数据库异常时能不能缓存 null？**
不能。数据库异常和数据不存在是两回事。异常时缓存 null 会把临时故障变成错误缓存，导致恢复后仍返回不存在。

### 39. Redis 适合做数据库吗？

**结论先行：**
Redis 可以存数据，但不适合替代通用关系型数据库。它更适合缓存、计数、排行榜、会话、队列、限流等高性能内存场景。

**分层展开：**

**优势**
- 高性能。
- 丰富数据结构。
- 原子操作。
- TTL 和过期机制。

**短板**
- 内存成本高。
- 查询能力弱。
- 事务能力有限。
- 持久化和一致性不如数据库。

**适合场景**
- 可重建缓存。
- 中间状态。
- 低延迟访问。
- 允许最终一致的数据。

**不适合场景**
- 强事务。
- 复杂查询。
- 大规模低价值冷数据。
- 需要严格审计的数据。

**面试追问：Redis 持久化开了就能当数据库吗？**
不能。持久化只解决部分恢复问题，不等于完整事务、强一致、复杂查询和数据治理能力。

## 四、线上治理与故障排查：发现问题、定位问题、解决问题

> 这一章回答“Redis 出问题怎么办”。按现象、指标、止血、根因治理来背，面试时会显得更像做过线上系统。

### 40. 什么是大 key？有什么危害？

**结论先行：**
大 key 是 value 很大或集合元素很多的 key，会造成阻塞、网络抖动、迁移困难、内存不均和故障恢复变慢。

**分层展开：**

**常见形态**
- String value 几 MB 甚至几十 MB。
- Hash/List/Set/ZSet 元素几十万甚至更多。
- 单个 key 下存储了过多业务对象。

**主要危害**
- 读取和删除耗时长，阻塞主线程。
- 网络传输大包导致延迟抖动。
- Cluster 中 slot 数据倾斜。
- RDB/AOF、主从复制、迁移成本变高。

**治理方案**
- 拆 key，例如按用户、业务、时间片分片。
- 删除用 `UNLINK` 或分批删除。
- 集合分页读取，避免一次拉全量。
- 定期扫描识别大 key。

**面试追问：怎么发现大 key？**
可以用 `redis-cli --bigkeys`、`MEMORY USAGE`、离线分析 RDB、监控慢查询和网络大包。线上执行扫描要控制频率，避免影响主库。

### 41. 什么是热 key？怎么处理？

**结论先行：**
热 key 是被超高频访问的 key，会导致单节点 CPU、网络或带宽打满。处理思路是发现、拆分、复制、本地缓存和限流降级。

**分层展开：**

**发现方式**
- 业务埋点统计 key 访问频率。
- Redis hotkeys 采样。
- 监控单节点 QPS、带宽、CPU 异常。
- 代理层统计。

**处理方式**
- 本地缓存热点数据，降低 Redis 压力。
- 热 key 拆分成多个副本 key，随机读。
- 读写分离，增加副本承担读流量。
- 对热点请求限流和降级。

**注意事项**
- 拆分副本会增加一致性复杂度。
- 本地缓存需要失效通知或短 TTL。
- 对强一致字段不要轻易复制缓存。

**面试追问：热点 key 能靠 Cluster 自动解决吗？**
不能。Cluster 解决的是数据分片，不会把单个 key 拆到多个节点。单 key 热点仍会集中在一个节点。

### 42. Redis 内存碎片是什么？怎么治理？

**结论先行：**
内存碎片是 Redis 释放的内存没有完全归还操作系统或无法被高效复用，表现为 RSS 明显高于 used_memory。

**分层展开：**

**产生原因**
- key 频繁创建和删除。
- value 大小差异大。
- jemalloc 分配器按不同 size class 管理内存。
- 大量过期和淘汰导致内存空洞。

**观察指标**
- `mem_fragmentation_ratio`。
- `used_memory`。
- `used_memory_rss`。

**治理方案**
- 开启 active defrag。
- 控制 value 大小。
- 避免频繁写入删除大对象。
- 必要时主从切换或重启释放内存。

**面试追问：mem_fragmentation_ratio 越低越好吗？**
不是。过低可能意味着使用了 swap 或 RSS 小于逻辑内存，反而危险。要结合 used_memory、RSS、系统内存和延迟综合判断。

### 43. 如何避免 Redis 缓存污染？

**结论先行：**
缓存污染是大量低价值或一次性数据进入缓存，挤掉高价值热点数据。治理核心是准入控制、TTL 分级和淘汰策略。

**分层展开：**

**产生原因**
- 爬虫或批处理扫描大量冷数据。
- 搜索结果页把一次性数据都写缓存。
- 预热范围过大。
- TTL 设置过长。

**治理方案**
- 只缓存高频访问数据。
- 对冷数据设置短 TTL。
- 对批量任务禁用缓存回填。
- 使用 TinyLFU、本地缓存准入或业务白名单。

**监控指标**
- key 新增速度。
- 命中率变化。
- evicted_keys。
- 热点 key 占比。

**面试追问：查询 DB 后一定要回填缓存吗？**
不一定。低频数据、批量扫描、后台任务、分页深翻数据不一定要回填，否则会污染缓存。

### 44. Redis 慢查询怎么排查？

**结论先行：**
Redis 慢查询排查要从命令复杂度、大 key、网络、持久化、CPU、内存和客户端连接池一起看，不能只盯 slowlog。

**分层展开：**

**命令层**
- 看 `SLOWLOG GET`。
- 排查 `KEYS`、`HGETALL`、`SMEMBERS`、`ZRANGE` 大范围查询。
- 避免 Lua 脚本执行过久。

**数据层**
- 检查大 key 和热 key。
- 检查 key 过期是否集中。
- 检查集合是否一次性返回大量元素。

**系统层**
- CPU 是否打满。
- 网络带宽是否打满。
- 磁盘 fsync 是否抖动。
- 是否发生 fork 或 AOF rewrite。

**客户端层**
- 连接池是否耗尽。
- 超时时间是否合理。
- 是否存在请求堆积和重试风暴。

**面试追问：慢查询日志记录网络耗时吗？**
不记录。slowlog 主要记录 Redis 执行命令的耗时，不包含客户端排队、网络传输和响应读取时间。

### 45. 线上 Redis CPU 飙高怎么办？

**结论先行：**
CPU 飙高优先判断是命令复杂度、热 key、连接风暴、Lua 脚本、过期淘汰还是持久化导致，再采取限流、拆分、优化命令和扩容。

**分层展开：**

**快速止血**
- 限制异常流量。
- 临时下线问题接口或降级。
- 切读副本或扩容读节点。

**定位原因**
- `INFO commandstats` 看命令分布。
- `SLOWLOG` 看慢命令。
- 监控 QPS、连接数、网络带宽。
- 查热 key 和大 key。

**根因治理**
- 替换高复杂度命令。
- 对集合分页或拆 key。
- 热点数据本地缓存。
- 限制 Lua 脚本耗时。
- 扩容分片。

**面试追问：Redis 是单线程，CPU 为什么可能超过 100%？**
Redis 还有后台线程和子进程，例如 AOF、lazy free、IO 线程、RDB/AOF rewrite 子进程。多核环境下进程整体 CPU 可能超过单核 100%。

### 46. Redis 连接数打满怎么办？

**结论先行：**
连接数打满通常是客户端连接池配置不合理、连接泄漏、慢请求堆积或突发流量导致，需要先止血再治理连接生命周期。

**分层展开：**

**现象**
- 新请求连接 Redis 失败。
- 客户端超时增多。
- Redis `connected_clients` 接近 `maxclients`。

**止血**
- 限流入口流量。
- 重启或隔离异常客户端。
- 临时提高 `maxclients`，前提是系统 fd 和内存允许。

**治理**
- 使用连接池，不要每个请求新建连接。
- 设置合理超时。
- 避免连接泄漏。
- 对慢命令和阻塞命令做治理。

**面试追问：连接池越大越好吗？**
不是。连接池太大会增加 Redis 上下文管理、客户端资源和突发并发压力。连接池大小要按服务实例数、QPS、命令耗时和 Redis 承载能力计算。

### 47. Redis 出现超时，可能是什么原因？

**结论先行：**
Redis 超时不一定是 Redis 慢，可能来自客户端排队、网络抖动、连接池耗尽、命令阻塞、持久化 fork、GC、带宽打满或下游重试放大。

**分层展开：**

**客户端侧**
- 连接池耗尽。
- 线程池排队。
- GC 停顿。
- 超时时间设置过短。

**网络侧**
- 跨机房访问。
- 网络丢包。
- 带宽打满。
- 大 value 传输。

**Redis 侧**
- 慢命令阻塞主线程。
- 大 key 删除或读取。
- fork、AOF fsync。
- CPU 打满、内存 swap。

**业务侧**
- 请求重试导致流量放大。
- 热点 key 集中访问。
- 缓存雪崩导致回源和 Redis 双重压力。

**面试追问：如何证明是 Redis 服务端慢还是客户端慢？**
对比客户端耗时、Redis slowlog、Redis latency monitor、网络 RTT、服务端 CPU 和连接池等待时间。如果客户端耗时高但 slowlog 没记录，往往是网络、排队或连接池问题。

### 48. 如果 Redis 命中率突然下降，你怎么排查？

**结论先行：**
命中率下降要先判断是流量变化、key 过期、缓存写入失败、容量淘汰、key 版本变化还是 Redis 故障导致。

**分层展开：**

**看现象**
- 命中率下降从什么时候开始。
- 是否伴随 DB QPS 上升。
- 是否集中在某些接口或 key 前缀。

**查变更**
- 是否有代码发布。
- key 规则是否变更。
- TTL 是否被改短。
- 是否有批量删除任务。

**查 Redis**
- `evicted_keys` 是否增加。
- `expired_keys` 是否异常。
- 内存是否达到 maxmemory。
- Cluster 是否有节点故障。

**查业务**
- 是否有新流量进入。
- 是否大量访问冷数据。
- 是否缓存回填失败。

**面试追问：命中率低一定是坏事吗？**
不一定。新用户、新商品、冷启动、活动流量变化都可能导致短期命中率低。要结合 DB 压力、接口延迟和业务场景判断。

### 49. Redis 故障时怎么保护数据库？

**结论先行：**
Redis 故障时最怕流量直接打爆数据库，要通过熔断、限流、本地兜底、降级和隔离保护 DB。

**分层展开：**

**入口限流**
- 对高 QPS 接口限流。
- 对非核心请求快速失败。

**本地兜底**
- 返回本地缓存旧值。
- 返回默认值或静态数据。
- 只保核心字段。

**回源保护**
- 控制 DB 并发。
- 请求合并。
- 热点 key 互斥回源。

**故障恢复**
- Redis 恢复后分批预热。
- 避免瞬间全量回填。
- 复盘命中率和容量。

**面试追问：Redis 恢复后为什么还要预热？**
如果缓存是空的，所有请求仍会回源 DB。恢复后要分批加载热点数据，让命中率逐步恢复，避免恢复阶段二次雪崩。

### 50. 你负责一个高并发系统，Redis 架构怎么设计？

**结论先行：**
我会从业务读写模型出发，设计“多级缓存 + Redis Cluster + 主从高可用 + 一致性补偿 + 热点治理 + 监控预案”的完整体系。

**分层展开：**

**业务分层**
- 核心强一致数据不放或少放 Redis。
- 高频读、可重建数据放 Redis。
- 热点数据加本地缓存。

**部署架构**
- Cluster 分片扩展容量。
- 每个 master 配 replica。
- 跨机器、跨机架部署。
- 客户端使用成熟 Cluster SDK。

**一致性设计**
- Cache Aside。
- 写 DB 后删缓存。
- MQ/binlog 异步补偿。
- TTL 兜底。

**稳定性设计**
- 热 key 识别和本地缓存。
- 大 key 拆分。
- 限流、熔断、降级。
- Redis 故障时保护 DB。

**可观测性**
- 命中率。
- QPS、延迟、慢查询。
- CPU、内存、碎片率。
- 主从延迟。
- keyspace 和大 key 扫描。

**面试追问：这个方案最大的风险是什么？**
最大风险通常不是 Redis 本身，而是缓存一致性、热点流量和故障时流量回源。架构上要默认 Redis 会慢、会挂、会丢部分缓存，然后让系统仍能保护核心链路。

### 51. 如何做 Redis 容量评估？

**结论先行：**
容量评估要从 key 数量、value 大小、数据结构开销、TTL、增长速度、复制副本、碎片率和峰值写入综合估算。

**分层展开：**

**数据规模**
- 预计 key 数量。
- 平均 value 大小。
- 最大 value 大小。
- 集合元素数量。

**额外开销**
- key 本身占内存。
- Redis 对象头、dict、编码结构开销。
- 内存碎片。
- 复制 backlog、client buffer、AOF rewrite buffer。

**副本和高可用**
- 主从至少双份内存。
- Cluster 多分片要预留倾斜空间。
- fork 时要预留写时复制内存。

**安全水位**
- 不建议长期超过 70% 到 80%。
- 大促或批处理前预留更多空间。
- 设置 maxmemory 和告警阈值。

**面试追问：如何估算一个 key 的真实内存？**
可以用 `MEMORY USAGE key` 抽样，再结合线上 key 数量、类型分布和业务增长率估算。精确容量需要压测和 RDB 离线分析。

### 52. Redis 如何做灰度和多环境隔离？

**结论先行：**
Redis 灰度隔离主要通过 key 前缀、独立实例、逻辑库、权限账号和流量路由实现，生产核心业务更推荐实例或集群级隔离。

**分层展开：**

**key 前缀隔离**
- 简单、成本低。
- 适合小规模灰度。
- 风险是误删和资源竞争。

**逻辑库隔离**
- 单机 Redis 支持多个 DB。
- Cluster 不支持多 DB。
- 生产大规模不推荐依赖逻辑库。

**实例隔离**
- 不同环境或业务使用不同 Redis。
- 资源和故障边界清晰。
- 成本更高。

**权限隔离**
- Redis ACL 控制命令和 key pattern。
- 降低误操作风险。

**面试追问：为什么大厂强调资源隔离？**
因为共享 Redis 很容易出现某个业务的大 key、热 key、慢命令或内存打满影响其他业务。隔离可以把故障限制在业务边界内。

## 五、项目实战表达：把 Redis 讲成项目能力

> 这一章用于简历和项目深挖。回答时一定从业务问题开始，落到数据模型、链路、风险、监控和收益。

### 53. 枫叶互动项目里 Redis 用在哪些核心场景？

**可以这样答：**

在枫叶互动广告投放平台里，Redis 不是单纯做缓存，主要有三类核心场景：

1. **分布式锁：** 广告数据拉取、素材同步、自动投放、报表预警等 Cron 任务需要保证多实例下单实例执行，避免重复创建广告、重复写入和重复通知。
2. **媒体 API 限流：** Facebook/TikTok/Snapchat Marketing API 有账号级、应用级、接口级限流，所以使用 Redis ZSET + Lua 做全局滑动窗口限流。
3. **缓存版本指纹：** 广告账号下 Campaign/AdGroup/Ad 配置频繁变更，通过账号级版本号生成缓存指纹，实现多账号查询缓存精准失效。

**面试官追问：为什么这几个场景都适合 Redis？**

- 分布式锁需要跨实例共享状态，Redis 的 `SET NX EX` 和 Lua 能比较轻量地解决。
- 限流需要低延迟计数和过期窗口，ZSET 天然适合按时间窗口清理。
- 缓存版本需要高频读写、小数据量、低延迟，Redis 比 DB 更适合。

### 54. 如何讲清楚 Redis 分布式锁的项目细节？

**业务背景：**
广告数据同步、素材同步、自动投放策略执行通常由定时任务触发。如果服务多副本部署，多个实例同时执行会导致重复调用媒体 API、重复写 MongoDB、重复发送通知，甚至重复创建广告。

**方案：**
- 使用 Redis 分布式锁控制任务单实例执行。
- 加锁时设置 TTL，避免实例宕机导致死锁。
- 长任务开启 watchdog，按锁过期时间的一定比例自动续期。
- 加锁失败时做随机抖动重试，避免多个实例同时竞争。
- 任务退出时释放锁，panic 场景也要 recover 并释放资源。

**风险和改进：**
- 释放锁必须校验 token，避免误删别人的锁。
- watchdog 只能续自己的锁，不能无脑 `EXPIRE`。
- 任务要有业务幂等，因为锁是稳定性手段，不是业务正确性的唯一保障。

**一句话总结：**
我会把 Redis 锁定位成“降低并发重复执行概率的工程手段”，最终还要靠任务状态机、唯一键、幂等消费和补偿机制兜底。

### 55. 如何讲 Redis ZSET + Lua 滑动窗口限流？

**业务背景：**
广告创建和报表拉取会调用媒体平台 API。平台一旦判定破频，可能会返回限流错误、进入冷却窗口，影响运营创建广告和数据回流。

**核心设计：**
- key 按平台、账号、接口维度拆分，例如 `rate:meta:account:{id}:create_ad`。
- ZSET 的 score 使用请求时间戳。
- 每次请求先删除窗口外数据，再统计窗口内请求数。
- 未超过阈值则写入当前请求并放行。
- 全部操作通过 Lua 保证原子性。

**重要细节：**
ZSET member 不能只用秒级时间戳。因为同一秒内多个请求 member 相同会被覆盖，导致统计值偏小。可以使用微秒时间戳，或者 `timestamp:random/requestID` 作为 member。

**降级策略：**
- 触发平台限流后，临时降低账号 QPS。
- 对写接口更保守，对读接口分级处理。
- 任务状态保留为可重试，不直接丢弃。
- 告警中展示平台错误码、账号、接口、请求量和限流命中数。

### 56. 缓存版本指纹怎么讲？

**业务背景：**
广告账号下 Campaign、AdGroup、Ad 配置经常由同步任务、广告创建任务、状态拉取任务更新。如果每次变更都直接删除大范围缓存，容易误删、成本高，也不利于多账号组合查询。

**方案：**
- 每个账号维护一个配置版本号，例如 `cfgver:{channel}:{account_id}`。
- 账号下关键配置变化时自增版本。
- 查询多账号数据时，批量读取账号版本号。
- 将 `accountID=version` 排序拼接后生成 hash，作为缓存 key 的一部分。
- 任一账号版本变化，组合查询缓存自然失效。

**为什么用 pipeline GET，不直接 MGET？**
在 Redis Cluster 下，如果多个 key 不在同一个 slot，`MGET` 可能受限；pipeline GET 更容易兼容跨 slot 场景，也能减少网络往返。

**面试表达亮点：**
这类方案比“更新就删缓存”更精细，适合多账号、多条件组合查询，能降低缓存误删和缓存雪崩风险。

### 57. 如何把知乎动态模板项目里的 Redis 讲出来？

**业务背景：**
知乎动态化模板平台需要按端类型、App 版本、场景、内容类型、AB 实验等条件匹配模板。模板更新后，客户端不能长期读取旧模板。

**方案：**
- 模板数据以 DB 为准，Redis 缓存热点模板和匹配结果。
- 更新模板后先删缓存，再更新 DB，再延迟删除一次。
- 缓存设置合理过期时间，避免删除失败导致长期脏读。
- 客户端侧有本地预置和业务接口兜底，服务端缓存异常时不影响基础体验。

**追问：延迟双删一定可靠吗？**
不绝对可靠。它解决的是常见并发读写下的脏缓存窗口，不能保证强一致。如果要求更强，可以用版本号、binlog 订阅、消息通知或读请求校验版本。模板配置这类场景通常允许短时间最终一致，所以延迟双删 + TTL 是性价比较高的方案。

**背景：**
业务量、痛点、原方案问题。

**方案：**
数据结构、key 设计、TTL、一致性、降级。

**难点：**
热点、大 key、并发重建、删除失败、故障恢复。

**结果：**
用数字说话：命中率、QPS、延迟、成本、稳定性。

**反思：**
方案边界和后续优化。

### 58. 你在项目里如何体现 Redis 使用的工程深度？

**结论先行：**
不要只说“用了 Redis 做缓存”，要讲清楚业务问题、数据模型、读写链路、一致性策略、异常处理、监控指标和最终收益。

**分层展开：**

**业务背景**
- 哪个接口 QPS 高。
- 原来瓶颈在哪里。
- 为什么数据库扛不住。

**技术方案**
- 选用什么数据结构。
- key 如何设计。
- TTL 如何设置。
- 是否多级缓存。

**一致性**
- 写 DB 后删缓存。
- MQ 或 binlog 补偿。
- 删除失败如何重试。
- 可接受的不一致窗口是多少。

**稳定性**
- 如何防穿透、击穿、雪崩。
- 如何处理热 key、大 key。
- Redis 故障如何降级。

**结果量化**
- 命中率提升。
- DB QPS 降低。
- P99 延迟降低。
- 故障次数下降。

**面试追问：能不能举一个简短项目表达？**
可以这样说：我在商品详情页引入 Redis + 本地缓存，详情主数据采用 Cache Aside，写库后通过 MQ 删除 Redis 和本地缓存；热点商品提前预热，TTL 加随机值，价格库存拆成独立缓存；上线后 Redis 命中率稳定在 95% 以上，DB 查询 QPS 降低约 80%，大促期间通过逻辑过期和回源互斥避免了热点击穿。

## 六、Redis 面试方法论、速记清单与回答模板

> 本模块统一收纳所有“怎么答、怎么想、怎么背”的内容。前面五章负责知识本体，这一章负责面试表达和临场组织。

### 方法 0：复习地图

```text
第一层：Redis 当前能力
    命令、数据类型、底层编码、持久化、过期/淘汰、主从、哨兵、Cluster、Pipeline、key 设计

第二层：核心能力原理分析
    为什么快、网络 IO、ZSet 跳表、缓存一致性、分布式锁、Lua/事务、fork、脑裂

第三层：实战与线上治理
    商品缓存、排行榜、限流、延迟队列、点赞、预热、本地缓存、大 key、热 key、慢查询、CPU、超时、故障保护 DB、项目表达
```

### 方法 1：Redis 题目怎么答才像资深后端？

**结论先行：**
Redis 面试不要只背命令，要围绕“为什么快、怎么保证一致性、怎么扛高并发、怎么处理故障、怎么做容量与成本权衡”来回答。

**分层展开：**

**基础层：数据结构与命令**
- String、Hash、List、Set、ZSet、Bitmap、HyperLogLog、Geo、Stream。
- 底层编码、时间复杂度、适用场景。

**性能层：单线程、高性能 IO 与内存模型**
- 单线程执行命令，避免锁竞争。
- IO 多路复用处理大量连接。
- 内存操作、紧凑编码、渐进式 rehash、惰性删除。

**可靠性层：持久化、复制、高可用**
- RDB 适合快照和恢复。
- AOF 适合更高数据安全。
- 主从复制、哨兵、Cluster 解决可用性和扩展性。

**工程层：缓存架构与一致性**
- 缓存穿透、击穿、雪崩。
- Cache Aside、延迟双删、异步补偿、消息队列。
- 分布式锁、限流、排行榜、延迟队列、会话存储。

**治理层：容量、监控、故障演练**
- 热 key、大 key、慢查询、内存碎片、淘汰策略。
- 指标监控、压测、熔断降级、预案。

**面试追问：**
如果面试官问“你线上怎么用 Redis”，不要泛泛说缓存，要讲清楚业务读写路径、缓存粒度、过期策略、一致性边界、失败兜底和监控指标。

### 方法 2：Redis 题通用决策框架

Redis 面试不要一上来背命令。先判断面试官问的是哪类问题，再套对应的回答模板。

#### 第一反应：先问 6 个问题

```text
1. 问的是数据结构选型，还是缓存架构？
2. 问的是读性能，还是写一致性？
3. 问的是单机能力，还是高可用/分片扩展？
4. 问的是业务设计，还是线上故障排查？
5. 问的是原子性/并发控制，还是容量/成本治理？
6. 这个 Redis 方案失败时，数据库和业务怎么兜底？
```

对应模板：

| 问题类型   | 优先想到                                    | 回答重点                              |
| ------ | --------------------------------------- | --------------------------------- |
| 为什么快   | 内存 + 单线程命令 + IO 多路复用 + 数据结构             | 别只说内存，要讲主线程、socket、Pipeline、慢命令边界 |
| 数据结构选型 | String/Has/ZSet/Bitmap/HLL/Bloom/Stream | 先讲业务需求，再讲结构能力和风险                  |
| 缓存一致性  | Cache Aside                             | 先更新 DB，再删缓存，删除失败要重试/补偿            |
| 缓存三大问题 | 穿透/击穿/雪崩                                | 空值/Bloom、互斥重建、随机 TTL/预热           |
| 并发控制   | 分布式锁/Lua/事务                             | 原子性、锁误删、过期时间、业务幂等                 |
| 限流     | 固定窗口/滑动窗口/令牌桶/ZSET+Lua                  | 单机 vs 全局，精度 vs 成本，降级和监控           |
| 高可用    | 主从/Sentinel/Cluster                     | 可用性、故障转移、分片扩容、客户端复杂度              |
| 持久化    | RDB/AOF/混合持久化                           | 恢复速度、数据丢失窗口、fork 和磁盘 IO 风险        |
| 性能排查   | 慢命令/大 key/热 key/连接/网络/fork              | 先影响面，再指标，再止血，再复盘                  |
| 架构设计   | 容量/QPS/热点/一致性/降级                        | 先约束量级，再选架构，不要直接堆中间件               |

#### Redis 题通用解题步骤

```text
Step 1：先定业务场景
Step 2：判断读多还是写多
Step 3：选择数据结构或缓存模式
Step 4：说明一致性和原子性边界
Step 5：说明高并发风险：热 key、大 key、击穿、雪崩
Step 6：说明故障兜底：降级、限流、本地缓存、DB 保护
Step 7：补监控指标：命中率、延迟、慢查询、内存、连接数、错误率
```

面试回答不要停在“我用了 Redis 缓存”，要讲成：

```text
业务为什么需要 Redis
key/value 怎么设计
读写路径是什么
一致性怎么保证
异常失败怎么补偿
高并发下有什么风险
线上怎么监控和止血
```

#### Redis 思想分类

| 思想 | 代表方案 | 解决问题 | 关键词 | 典型题 |
|---|---|---|---|---|
| 数据结构选型思想 | String/Hash/ZSet/Stream 等 | 用合适结构表达业务 | 对象、排行、去重、计数、消息 | 用户缓存、排行榜、延迟队列 |
| 旁路缓存思想 | Cache Aside | 缓存与 DB 最终一致 | 先读缓存、miss 查 DB、写 DB 后删缓存 | 缓存一致性 |
| 热点保护思想 | 互斥重建/逻辑过期/本地缓存 | 防止热点 key 打爆 DB | 击穿、热 key、预热 | 商品详情、播放页 |
| 防穿透思想 | 空值缓存/Bloom Filter | 拦截不存在请求 | 非法 ID、不存在数据 | 商品/用户不存在 |
| 原子执行思想 | Lua/事务/SET NX EX | 多步操作不可被插入 | 限流、锁释放、扣减 | 分布式锁、滑动窗口 |
| 时间窗口思想 | ZSet/计数器/令牌桶 | 控制请求频率 | QPS、窗口、突发 | 媒体 API 限流 |
| 高可用思想 | 主从/Sentinel/Cluster | 故障转移和容量扩展 | 副本、分片、slot | Redis 架构选型 |
| 异步补偿思想 | MQ/binlog/重试任务 | 删除失败、同步失败可恢复 | 最终一致、补偿、重试 | 缓存删除、ES 同步 |
| 治理排障思想 | slowlog/bigkeys/hotkeys/监控 | 定位线上问题 | 延迟、CPU、连接、内存 | Redis 超时/CPU 高 |

#### 快速决策树

```text
题目问“存什么结构”？
    对象整体读写 -> String + JSON/ProtoBuf
    字段级读写 -> Hash
    排名/范围/延迟 -> ZSet
    去重集合/交并差 -> Set
    海量 0/1 状态 -> Bitmap
    近似 UV -> HyperLogLog
    防穿透/可能存在 -> Bloom Filter
    轻量消息流 -> Stream

题目问“缓存和 DB 一致性”？
    常规场景 -> Cache Aside：读缓存，miss 查 DB 回填；写 DB 后删缓存
    删除失败 -> MQ/binlog/重试表补偿
    极短不一致不能接受 -> 版本号/强校验/不走缓存

题目问“高并发缓存风险”？
    查不存在 -> 穿透：空值/Bloom
    热点 key 过期 -> 击穿：互斥锁/逻辑过期/热点不过期
    大量 key 同时过期 -> 雪崩：随机 TTL/分批预热/降级

题目问“并发原子性”？
    单 key 加锁 -> SET NX EX + token + Lua 删除
    多步读改写 -> Lua
    批量发送减少 RTT -> Pipeline，但不保证原子性
    事务排队执行 -> MULTI/EXEC，但不是数据库事务

题目问“限流”？
    单机简单限制 -> 本地滑动窗口/令牌桶
    多实例全局限制 -> Redis ZSET + Lua
    允许突发 -> 令牌桶
    精确窗口统计 -> 滑动窗口

题目问“架构选型”？
    数据量小、可接受短暂不可用 -> 单机/主从
    需要自动故障转移 -> Sentinel
    单实例容量或吞吐不够 -> Cluster
    极热点读 -> 本地缓存 + Redis + DB 多级缓存

题目问“线上变慢/超时”？
    先看影响面和指标
    再查 slowlog、大 key、热 key、连接数、CPU、内存、fork、网络、客户端连接池
    先止血：限流、降级、摘流量、禁用重命令、扩容
    后复盘：key 设计、容量、监控、预案
```

#### 常见 Redis 业务题模板组合表

| 业务题 | 模板组合 |
|---|---|
| 商品/剧集详情缓存 | Cache Aside + 随机 TTL + 热点互斥重建 + 降级 |
| 用户/账号基础信息 | String 或 Hash + TTL + 删除缓存重试 |
| 排行榜 | ZSet + `ZINCRBY/ZREVRANGE/ZREVRANK` + 大 key 治理 |
| 媒体 API 限流 | Redis ZSET + Lua + 账号/API 维度 key + 破频降级 |
| 分布式锁 | `SET NX EX` + token + Lua 删除 + watchdog + 业务幂等 |
| 延迟队列 | ZSet score=执行时间 + Lua 取删 + 重试/死信 |
| 点赞系统 | Set/Bitmap 去重 + 计数 + 异步落库 |
| 防缓存穿透 | Bloom Filter + 空值缓存 + 参数校验 |
| 缓存预热 | 分批扫描 + 限速 + 随机 TTL + 可回滚 |
| Redis 故障保护 DB | 本地缓存兜底 + 限流降级 + 熔断 + 读 DB 保护 |
| Redis CPU 高 | 慢命令 + 大 key + 热 key + Lua + 连接风暴排查 |
| Redis 内存高 | 大 key + 过期策略 + 淘汰策略 + 碎片率 + 容量评估 |

#### 面试前速记口诀

```text
先问场景，再选结构；
读多看缓存，写多看一致性；
高并发先防穿透、击穿、雪崩；
多步原子用 Lua，批量 RTT 用 Pipeline；
锁只防重复，正确性靠幂等；
限流先分单机和全局；
容量不够上 Cluster，可用性不够看 Sentinel；
线上慢先看慢命令、大 key、热 key、连接、fork、网络；
任何 Redis 方案都要说 DB 兜底、降级和监控。
```

#### 二、Redis 基础与数据结构

### 方法 3：Redis 高频题速记清单

1. Redis 快：内存、单线程、IO 多路复用、高效结构、后台线程。
2. 单线程：命令执行串行，网络连接并发。
3. String：SDS，二进制安全，O(1) 获取长度。
4. Hash：适合字段级读写，但要防大 key。
5. ZSet：dict + skiplist，适合排行榜和范围查询。
6. 跳表：实现简单、范围查询友好、平均 O(logN)。
7. 缓存一致性：先写 DB，再删缓存，失败重试，TTL 兜底。
8. 穿透：空值缓存、Bloom Filter、参数校验。
9. 击穿：互斥锁、逻辑过期、热点不过期。
10. 雪崩：TTL 随机、多级缓存、限流降级、高可用。
11. 分布式锁：`SET NX EX` + 唯一值 + Lua 解锁。
12. RDB：快照，恢复快，可能丢数据。
13. AOF：命令日志，安全性更好，文件更大。
14. 主从复制：全量 + 增量，异步复制有延迟。
15. Sentinel：自动故障转移，不解决分片。
16. Cluster：16384 slot，支持水平扩容。
17. 大 key：阻塞、网络抖动、迁移慢、内存倾斜。
18. 热 key：单节点打满，本地缓存、拆分副本、限流。
19. 淘汰策略：LRU/LFU/random/TTL/noeviction。
20. 慢查询：命令复杂度、大 key、Lua、网络、持久化、客户端都要看。

### 方法 4：面试回答模板

### 模板 1：原理类题

**结论：**
先用一句话回答核心判断。

**原因：**
从数据结构、复杂度、线程模型、IO 模型解释。

**场景：**
说明适合什么业务，不适合什么业务。

**风险：**
指出线上坑，例如大 key、热 key、一致性、阻塞。

**治理：**
给出工程方案和监控指标。

### 模板 2：场景设计题

**目标：**
明确 QPS、延迟、一致性、容量和可用性目标。

**读链路：**
本地缓存 -> Redis -> DB。

**写链路：**
更新 DB -> 删除缓存 -> 异步补偿 -> TTL 兜底。

**异常链路：**
Redis 故障、DB 故障、缓存雪崩、热点 key、大 key。

**监控：**
命中率、P99、慢查询、DB QPS、Redis CPU/内存/网络、主从延迟。

### 模板 3：项目深挖题
