# 06 - 缓存设计文档规范（KV 缓存 + 上下游）

> 规定 `docs/cache/cache_design.md` + `cache_helpers.md` 的内容结构。

---

## 1. 定位

缓存设计文档要让 AI 能回答：

1. **整体策略**：哪些数据走 Redis、为什么、与 MySQL 的关系
2. **每个 Key**：模式、类型、TTL、写入时机、读取时机
3. **每个 Hash 的每个 field**：含义、单位、与 MySQL 的等式映射
4. **三级缓存（如有）**：L1 本地 / L2 Redis / L3 MySQL 的命中链
5. **写入语义**：双写、增量 flush、MQ 异步落库的精确实现
6. **限流算法**：固定窗口 / 滑动窗口 / Token Bucket，以及 INCR + 条件 EXPIRE 的优化
7. **失效策略**：TTL 过期 vs 主动 DEL vs 缓存预热

精度要求：**Hash field 级**，每个 field 的语义不能模糊。

---

## 2. cache_design.md 顶层结构（强制）

cache_design.md 必须按以下章节顺序组织：

- `## 1.` 缓存架构总览（Redis-first 策略图 + 三级缓存层级 + 各类数据缓存路径表 + **读写非对称降级语义登记表**）
- `## 2.` Redis Key 设计（按用途分组：热数据 / 状态数据 / 频率限制 / 分布式锁 / Flush 脏标记 + Hash 字段映射；并含 **TTL 随机抖动登记 / 值编码登记 / Hash 分桶登记 / 回源保护三类分治登记 / 紧凑编码阈值登记 / 概率·位图结构登记**）
- `## 3.` 本地缓存层（如有）（实例清单 / TTL / bucket 数 / 失效策略 + **本地缓存镜像登记**：镜像表清单 / WAL 读写池容量 / 三层 fail-open + **本地缓存准入条件 + 排除清单登记**）
- `## 4.` Flush 机制（Dirty Set / SPOP 算法 / 一致性校验 / 并行 + **写路径削峰登记**：高频写 Redis-first + delta 增量 flush）
- `## 5.` MQ 异步落库（如有）（消息结构 / 攒批策略 / 失败处理）
- `## 6.` 缓存预热（触发时机 / 预热顺序 / 部分失败降级）
- `## 7.` 缓存完整性巡检（巡检任务 / 检测算法 / 修复策略）
- `## 8.` 与 MySQL 的同步矩阵
- `## 9.` 分片可扩展登记（如有多分片 / 容量需线性扩展）（**大 key 治理 / 热 key 治理（读热·写热分治）/ 多分片利用**：容量线性律三前提的工程填空）

