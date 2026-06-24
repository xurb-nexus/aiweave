# 缓存设计

> 规范来源：`aiweave/docs-spec/06_cache_design_spec.md`

## 1. 缓存架构总览

### 1.1 Redis-first 策略图

```
{ASCII：缓存层 → flush / MQ / 自然过期 → 持久化层}
```

### 1.2 三级缓存层级（如有 L1 本地缓存）

```
L1（gcache，0 网络）→ L2（Redis）→ L3（MySQL）
```

### 1.3 各类数据的缓存路径表

| 数据类别 | 缓存策略 | 真相源 |
|---------|---------|--------|
| ... | ... | ... |

### 1.4 读写非对称降级语义登记

> 登记每个缓存的**定位**与对应**降级方向**（熔断打开 / 失败时）。写永远 best-effort（失败静默）；读的降级方向取决于该缓存是加速层还是核心数据源——按下表逐缓存填空，不要一刀切。

| 缓存（Key 命名空间） | 定位 | 读 miss / 故障时降级方向 | 写故障时 |
|---------------------|------|------------------------|---------|
| `{ns}` | 加速层（背后有 DB 兜底） | 静默穿透：返回"未命中" → 自然回源数据源（宕机与未命中行为同构） | 静默跳过，返回 nil |
| `{ns-N}` | 核心数据源（miss = 拒绝，无兜底） | 快速失败（如返 `{fail-code}`），**绝不穿透数据源**（穿透会返脏/返空） | 静默跳过，返回 nil |

> 决策方法（"加速层 vs 数据源"如何判定、为何不可一刀切）见 [design-spec/05_caching_design.md §3.2](../../../design-spec/05_caching_design.md)。

## 2. Redis Key 设计

### 2.1 热数据缓存（不过期，由管理操作双写维护）

| Key 模式 | 类型 | 说明 | 写入时机 |
|---------|------|------|---------|
| ... | ... | ... | ... |

### 2.2 状态数据缓存

#### 2.2.1 `{state-1}:{entityId}:{actionId}` Hash 完整字段映射

| Field | 类型 | 单位 | 说明 | 写入时机 | 读取时机 | 与 MySQL 的关系 |
|-------|------|------|------|---------|---------|---------------|
| total | int64 | 次 | ... | ... | ... | ... |
| used | int64 | 次 | ... | ... | ... | ... |
| delta | int64 | 次 | 未提交增量 | ... | flush 后 HDEL | flush 时增量写入 MySQL |

**delta 字段语义详解**：

```
{核心写入动作}流程（service.{Method-N}）：
  HINCRBY ... used 1
  HINCRBY ... delta 1
  SADD ...:dirty {entry}

flush 流程：
  delta = HGET ... delta
  HDEL ... delta
  MySQL: UPDATE ... SET used = LEAST(used + delta, total)
```

**关键不变量**：`Redis.used = MySQL.SUM(used) + Redis.delta`。

#### 2.2.2 `{state-2}:{actor-id}` Hash 完整字段映射
（同 §2.2.1）

#### 2.2.3 `{stats-key}:{...}` Hash 完整字段映射
（同上）

### 2.3 频率限制

#### 2.3.1 多层限流总览

| 层级 | 限流维度 | 阈值来源 | Redis Key | 触发时返回 |
|------|---------|---------|-----------|----------|
| L1 | 实体级 QPS | ... | `{rl}:qps:{...}` | ... |
| L2 | 实体级日调用量 | ... | `{rl}:daily:total:{...}` | ... |
| L3 | 动作级日调用量 | ... | `{rl}:daily:{...}` | ... |

#### 2.3.2 INCR + 条件 EXPIRE 优化

```
count = INCR {rl}:qps:{...}
if count == 1: EXPIRE {rl}:qps:{...} 1
else if count % 100 == 0: EXPIRE {rl}:qps:{...} 1
if count > limit: return 122
```

### 2.4 {受限模块}数据（如有）

### 2.5 分布式锁

| Key 模式 | TTL | 用途 |
|---------|-----|------|
| `lock:{name}` | ... | ... |

