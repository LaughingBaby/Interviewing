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
缓存与数据库在常规高并发架构下很难低成本做到绝对强一致。工程上通常把数据库作为唯一事实源，采用 Cache Aside，通过“先更新数据库，再删除缓存”，配合删除失败重试、MQ/CDC 异步补偿、延迟二次删除、版本号校验和 TTL 兜底，实现最终一致或有界不一致。

```text
缓存与数据库一致性
├── 一致性目标分级：强一致 / 最终一致 / 有界不一致
├── Cache Aside 读写流程
├── 为什么先更新数据库，再删除缓存
├── 删除缓存失败如何补偿
├── 并发读回填旧值问题
│   └── 延迟二次删除
├── 本地缓存 + Redis + DB 的多级一致性
└── 高并发下的缓存重建与性能权衡
```

完整“缓存一致性与多级缓存实战”上位方法论已沉淀到 `技术｜分布式与高并发架构面试题.md` 的专题模块；本题重点保留 Redis 缓存落地视角。

**一、为什么会有一致性问题？**

没有缓存时，数据只有数据库一份，不存在缓存一致性问题；引入 Redis 后，数据变成：

```text
Redis：派生副本
DB：真实数据源
```

如果再引入本地缓存，同一份数据可能同时存在于：

```text
应用实例 A 的本地缓存
应用实例 B 的本地缓存
Redis 集群
MySQL / MongoDB
```

只要数据存在多个副本，就必然要回答：先更新谁、失败怎么办、并发读写怎么办、消息丢失怎么办、副本延迟怎么办。所以一致性问题的本质不是 Redis 独有，而是多副本系统在性能、可用性和一致性之间做权衡。

**二、一致性目标要先分级**

**强一致性**
- 写入完成后，后续读取必须立即看到最新值。
- 适合余额、支付状态、订单状态流转、库存最终扣减、权限安全校验。
- 通常依赖数据库事务、条件更新、锁、版本号、状态机和关键读取绕过缓存。
- 普通 Redis 缓存方案不应承担最终正确性。

**最终一致性**
- 短时间允许读到旧值，但经过重试、过期或异步同步后最终一致。
- 适合用户资料、商品介绍、文章内容、普通配置展示、非核心统计。
- Cache Aside 主要解决这类场景。

**有界不一致**
- 允许短时间旧数据，但必须限制在业务可接受窗口内。
- 例如最多旧 1 秒、5 秒，或旧到本地缓存短 TTL 到期。
- 多级缓存常见目标不是瞬时强一致，而是把不一致窗口控制住。

**三、Cache Aside 读路径**

```text
读取数据
   │
   ▼
读取缓存
   │
   ├── 命中：直接返回
   └── 未命中
          │
          ▼
       查询数据库
          │
          ▼
       写入缓存
          │
          ▼
         返回
```

```go
func GetUser(ctx context.Context, id int64) (*User, error) {
	key := fmt.Sprintf("user:%d", id)

	user, err := redisGetUser(ctx, key)
	if err == nil {
		return user, nil
	}

	user, err = queryUserFromDB(ctx, id)
	if err != nil {
		return nil, err
	}

	// 写缓存失败通常不影响本次读请求返回，但要记录日志和指标。
	_ = redisSetUser(ctx, key, user, 30*time.Minute)
	return user, nil
}
```

**四、Cache Aside 写路径**

推荐流程：

```text
更新数据库
   │
   ▼
数据库事务提交成功
   │
   ▼
删除缓存
   │
   ├── 成功：结束
   └── 失败：重试 / MQ / CDC 补偿
```

```go
func UpdateUser(ctx context.Context, user *User) error {
	if err := updateUserInDB(ctx, user); err != nil {
		return err
	}

	key := fmt.Sprintf("user:%d", user.ID)
	if err := redisDelete(ctx, key); err != nil {
		enqueueCacheInvalidation(key)
	}

	return nil
}
```

**五、为什么不是先删除缓存，再更新数据库？**

假设初始状态：

```text
数据库：age = 18
缓存：  age = 18
```

如果先删缓存再更新数据库，可能出现：

```text
写请求 W                         读请求 R
   │                                │
   │ 删除缓存                        │
   │                                │ 读取缓存未命中
   │                                │ 查询数据库，读到旧值 18
   │                                │ 把 18 写回缓存
   │ 更新数据库为 19                 │
   ▼                                ▼
```

最终：

```text
数据库：19
缓存：  18
```

先删除缓存会制造一个缓存空窗期，并发读可能把数据库旧值重新填回缓存。

**六、为什么不是更新数据库后直接更新缓存？**

并发写可能乱序覆盖：

```text
W1 更新数据库为 1
W2 更新数据库为 2
W2 更新缓存为 2
W1 因网络较慢，最后更新缓存为 1
```

最终：

```text
数据库：2
缓存：  1
```

此外，缓存可能是复杂聚合结果，一次数据库写可能影响多个缓存 key，直接更新缓存的维护成本高。删除缓存是幂等动作，`DEL key` 执行一次或多次最终效果相同，更适合重试和异步补偿。

**七、先更新数据库，再删除缓存是否绝对安全？**

不是。它仍然存在并发读回填旧值的竞态。关键不是写请求顺序错了，而是读请求的“查 DB”和“写缓存”不是原子操作，可能横跨写请求的更新和删除过程。

假设缓存已经过期：

```text
数据库：旧值 18
缓存：不存在
```

可能出现：

```text
读请求 R                                写请求 W
   │                                       │
   │ 1. 读 Redis，缓存未命中                │
   │                                       │
   │ 2. 查询数据库                          │
   │    此时 DB 还是 18，R 读到旧值 18       │
   │                                       │
   │                                       │ 3. 更新数据库为 19
   │                                       │
   │                                       │ 4. 删除 Redis
   │                                       │    此时 Redis 本来就不存在
   │                                       │
   │ 5. R 的 DB 查询返回较慢                 │
   │    在 W 删除缓存之后，才把 18 写入 Redis │
   ▼                                       ▼
```

最终：

```text
数据库：19
缓存：  18
```

问题关键是：旧值不是写请求删除缓存前就已经存在于 Redis，而是读请求先从 DB 读到了旧值，随后在写请求删除缓存之后，才把旧值回填进 Redis。

**八、延迟二次删除怎么理解？**

延迟二次删除不是独立的一致性方案，而是 Cache Aside 的并发补偿手段。更推荐的工程流程是：

```text
更新数据库
   │
   ▼
立即删除缓存
   │
   ▼
发送延迟消息
   │
   ▼
等待一段时间
   │
   ▼
再次删除缓存
```

第一次删除解决正常缓存失效；第二次删除负责清理并发竞态中被旧读请求重新回填的旧缓存。

不建议在主请求里直接 `sleep`：

```go
updateDB()
deleteCache()
time.Sleep(500 * time.Millisecond)
deleteCache()
```

原因：
- 增加接口响应时间。
- 占用 goroutine 和连接资源。
- sleep 时间难以准确设置。
- 服务重启后第二次删除可能丢失。
- 失败后没有可靠重试。

更推荐使用延迟 MQ、任务表、Redis Stream 或 CDC 消费链路做第二次删除。

**延迟多久？**

不能机械背 500ms 或 1s，应该覆盖可能产生旧回填的最长链路：

```text
数据库查询 P99
+ 缓存回填耗时
+ 网络抖动
+ 读从库复制延迟
+ 安全余量
```

如果系统读从库，还要考虑主从延迟导致“缓存未命中后读到从库旧值再回填”。这时可以结合写后短时间读主库、读写粘滞、版本号校验和复制延迟监控。

**如果算出来要延迟 5 到 12 秒，延迟双删还有意义吗？**

有意义，但它的定位会变窄：它不再是一个优雅的主一致性方案，而更像兜底补偿手段。

它的价值是把错误缓存的存活时间从 TTL 级别缩短到秒级补偿窗口：

```text
没有延迟二次删除：
旧缓存可能一直存在到 TTL 到期，例如 10 分钟、30 分钟

有延迟二次删除：
旧缓存大概率在 5 到 12 秒后被清掉
```

如果业务能接受几秒级旧数据，Cache Aside + 延迟二次删除 + TTL 仍然是性价比较高的方案。

如果业务不能接受 5 到 12 秒旧数据，就不能主要依赖延迟双删，而要升级一致性方案：

- 写后立刻读场景：写后短时间读主库、读写粘滞或绕过缓存。
- 高并发热点数据：缓存 value 带版本号，配合 Lua 防止旧版本覆盖新版本。
- 多级缓存场景：MQ/CDC 广播失效本地缓存，并设置较短 L1 TTL。
- 更高一致性场景：用数据库事务、条件更新、状态机、幂等号兜住最终正确性，缓存只做展示或加速。

面试表达可以这样说：

> 如果延迟时间计算出来要 5 到 12 秒，我会认为延迟双删仍然有兜底价值，因为它能把旧缓存从 TTL 级别缩短到秒级清理。但这也说明读链路 P99、网络抖动或主从复制延迟偏长，不能只依赖延迟双删。如果业务对写后读一致性敏感，我会配合写后读主、版本号校验、Lua 防旧值覆盖或 CDC 失效；强一致场景则不把缓存作为最终判断依据。

**延迟双删能保证强一致吗？**

不能。第二次删除也可能失败，延迟消息可能丢失，删除后仍可能有更晚的旧请求回填，数据库副本延迟也可能超过预估。它只能降低旧值回填概率、缩短不一致窗口，不能提供严格强一致。

**九、删除缓存失败怎么办？**

删除失败是 Cache Aside 最关键的异常场景：

```text
数据库：新值
缓存：  旧值
```

不能只打印日志或只等 TTL，应该把删除缓存做成可靠、可重试的动作。

**方案 1：同步有限重试**
- 删除失败后指数退避重试 2 到 3 次。
- 仍失败则进入异步补偿。
- 不能无限同步重试，否则拖垮写接口。

**方案 2：MQ 异步重试**
- 数据库更新成功。
- 删除缓存失败或统一发送缓存失效消息。
- 消费者删除缓存，失败则重试或进入死信队列。
- 消息包含缓存 key、业务类型、数据 ID、版本号、事件 ID、重试次数。

**方案 3：事务 Outbox**

解决“数据库提交成功，但应用在发送 MQ 前崩溃”的问题：

```sql
BEGIN;

UPDATE user
SET name = 'new_name',
    version = version + 1
WHERE id = 1001;

INSERT INTO outbox_event(
    event_type,
    aggregate_id,
    payload,
    status
) VALUES (
    'USER_CACHE_INVALIDATE',
    1001,
    '{"key":"user:1001"}',
    'PENDING'
);

COMMIT;
```

业务数据和失效事件在同一个数据库事务中提交，后台 Publisher 再投递 MQ。

**方案 4：binlog / Change Stream / CDC**

- MySQL 可订阅 binlog。
- MongoDB 可通过 Change Stream 感知变更。
- CDC 消费程序统一删除 Redis 或通知本地缓存失效。
- 优点是业务代码解耦；缺点是链路更长、存在传播延迟、要处理重复事件和消费积压。

**方案 5：TTL 最终兜底**