> **高性能技术点的"表示记录槽位"（强制 / 登记位而非方法论）**：cache_design.md 除上述基础结构外，必须含以下登记位——工程在此填空"自己选了什么"，决策树 / 权衡 / 选型理由全部留在 [`design-spec/05_caching_design.md`](../design-spec/05_caching_design.md)，每个登记位末尾附一行反向引用，**不复制方法论**：
>
> | 登记槽位 | 落点章节 | 登记内容（占位符化） | 反向引用 |
> | --- | --- | --- | --- |
> | 读写非对称降级语义 | §1.4 | 每个缓存的定位（加速层 / 数据源）→ 降级方向（加速层 miss 静默穿透 / 数据源 miss 快速失败） | design-spec/05 §3.2 |
> | 写策略 | §4.x | 每个缓存采用哪种写策略（cache-aside 失效优先 / write-through 同步写穿 / write-back 写回异步 flush）+ 触发理由 | design-spec/05 §4 写策略轴 |
> | TTL 随机抖动 | §2.x | base TTL + 抖动比例 `{jitter-pct}` | design-spec/05 §3.3 |
> | 值编码 | §2.x | 是否压缩 + 小值不压阈值 `{min-compress-bytes}` + value 写入上限 `{max-value-KB}` + 1 字节自描述头 | design-spec/05 §3.3 |
> | Hash 分桶 | §2.x | 桶数 `{bucket-N}` + key/field/value 形态 | design-spec/05 §3.3 |
> | 回源保护三类分治 | §2.x | 逐 Key 登记武器（single-flight / TTL 抖动 / XFetch）+ XFetch `{delta}` / `{beta}` | design-spec/05 §3.4 |
> | 紧凑编码阈值 | §2.x | 各集合类型 `listpack/intset/ziplist` 阈值（元素数 `{max-entries}` / 值长 `{max-value-len}`） | design-spec/05 §3.5 |
> | 概率/位图结构 | §2.x | 哪些"计数/存在/集合"需求用 HyperLogLog / Bitmap / Roaring | design-spec/05 §3.5 |
> | 本地缓存镜像 | §3.x | 镜像表清单 + WAL 读池 `min(max(2×NumCPU,16),64)` / 写池 `MaxOpenConns=1` + 三层 fail-open + **原子发布不变量（`ready` 仅 COMMIT 后置位）** | design-spec/05 §3.1 / §9.4（并发细节指 design-spec/03 §9.4） |
> | 本地缓存准入 + 排除清单 | §3.x | L1 进程内 KV 六条准入（有界小 / 低变更 / 容忍陈旧 / 高频读 / 非强一致 / 低基数）+ 排除清单（高基数 / 高频变更 / 强一致 / 大对象） | design-spec/05 §3.1 |
> | 写路径削峰 | §4.x | 哪些高频写走 Redis-first + delta 增量 flush（周期 `{flush-interval}`） | design-spec/01 §3.2 + design-spec/04 |
> | 分片可扩展 | §9 | 大 key 治理（识别 `{big-value}` / 拆分 / `UNLINK` / 检测）+ 热 key 治理（读热 L1/读副本/`{hotkey}:{0..K}` 打散；写热 本地聚合 flush/分片计数）+ 多分片（Cluster/一致性哈希 / Hash Tag 纪律 / 按分片分组并行 / 禁 `mod N`） | design-spec/05 §3.6 |
> | 批量自动分批 | cache_helpers.md | MGET/MSET 超 `{batch-threshold}` 自动 Pipeline 分批 | design-spec/05 §3.3 |

> **完整章节骨架见** [`templates/docs/cache/cache_design.md`](../templates/docs/cache/cache_design.md)。下面各节给出每个章节的填充精度与必填要素。

---

## 3. §1 缓存架构总览

### 3.1 Redis-first 策略图

ASCII 图示意全部数据写入路径：

```
┌─────────────────────────────────────────────────────────────────┐
│                     缓存层（Redis）                              │
│                                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                      │
│  │ 热数据  │  │ 状态数据  │  │ 限流数据  │                      │
│  ├──────────┤  ├──────────┤  ├──────────┤                      │
│  │ {ns}:info │  │ {state-1}:*  │  │ {rl}:qps:* │                      │
│  │ ...      │  │ {state-2}:*│  │ rl:daily │                      │
│  └──────────┘  └──────────┘  └──────────┘                      │
└─────────────────────────────────────────────────────────────────┘
        │                    │                    │
        │ flush (5min)       │ MQ 异步             │ 自然过期
        ▼                    ▼                    ▼
   db_{example_core}      Kafka            （无持久化）
                        │
                        ▼
                  db_{example_call_log}
```

### 3.2 关键设计断言

明确写出 4-6 条核心设计原则：

- "核心业务链路 0 次 MySQL 查询"
- "状态数据先写 Redis，定时任务 5min flush"
- "业务事件流水走 Kafka 异步 INSERT，不走 Redis"
- "限流数据仅 Redis，不持久化"

---

## 4. §2.x Redis Key 设计

### 4.1 通用表格格式

每节用一张总览表 + 关键 Hash 的字段详情子节：

```markdown
### 2.X 热数据缓存（不过期，由管理操作双写维护）

| Key 模式 | 类型 | 说明 | 写入时机 |
|---------|------|------|---------|
| `{ns}:info:{entityId}` | Hash | Key 基本信息 | Admin 操作 + 全量缓存预热 |
| `{ns}:ips:{entityId}` | Set | IP 白名单集合 | Admin 操作 + 全量缓存预热 |
| `{ns2}:info:{actor-id}` | Hash | {核心实体信息} | Admin 操作 + 全量缓存预热 |
```

### 4.2 Hash 字段映射子节（最关键）

每个 Hash Key 都必须有专门子节：