实现：`SET lock:{name} 1 NX EX {ttl}`，defer DEL，崩溃 TTL 兜底。

### 2.6 Flush 脏标记

| Key 模式 | 类型 | 添加时机 | 移除时机 |
|---------|------|---------|---------|
| ... | ... | ... | ... |

### 2.7 TTL 随机抖动登记

> 登记每个有 TTL 的 Key 的 base TTL 与抖动比例（实际 TTL = base + rand[0, base × `{jitter-pct}`)，打散集体过期防雪崩）。

| Key 模式 | base TTL | 抖动比例 `{jitter-pct}` |
|---------|---------|------------------------|
| `{ns}` | `{base-ttl}` | `{jitter-pct}` |

> 决策方法（为何随机抖动、抖动比例怎么定）见 [design-spec/05_caching_design.md §3.3](../../../design-spec/05_caching_design.md)。

### 2.8 值编码登记

> 登记每个存结构化大值的 Key 的编码策略。value 首字节为 1 字节自描述头（标识是否压缩），读侧据此解码。

| Key 模式 | 是否压缩 | 小值不压阈值 `{min-compress-bytes}` | value 写入上限 `{max-value-KB}` | 自描述头 |
|---------|---------|-----------------------------------|--------------------------------|---------|
| `{ns}` | 是 / 否 | `{min-compress-bytes}`（小于即不压，避免膨胀） | `{max-value-KB}`（超限静默跳过写入） | 1 字节 flag（`0x00` 原文 / `0x01` 压缩） |

> 决策方法（双阈值含义、自描述头为何 1 字节）见 [design-spec/05_caching_design.md §3.3](../../../design-spec/05_caching_design.md)。

### 2.9 Hash 分桶登记（海量小映射）

> 海量小映射用 Hash 分桶替代海量独立 String key（省 robj/dictEntry/TTL 元数据）。登记桶数与 key/field/value 形态。

| 映射 | 桶数 `{bucket-N}` | bucket 计算 | key 形态 | field 形态 | value 形态 |
|------|------------------|------------|---------|-----------|-----------|
| `{mapping}` | `{bucket-N}` | `crc32(field) % {bucket-N}` | `{ns}:{bucket}` | `{field-key}` | `{packed-value}` |

> 决策方法（桶数怎么定以保 listpack 编码、为何 CRC32 取模）见 [design-spec/05_caching_design.md §3.3](../../../design-spec/05_caching_design.md)。

### 2.10 回源保护三类分治登记

> 三类"回源风暴"是不同失效形态，各有专属武器，叠加使用不可互相替代。逐 Key 登记**用了哪类武器**：并发 miss 同一 key → single-flight；同批 key 集体过期 → TTL 随机抖动（参数见 §2.7）；单热 key 到期悬崖 → XFetch 概率提前重算。XFetch 需与值一起缓存 `{delta}`（上次回源耗时）并登记 `{beta}`（提前早晚系数，默认 1，越大越早刷）。

| Key 模式 | 回源风暴形态 | 武器 | single-flight？ | TTL 抖动？ | XFetch `{delta}` / `{beta}` |
|---------|------------|------|----------------|-----------|----------------------------|
| `{ns}` | 并发 miss 同一 key | single-flight 合并回源 | 是 | — | — |
| `{ns-N}` | 同批 key 集体过期（雪崩） | TTL 随机抖动 | — | 是（比例见 §2.7） | — |
| `{hot-ns}` | 单热 key 到期悬崖（击穿） | XFetch 概率提前重算 | 是 | 是 | 缓存 `{delta}` + `{beta}` |

> 决策方法（三类失效形态如何区分、为何正交叠加、XFetch 触发条件）见 [design-spec/05_caching_design.md §3.4](../../../design-spec/05_caching_design.md)。

### 2.11 紧凑编码阈值登记

> Redis 集合类型在元素数 / 值长低于阈值时用紧凑编码（`listpack` / `intset` / `ziplist`），超阈值退化为 hashtable/skiplist，内存翻数倍。登记**主动把单 key 元素数 / 值长压在阈值内**的目标阈值（与 §2.9 Hash 分桶"控桶体积保 listpack"同源）。