所有缓存都应设置合理 TTL。但 TTL 只能做最后兜底，不能作为唯一一致性方案。否则 TTL 是 30 分钟，就可能出现长达 30 分钟的旧数据窗口。

**十、多级缓存如何保证一致性？**

典型多级缓存：

```text
请求 -> L1 本地缓存 -> L2 Redis -> DB
```

读路径：

```text
读 L1
  ├── 命中：返回
  └── 未命中：读 Redis
        ├── 命中：写 L1，返回
        └── 未命中：读 DB，写 Redis，写 L1，返回
```

写路径：

```text
更新数据库并提交
   │
   ▼
删除 Redis
   │
   ▼
发布缓存失效事件
   ├── 实例 A 删除 L1
   ├── 实例 B 删除 L1
   └── 实例 C 删除 L1
```

本地缓存分散在多个应用进程中，不能像 Redis 一样删除一个共享 key 就结束。需要通过 MQ、Redis Pub/Sub、Redis Stream、配置中心事件或 CDC 通知所有实例失效。

**本地缓存实践建议**
- 容量小。
- TTL 短。
- 只缓存超热点数据。
- 主动失效 + TTL 兜底。
- 关键请求可绕过 L1。

Redis Pub/Sub 可以做本地缓存失效，但通常不保证离线消息和可靠重试；重要业务更适合可靠 MQ、Redis Stream 或 CDC。

**十一、高并发下还要解决缓存重建**

删除缓存后，大量请求可能同时未命中并打到 DB，这就是缓存击穿。一致性方案如果不兼顾重建性能，可能把数据库打垮。

**单机 singleflight**
- 同一应用实例内，相同 key 只允许一个请求查 DB 并重建缓存。
- 其他请求等待结果。

**分布式互斥重建**
- 缓存未命中后尝试获取 Redis 锁。
- 获取成功者查 DB、写缓存、释放锁。
- 获取失败者短暂等待后重试缓存。
- 锁必须有过期时间，释放锁要校验唯一 token。

**逻辑过期**

超热点 key 可以不让物理 key 立即过期，而是在 value 中保存逻辑过期时间：

```json
{
  "expireAt": 1720000000,
  "data": {
    "id": 1001,
    "name": "Tom"
  }
}
```

数据已逻辑过期时，先返回旧值，再后台异步重建。优点是接口延迟稳定、保护 DB；代价是允许短时间返回旧数据。

**十二、版本号如何降低旧值覆盖？**

对于一致性要求更高的缓存，可以给数据增加版本：

```text
value = { version: 105, data: ... }
```

数据库每次更新：

```sql
UPDATE user
SET name = ?,
    version = version + 1
WHERE id = ?;
```

缓存写入时只允许新版本覆盖旧版本：

```text
准备写入 version=104
Redis 已存在 version=105
拒绝旧版本写入
```

这个判断通常需要 Lua 脚本在 Redis 内原子完成。版本号比单纯延迟双删更严谨，但实现和维护成本更高。

**十三、不同业务怎么选？**

**普通商品详情、用户资料**
- Cache Aside。
- 先更新数据库，再删除 Redis。
- 删除失败重试。
- 设置 TTL。

**高并发热点数据**
- 延迟二次删除。
- singleflight 或分布式锁重建。
- TTL 随机抖动。
- 版本号。
- MQ / CDC 补偿。

**本地缓存 + Redis + DB**
- 数据库为事实源。
- Redis 使用 Cache Aside。
- MQ 广播 L1 缓存失效。
- 本地缓存短 TTL。
- 失效消息幂等处理。

**余额、支付、库存最终结果**
- 不能单纯依赖缓存一致性方案。
- 使用数据库事务、乐观锁/悲观锁、条件更新、幂等、账本和状态机。
- 缓存只做查询加速。

**面试 1 分钟回答：**
缓存和数据库在常规高并发架构下很难低成本做到绝对强一致，通常采用 Cache Aside 实现最终一致，并把数据库作为唯一可信数据源。读请求先查缓存，未命中查数据库并回填；写请求通常先更新数据库，事务提交成功后删除缓存，而不是直接更新缓存。删除失败要通过同步重试、MQ、事务 Outbox、binlog 或 Mongo Change Stream 做异步补偿，TTL 只作为最后兜底。即使先更新数据库再删除缓存，也可能出现并发旧读回填旧缓存，可以通过异步延迟二次删除、版本号校验降低风险。如果还有本地缓存，需要通过 MQ 或广播通知所有应用实例失效，并给本地缓存设置较短 TTL。高并发缓存失效时，还要用 singleflight、分布式锁或逻辑过期防止大量请求同时打到数据库。

**面试追问：缓存和数据库能做到绝对一致吗？**
普通 Redis + 数据库双写很难低成本做到绝对强一致，因为二者不在同一个本地事务中。工程上通常选择数据库作为事实源，通过缓存失效、重试、MQ、CDC 和 TTL 实现最终一致。真正要求强一致的余额、支付、库存等数据，应以数据库事务、锁和条件更新为准，缓存只做查询加速。

**面试追问：只设置 TTL 可以吗？**
不建议。TTL 只能保证最终恢复，不能限制正常情况下的不一致窗口。如果 TTL 是 30 分钟，删除失败就可能持续返回 30 分钟旧值。正确方式是主动删除、失败重试、异步补偿，TTL 只做最后兜底。

### 22. 缓存穿透、击穿、雪崩分别是什么？怎么解决？

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

### 23. Redis 分布式锁如何设计？生产中会遇到哪些问题？

**结论先行：**
Redis 分布式锁本质上是一个“带 TTL 的互斥租约”，适合跨服务实例做短时间互斥，例如缓存重建、定时任务抢占、防重复提交和非核心资源串行化。生产级 Redis 锁不能只会 `SET NX EX`，还要处理唯一 owner token、安全释放、TTL、续期、抢锁失败退避、故障切换、旧持锁者继续写、fencing token、业务幂等和监控告警。

```text
Redis 分布式锁生产实践
├── 锁的本质：所有权、互斥、访问顺序
├── 基础实现：SET NX PX + 唯一 owner token
├── 安全释放：Lua 校验 token 后删除，Redis 8.4+ 可用 DELEX IFEQ
├── TTL 与续期：覆盖 P99，但必须有最大持锁时间
├── 抢锁失败：退避、抖动、最大等待时间、context 取消
├── 生产问题：锁过期、主从切换、旧持锁者、热点锁、多锁死锁
├── fencing token：防止旧持锁者恢复后继续写
├── Redlock 评价：提升容错，但不是强一致银弹
└── 场景选型：什么时候用 Redis 锁，什么时候不用
```

**一、锁的横向视角摘要**

锁不是业务正确性的魔法开关。锁本质上是所有权与访问顺序协议，解决“谁能进入临界区”，但不能单独解决重复执行、锁过期后旧客户端继续写、消息重复、网络超时或数据库部分提交。

生产级正确性通常是：

```text
锁 + 数据库原子操作 + 幂等 + 唯一约束 + 版本号 + fencing token + 补偿
```

横向对比可以记住：

- Go 锁解决单进程 goroutine 共享内存。
- MySQL/MongoDB 更适合用事务、条件更新、唯一约束、状态机保护最终数据。
- Redis 锁适合跨实例短时互斥。
- Kafka 更像按 key 分区串行化处理，不是通用锁服务。
- ES 用 `seq_no + primary_term` 做乐观并发，不适合当强一致事实源。
- etcd/ZooKeeper 更适合强协调、选主和严格分布式锁。

完整“锁机制与并发控制体系”已沉淀到 `技术｜分布式与高并发架构面试题.md` 的专题模块；本题只展开 Redis 锁生产实践。

**二、Redis 锁基础实现**

加锁：

```redis
SET lock:order:1001 randomToken NX PX 10000
```

含义：

- `NX`：锁不存在时才能成功。
- `PX/EX`：设置过期时间，防止客户端宕机后死锁。
- `randomToken`：本次锁请求的唯一所有者标识。

Go 中不建议使用“线程 ID”。Go 没有稳定暴露给业务代码的 goroutine ID。实际可以使用：

```text
128 位以上安全随机数
或 UUID
+ 可选 instance_id / request_id，便于排查
```

真正参与所有权判断的是随机 owner token，不是机器名、线程 ID 或 goroutine ID。

**三、加锁结果必须区分三种情况**

```text
Redis 返回成功：可以进入临界区
Redis 返回锁已存在：没有获得锁
Redis 请求超时：结果未知
```

请求超时时，不能默认认为没加上，也不能默认认为加上了。更稳妥的策略是：

```text
加锁结果不确定
    │
    ├── 不进入临界区
    └── 使用相同 token 尝试安全清理或等待 TTL 过期
```

**四、安全释放锁**

不能直接：

```redis
DEL lock:order:1001
```

因为可能出现：

```text
A 获得锁
A 执行时间过长，锁过期
B 获得同一个锁
A 执行结束
A 直接 DEL，把 B 的锁删除
```

正确方案是比较 token 后删除。

Redis 8.4+ 可以使用：

```redis
DELEX lock:order:1001 IFEQ randomToken
```

多数生产环境仍可能使用 Redis 6/7/8.0，通用做法是 Lua：

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
end
return 0
```

**五、续期怎么做？**

续期也不能直接：

```redis
PEXPIRE lockKey 10000
```

否则可能给别人的锁续期。必须原子校验 owner token：

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("PEXPIRE", KEYS[1], ARGV[2])
end
return 0
```

生产中常见模式：

```text
TTL = 9 秒
每 3 秒尝试续期
最大持锁时间 = 业务硬上限
```

这只是示例，核心原则是：

- 续期间隔明显小于 TTL，例如 TTL 的 1/3。
- 续期失败时尽快停止业务或进入补偿逻辑。
- 限制最大续期时长，不能无限续期。
- 业务结束立即取消续期 goroutine。
- 续期必须校验 owner token。

看门狗只能降低锁过期概率，不能保证绝对安全。进程长时间 STW、机器卡顿、网络分区时，续期线程也可能无法正常执行。

**六、锁的 TTL 怎么设置？**

不要简单背“统一 30 秒”。应该根据真实链路：

```text
TTL > 业务执行 P99
    + 最慢下游调用超时
    + 允许的重试时间
    + 调度 / GC / 网络抖动余量
```

例如：

```text
正常 P99：800ms
数据库超时：1s
没有内部重试
抖动预留：1s
```

初始 TTL 可以考虑 3 到 5 秒，而不是机械设置 30 秒。最终必须根据监控调整。

如果任务耗时无法明确上界，例如第三方生成大文件、长时间 AI 任务、批量数据迁移、人工审批，就不应该只依赖固定 TTL，而应改成：

```text
任务状态机 + worker owner + heartbeat + lease + fencing token + 可恢复执行
```

**七、抢锁失败怎么处理？**

不能无脑死循环：

```go
for {
    tryLock()
}
```

这会造成 Redis 热点和重试风暴。应该使用：

```text
最大等待时间 + 指数退避 + 随机抖动 jitter + context 取消
```

例如：

```text
第一次等待 20 到 40ms
第二次等待 40 到 80ms
第三次等待 80 到 160ms
超过总等待时间则失败
```