```markdown
#### 2.X.Y `{state-1}:{entityId}:{actionId}` Hash 完整字段映射

此 Key 是该 EntityID 对该接口的**所有 {核心写入维度} 记录的聚合值**（MySQL 中可能有多条 `{example_table_*}` 记录，Redis 中只存一个聚合 Hash）。

| Field | 类型 | 单位 | 说明 | 写入时机 | 读取时机 | 与 MySQL 的关系 |
|-------|------|------|------|---------|---------|---------------|
| `{counter-total}` | int64 | {单位} | {总量语义} | {核心写入动作}时聚合写入；cache_warmup | {校验步骤} | = `SUM({example_table_*}.{amount-field}) WHERE {time-cond}` |
| `{counter-used}` | int64 | {单位} | {已用量语义} | {核心写入动作} HINCRBY +1 | {校验步骤} | ≈ `SUM({example_table_*}.{used-field})` + {delta-field} |
| ... |

> 真实业务示例（按业务核心字段替换 `{counter-*}` / `{amount-field}` / `{delta-field}` 等占位符）见末尾 §15 参考示例。

**`{delta-field}` 字段语义详解**：

\`\`\`
{核心写入动作}流程（{module}.{state-report-method}）：
  HINCRBY {state-1}:{entityId}:{actionId} {counter-used} 1     ← 实时值 +1
  HINCRBY {state-1}:{entityId}:{actionId} {delta-field} 1      ← 未提交增量 +1
  SADD {state-1}:dirty {entityId}:{actionId}                   ← 标记为脏

flush_state_to_db 流程：
  {delta-field} = HGET {state-1}:{entityId}:{actionId} {delta-field}   ← 读取增量
  HDEL {state-1}:{entityId}:{actionId} {delta-field}                   ← 重置增量
  MySQL: UPDATE {example_table_*} SET {used-field} = LEAST({used-field} + {delta-field}, {amount-field})
\`\`\`

**关键不变量**：`Redis.{counter-used} = MySQL.SUM({used-field}) + Redis.{delta-field}`。flush 后 `{delta-field}` 被清零。
```

### 4.3 必填字段（Hash 字段表）

每个 field 必填：

| 列 | 必填理由 |
|----|---------|
| Field | 字段名一字不差 |
| 类型 | int64 / string 等 |
| 单位 | "分" vs "元" / "次" vs "千次" / 时间格式 |
| 写入时机 | 哪个动作（业务校验/状态变更/管理）触发写入 |
| 读取时机 | 哪个步骤读取 |
| 与 MySQL 的关系 | SQL 语义的等式（SUM / MIN 等） |

### 4.4 单位规则（容易出错的点）

```markdown
**单位转换规则**：Redis 与 MySQL 单位可能不同，必须显式声明换算关系（如 `Redis.{counter} / 100 ≈ MySQL.{amount-field}`，flush 后精确对齐）。
```

任何单位转换都必须**显式写出**。AI 在生成 Service 代码时如果漏掉 *100 / /100，会产生重大 bug。

---

## 5. §2.3 频率限制（专门子节）

### 5.1 多层限流总览

```markdown
| 层级 | 限流维度 | 阈值来源 | Redis Key | 触发时返回 |
|------|---------|---------|-----------|----------|
| L1 | 实体级 QPS | `{ns}:info:{entityId}.qpsLimit` | `{rl}:qps:{entityId}:{actionId}` | {N} |
| L2 | 实体级日调用量 | `{ns}:info:{entityId}.dailyLimit` | `{rl}:daily:total:{entityId}:{date}` | {N} |
| L3 | 动作级日调用量 | `{ns4}:info:{actionId}.dailyLimit` | `{rl}:daily:{entityId}:{actionId}:{date}` | {N} |
```

### 5.2 INCR + 条件 EXPIRE 优化（伪码）

```markdown
**关键优化**：只在 INCR 返回 1（首次自增）时才调用 EXPIRE，避免每次请求都执行 EXPIRE。

\`\`\`
count = INCR {rl}:qps:{entityId}:{actionId}
if count == 1:
    EXPIRE {rl}:qps:{entityId}:{actionId} 1    ← 仅首次自增时设置过期
else if count % 100 == 0:
    EXPIRE {rl}:qps:{entityId}:{actionId} 1    ← 每 100 次兜底补设过期
if count > qpsLimit:
    return {N}
\`\`\`

**边界说明**：固定窗口在边界时刻最多允许 2×qpsLimit 的突发，对场景可接受。
```

### 5.3 算法选择必须显式说明