| 集合 Key | 编码类型 | 元素数阈值 `{max-entries}` | 值长阈值 `{max-value-len}` | 压在阈值内的手段 |
|---------|---------|--------------------------|---------------------------|----------------|
| `{ns}` | `listpack` / `intset` / `ziplist` | `{max-entries}` | `{max-value-len}` | 分桶 / 分片 / 控成员数 |

> 决策方法（各编码退化阈值、为何控在阈值内）见 [design-spec/05_caching_design.md §3.5](../../../design-spec/05_caching_design.md)。

### 2.12 概率/位图结构登记

> 需求是"统计基数 / 判断存在 / 集合运算"而非取回原值时，别存原始集合，改用概率 / 位图结构。登记每个"计数 / 存在 / 集合"需求选了哪种结构。

| 需求 | 结构 | Key 模式 | 误差 / 取舍 |
|------|------|---------|------------|
| 海量去重计数（可容忍 ~1% 误差） | HyperLogLog | `{hll-ns}` | 固定约 `{HLL-bytes}`，不可取回成员 |
| 稠密整数 ID 的 membership / 标志位 | Bitmap | `{bm-ns}` | 1 bit/元素，适稠密 |
| 稀疏整数 ID 集 + 快速交并 | Roaring Bitmap | `{roaring-ns}` | 分块压缩 + 快速 AND/OR，适稀疏 |

> 决策方法（精确值 vs 概率结构如何判定、HLL/Bitmap/Roaring 适用边界）见 [design-spec/05_caching_design.md §3.5](../../../design-spec/05_caching_design.md)。

## 3. 本地缓存层（如有）

### 3.1 实例清单

| 实例 | 活跃 TTL | 最大 TTL | 桶数 | 用途 |
|------|---------|---------|------|------|
| ... | ... | ... | ... | ... |

### 3.2 进入本地缓存的判定标准
1. 数据量 ≤ 万级
2. 变更频率低
3. 高频读取
4. 短期不一致可接受

### 3.3 失效策略

### 3.4 本地缓存镜像登记（近静态小表，如有进程内 SQLite 镜像）

> 近静态小维度表（≤ 万级、可容忍陈旧、被高频读）可整表镜像进进程内嵌入式库，本地命中绕过 Redis 与一切外层缓存。登记镜像了哪些表 + WAL 读写池容量 + 三层 fail-open 跌落链。

镜像的表清单：

| 镜像表 | 数据量级 | 刷新周期 | 命中链 |
|--------|---------|---------|--------|
| `{mirror-table}` | `{row-count}` | `{refresh-interval}` | 本地（就绪）→ Redis → 数据源 |

WAL 单写多读连接池（嵌入式库进程内 CPU-bound 访问，按核数定容，非网络池量级）：

| 池 | MaxOpenConns | 策略 |
|----|-------------|------|
| 读池 | `min(max(2×NumCPU, 16), 64)` | 长存保温（连接复用 → 保住私有 page cache） |
| 写池 | `1`（`MaxOpenConns=1`） | 把写并发在 Go 池层钳成串行，根除 `database is locked` |

三层 fail-open（每层故障安全跌落下一层）：本地（就绪）> Redis > 数据源。

**原子发布不变量（强制）**：`ready` 标志**只能在整表替换事务 COMMIT 成功后**置位；读者只读 `ready`，WAL 读事务开始即钉住最末已提交帧快照 → 要么走远端 / 读完整旧版本、要么读完整新版本，**绝无半截**。COMMIT 前置位 / 跨刷新持长读事务即撕裂可见性（`R-LOCALMIRROR-STALE-READY`）。

> 决策方法（何时该本地镜像、整表原子替换语义、组件骨架与原子发布不变量）见 [design-spec/05_caching_design.md §3.1 / §9.4](../../../design-spec/05_caching_design.md)；WAL 读写池定容与串行写的并发细节见 [design-spec/03_concurrency_design.md §9.4](../../../design-spec/03_concurrency_design.md)。