如果业务不是必须等待锁，建议快速失败或降级，避免所有请求一起阻塞。

**八、Redis 锁必须面对的生产问题**

**问题 1：锁过期，但业务没有完成**

```text
A 获得锁
A 执行慢 SQL / RPC 卡顿 / 进程暂停
锁过期
B 获得锁
A 和 B 同时执行
```

解决不是单选，而是组合：

```text
合理 TTL + 有上限自动续期 + 业务幂等 + 数据库条件更新 + fencing token
```

**问题 2：Redis 主从故障切换导致锁丢失**

Redis 主从复制是异步的，可能出现：

```text
A 在主节点加锁成功
锁还没有复制到从节点
主节点宕机
从节点提升为主
B 再次获得同一个锁
```

因此单 Redis 锁更适合：

- 重复执行只产生额外成本。
- 业务有幂等兜底。
- 极少数故障下允许重复。
- 锁用于缓存重建、防抖、非核心任务。

**问题 3：旧持锁者恢复后继续写**

```text
A token=100 获得锁
A 暂停
锁过期
B token=101 获得锁并完成写入
A 恢复，使用旧结果覆盖 B
```

随机 UUID 只能防止 A 删除 B 的锁，不能防止 A 继续写数据库、文件或外部系统。这时需要 fencing token。

**问题 4：锁热点**

```text
lock:global
lock:all_orders
lock:all_products
```

所有请求竞争同一把锁，系统最终退化成单线程。应尽量按业务资源拆分：

```text
lock:order:{orderId}
lock:product:{productId}
lock:user:{userId}
```

但锁粒度过细会增加多锁管理和死锁风险，所以要围绕业务不变量设计。

**问题 5：多把锁死锁**

```text
请求 A：先锁用户，再锁订单
请求 B：先锁订单，再锁用户
```

实践建议：

- 尽量避免嵌套分布式锁。
- 多资源锁按固定顺序获取。
- 每次获取都有超时。
- 获取部分失败后立即释放已获得锁。
- 优先重新设计数据模型，使用单一聚合根或数据库事务。

**问题 6：Redis 锁不公平**

`SET NX` 不保证先到先得。高竞争时可能有请求长期抢不到锁。对公平性要求高时，可考虑队列串行化、ZooKeeper 顺序节点、Redis Stream 排队或 Kafka partition 排队，但公平通常会增加复杂度和延迟。

**问题 7：拿锁后不做二次检查**

缓存重建场景中，获得锁后必须再次检查缓存：

```text
缓存未命中
   │
   ▼
获取重建锁
   │
   ▼
再次检查缓存
   │
   ├── 已存在：直接返回
   └── 仍不存在：查询 DB 并回填
```

否则可能出现 A 重建完缓存释放锁，B 随后获得锁，又重复查 DB 重建一次。

**九、什么是 fencing token？**

fencing token 是每次成功获得锁时生成的单调递增序号：

```text
第一次持锁：fence = 100
第二次持锁：fence = 101
第三次持锁：fence = 102
```

下游资源记录已经接受的最大 token：

```text
当前最大 fence = 101
收到 fence=100：拒绝
收到 fence=102：接受
```

数据库可以这样防止旧持锁者写入：

```sql
UPDATE resource
SET value = ?,
    fence_token = ?
WHERE id = ?
  AND fence_token < ?;
```

要区分两个 token：

```text
owner token：UUID，用于判断“锁是不是我的”
fencing token：单调递增数字，用于判断“我是不是最新持锁者”
```

如果业务真的依赖 fencing token 保证严格正确性，递增 token 最好来自能提供可靠顺序的数据库或共识系统，而不是普通 Redis 主从实例。

**十、Redlock 怎么评价？**

Redlock 使用多个相互独立的 Redis master，例如 5 个节点。客户端尝试加锁：

```text
成功节点数 >= 3
并且加锁耗时 < TTL
```

才认为锁成功。失败则清理所有可能已经获得的锁。

我的工程评价是：不建议把 Redlock 作为普通业务默认方案。

原因：

- 需要维护多个相互独立的 Redis master。
- 加锁链路延迟更高。
- 故障恢复、持久化和时间漂移模型复杂。
- 最终仍然建议加入 fencing token。
- 如果业务真的要求严格一致，etcd、ZooKeeper 或数据库事务通常更直接。
- 如果业务允许极低概率重复，单 Redis 锁 + 幂等通常已经足够。

选型建议：

```text
缓存重建、接口防抖：
    单 Redis 锁

普通定时任务，允许偶发重复：
    Redis 锁 + 幂等

严格单 Leader、集群协调：
    etcd / ZooKeeper

库存、余额、订单状态：
    数据库事务 / 条件更新 / 唯一约束

跨系统长任务：
    租约 + fencing token + 幂等状态机
```

**十一、生产监控要看什么？**

- 锁获取成功率。
- 锁获取失败率。
- 获取锁等待时间。
- 锁持有时间 / TTL 比例。
- 续期成功和失败次数。
- 解锁 token 不匹配次数。
- 锁过期后业务仍在运行次数。
- 热点锁 TopN。
- 单 key 抢锁 QPS。

这些指标能帮助判断：锁粒度是否过大、TTL 是否过短、是否出现旧持锁者、是否存在重试风暴。

**十二、生产中怎么选，而不是哪里都加 Redis 锁？**

**缓存重建**

推荐：

```text
Redis 锁 + 获得锁后二次检查缓存 + 较短 TTL + singleflight + DB 限流
```

重复重建通常只增加 DB 压力，不破坏核心数据，因此 Redis 锁很合适。

**支付回调**

不推荐 Redis 锁作为核心保障。推荐：

```text
业务流水号唯一索引 + 状态条件更新 + 事务 + 幂等返回
```

Redis 锁最多用于减轻瞬时重复请求，不能成为最终防线。

**库存扣减**

推荐：

```sql
UPDATE product
SET stock = stock - 1
WHERE id = ?
  AND stock > 0;
```

或使用事务行锁。不要因为拿到 Redis 锁，就认为库存一定不会超卖。

**同一订单事件按顺序处理**

推荐：

```text
Kafka 按 orderId 分区 + 单分区串行消费 + 数据库状态机 + 幂等消费
```

通常比每条消息都抢 Redis 锁更自然。

**ES 文档并发更新**

推荐：

```text
读取 seq_no + primary_term
条件更新
冲突后重新读取并合并
```

而不是围绕 ES 文档再设计 Redis 锁。

**集群只允许一个调度器执行**

```text
允许偶发重复：Redis lease + 任务幂等
不允许双 Leader：etcd / ZooKeeper leader election + fencing token
任务结果存 MySQL：也可以通过数据库抢占任务记录
```

**面试 1 分钟回答：**
Redis 分布式锁本质是一个带 TTL 的互斥租约，基础实现是 `SET key token NX PX ttl`，token 必须唯一，释放和续期都要校验 token，老版本用 Lua，Redis 8.4+ 可以用 `DELEX key IFEQ token` 做安全删除。TTL 要覆盖业务 P99 和下游抖动，但不能无限长；复杂任务要有续期、最大持锁时间和状态机。生产里还要处理加锁超时结果未知、抢锁失败退避、主从切换锁丢失、锁过期后旧持锁者继续写、热点锁和多锁死锁。Redis 锁适合缓存重建、任务抢占、防重复提交这类短时互斥，不能替代数据库事务、幂等和条件更新。强一致场景要用数据落点的原子规则兜底，必要时用 etcd/ZooKeeper 或 fencing token。

**面试追问：Redis 锁一定安全吗？**
不绝对安全。单 Redis 锁在主从异步复制、故障切换、GC 停顿、网络分区、锁过期等情况下都可能失效。生产上不能只靠锁保证正确性，要结合业务幂等、数据库条件更新、版本号、fencing token 和补偿机制。

**面试追问：锁过期业务还没执行完怎么办？**
会出现并发执行风险。解决思路是合理 TTL、有上限续期、任务拆分、状态机、业务幂等、数据库条件更新和 fencing token。不能只靠看门狗兜住所有一致性问题。

### 24. Redis 事务和数据库事务有什么区别？

**结论先行：**
Redis 事务通过 `MULTI/EXEC` 保证一组命令在服务端按顺序连续执行，但它不是 MySQL/MongoDB 那种完整数据库事务，不支持运行时错误自动回滚，也不负责跨系统一致性。面试里要把 Redis、MySQL、MongoDB 的事务目标区分清楚：Redis 解决“多条 Redis 命令连续执行”，MySQL 解决“关系型数据强一致变更”，MongoDB 解决“文档模型下少量跨文档/跨集合一致变更”。

**分层展开：**

**1. Redis 事务解决什么问题**
- 把多条 Redis 命令放入队列，最后由 `EXEC` 一次性按顺序执行。
- 执行期间不会被其他客户端命令插入，具备“连续执行”的隔离效果。
- 配合 `WATCH` 可以做乐观锁：如果被监控 key 在 `EXEC` 前被别人改过，本次事务失败。
- 适合简单的缓存状态更新、计数、队列元数据变更等 Redis 内部原子组合。

**2. Redis 事务如何使用**

```redis
WATCH stock:sku:1001
GET stock:sku:1001
MULTI
DECR stock:sku:1001
LPUSH order:queue order_123
EXEC
```

- `WATCH`：监控 key，提供乐观锁语义。
- `MULTI` 开启事务。
- 普通命令进入队列，不会立即执行。
- `EXEC` 提交队列。
- `DISCARD` 放弃事务队列。
- `UNWATCH` 取消监控。

**3. Redis 事务的关键限制**
- 命令运行时报错，不会回滚前面已经执行的命令。
- 命令入队时的语法错误会导致事务整体不执行。
- 不支持复杂条件分支，复杂原子逻辑通常更适合 Lua。
- 不保证跨 Redis、DB、MQ 的一致性。
- 不等于关系型数据库的 ACID 事务。

**4. Redis、MySQL、MongoDB 事务对比**

| 组件 | 事务设计目标 | 原子性范围 | 隔离/并发控制 | 回滚能力 | 典型使用场景 |
| --- | --- | --- | --- | --- | --- |
| Redis | 多条 Redis 命令连续执行，减少并发插入 | 单实例内一组命令 | 单线程顺序执行 + `WATCH` 乐观锁 | 不支持运行时错误自动回滚 | 简单计数、队列元数据、库存预扣、缓存状态切换 |
| MySQL/InnoDB | 关系型数据强一致变更 | 一个本地事务内多行多表 | MVCC、行锁、next-key lock、隔离级别 | 支持 rollback，依赖 undo log | 订单、库存、账务、权限、工单状态 |
| MongoDB | 文档模型下补充跨文档一致性 | 单文档天然原子，多文档事务可跨集合 | WiredTiger MVCC、snapshot、write conflict retry | 支持事务 abort | 配置发布、任务状态、少量跨集合一致变更 |

**5. 和 MySQL 事务的核心差异**
- MySQL 事务追求完整 ACID，Redis 事务更像“命令队列 + 连续执行”。
- MySQL 有 undo log 支持回滚，Redis 执行中某条命令失败不会自动撤销前面命令。
- MySQL 事务可以通过隔离级别控制并发读写现象，Redis 事务主要依赖单线程执行和 `WATCH` 乐观锁。
- MySQL 是事实数据源时，余额、订单、库存最终正确性应落在 MySQL 事务、唯一约束、条件更新和状态机上。