固定窗口 / 滑动窗口 / Token Bucket / Leaky Bucket，选哪个 + 为什么。

---

## 6. §2.5 分布式锁（专门子节）

```markdown
### 2.5 分布式锁

| Key 模式 | TTL | 用途 |
|---------|-----|------|
| `lock:{integrity_task}` | 5min | 缓存巡检任务互斥 |
| `lock:{flush_task}` | 30min | 日统计 flush 任务互斥 |

**实现**：

\`\`\`
SET lock:{name} 1 NX EX {ttl}
→ OK：获取成功
→ nil：已被其他进程持有，放弃本次
\`\`\`

**释放**：

\`\`\`
DEL lock:{name}
\`\`\`

任务完成后必须释放（用 defer）。崩溃时由 TTL 兜底。
```

---

## 7. §2.6 Flush 脏标记

```markdown
### 2.6 Flush 脏标记

| Key 模式 | 类型 | 添加时机 | 移除时机 |
|---------|------|---------|---------|
| `{state-1}:dirty` | Set | {核心写入动作} SADD `{entityId}:{actionId}` | flush 任务 SPOP |
| `{state-2}:dirty` | Set | {核心写入动作} SADD `{actor-id}` | flush 任务 SPOP |
| `{stats-key}:dirty` | Set | Kafka 消费端 SADD | flush_daily_stats SMEMBERS |

**SPOP 模式 vs SMEMBERS 模式的选择**：

- 需要多进程并行：用 SPOP（原子弹出）
- 需要全量遍历：用 SMEMBERS（不删除，遍历完后用 SADD 重写"剩余未处理"）
```

---

## 8. §3 本地缓存层（如有）

### 8.1 实例清单

```markdown
| 实例 | 活跃 TTL | 最大 TTL | 桶数 | 用途 |
|------|---------|---------|------|------|
| `{LocalCache1}` | 15min | 30min | 36 | 热数据 |
| `{LocalCache2}` | 15min | 30min | 36 | 关联实体名单 |
| `{LocalCache3}` | 45min | 10h | 12 | {受限模块}规则 |
```

### 8.2 进入本地缓存的判定标准

```markdown
**何时使用本地缓存**：满足以下全部条件时考虑：

1. 数据量 ≤ 万级（避免 OOM）
2. 变更频率低（管理操作触发，非业务运行时变更）
3. 高频读取（核心业务链路 / 限流计算）
4. 短期不一致可接受（TTL 内允许稍旧）
```

### 8.3 失效策略

| 触发 | 动作 |
|------|------|
| 数据变更（管理操作） | 主动 Delete 本地缓存对应 Key |
| TTL 到期 | 自动失效，下次访问回源 Redis |
| Redis 故障 | 本地缓存继续服务（兜底） |

---

## 9. §4 Flush 机制

### 9.1 Dirty Set + SPOP 算法

```markdown
**算法描述**：

1. 业务写操作时：
   - HINCRBY xxx delta n
   - SADD xxx:dirty {key}

2. flush 任务（每 N 分钟）：
   - 循环 SPOP xxx:dirty
   - 对每个 popped key：
     - HGET delta
     - HDEL delta
     - UPDATE MySQL（增量语义）
     - 失败时 SADD xxx:dirty 回滚（可选，看一致性级别）

**优势**：
- 多进程天然安全（SPOP 原子）
- 增量语义（不是覆盖）
- flush 失败时数据仍在 Redis 实时值中
```

### 9.2 一致性校验

flush 完成后做 **样本对账**：

```markdown
flush 后随机抽取 N 个 key：
  redis_{counter-used} = HGET key {counter-used}
  mysql_{counter-used} = SUM(SELECT {used-field} FROM {example_table_*} WHERE {biz-key-1}=? AND {biz-key-2}=?)
  if abs(redis_{counter-used} - mysql_{counter-used}) > 误差阈值：
      告警 + 触发完整性巡检
```

### 9.3 §4.x 写策略登记（表示记录槽位）

`§4.x` 是工程填空记录（非方法论）：每个缓存登记**采用哪种写策略**及触发理由。写策略决定"何时让缓存与数据源一致"，是 §9 Flush 机制（write-back 的落地）与 §2 写入时机（cache-aside / write-through 的落地）之上的统一选择记录。槽位**只记选了什么 + 触发**，何时该选哪种的决策树不复制，反向引用 [`design-spec/05_caching_design.md §4 写策略轴`](../design-spec/05_caching_design.md)。