### 3.5 本地缓存准入条件 + 排除清单登记

> L1 进程内 KV **每实例各存一份（内存 × N 实例）、只靠 TTL 自然失效、无跨实例失效广播**——这两条根因推出准入门槛，须**同时满足**。逐 L1 实例登记是否满足六条准入，并显式登记排除清单（命中即不放 L1，退 L2 Redis）。

准入条件（六条须同时满足）：

| 准入条件 | 本实例是否满足 | 根因 |
|---------|--------------|------|
| 数据量有界且小（万级） | 是 / 否 | L1 每实例各一份 = 内存 × N 实例 |
| 低变更频率 | 是 / 否 | 无跨实例失效广播 → 变更越频实例间越不一致 |
| 可容忍陈旧（最多一个 TTL） | 是 / 否 | 每实例靠 TTL 自然过期 |
| 高频读 | 是 / 否 | 用大量读摊薄"× N 实例内存"成本 |
| 非强一致 / 不要求 read-your-writes | 是 / 否 | L1 无法保证全实例瞬时一致 |
| 低基数判定类 key | 是 / 否 | 非每实体 / 每会话 / 组合键 |

排除清单（命中即不放 L1，走 L2 Redis）：

| 排除类型 | 原因 |
|---------|------|
| 高基数 key（每实体 / 每会话 / 组合键） | 内存 × N 爆 |
| 高频变更（计数器 / 限流 / 累加类状态） | 陈旧 + 实例发散 |
| 强一致 / read-your-writes 需求 | L1 无法保证瞬时一致 |
| 大对象 | 占内存 × N |

> 决策方法（准入门槛根因、为何排除这几类、本地 SQLite 镜像更窄的准入）见 [design-spec/05_caching_design.md §3.1](../../../design-spec/05_caching_design.md)。

## 4. Flush 机制

### 4.1 Dirty Set 设计
### 4.2 SPOP 增量 flush 算法
### 4.3 一致性校验
### 4.4 单进程 vs 多进程并行

### 4.5 写路径削峰登记

> 登记哪些高频写走 Redis-first + delta 增量 flush（热路径只写内存态原子命令 + 脏标记，由定时任务按周期 `{flush-interval}` 把 delta 增量落库，热路径 0 落库）。

| 高频写动作 | 热路径写法 | flush 周期 `{flush-interval}` | delta 不变量 |
|-----------|-----------|------------------------------|-------------|
| `{write-action}` | HINCRBY `{counter}` +1 + HINCRBY `{delta-field}` +1 + SADD `{dirty-set}` | `{flush-interval}` | `Redis.{counter} = 数据源.SUM(...) + Redis.{delta-field}` |

> 写路径削峰编排（何时该 Redis-first、delta 摊销机制）见 [design-spec/01_io_design.md §3.2](../../../design-spec/01_io_design.md)；最终一致 + 边界兜底的事务取舍见 [design-spec/04_transaction_design.md](../../../design-spec/04_transaction_design.md)。

### 4.6 写策略登记

> 登记每个缓存采用哪种写策略（cache-aside 失效优先 / write-through 同步写穿 / write-back 写回异步 flush）及触发理由。**只登记选了什么 + 触发；何时该选哪种的决策方法见下方引用。**

| 缓存（Key 命名空间） | 写策略 | 触发理由 |
|---------------------|--------|---------|
| `{ns}` | cache-aside 失效优先（改库后删缓存） | 默认（读多写少、容忍短窗陈旧） |
| `{ns-N}` | write-through 同步写穿（同步写缓存 + 库） | 写后即读 / 一致性敏感 |
| `{state-1}` | write-back 写回（写缓存 + delta 增量异步 flush，见 §4.1-§4.5） | 计数热点削峰（高频写聚合，热路径 0 落库） |

> 决策方法（三种写策略如何按"写后即读 / 一致性敏感 / 计数热点"判定，强一致延迟双删 / 订阅 binlog 升级）见 [design-spec/05_caching_design.md §4 写策略轴](../../../design-spec/05_caching_design.md)；write-back 削峰编排见 [design-spec/01_io_design.md §3.2](../../../design-spec/01_io_design.md) + [design-spec/04_transaction_design.md](../../../design-spec/04_transaction_design.md)。