**6. 和 MongoDB 事务的核心差异**
- MongoDB 单文档更新天然原子，很多场景通过嵌入式建模避免事务。
- MongoDB 多文档事务适合少量跨文档/跨集合一致变更，但成本比单文档写高。
- Redis 事务通常只管理缓存/内存状态，MongoDB 事务管理持久化文档数据。
- MongoDB 事务仍要处理 `TransientTransactionError`、`UnknownTransactionCommitResult` 等重试场景。

**使用建议**
- Redis 内部简单组合：可以用 `MULTI/EXEC`。
- 需要判断、循环、校验后删除：优先 Lua，例如分布式锁释放、滑动窗口限流。
- 需要最终数据正确：优先 MySQL/MongoDB 的条件更新、事务、唯一约束和状态机。
- 跨系统一致性不要依赖 Redis 事务。
- 写 DB + 删缓存：不能用 Redis 事务解决，应该用 Cache Aside、重试、MQ、Outbox、binlog/Change Stream 补偿。

**工程面试表达：**
Redis 事务不是为了替代数据库事务，而是为了让 Redis 内部多条命令连续执行。真正的数据一致性要看数据落点：关系型强一致用 MySQL 事务，文档型聚合根优先用 Mongo 单文档原子更新，少量跨文档再用 Mongo 事务；缓存只做加速，不能当事实源。

**面试追问：为什么 Redis 事务不支持回滚？**
Redis 命令执行失败通常是编程错误或类型错误，官方设计上更强调简单和性能。如果需要复杂事务语义，应该由业务或数据库保证。

**面试追问：Redis 事务、Lua、Pipeline 怎么选？**
- 只想减少网络往返：Pipeline。
- 要多条命令连续执行，但逻辑简单：`MULTI/EXEC`。
- 要校验 token、判断库存、滑动窗口等复杂原子逻辑：Lua。
- 要保证订单、库存、资金等最终正确：数据库事务/条件更新，而不是 Redis 事务。

### 25. Lua 脚本有什么优势和风险？

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

### 26. Redis fork 子进程时有什么风险？

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

### 27. Redis Cluster 脑裂是什么？怎么避免？

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

### 28. 如何设计一个高并发商品详情缓存？

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

### 29. 如何用 Redis 设计排行榜？

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

### 30. 如何用 Redis 做限流？

**结论先行：**
Redis 可以实现固定窗口、滑动窗口、令牌桶、漏桶和并发数限制。简单低成本场景用 `INCR + EXPIRE`，精确窗口用 `ZSet + Lua`，允许突发用令牌桶，要求匀速排队用漏桶。工程上要先确定限流维度、时间窗口、失败策略和兜底方案，不是所有限流都必须用 Redis；单机限流、网关限流、Nginx/Envoy、Sentinel、令牌桶库、MQ 削峰都可以做限流。

**分层展开：**

**1. 限流先明确四个问题**
- 限谁：用户、IP、账号、租户、广告账户、接口、平台、全局。
- 限什么：QPS、并发数、每日总量、外部 API 配额、任务消费速度。
- 超限怎么办：直接拒绝、排队等待、降级返回、异步重试、进入死信。
- 放在哪里：本机、网关、Redis 全局、服务端队列、MQ、第三方 SDK。

**2. 固定窗口：`INCR + EXPIRE`**

适合：简单接口限流、低成本粗粒度保护，例如每个用户每分钟最多 100 次。

```redis
INCR rate:user:1001:202607121430
EXPIRE rate:user:1001:202607121430 70
```

- key 按限流维度和窗口时间命名。
- 第一次 `INCR` 后设置过期时间。
- 查询计数是否超过阈值。
- 优点：实现简单，性能好，内存少。
- 缺点：窗口边界有突刺，比如 10:00:59 和 10:01:00 各打满一次，瞬时流量可能翻倍。

更严谨的固定窗口 Lua 思路：

```lua
local current = redis.call("INCR", KEYS[1])
if current == 1 then
  redis.call("EXPIRE", KEYS[1], ARGV[2])
end
if current > tonumber(ARGV[1]) then
  return 0
end
return 1
```

- `KEYS[1]`：限流 key。
- `ARGV[1]`：窗口内最大请求数。
- `ARGV[2]`：key 过期秒数。

**3. 滑动窗口：`ZSet + Lua`**

适合：精确控制最近 N 秒/N 分钟请求数，尤其适合外部媒体 API 配额。

- 用 ZSet 存请求时间戳。
- 删除窗口外数据。
- 统计窗口内请求数。
- 判断是否放行。

核心命令：

```redis
ZREMRANGEBYSCORE rate:facebook:act_1001:campaign_list:1s -inf now-1000
ZCARD rate:facebook:act_1001:campaign_list:1s
ZADD rate:facebook:act_1001:campaign_list:1s now requestId
EXPIRE rate:facebook:act_1001:campaign_list:1s 2
```

- `ZREMRANGEBYSCORE`：清理窗口外请求。
- `ZCARD`：统计当前窗口内请求量。
- `ZADD`：未超限才写入当前请求。
- `EXPIRE`：避免冷 key 常驻。
- 多条命令必须用 Lua 包起来，否则并发下可能多个请求同时看到未超限而全部放行。

单窗口 Lua 思路：

```lua
redis.call("ZREMRANGEBYSCORE", KEYS[1], "-inf", ARGV[1] - ARGV[2])
local count = redis.call("ZCARD", KEYS[1])
if count >= tonumber(ARGV[3]) then
  return 0
end
redis.call("ZADD", KEYS[1], ARGV[1], ARGV[4])
redis.call("PEXPIRE", KEYS[1], ARGV[2] + 1000)
return 1
```

- `ARGV[1]`：当前毫秒时间戳。
- `ARGV[2]`：窗口大小，单位毫秒。
- `ARGV[3]`：窗口最大请求数。
- `ARGV[4]`：本次请求唯一 ID。

**4. 多层滑动窗口：外部 API 工程示例**

场景：调用 Facebook 获取 campaign 列表，平台限制：
- 每秒最多 60 次。
- 每 5 分钟最多 3000 次。
- 每天最多 10000 次。

限流 key 设计：

```text
rate:facebook:{account_id}:campaign_list:1s
rate:facebook:{account_id}:campaign_list:5m
rate:facebook:{account_id}:campaign_list:1d
```

工程执行逻辑：
- 请求进入时生成 `requestId`。
- 同一个 Lua 脚本检查三个窗口。
- 任一窗口超限，返回拒绝和剩余等待时间。
- 三个窗口都通过，才把本次请求写入三个 ZSet。
- 调用 Facebook 成功/失败都算一次消耗，避免失败重试击穿平台配额。
- 平台返回限流错误时，动态降低该账号/接口的阈值，进入冷却窗口。

伪代码：

```go
allowed, retryAfter, err := limiter.Allow(ctx, []Window{
    {Key: "rate:facebook:act_1001:campaign_list:1s", Limit: 60, Window: time.Second},
    {Key: "rate:facebook:act_1001:campaign_list:5m", Limit: 3000, Window: 5 * time.Minute},
    {Key: "rate:facebook:act_1001:campaign_list:1d", Limit: 10000, Window: 24 * time.Hour},
})
if err != nil {
    // Redis 异常时按业务重要性选择 fail open 或 fail closed
    return err
}
if !allowed {
    return ErrRateLimited{RetryAfter: retryAfter}
}
return facebookClient.ListCampaigns(ctx, accountID)
```

**5. 令牌桶：允许突发，长期平均受控**

适合：接口允许短时间突发，但长期 QPS 不能超过平均速率，例如每秒补充 100 个 token，桶容量 500。

设计思想：

```text
系统按固定速率生成 token
        │
        ▼
Token Bucket，最多存 capacity 个 token
        │
        ▼
请求到来，先消耗 token
        │
        ├── 有 token：放行
        └── 无 token：拒绝 / 等待 / 降级
```

核心参数：
- `rate`：每秒生成多少 token，决定长期平均速率。
- `capacity`：桶最多存多少 token，决定最大突发能力。
- `requested`：每次请求消耗多少 token，普通请求一般是 1。

例如 `rate=100/s，capacity=500`：
- 长期平均每秒最多 100 个请求。
- 低峰期可以积攒 token。
- 突发时最多可以瞬间放行约 500 个请求。

- 按固定速率生成令牌。
- 请求消耗令牌。
- 允许一定突发流量。
- 桶容量决定最大突发。
- Redis 中通常存 `tokens` 和 `lastRefillTime`，用 Lua 原子计算补充和扣减。

适合场景：
- 用户接口 QPS 控制。
- 允许瞬时峰值的网关限流。
- 外部 API 在短期突发不敏感、只看平均速率时。

Go 本地令牌桶实现：

```go
package main

import (
	"context"
	"fmt"
	"time"

	"golang.org/x/time/rate"
)

func main() {
	// 每秒生成 5 个 token，桶容量为 10，表示平均 QPS=5，最大突发=10。
	limiter := rate.NewLimiter(rate.Limit(5), 10)

	for i := 0; i < 20; i++ {
		if limiter.Allow() {
			fmt.Println("allowed", i)
		} else {
			fmt.Println("limited", i)
		}
		time.Sleep(100 * time.Millisecond)
	}

	// Wait 会等待 token，适合愿意排队的调用；ctx 控制最多等待多久。
	ctx, cancel := context.WithTimeout(context.Background(), 200*time.Millisecond)
	defer cancel()

	if err := limiter.Wait(ctx); err != nil {
		fmt.Println("wait token failed:", err)
		return
	}
	fmt.Println("allowed after wait")
}
```

Redis 全局令牌桶 Lua：

```lua
-- KEYS[1]：令牌桶 key
-- ARGV[1]：当前毫秒时间 now
-- ARGV[2]：生成速率 rate，每秒生成多少 token
-- ARGV[3]：桶容量 capacity
-- ARGV[4]：本次请求消耗 token 数，一般是 1
-- ARGV[5]：key 过期时间，秒

local bucket = redis.call("HMGET", KEYS[1], "tokens", "last_refill")

local tokens = tonumber(bucket[1])
local last_refill = tonumber(bucket[2])

local now = tonumber(ARGV[1])
local rate = tonumber(ARGV[2])
local capacity = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])
local ttl = tonumber(ARGV[5])

if tokens == nil then
  tokens = capacity
end

if last_refill == nil then
  last_refill = now
end

local delta = math.max(0, now - last_refill)
local refill = delta / 1000 * rate

tokens = math.min(capacity, tokens + refill)
last_refill = now

local allowed = 0

if tokens >= requested then
  tokens = tokens - requested
  allowed = 1
end

redis.call("HMSET", KEYS[1], "tokens", tokens, "last_refill", last_refill)
redis.call("EXPIRE", KEYS[1], ttl)

return {allowed, tokens}
```

Go 调用参数示例：