```markdown
| 缓存（Key 命名空间） | 写策略 | 触发理由 |
|---------------------|--------|---------|
| `{ns}` | cache-aside 失效优先（改库后删缓存） | 默认（读多写少、容忍短窗陈旧） |
| `{ns-N}` | write-through 同步写穿（同步写缓存 + 库） | 写后即读 / 一致性敏感 |
| `{state-1}` | write-back 写回（写缓存 + delta 增量异步 flush，见 §4 / §9） | 计数热点削峰（高频写聚合，热路径 0 落库） |
```

> 决策方法（三种写策略如何按"写后即读 / 一致性敏感 / 计数热点"判定、强一致延迟双删 / 订阅 binlog 的升级）见 [`design-spec/05_caching_design.md §4 写策略轴`](../design-spec/05_caching_design.md)；write-back 削峰编排见 [`design-spec/01 §3.2`](../design-spec/01_io_design.md) + [`design-spec/04`](../design-spec/04_transaction_design.md)。

---

## 10. §5 MQ 异步落库

```markdown
### 5.1 消息结构（与 Kafka 协议一致）

\`\`\`go
type {Example}Msg struct {
    {IdempotentKey}  string  `json:"{idempotent-key}"`
    EntityID         string  `json:"entityId"`
    ActionID         string  `json:"actionId"`
    StatusCode       int     `json:"statusCode"`
    Billed       int8    `json:"billed"`     // 1=计入状态 0=不计入
    Cost         int64   `json:"cost"`        // 单位：分
    CalledAt     int64   `json:"calledAt"`    // Unix timestamp
    ...
}
\`\`\`

### 5.2 攒批策略

- 攒满 200 条 OR 等待 5 秒（先到先触发）
- 按 `calledAt` 月份分组 → 写到对应月分片
- 同时 HINCRBY {stats-key}:* 更新日统计

### 5.3 失败处理

- MySQL INSERT 失败：重试 3 次（间隔 1s/2s/4s）
- 3 次仍失败：逐条发送到死信队列 `{example-topic}_dlq`
- Redis 更新失败：WARN 日志，不影响 MySQL
```

---

## 11. §6 缓存预热

```markdown
### 6.1 触发时机

- 首次部署
- Redis 故障恢复
- 数据迁移后

### 6.2 预热顺序

1. 资源目录（{ns4}:info）
2. 路径映射（{ns5}:*）
3. {核心实体信息}（{ns2}:info）
4. 实体信息（{ns}:info、{ns}:ips、{ns6}）
5. 状态数据（{state-1} / {state-2} / {state-3}）
6. 权限位（{ns3}:*）

按"被依赖程度"排序：先预热被多模块依赖的数据。
```

---

## 12. cache_helpers.md（如有 L1 本地缓存层）

### 12.1 顶层结构

```markdown
# 缓存辅助函数设计

## 1. 三级缓存访问标准模板（L1 → L2 → 拒绝）
   伪码：先查本地，miss 时查 Redis，再 miss 时返回错误（不回源 MySQL）

## 2. gcache.BucketCache API 规范
   导出方法：Get / Set / Delete / Has

## 3. 5 个辅助函数完整签名与伪代码
   loadEntityInfo / loadUserInfo / loadResourceInfo / loadIPWhitelist / loadPartnerState

## 4. 缓存 Struct 定义
   entityInfo / userInfo / resourceInfo / partnerStateInfo

## 5. 类型转换工具（Redis Hash → Go struct）

## 6. 调用关系总览

## 7. 缓存失效策略

## 8. 批量自动分批登记
   登记 MGET/MSET 超 `{batch-threshold}` 自动 Pipeline 分批的阈值与未命中处理；
   决策方法反向引用 design-spec/05 §3.3
```

### 12.2 函数签名精度

每个 helper 函数必须给出：

```markdown
\`\`\`go
// LoadEntityInfo 从 L1 → L2 加载 Key 信息
//
// 流程：
// 1. 查 L1 {LocalCache1}（本地）
// 2. miss → HGETALL {ns}:info:{entityId}
// 3. 解析为 keyInfoCache struct
// 4. 写入 L1
// 5. 返回 *keyInfoCache, found bool
//
// 不回源 MySQL：缓存预热保证 Redis 完整性
func LoadEntityInfo(ctx *gin.Context, entityId string) (*EntityInfoCache, bool)
\`\`\`
```