## 5. MQ 异步落库（如有）

### 5.1 消息结构
### 5.2 攒批策略
### 5.3 失败处理

## 6. 缓存预热

### 6.1 触发时机
### 6.2 预热顺序
### 6.3 部分缓存失败的降级

## 7. 缓存完整性巡检

### 7.1 巡检任务的目标
### 7.2 检测算法
### 7.3 修复策略

## 8. 与 MySQL 的同步矩阵

| 数据类别 | Redis 主存储 | MySQL 主存储 | flush 方向 | 一致性级别 |
|---------|-------------|-------------|-----------|----------|
| ... | ... | ... | ... | ... |

## 9. 分片可扩展登记（如有多分片 / 容量需线性扩展）

> **容量线性律**：缓存容量随 Redis 分片数线性扩展，三前提缺一不可——① key 在分片间均匀分布；② 无大 key；③ 无热 key。大 key / 热 key / 倾斜正是击穿分片扩展性的元凶。本节登记**为达成线性扩展，对大 key / 热 key / 多分片各做了什么治理**。

### 9.1 大 key 治理登记

> 单 value 过大会占单分片内存无法跨分片拆、阻塞 Redis 单线程、`DEL` / slot 迁移卡顿。登记识别阈值与治理手段。

| Key 模式 | 识别阈值 `{big-value}` | 拆分方式 | 删除方式 | 检测方式 |
|---------|----------------------|---------|---------|---------|
| `{ns}` | String > `{big-value}` / 集合元素数 > `{big-entries}` / 总字节 > `{big-bytes}` | 分桶 / `{key}:{part}` 分块 | `UNLINK`（异步惰性删，不用 `DEL`）；大 key 过期分批清 | `--bigkeys` / `MEMORY USAGE` 采样巡检 + 告警 |

### 9.2 热 key 治理登记（按读热 / 写热分治）

> 单 key QPS 远超均值会让单分片 CPU 饱和、加分片也分不走。读热与写热用相反手段。登记每个热 key 的治理。

| 热 key 模式 | 读热 / 写热 | 治理手段 | 打散副本数 `{K}` |
|------------|------------|---------|-----------------|
| `{hotkey}` | 读热 | L1 本地缓存挡读流量（最有效，准入见 §3.5）/ 读副本 / `{hotkey}:{0..K}` 随机读打散到不同分片 | `{K}` |
| `{counter}` | 写热 | 本地聚合 + 批量 flush（见 §4.5）/ `{counter}:{0..K}` 分片计数、读时求和 | `{K}` |

检测方式：`--hotkeys`（需 `maxmemory-policy=LFU`）/ `OBJECT FREQ` / 客户端采样单 key QPS。

### 9.3 多分片利用登记

> 登记分片模型、Hash Tag 纪律、跨分片批量并行、扩缩容迁移策略。

| 项 | 本工程选型 | 纪律 |
|----|----------|------|
| 分片模型 | Redis Cluster（16384 slot，CRC16(key)%16384）/ 客户端一致性哈希 | key 设计让散列均匀、无倾斜 |
| Hash Tag `{tag}` | 是否使用 | 仅在必须跨 key 原子（MGET/事务/Lua）时用；tag 基数要大，避免热点 slot |
| 跨分片批量 | MGET/Pipeline 按分片分组并行 | 禁串行逐分片（聚合器"按实例分组批量读"） |
| 扩缩容迁移 | 一致性哈希（只迁 K/N）/ Cluster slot 迁移 | **禁裸 `mod N`**（全量重哈希） |

> 决策方法（容量线性律三前提、大 key / 热 key 危害与治理选型、Hash Tag 双刃、为何禁 `mod N`）见 [design-spec/05_caching_design.md §3.6](../../../design-spec/05_caching_design.md)；跨分片批量并行编排见 [design-spec/01_io_design.md §3.1](../../../design-spec/01_io_design.md)。


---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