```go
// 每秒补充 60 个 token，桶容量 120，适合允许短时突发的外部 API 调用。
allowed, tokens := evalTokenBucket(
	ctx,
	"rate:token:facebook:act_1001:campaign_list",
	time.Now().UnixMilli(),
	60,
	120,
	1,
	300,
)
```

**6. 漏桶：匀速流出，削峰排队**

适合：要求下游处理速度稳定，宁愿排队也不想突发打爆下游。

设计思想：

```text
请求进入桶
    │
    ▼
Leaky Bucket，有固定容量
    │
    ▼
按固定速率漏出
    │
    ▼
真正执行业务 / 调用下游
```

核心参数：
- `capacity`：桶容量，也就是最大排队数量。
- `leakRate` 或 `interval`：漏出速率，决定下游看到的处理速度。
- `maxDelay`：最大可接受排队等待时间，超过就拒绝。

例如 `capacity=1000，leakRate=100/s`：
- 最多排队 1000 个请求。
- 下游每秒稳定处理 100 个请求。
- 超过桶容量或最大等待时间就拒绝，避免排队无限增长。

- 请求进入桶，按固定速率流出。
- 桶满则拒绝或丢弃。
- 更像队列削峰，不强调突发能力。
- Redis 可以存队列和调度时间，但高可靠排队更建议 MQ。

适合场景：
- 短信/邮件发送。
- 外部 API 慢速消费。
- 批量任务稳定出队。

Go 本地漏桶实现：

```go
package main

import (
	"fmt"
	"time"
)

type LeakyBucket struct {
	queue chan int
}

func NewLeakyBucket(capacity int, interval time.Duration) *LeakyBucket {
	b := &LeakyBucket{
		queue: make(chan int, capacity),
	}

	go func() {
		ticker := time.NewTicker(interval)
		defer ticker.Stop()

		for range ticker.C {
			select {
			case req := <-b.queue:
				fmt.Println("process request:", req)
			default:
			}
		}
	}()

	return b
}

func (b *LeakyBucket) Allow(req int) bool {
	select {
	case b.queue <- req:
		return true
	default:
		return false
	}
}

func main() {
	// 桶容量 5，每 200ms 漏出 1 个请求，即每秒处理 5 个。
	bucket := NewLeakyBucket(5, 200*time.Millisecond)

	for i := 0; i < 20; i++ {
		if bucket.Allow(i) {
			fmt.Println("enqueue", i)
		} else {
			fmt.Println("reject", i)
		}
		time.Sleep(50 * time.Millisecond)
	}

	time.Sleep(2 * time.Second)
}
```

Redis 全局漏桶 Lua：

```lua
-- KEYS[1]：漏桶 key
-- ARGV[1]：当前毫秒时间 now
-- ARGV[2]：漏出间隔 intervalMs，例如每 100ms 处理 1 个
-- ARGV[3]：最大排队等待时间 maxDelayMs
-- ARGV[4]：key 过期时间，秒

local now = tonumber(ARGV[1])
local interval = tonumber(ARGV[2])
local max_delay = tonumber(ARGV[3])
local ttl = tonumber(ARGV[4])

local last_time = tonumber(redis.call("GET", KEYS[1]))

if last_time == nil then
  last_time = now
end

local scheduled_time = math.max(now, last_time) + interval
local delay = scheduled_time - now

if delay > max_delay then
  return {0, delay}
end

redis.call("SET", KEYS[1], scheduled_time, "EX", ttl)

return {1, delay}
```

Go 调用参数示例：

```go
allowed, delay := evalLeakyBucket(
	ctx,
	"rate:leaky:facebook:act_1001:campaign_list",
	time.Now().UnixMilli(),
	100,  // 每 100ms 漏出 1 个，即每秒 10 个
	1000, // 最多允许排队 1 秒
	300,
)

if !allowed {
	return ErrRateLimited
}

if delay > 0 {
	time.Sleep(time.Duration(delay) * time.Millisecond)
}

return callFacebookAPI()
```

**7. 并发数限制：计数器或信号量**

适合：限制“同时执行中的任务数”，不是限制 QPS。

```redis
INCR running:facebook:campaign_list
EXPIRE running:facebook:campaign_list 60
DECR running:facebook:campaign_list
```

- 进入时 `INCR`，超过并发上限则拒绝并 `DECR`。
- 退出时 `DECR`。
- 必须设置 TTL，防止进程异常导致计数不释放。
- 更严谨可以用 ZSet 存执行中的 requestId 和过期时间，定期清理超时请求。

**工程建议**
- 限流逻辑用 Lua 保证原子。
- key 设置 TTL，避免无限增长。
- 高 QPS 全局限流不要让 Redis 成为瓶颈，可结合本地限流。
- key 建议包含平台、账号、接口和窗口，例如 `rate:facebook:{account_id}:campaign_list:1s`，方便隔离不同媒体账号和接口。
- 限流失败策略要明确：核心保护下游时 fail closed，用户体验优先时可 fail open + 本地限流兜底。
- 限流命中要打点：限流 key、接口、账号、窗口、阈值、当前计数、拒绝次数。
- Redis 集群下同一个限流维度要落到同一 hash slot，可使用 hash tag，例如 `rate:{facebook:act_1001:campaign_list}:1s`。
- 大规模滑动窗口会消耗较多内存，窗口越大、请求越多，ZSet 元素越多；高 QPS 可用令牌桶或本地预分配 token 降低 Redis 压力。

**高并发下如何保护数据库和下游 RPC**

限流不是只在 Redis 做一个 key，而是分层保护：

```text
入口网关粗限流
    ↓
服务本地令牌桶 / semaphore
    ↓
Redis 全局账号/API 限流
    ↓
连接池 / worker pool 有界并发
    ↓
DB 条件更新 / MQ 削峰 / 下游降级
```

保护 MySQL 回源：
- Redis 缓存 miss 后，不要让所有请求直接查 DB。
- 本地令牌桶限制回源 QPS。
- semaphore 限制同时查 DB 的 goroutine 数。
- 热点 key 用 singleflight 合并请求。
- DB 侧用连接池上限、慢 SQL 告警和条件更新兜底。

保护下游 RPC / 外部 API：
- 单实例用本地 limiter 控制当前进程调用速率。
- 多实例共享额度用 Redis 滑动窗口或令牌桶。
- 账号/API/租户维度拆 key，避免互相影响。
- 下游返回限流错误时，动态降速并进入冷却窗口。
- 超限请求可以拒绝、延迟重试、投递 MQ 或降级返回。

**只有 Redis 可以做限流吗？不是。**

| 方案 | 典型实现 | 优点 | 缺点 | 适合场景 |
| --- | --- | --- | --- | --- |
| Go 本地限流 | `golang.org/x/time/rate` 令牌桶、`semaphore` 并发控制 | 极快，无网络开销，延迟最低 | 单机生效，多实例不共享额度 | 单实例保护、客户端 SDK、Redis 故障兜底 |
| Redis 全局限流 | `INCR/EXPIRE`、`ZSet + Lua`、令牌桶 Lua | 多实例共享额度，灵活按用户/账号/API 限制 | 有网络开销，Redis 可能成瓶颈 | 多实例服务、外部 API 账号级配额 |
| Nginx 限流 | `limit_req`、`limit_conn` | 接入层简单高效 | 业务维度有限，动态规则弱 | IP、URI 粗粒度入口保护 |
| Envoy/API Gateway | local/global rate limit service | 统一入口治理，规则集中 | 需要网关体系和配置治理 | 微服务统一入口、租户/API 配额 |
| Sentinel/Hystrix 类库 | QPS、并发、熔断、热点参数 | 与服务治理结合，降级能力强 | 语言/框架绑定，跨实例全局能力取决于后端 | Java 微服务、热点接口保护 |
| MQ/队列削峰 | Kafka/RocketMQ/任务队列 | 平滑流量，保护下游，可靠重试 | 不是实时拒绝，延迟增加 | 批量任务、异步创建、报表拉取 |
| 数据库条件更新 | `UPDATE quota SET used=used+1 WHERE used<limit` | 强一致，和业务数据同事务 | 性能低，不适合高 QPS 入口 | 资金、库存、强配额扣减 |
| CDN/WAF/云网关 | 云厂商限流、Bot 防护 | 抗大流量，离用户近 | 业务细粒度弱，成本和配置依赖平台 | 公网入口、防刷、防攻击 |

**性能和成本粗略判断**
- 本地限流最快：内存操作，通常纳秒到微秒级，但只能保护单实例。
- Nginx/网关限流很快：适合入口粗粒度限流。
- Redis 限流多一次网络往返：通常毫秒内，但热 key、高 QPS Lua、ZSet 大窗口会带来 CPU 和内存压力。
- DB 限流最重：有锁、事务和磁盘成本，不适合普通接口入口限流，只适合强一致额度扣减。
- MQ 削峰不追求立即返回，核心价值是把突发流量变成可控消费速度。

**面试 1 分钟回答：**
Redis 限流常见有固定窗口、滑动窗口、令牌桶、漏桶和并发数限制。固定窗口用 `INCR + EXPIRE`，简单高效但有边界突刺；精确滑动窗口用 `ZSet + Lua`，按时间戳清理窗口外请求并统计窗口内数量；令牌桶适合允许突发但控制长期平均 QPS；漏桶适合匀速排队保护下游。像 Facebook campaign 列表这种外部 API，我会按平台、账号、接口做多层窗口，例如 `1s/60`、`5m/3000`、`1d/10000`，用 Lua 保证多个窗口检查和写入原子性。限流不只有 Redis，本地令牌桶、Nginx/网关、Sentinel、MQ 削峰、数据库条件更新都能做，选择时看是否需要全局共享额度、精度、延迟、成本和失败兜底。

**面试追问：为什么限流要用 Lua？**
因为限流通常包含读计数、判断、写计数、设置过期等多个步骤。Lua 能保证这些步骤在 Redis 中原子执行，避免并发穿透。

**面试追问：Redis 限流失败时应该放行还是拒绝？**
看业务目标。如果限流是为了保护第三方 API 或数据库，Redis 异常时倾向 fail closed 或本地小额度放行，避免下游被打爆；如果是非核心用户体验接口，可以 fail open，同时打告警并启用本地限流兜底。不能默认所有场景都放行。

**面试追问：高 QPS 下 Redis 限流会不会成为瓶颈？**
会。热点限流 key、复杂 Lua、大 ZSet 窗口都会增加 Redis CPU、内存和网络压力。优化方式包括本地预限流、令牌预取、分维度拆 key、缩短窗口、聚合上报、网关粗限流 + Redis 精限流，以及给 Redis 做分片和监控。

### 31. 如何用 Redis 实现延迟队列？

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

### 32. 如何用 Redis 实现点赞系统？

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

### 33. 如何设计 Redis 缓存预热？

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

### 34. Redis 和本地缓存怎么配合？

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

### 35. 缓存数据要不要存 null？

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

### 36. Redis 适合做数据库吗？

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

### 线上 Redis 生产故障总手册：从告警到恢复

> 这是一张总流程图。第 41～46 题分别展开慢查询、CPU、连接数、超时、命中率和 Redis 故障保护 DB；实际排障时先按本节确定故障分支，再进入对应小节。

#### 1. 先明确故障目标：保护什么、影响谁