---

## 13. 与 schema/database_design.md 的关系

| 内容 | cache_design.md | database_design.md |
|------|-----------------|--------------------|
| Redis Key 总览注册表 | 完整 | 简略（指向 cache_design） |
| Hash 字段映射 | 完整 | 不写 |
| 写入时机的伪码 | 完整 | 不写 |
| MySQL DDL | 不写 | 完整 |
| GORM Model 规范 | 不写 | 完整 |

**禁止重复**：同一信息只能在一处定义，另一处用 `[详见 cache_design.md §2.X](...)` 引用。

---

## 14. 维护

| 触发 | 动作 |
|------|------|
| 新增 Redis Key | §2.x 表格新增一行 + 关键 Hash 新增字段映射子节 + database_design.md §4 同步 |
| 修改 Hash field 语义 | §2.x.y Hash 字段映射 + 受影响的 Service / 定时任务文档 |
| 调整 TTL | §2.x 表格更新 + 受影响的限流/缓存预热说明 |
| 调整写策略（cache-aside ↔ write-through ↔ write-back） | §4.x 写策略登记更新一行 + 同步受影响的 §2 写入时机 / §9 Flush 机制 |
| 删除 Key | §2.x / §3（如有本地缓存）/ §4（如有脏标记）全部清理 |

---

## 15. 参考示例（仅示意，落地按业务替换）

> ⚠️ 以下为示意，规范本体（§1-§14）已用占位符表达；落地工程时按业务语义替换占位符，不得直接照搬本节具体业务名作为规范。
>
> 示例服从 [`PRINCIPLES.md §12 占位符规则`](../PRINCIPLES.md#12-占位符规则强制--双轨结构) 的"参考示例段豁免"。

### 15.1 §4.2 Hash 字段映射（示例：某计数 / 额度类业务）

`acc:quota:{userId}:{actionId}` Hash 字段：

| Field | 类型 | 单位 | 说明 | 写入时机 | 读取时机 | 与 MySQL 的关系 |
|-------|------|------|------|---------|---------|---------------|
| total | int64 | 次 | 总额度 | 状态上报时聚合写入；cache_warmup | 校验步骤 ⑪ | = `SUM(quota.total_amount) WHERE expire_at > NOW()` |
| used | int64 | 次 | 已使用量 | 状态上报 HINCRBY +1 | 校验步骤 ⑪ | ≈ `SUM(quota.used_amount)` + delta |
| delta | int64 | 次 | 未提交增量 | 状态上报 HINCRBY +1 | flush 任务 HGET → HDEL | flush 时增量写入 MySQL |
| expire_at | string | datetime | 最早到期时间 | 状态上报时取最早 | 校验步骤 ⑪ | = `MIN(quota.expire_at)` |

**关键不变量**：`Redis.used = MySQL.SUM(quota.used_amount) + Redis.delta`

### 15.2 §4.4 单位转换规则（示例）

如金额类场景：

```
Redis 存"分"（int64），MySQL 存"元"（decimal(10,2)）：
  Redis.balance_in_cents / 100 ≈ MySQL.balance
```

如计数类场景通常 Redis 和 MySQL 同单位（次/条），无需转换。

### 15.3 §9.2 一致性校验伪码（示例）

```
flush 后随机抽取 N 个 key：
  redis_used = HGET acc:quota:userId:actionId used
  mysql_used = SUM(SELECT used_amount FROM quota WHERE app_key=? AND api_code=?)
  if abs(redis_used - mysql_used) > 误差阈值：
      告警 + 触发完整性巡检
```

### 15.4 占位符 → 业务名 对照表（示例）

| 占位符 | 本节示例值 |
| --- | --- |
| `{counter-total}` / `{counter-used}` / `{delta-field}` | total / used / delta |
| `{amount-field}` / `{used-field}` | total_amount / used_amount |
| `{time-cond}` | `expire_at > NOW()` |
| `{example_table_*}` | quota |
| `{biz-key-1}` / `{biz-key-2}` | app_key / api_code |
| `{核心写入动作}` | 状态上报 |
| `{核心写入维度}` | 未过期额度记录 |
| `{校验步骤}` | 校验步骤 ⑪ |


---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