收到 Redis 告警后，先用 1～3 分钟回答四个问题：

1. **影响面：** 是单个接口、单个业务、单个 Redis 节点，还是所有依赖 Redis 的服务？
2. **业务后果：** 是延迟升高、请求失败、缓存 miss、数据暂时不可见，还是写入/扣减等核心操作失败？
3. **资源位置：** 是应用连接池、网络、Redis 主线程、后台持久化，还是数据库被回源流量打高？
4. **时间线：** 故障开始前是否发生发布、扩容、配置变更、批处理、缓存清理、主从切换或流量活动？

不能看到 `CPU 高` 就直接扩容，也不能看到 `Redis 超时` 就直接重启。先把应用、Redis 和 DB 的时间线对齐，才能避免错误止血。

#### 2. 第一阶段：快速确认与止血

**应用侧先看：**

- Redis 调用 QPS、成功率、P50/P95/P99、超时率和错误类型。
- 连接池获取等待时间、活跃连接、空闲连接和重试次数。
- 哪些接口、服务实例、租户或 key 前缀异常。
- Redis 失败后是否大量回源 DB，DB QPS、CPU、连接池等待是否同步上涨。

**Redis 侧先看：**

```redis
INFO server
INFO clients
INFO memory
INFO stats
INFO commandstats
INFO persistence
INFO replication
SLOWLOG GET 20
LATENCY LATEST
```

**止血动作必须和现象对应：**

| 现象 | 优先止血动作 | 不要直接做的事 |
|---|---|---|
| 新版本发布后延迟或错误突增 | 暂停发布、关闭 feature flag、回滚版本 | 先加 Redis 连接数或盲目扩容 |
| Redis 故障导致 DB 回源上涨 | 限制回源并发、接口限流、非核心功能降级、本地旧值兜底 | 让所有请求继续重试和回源 |
| 某个调用方重试风暴 | 按应用/接口摘流、限制重试次数、增加退避 | 继续放大重试次数 |
| 单节点 CPU 或带宽打满 | 热 key 分流、暂停扫描/批处理、拆大 key 或限制大请求 | 只把 `maxclients` 调大 |
| 连接数接近上限 | 找异常客户端并摘流，收紧连接池上限 | 直接无限提高 `maxclients` |
| 主从延迟或副本异常 | 从读节点摘流；强一致读取临时走主节点 | 继续把实时读发给落后副本 |
| 大 key/慢命令阻塞 | 停止问题命令来源，改分页、拆 key，删除用 `UNLINK` | 在线执行 `KEYS`、`HGETALL` 或大范围集合运算 |
| 持久化或磁盘抖动 | 暂停低优先级批量写，检查磁盘和 fork 压力 | 未确认数据安全前直接删除 AOF/RDB |

**止血的判断原则：**

- 限流发生在网关、服务入口、消费者或 Redis 调用封装层，目标是减少进入 Redis 的请求量。
- 降级只关闭非核心能力，例如推荐、报表、预热、非实时统计；不能把扣库存、投放状态等核心写操作静默丢弃。
- 本地缓存适合短时间兜底可重建数据；旧值要标记允许短暂过期，不能把错误数据伪装成最新数据。
- 主从切换或读主只解决读副本不可用/延迟问题，不能解决主节点本身 CPU、内存或网络已打满的问题。
- 任何重试都必须有次数上限、退避和熔断，否则 Redis 故障会被重试流量放大。

#### 3. 第二阶段：按现象进入排查分支

```text
Redis 请求变慢或失败
        |
        +-- 连接池等待高？ -> 查连接泄漏、池配置、实例数、阻塞连接
        |
        +-- slowlog/commandstats 高？ -> 查慢命令、Lua、返回结果大小
        |
        +-- CPU 高但慢命令不明显？ -> 查热 key、过期淘汰、扫描、持久化
        |
        +-- 网络 RTT/重传/带宽异常？ -> 查跨机房、丢包、大 value、大响应
        |
        +-- 命中率下降且 DB 被打高？ -> 查 key 规则、TTL、淘汰、回填和流量结构
        |
        +-- 副本延迟/故障？ -> 查复制链路、主从状态、读路由和故障转移
        |
        +-- 内存接近上限？ -> 查大 key、碎片、增长速度、淘汰和容量规划
```

#### 4. 第三阶段：最小证据集

排查不能只凭截图或单点指标。至少保留以下证据，并记录采集时间：

| 维度 | 需要保留的证据 | 用来回答的问题 |
|---|---|---|
| 请求端 | 接口 QPS、P99、错误码、Redis span、pool wait、重试数 | 请求在哪里变慢或失败？ |
| 命令端 | `SLOWLOG`、`INFO commandstats`、命令调用量和平均耗时 | 哪类命令变重或变多？ |
| 数据端 | key 前缀、`TYPE`、`MEMORY USAGE`、集合规模、TTL | 是大 key、热 key、过期还是数据模型问题？ |
| 实例端 | CPU、内存、RSS、碎片率、连接、带宽、blocked clients | Redis 自身哪个资源到瓶颈？ |
| 持久化 | RDB/AOF rewrite、fork、fsync、磁盘延迟 | 是否是后台任务造成抖动？ |
| 高可用 | 主从 offset、复制延迟、角色、故障转移日志 | 是否是复制或路由问题？ |
| 变更 | 发布、配置、运维命令、批任务、活动流量 | 故障是否由近期变化触发？ |

常用的安全检查命令：

```redis
TYPE some:key
TTL some:key
MEMORY USAGE some:key
OBJECT ENCODING some:key
INFO keyspace
CLIENT LIST
ROLE
```

生产环境使用这些命令时，要避免对未知大 key 执行全量读取；`MEMORY USAGE`、`TYPE`、`TTL` 也要控制采样范围，不能把排查命令变成新的压力来源。

#### 5. 第四阶段：根因定位后的修复选择

| 根因 | 短期修复 | 长期治理 |
|---|---|---|
| 错误命令或错误参数 | 回滚、关闭开关、限制请求规模 | SDK 封装禁用危险命令，增加参数上限和代码扫描 |
| 大 key | 分批读取/删除，`UNLINK`，暂停生产者 | 按用户、时间或业务维度拆 key，限制集合规模 |
| 热 key | 本地缓存、副本 key、限流 | 热点识别、分层缓存、热点数据预案 |
| 缓存雪崩/命中率下降 | 保护 DB、分批预热、延长核心数据 TTL | TTL 抖动、singleflight、缓存准入和回填监控 |
| 连接池过大或泄漏 | 摘异常实例、收紧池上限、重启泄漏实例 | 按实例数计算连接预算，连接池指标和压测校准 |
| 内存/碎片/淘汰 | 清理低价值数据、扩容、降低写入 | 容量水位、内存模型、active defrag 和拆实例 |
| 主从延迟 | 摘除落后副本，核心读走主 | 复制链路监控、读路由策略、故障演练 |
| 单实例能力不足 | 分流、扩容、迁移 slot | Cluster 分片、业务隔离、容量和增长规划 |

#### 6. 恢复阶段：避免“恢复即二次故障”

1. 先确认 Redis 延迟、错误率、连接数和 CPU 已恢复，而不是只看进程存活。
2. 先恢复核心读写，再逐步恢复非核心接口、批任务和预热任务。
3. 缓存为空时按热点、租户、接口分批预热，限制回源并发和写入速率。
4. 主从恢复后确认复制 offset、延迟和数据追平，再把副本加入读流量。
5. 观察至少一个完整业务高峰或 TTL 周期，确认命中率、DB 压力和淘汰没有再次异常。

#### 7. 复盘必须回答的五个问题

- 为什么监控能发现 Redis 异常，却不能快速定位到具体应用、命令或 key 前缀？
- 为什么故障没有被限流、熔断、隔离或降级挡住？
- 是否存在危险命令、大 key、热 key、连接池无上限或重试无上限？
- Redis 恢复后是否会导致缓存击穿、DB 雪崩或流量瞬时回灌？
- 是否需要补充容量水位、命令审计、key 采样、演练和自动化止血能力？

**面试中的完整回答模板：**

> 我会先确认影响范围和时间线，同时看应用 Redis 调用耗时、连接池等待、错误类型、Redis `slowlog/commandstats/latency` 以及 DB 回源压力。根据现象做定向止血：新发布先回滚，异常调用方限流并关闭重试，非核心功能降级，Redis 故障时限制回源并保护 DB，副本延迟则摘除副本。业务稳定后，再从请求端、命令端、数据端、实例资源、持久化和主从复制六个方向定位，最后按根因选择改命令、拆 key、治理热 key、调整 TTL、修复连接池、扩容分片或调整架构。恢复时分批放量并验证命中率、DB 压力和复制延迟，最后补监控、限额和故障演练。

### 37. 什么是大 key？有什么危害？

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

### 38. 什么是热 key？怎么处理？

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

### 39. Redis 内存碎片是什么？怎么治理？

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

### 40. 如何避免 Redis 缓存污染？

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

### 41. Redis 慢查询怎么排查？

**结论先行：**
Redis 慢查询排查不能只看 `SLOWLOG`。生产上要先确认“慢”发生在哪里：客户端排队、网络传输、Redis 执行命令、返回大结果、还是 fork/fsync 等系统抖动。排查路径是先止血，再按请求端、Redis 自身、数据层和系统层逐层定位。

**分层展开：**

**常见现象**
- 接口 P95/P99 延迟突然升高，Redis 调用耗时从几毫秒变成几十/几百毫秒。
- 客户端出现 `read timeout`、`context deadline exceeded`。
- DB QPS 可能被带高，因为 Redis 慢导致缓存读失败或降级回源。
- 监控上 Redis `instantaneous_ops_per_sec`、CPU、带宽、`connected_clients`、慢查询数异常。

**一般怎么发现**
- APM/链路追踪看到 Redis span 耗时升高。
- 客户端埋点看到 Redis 命令耗时、连接池等待时间升高。
- Redis `SLOWLOG GET` 出现慢命令。
- Redis latency monitor 出现 command、fork、aof、expire-cycle 等事件。
- 业务报警：接口超时率、错误率、DB 回源量、缓存命中率异常。

**第一步：先判断是不是 Redis 执行慢**

```redis
SLOWLOG GET 20
SLOWLOG LEN
INFO commandstats
LATENCY LATEST
LATENCY DOCTOR
```

- 如果客户端耗时高，但 `SLOWLOG` 没有记录，优先怀疑连接池等待、网络、大 value 传输、客户端 GC 或请求排队。
- 如果 `SLOWLOG` 里有慢命令，继续看命令类型、key、执行耗时和出现时间。
- `SLOWLOG` 只记录 Redis 执行命令的时间，不包含客户端排队、网络传输和结果读取时间。

**第二步：从命令层排查**

- 是否出现 `KEYS`、`FLUSHALL`、`SORT`、`SUNION`、`SINTER`、`HGETALL`、`SMEMBERS`、大范围 `ZRANGE/ZREVRANGE`。
- 是否有 Lua 脚本执行过久，脚本里循环处理大量 key 或大集合。
- 是否有批量 `MGET/MSET` 一次带太多 key。
- 是否一次返回大量数据，服务端执行不慢，但网络传输和客户端反序列化很慢。

治理动作：
- `KEYS` 改为 `SCAN`，并控制 `COUNT` 和执行频率。
- 大集合全量读取改成分页读取，例如 `HSCAN/SSCAN/ZSCAN` 或 `ZRANGE start stop`。
- 大 Lua 拆小，避免在脚本中处理不可控数据量。
- 批量命令控制 batch size，例如每批 100～500 个 key，根据 value 大小压测调整。

**第三步：从数据层排查**

- 是否存在大 key：`redis-cli --bigkeys`、`MEMORY USAGE key`、离线分析 RDB。
- 是否存在热 key：业务埋点、代理层统计、`redis-cli --hotkeys`。
- 是否 key 集中过期，导致 expire cycle 抖动。
- 是否 value 变大，例如缓存从摘要字段变成完整对象列表。

治理动作：
- 大 key 拆分：按用户、时间片、业务维度分片。
- 热 key 加本地缓存、副本 key、读写分离或限流。
- 集中过期加随机 TTL，避免同一时间大量过期。
- 删除大 key 用 `UNLINK`，不要直接 `DEL`。

**第四步：从请求端排查**

- 连接池是否耗尽，是否有获取连接等待时间。
- 是否有重试风暴，超时后立即重试导致 Redis 压力翻倍。
- 客户端超时时间是否过短，导致正常抖动被放大成失败。
- Go 服务是否发生 GC 停顿、goroutine 堆积、worker pool 阻塞。

治理动作：
- 连接池设置最大连接数、最大等待时间、空闲连接回收。
- 重试必须加退避和最大次数，Redis 超时时不要无脑重试。
- 对非核心缓存读允许降级，本地缓存兜底。
- 对热点接口加本地 limiter，避免异常流量继续打 Redis。

**第五步：从 Redis 自身和系统层排查**

- CPU 是否接近单核打满。
- 网络入/出带宽是否打满。
- 是否正在 RDB、AOF rewrite、fork。
- 是否 AOF `appendfsync always/everysec` 遇到磁盘抖动。
- 是否内存接近上限、发生频繁淘汰、swap。

```redis
INFO stats
INFO cpu
INFO memory
INFO persistence
INFO clients
```

治理动作：
- fork/AOF 抖动明显时，评估持久化策略、磁盘性能和实例内存规模。
- 网络打满时，减少大 value 返回、拆分热点、扩容分片。
- CPU 打满时，治理慢命令、热 key、Lua，必要时拆分实例。

**生产止血顺序**
1. 先限流或降级异常接口，阻止 Redis 压力继续扩大。
2. 如果是明确慢命令，快速回滚发布或关闭对应任务。
3. 如果是热 key，临时开启本地缓存或副本 key。
4. 如果是大 key 删除/读取，暂停任务，改为分批或异步删除。
5. 如果 Redis 实例资源打满，临时扩容读副本、拆分流量或迁移热点。

**面试追问：慢查询日志记录网络耗时吗？**
不记录。slowlog 主要记录 Redis 执行命令的耗时，不包含客户端排队、网络传输和响应读取时间。

### 42. 线上 Redis CPU 飙高怎么办？

**结论先行：**
Redis CPU 飙高要先判断是“流量变多”还是“单次命令变重”。生产排查要把 CPU 飙高和 QPS、慢查询、命令分布、热 key、大 key、过期淘汰、Lua、持久化事件放在同一时间线里看。

**分层展开：**

**常见现象**
- Redis 实例 CPU 使用率持续高位，单核接近 100%。
- 请求延迟升高，慢查询增加。
- 某个 Cluster 节点明显比其他节点忙。
- 带宽、QPS、连接数可能同步升高，也可能 QPS 没升但 CPU 升高。

**一般怎么发现**
- Redis 监控 CPU 报警。
- APM 看到 Redis 调用延迟升高。
- `INFO commandstats` 某类命令调用次数或耗时异常。
- `SLOWLOG` 出现高复杂度命令或 Lua 脚本。
- 业务发布、定时任务、活动流量开始后 CPU 升高。

**第一步：确认 CPU 飙高形态**

```redis
INFO cpu
INFO stats
INFO commandstats
SLOWLOG GET 20
```

判断重点：
- QPS 是否升高：如果 QPS 升高，大概率是流量、热 key 或连接风暴。
- QPS 没升但 CPU 升高：大概率是命令变重、大 key、Lua、过期淘汰、持久化事件。
- 是否只有单个节点高：大概率是热 key、slot 倾斜或某类业务 key 集中。

**第二步：从请求端看流量来源**

- 哪个服务、接口、定时任务调用 Redis 激增。
- 是否刚发布新版本，新增了缓存回填、批量扫描或大范围读取。
- 是否有重试风暴，超时后多实例同时重试。
- 是否有后台任务和在线流量抢同一个 Redis。

处置：
- 对异常接口限流、降级或回滚。
- 暂停批处理、预热、扫描类任务。
- 重试增加退避，必要时关闭 Redis 失败重试。

**第三步：从命令层定位高 CPU 命令**

- `INFO commandstats` 看 `cmdstat_xxx` 的 calls、usec、usec_per_call。
- `SLOWLOG GET` 看是否有 `HGETALL`、`SMEMBERS`、`ZRANGE` 大范围、`SUNION`、Lua。
- 查看是否有大量 `SCAN`，虽然 `SCAN` 非阻塞，但高频扫描也会消耗 CPU。

处置：
- 高复杂度命令改分页、拆 key 或异步任务。
- Lua 设置输入规模上限，禁止脚本里做大循环。
- 批量操作限制 batch size。

**第四步：从数据层看热 key、大 key、过期淘汰**

- 热 key 会让单个节点 CPU 和带宽打满。
- 大 key 会让单条命令消耗大量 CPU 和网络。
- 大量 key 同时过期会让过期删除周期占用 CPU。
- maxmemory 触发频繁淘汰时，也会增加 CPU 消耗。

```redis
INFO keyspace
INFO memory
INFO stats
```

重点看：
- `expired_keys` 是否突增。
- `evicted_keys` 是否突增。
- `used_memory` 是否接近 `maxmemory`。
- 某些 key 前缀是否访问异常。

处置：
- 热 key 本地缓存、副本 key、读副本、限流。
- 大 key 拆分，删除用 `UNLINK`。
- TTL 增加随机抖动。
- 容量不足时扩容或调整淘汰策略。

**第五步：从 Redis 自身事件看 fork、AOF、后台线程**

- RDB/AOF rewrite fork 期间可能带来 CPU 和内存压力。
- AOF rewrite、lazy free、active defrag 也会消耗 CPU。
- Redis 6+ IO 线程、后台线程使进程总 CPU 可能超过 100%。

```redis
INFO persistence
INFO memory
LATENCY LATEST
```

处置：
- 业务高峰避开 rewrite、bgsave。
- 大实例拆小，降低 fork 成本。
- active defrag 根据碎片率和 CPU 情况谨慎开启或调参。

**生产止血顺序**
1. 限流/熔断异常调用方，暂停批量任务。
2. 快速查看 `commandstats + slowlog`，定位是否有明显问题命令。
3. 如果是热 key，启用本地缓存或读副本分流。
4. 如果是大 key/大范围命令，回滚或改分页。
5. 如果单节点长期高，做数据拆分、slot 迁移或扩容分片。

**面试追问：Redis 是单线程，CPU 为什么可能超过 100%？**
Redis 还有后台线程和子进程，例如 AOF、lazy free、IO 线程、RDB/AOF rewrite 子进程。多核环境下进程整体 CPU 可能超过单核 100%。

### 43. Redis 连接数打满怎么办？

**结论先行：**
Redis 连接数打满通常不是单纯把 `maxclients` 调大就结束，而是要排查调用方连接池、连接泄漏、慢请求堆积、短连接风暴和异常重试。生产上先保护 Redis 和核心链路，再定位是哪批客户端占满连接。

**分层展开：**

**现象**
- 新请求连接 Redis 失败。
- 客户端超时增多。
- Redis `connected_clients` 接近 `maxclients`。
- 业务日志出现 `ERR max number of clients reached`、`connection refused`、`connection pool timeout`。
- Redis 端 `blocked_clients`、客户端连接池等待时间可能升高。

**一般怎么发现**
- Redis `INFO clients` 看到 `connected_clients` 接近上限。
- 业务侧连接池等待、获取连接超时报警。
- 实例连接数曲线突然上升，常和发布、扩容、重试风暴同时发生。
- `CLIENT LIST` 看到某些来源 IP 连接数异常。

**第一步：确认连接数和客户端来源**

```redis
INFO clients
CONFIG GET maxclients
CLIENT LIST
```

重点看：
- `connected_clients` 是否接近 `maxclients`。
- `blocked_clients` 是否异常。
- `CLIENT LIST` 中哪个 `addr`、`name`、`cmd` 占连接多。
- 是否大量连接处于 idle 很久，说明可能连接泄漏或池未回收。

**第二步：从请求端排查连接池**

- 是否每个请求都新建 Redis client，没有复用连接池。
- 服务实例数是否扩容后，每个实例连接池太大，连接总数超过 Redis 承载。
- 连接池是否没有设置最大连接数、最大空闲数、最大等待时间。
- pipeline、pubsub、阻塞命令是否长期占用专用连接。
- Go 代码是否忘记关闭 `Rows` 类似资源；Redis 场景一般是 pipeline/pubsub/lock watch 等对象未正确释放。

估算方式：

```text
Redis 总连接数 ≈ 服务实例数 × 单实例连接池上限 + 其他后台任务连接
```

如果服务 100 个实例，每个实例 Redis pool 100，理论最大就是 10000 条连接，可能直接打满 Redis。

**第三步：从 Redis 命令和数据层排查连接堆积**

- 慢命令导致连接被长时间占用。
- 大 key 返回慢，客户端读响应慢，连接释放慢。
- `BLPOP/BRPOP/XREAD BLOCK` 等阻塞命令占住连接。
- Lua 脚本或大范围查询阻塞主线程，其他连接排队。

处置：
- 治理慢命令、大 key、热 key。
- 阻塞命令使用独立连接池，避免占满普通请求连接。
- 读写流量拆分，后台任务和在线请求隔离 Redis 实例或连接池。

**第四步：从系统层确认上限是否真的够**

- Redis `maxclients`。
- 操作系统 `ulimit -n`。
- Redis 进程可用 fd。
- 每条连接的内存开销。

临时提高 `maxclients` 前必须确认 fd 和内存足够，否则会从“连接数问题”变成“系统资源问题”。

**生产止血顺序**
1. 用 `CLIENT LIST` 找出异常来源 IP 或应用名。
2. 对异常服务限流、摘流量、重启异常实例或回滚发布。
3. 必要时临时提高 `maxclients`，同时确认 `ulimit -n` 和内存。
4. 暂停批处理、扫描、阻塞消费等非核心任务。
5. 事后调整连接池上限、等待超时、空闲回收和实例扩容策略。

**面试追问：连接池越大越好吗？**
不是。连接池太大会增加 Redis 上下文管理、客户端资源和突发并发压力。连接池大小要按服务实例数、QPS、命令耗时和 Redis 承载能力计算。

### 44. Redis 出现超时，可能是什么原因？

**结论先行：**
Redis 超时不等于 Redis 服务端一定慢。生产排查要把一次 Redis 调用拆成“获取连接 → 发送命令 → Redis 排队执行 → 网络返回 → 客户端反序列化”，逐段确认耗时在哪里。

**分层展开：**

**常见现象**
- 客户端出现 `i/o timeout`、`read timeout`、`context deadline exceeded`。
- 接口错误率升高，P99 抖动明显。
- Redis slowlog 可能有记录，也可能完全没有。
- 连接池等待时间、网络 RTT、CPU、带宽、Redis 延迟事件可能有一个或多个异常。

**一般怎么发现**
- 链路追踪看到 Redis span 超过超时阈值。
- 客户端 Redis SDK 指标显示 pool wait、dial、read、write 耗时。
- Redis 监控出现 latency、CPU、connected_clients、blocked_clients 异常。
- 网络监控看到丢包、重传、带宽打满。

**第一步：先分清是哪一种超时**

```text
获取连接超时：连接池满、连接泄漏、慢请求占连接。
建立连接超时：网络、防火墙、Redis 连接数满、跨机房抖动。
写超时：客户端到 Redis 网络异常或服务端收包慢。
读超时：Redis 执行慢、返回大包、网络慢、客户端读慢。
context 超时：上游链路预算不足或排队太久。
```

**第二步：从请求端排查**

- 连接池等待时间是否升高。
- 单实例并发是否突然上升。
- 是否有重试风暴，失败后立即重试。
- Go 服务是否 GC、goroutine 堆积、CPU 被打满。
- context 超时是否设置过短，比如业务总超时 100ms，却给 Redis 留了 5ms。

处置：
- 调整连接池上限和等待时间，但不能盲目放大。
- Redis 调用失败重试要加退避、限次和熔断。
- 非核心缓存读失败时降级，不要继续放大请求。
- 将 Redis 超时预算纳入整条链路预算。

**第三步：从 Redis 自身排查**

```redis
SLOWLOG GET 20
LATENCY LATEST
INFO commandstats
INFO clients
INFO memory
INFO persistence
INFO stats
```

重点看：
- 是否有慢命令、大 key、Lua。
- CPU 是否打满。
- `connected_clients`、`blocked_clients` 是否异常。
- 是否发生 fork、AOF fsync、rewrite。
- 是否内存不足、swap、淘汰频繁。

处置：
- 慢命令分页、拆 key、回滚问题发布。
- 大 key 读取/删除改分批或 `UNLINK`。
- 持久化抖动评估磁盘、fork 成本和大实例拆分。

**第四步：从网络层排查**

- 应用和 Redis 是否跨机房、跨可用区。
- 是否有丢包、重传、网络 RTT 抖动。
- Redis 出口或入口带宽是否打满。
- 是否返回大 value，导致网络传输耗时长。

处置：
- 服务和 Redis 尽量同机房/同可用区。
- 大 value 拆分或压缩，避免一次返回巨大对象。
- 热点读走本地缓存或读副本。

**第五步：从业务层排查**

- 是否缓存雪崩，大量 key 同时过期。
- 是否热点 key 集中。
- 是否新发布导致 key 命名变化，命中率下降。
- 是否后台任务扫描 Redis 或批量回填。

处置：
- TTL 增加随机值。
- 热点 key 本地缓存。
- 后台任务限速，和在线 Redis 隔离。
- 缓存失败时保护 DB，避免双重故障。

**生产止血顺序**
1. 先看客户端错误类型，区分 pool timeout、dial timeout、read timeout。
2. 同时看 `slowlog/latency/commandstats`，判断 Redis 是否真的慢。
3. 如果 Redis 不慢，重点查连接池、网络、客户端 GC 和大包传输。
4. 如果 Redis 慢，按慢查询/CPU/大 key/热 key 路径处理。
5. 加限流、熔断、降级，避免超时重试把故障放大。

**面试追问：如何证明是 Redis 服务端慢还是客户端慢？**
对比客户端耗时、Redis slowlog、Redis latency monitor、网络 RTT、服务端 CPU 和连接池等待时间。如果客户端耗时高但 slowlog 没记录，往往是网络、排队或连接池问题。

### 45. 如果 Redis 命中率突然下降，你怎么排查？

**结论先行：**
Redis 命中率突然下降时，核心不是先怪 Redis，而是先确认“哪些接口、哪些 key 前缀、从什么时候开始下降”。生产排查要把命中率、DB 回源、key 过期/淘汰、发布变更、流量结构和缓存回填链路串起来看。

**分层展开：**

**常见现象**
- 缓存命中率从 90%+ 降到 60%、40%。
- DB QPS、慢 SQL、连接池等待时间上升。
- 接口延迟升高，Redis QPS 可能升高，也可能 miss 后回源变多。
- `evicted_keys`、`expired_keys`、缓存写失败、key 数量变化可能异常。

**一般怎么发现**
- 缓存命中率监控报警。
- DB QPS/CPU/慢查询报警。
- APM 看到大量请求走 DB 回源链路。
- 业务指标异常，例如首页、广告计划列表、素材列表加载变慢。

**第一步：先定位影响范围**

```text
是全局命中率下降？
还是某个服务/接口下降？
是某类 key 前缀下降？
是某个 Redis 节点/slot 下降？
是读请求变了，还是缓存数据没了？
```

需要拉齐时间线：
- 命中率从什么时候开始降。
- 同一时间是否有发布、扩容、活动、批处理、缓存清理、Redis 故障切换。
- DB QPS 是否同步上升。

**第二步：从请求端看 key 规则和读写路径**

- 代码发布后 key 前缀、版本号、参数拼接是否变化。
- 读缓存和写缓存是否使用同一个 key。
- 是否新增了查询参数，导致 key 离散度变高。
- 是否灰度期间新老版本 key 不兼容。
- 是否缓存回填失败但没有报警。

典型事故：

```text
旧版本读写 key：campaign:{accountId}:list
新版本读 key：campaign:v2:{accountId}:list
写入仍然写旧 key

结果：读永远 miss，DB 回源升高。
```

处置：
- 回滚 key 规则变更，或同时兼容新老 key。
- 给缓存写入失败加报警。
- key 生成逻辑收敛到统一方法，避免读写两套拼接。

**第三步：从 Redis 数据层看过期、淘汰和容量**

```redis
INFO stats
INFO memory
INFO keyspace
CONFIG GET maxmemory
CONFIG GET maxmemory-policy
```

重点看：
- `expired_keys` 是否突增：可能 TTL 过短或集中过期。
- `evicted_keys` 是否突增：可能内存打满触发淘汰。
- `used_memory` 是否接近 `maxmemory`。
- keyspace 中 key 数量是否明显下降。
- 淘汰策略是否误伤热点，例如 `allkeys-lru` 下大量冷数据污染缓存。

处置：
- TTL 加随机抖动，避免同一时刻大量过期。
- 对核心热点 key 设置更合理 TTL 或预热策略。
- 清理低价值缓存，避免污染。
- 扩容 Redis 或拆分实例。

**第四步：从业务流量看是不是访问模式变了**

- 活动、新用户、新广告账户、新地区流量进入，冷数据访问增加。
- 爬虫、批处理、后台任务扫描大量冷数据。
- 搜索、分页深翻、报表类请求把大量低频数据写入缓存。
- 某些接口参数过于离散，导致缓存 key 数量暴涨但复用率低。

处置：
- 冷数据不回填缓存，或设置短 TTL。
- 后台任务禁用缓存回填，避免污染在线缓存。
- 搜索/分页类缓存做准入控制，只缓存高频页或核心条件。
- 对异常来源限流。

**第五步：从缓存回填链路看是否写不进去**

- DB 查询成功后是否真的写 Redis。
- Redis 写是否超时、被熔断、被降级。
- 序列化失败、value 过大、pipeline 部分失败是否被吞掉。
- 写缓存 TTL 是否设置错误，例如从 1 小时变成 1 秒。

处置：
- 回填失败必须打日志和指标。
- pipeline 批量写要检查每个结果。
- TTL 统一配置，避免魔法数字散落。
- 对核心 key 做主动预热和定时巡检。

**第六步：从 Redis 架构事件看**

- Cluster 是否发生故障切换，导致短期读到空节点或客户端路由异常。
- 主从延迟是否导致读副本读不到刚写入的数据。
- 是否误执行批量删除、flush、迁移脚本。
- 是否某些 slot 迁移期间客户端处理 MOVED/ASK 异常。

处置：
- 检查 Redis 变更记录和运维操作。
- 客户端 SDK 使用成熟 Cluster 模式。
- 读副本场景关注复制延迟，强一致读走主库或降级策略。

**生产止血顺序**
1. 先确认 DB 是否被打高，必要时立即限流和降级，保护数据库。
2. 定位下降范围：全局、接口、key 前缀、节点。
3. 查同时间发布和配置变更，尤其是 key 规则、TTL、缓存回填。
4. 查 Redis `expired_keys/evicted_keys/used_memory/keyspace`。
5. 对热点数据临时预热，对冷流量和批任务限流。
6. 修复根因后分批预热，避免恢复瞬间再次打爆 DB。

**面试追问：命中率低一定是坏事吗？**
不一定。新用户、新商品、冷启动、活动流量变化都可能导致短期命中率低。要结合 DB 压力、接口延迟和业务场景判断。

### 46. Redis 故障时怎么保护数据库？

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

### 47. 你负责一个高并发系统，Redis 架构怎么设计？

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

### 48. 如何做 Redis 容量评估？

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

### 49. Redis 如何做灰度和多环境隔离？

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

### 50. 枫叶互动项目里 Redis 用在哪些核心场景？

**可以这样答：**

在枫叶互动广告投放平台里，Redis 不是单纯做缓存，主要有三类核心场景：

1. **分布式锁：** 广告数据拉取、素材同步、自动投放、报表预警等 Cron 任务需要保证多实例下单实例执行，避免重复创建广告、重复写入和重复通知。
2. **媒体 API 限流：** Facebook/TikTok/Snapchat Marketing API 有账号级、应用级、接口级限流，所以使用 Redis ZSET + Lua 做全局滑动窗口限流。
3. **缓存版本指纹：** 广告账号下 Campaign/AdGroup/Ad 配置频繁变更，通过账号级版本号生成缓存指纹，实现多账号查询缓存精准失效。

**面试官追问：为什么这几个场景都适合 Redis？**

- 分布式锁需要跨实例共享状态，Redis 的 `SET NX EX` 和 Lua 能比较轻量地解决。
- 限流需要低延迟计数和过期窗口，ZSET 天然适合按时间窗口清理。
- 缓存版本需要高频读写、小数据量、低延迟，Redis 比 DB 更适合。

### 51. 如何讲清楚 Redis 分布式锁的项目细节？

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

### 52. 如何讲 Redis ZSET + Lua 滑动窗口限流？

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

### 53. 缓存版本指纹怎么讲？

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

### 54. 如何把知乎动态模板项目里的 Redis 讲出来？

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

### 55. 你在项目里如何体现 Redis 使用的工程深度？

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
