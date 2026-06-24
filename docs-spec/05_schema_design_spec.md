# 05 - 数据库设计文档规范（DDL + 上下游）

> 规定 `docs/schema/database_design.md` 的内容结构。这是 AI 重建数据层的唯一真相源，精度要求最高。

---

## 1. 定位

`database_design.md` 必须做到：

1. **AI 仅凭这一篇就能生成全部 GORM Model**（字段名 / 类型 / nullable / 索引 / 字符集都要写到）
2. **AI 仅凭这一篇就能 CREATE TABLE**（DDL 完整，不留省略）
3. **缓存设计、Service 实现、定时任务设计都从这里取字段语义**

精度上限：**字段级**。任何模糊（"几个状态字段"、"必要的索引"）都视为不合格。

---

## 2. 顶层结构（强制）

database_design.md 必须按以下章节顺序组织：

- `## 1.` 数据库概览（设计哲学 + 表清单总览）
- `## 2.` 分库设计（策略 + 各库清单与特点 + 容量预估 + `§2.4` 各库读写模式 / 持久化级别登记）
- `## 3.` 表结构 DDL（**强制：每张表的完整 CREATE TABLE**；`§3.0` 数据访问层范式 / 分表路由登记，含 `§3.0.3` 批量写方法登记）
- `## 4.` Redis Key 设计（**强制：所有 Key 模式的注册表**，含 Hash 字段映射）
- `## 5.` 与 MySQL 的关系（如有 Redis-first 设计）
- `## 6.` GORM Model 层设计规范（包命名 / 编码约定 / 软删除 / 敏感字段 / 动态分表 / 类型映射 + `§6.6` 对外 ID 模型登记）
- `## 7.` 索引设计（**强制**：每张表索引清单 + 选择理由；`§7.1` 高频查询索引支撑登记[覆盖索引 / ICP / 前缀索引 / 复合列序]、`§7.2` 不过度索引[写放大]登记）
- `## 8.` 数据生命周期（如有过期/归档）

> **完整章节骨架见** [`templates/docs/schema/database_design.md`](../templates/docs/schema/database_design.md)。下面各节给出每个章节的填充精度与必填要素。

---

## 3. §1 表清单（必填模板）

```markdown
### 表清单

| #  | 表名                  | 说明         | 数据量预估 | 所属库               |
| -- | -------------------- | ----------- | --------- | -------------------- |
| 1  | `{example_table_actor}`    | {核心实体}表        | 万级       | db_{example_core}       |
| 2  | `{example_table_entity}`     | EntityID 表    | 万级       | db_{example_core}       |
| ...
```

每行**必填**：表名 / 说明 / 数据量级 / 所属库。**不能含糊**——"百万级"和"亿级"对应的设计决策天差地别。

---

## 4. §2 分库设计

每个库一个 ASCII 框：

```
┌─────────────────────────────────────────────────────────────────┐
│                     MySQL 实例 1                                │
│                  db_xxx_core（核心库）                           │
│                                                                 │
│  table_1                10万级   ← 用途简述                       │
│  table_2                ...                                     │
│                                                                 │
│  特点：（数据量级 / 读写模式 / 缓存策略）                          │
│  总数据量：< 50 万行                                              │
│  写入频率：（低 / 中 / 高）                                        │
└─────────────────────────────────────────────────────────────────┘
```

**必填字段**：库名、表清单、特点描述（数据量、读写模式、缓存策略）、写入频率。

### 4.1 §2.4 各库读写模式 / 持久化级别登记（表示记录槽位）

`§2.4` 是工程填空记录（非方法论）：每个物理库登记读写模式 + 持久化 / 复制级别。若 §2.2 各库 ASCII 框已含"读写模式 / 持久化级别"则只补一行指针，否则用下表登记：

```markdown
| 库名 | 量级 | 读写模式 | `innodb_flush_log_at_trx_commit` | 复制级别 |
|------|------|----------|----------------------------------|----------|
| `db_{example_core}` | {量级} | {读为主 / 读写均衡 / 写为主} | {1 / 2} | {半同步 / 异步} |
```

> 决策方法（按读写模式分库 + 持久化分级）见 [`design-spec/02_data_model_design.md §3.2`](../design-spec/02_data_model_design.md)；本节只登记选择结果，不复制决策树。

### 4.2 §2.5 读副本路由登记（表示记录槽位）

`§2.5` 是工程填空记录（非方法论），与 §2.4 复制级别相邻互补（§2.4 登记"每库复制级别"，本节登记"读流量怎么路由到副本"）：登记复制拓扑 / 只读连接 / read-your-writes 必走主库的接口清单 / 副本延迟阈值（超阈值摘副本）：

```markdown
| 登记项 | 取值 |
|--------|------|
| 复制拓扑 | `{主 → N 副本 / 级联}`（哪几个物理库有读副本） |
| 只读连接 | `{只读 DSN / 连接池}`（路由读流量的只读句柄） |
| read-your-writes 必走主库的接口清单 | `{接口/方法清单}`（刚写就读 → 该请求回主 / 会话粘主 / 等版本；其余默认路由副本） |
| 副本延迟阈值（超阈值摘副本） | `{lag-threshold}`（延迟超阈值 → 该副本摘出读池，退主库 / 他副本，不读过旧数据） |
```

> 决策方法（热点读先走缓存、海量非热点读才上副本、read-your-writes 回主、副本延迟取舍）见 [`design-spec/02_data_model_design.md §3.7`](../design-spec/02_data_model_design.md)；本节只登记选择结果（复制拓扑 / 只读连接），不复制决策树。

---

## 5. §3 表结构 DDL（精度核心）

### 5.0 §3.0 数据访问层范式 / 分表路由登记（表示记录槽位）

`§3.0` 是 §3 表结构之前的工程填空记录（非方法论），两张登记表：

```markdown
#### 3.0.1 Repository 四件套登记
| 表名 | `getFromDB`（私有纯查询） | `GetBy{X}`（缓存编排） | `Preload`（预取注册） | `GetBy{X}List`（批量） |
|------|--------------------------|------------------------|----------------------|------------------------|
| `{table_1}` | {已实现 / —} | {已实现 / —} | {已实现 / —} | {已实现 / —} |

#### 3.0.2 分表路由收敛 + shard 分组并行登记
| 登记项 | 取值 |
|--------|------|
| 分表算法唯一函数 | `{ModTable(base, shardKey, count, width)}`（缓存 key 用逻辑表名，物理名只查询期算） |
| 批量查询 shard 分组约定 | 按 shard 分组、只查有请求的物理表、单 shard 直查 / 多 shard 并行 IN |

#### 3.0.3 批量写方法登记
| 表名 | 批量 Upsert 方法 | 冲突子句 | 大批量装载 | 落地状态 |
|------|------------------|----------|------------|----------|
| `{table_1}` | `{...UpsertList}` | `{ON DUPLICATE KEY UPDATE / ON CONFLICT DO UPDATE / —}` | `{LOAD DATA / COPY / —}` | {已实现 / —} |
```

- `3.0.1` 登记每表 Repository 四件套（私有纯查询 `getFromDB` + 公有缓存编排 `GetBy{X}` + 预取注册 `Preload` + 成对批量 `GetBy{X}List`）已实现哪几件。
- `3.0.2` 登记分表算法收敛到的唯一函数，及"批量查询按 shard 分组、只查有请求的物理表、多 shard 并行"的约定与落地方法。
- `3.0.3` 登记数据访问层的批量写方法：批量 Upsert（`ON DUPLICATE KEY UPDATE` / `ON CONFLICT DO UPDATE` 一次往返，取代 select 判有无再 insert/update）+ 大批量装载（`LOAD DATA` / `COPY`）。只登记方法名 / 冲突子句 / 落地状态。

> 决策方法见 [`design-spec/02_data_model_design.md §3.1 / §3.2 / §3.3`](../design-spec/02_data_model_design.md)（§3.1 数据访问层范式含批量写末项）；写路径极致编排见 [`design-spec/01_io_design.md §3.2`](../design-spec/01_io_design.md)。本节只登记实现状态与函数名，不复制范式 / 三段式方法论。

### 5.1 每张表的完整规格

```markdown
### 3.N {table_name}

**用途**：（一句话说明这张表存什么）

**所属库**：db_xxx_yyy

**预估数据量**：万级 / 十万级 / 百万级 / 亿级

**写入特征**：（如：管理操作低频写 / 状态聚合定时 flush 批量更新 / Kafka 异步 INSERT）

**完整 DDL**：

\`\`\`sql
CREATE TABLE `{example_table}` (
    `id` bigint unsigned NOT NULL AUTO_INCREMENT,
    `field_1` varchar(32) NOT NULL DEFAULT '' COMMENT '字段说明',
    `field_2` int NOT NULL DEFAULT '0' COMMENT '字段说明',
    `created_at` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    `deleted` tinyint NOT NULL DEFAULT '0' COMMENT '0=正常, 1=已删除',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_field_1` (`field_1`),
    KEY `idx_field_2` (`field_2`),
    KEY `idx_created` (`created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='表注释';
\`\`\`

**字段说明**：

| 字段 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| id | bigint unsigned | 是 | AUTO_INCREMENT | 主键 |
| field_1 | varchar(32) | 是 | '' | 字段含义、取值范围 |
| ... |

**索引**：（如索引复杂，单独列出）
- `uk_field_1` — 唯一约束，业务保证
- `idx_field_2` — 加速 X 场景查询

**关联**：
- 上游：哪些表 / Redis Key / 接口写入本表
- 下游：哪些查询读取本表

**生命周期**：
- 永久保留 / N 月归档 / 软删除

**Redis 缓存**（如有）：
- Key 模式：`xxx:{...}`
- 同步方式：双写 / 定时 flush / MQ 异步

```

### 5.2 必填精度

- **DDL 必须完整**：列定义 / 默认值 / 索引 / 字符集 / 引擎 / 注释
- **每个字段必须有 COMMENT**：MySQL 注释字段也要写到，AI 生成代码时会提取
- **索引必须显式**：每个索引的目的（加速哪类查询）
- **关联关系必须双向**：上游谁写、下游谁读

### 5.3 不允许的写法

```sql
-- ❌ 缺 COMMENT
`status` tinyint NOT NULL DEFAULT 0,

-- ❌ 不写 NOT NULL（默认 NULL）
`field` varchar(32),

-- ❌ 不写 ENGINE / CHARSET
CREATE TABLE `tbl` (...);

-- ❌ 字段类型省略长度
`name` varchar,
```

---

## 6. §4 Redis Key 设计（关键）

### 6.1 必填子节

```markdown
## 4. Redis Key 设计

### 4.1 热数据缓存
（强制：列出所有 Key 模式 + 类型 + 写入时机）

### 4.2 状态数据缓存
（同上）

### 4.3 频率限制
（同上）

### 4.4 Flush 脏标记
（同上）

### 4.5 {受限模块}数据（如有）
（同上）

### 4.6 Redis Hash 字段完整映射（最关键）
（每个 Hash Key 的每个 field 一张表）
```

### 6.2 §4.1-§4.5 标准表格

```markdown
| Key 模式 | 类型 | TTL | 说明 | 写入时机 |
|---------|------|-----|------|---------|
| `{ns}:info:{entityId}` | Hash | 永久 | Key 基本信息 | Admin 操作 + 缓存预热 |
| `{rl}:qps:{entityId}:{actionId}` | String | 1s | QPS 限流 | 核心业务链路 INCR |
```

**必填字段**：Key 模式（占位符明确）、类型（String/Hash/Set/ZSet/List）、TTL（永久/秒数/根据条件）、说明、写入时机。

### 6.3 §4.6 Hash 字段完整映射（最关键，AI 实现 Service 时必查）

```markdown
#### 4.6.1 `{state-1}:{entityId}:{actionId}` Hash 完整字段映射

| Field | 类型 | 单位 | 说明 | 写入时机 | 读取时机 | 与 MySQL 的关系 |
|-------|------|------|------|---------|---------|---------------|
| {counter-total} | int64 | {单位} | {总量语义} | {核心写入动作}时聚合写入 | {校验步骤} 判定 {剩余量} | = `SUM({example_table_*}.{amount-field}) WHERE {time-cond}` |
| {counter-used} | int64 | {单位} | {已用量语义} | {核心写入动作} HINCRBY +1 | {校验步骤} | ≈ `SUM({example_table_*}.{used-field})` + {delta-field} |
| {time-field} | string | datetime | {时间字段语义} | {核心写入动作}时取最早 | {校验步骤} | = `MIN({example_table_*}.{time-field})` |
| {delta-field} | int64 | {单位} | 未提交增量 | {核心写入动作} HINCRBY +1 | flush 任务 HGET → HDEL | flush 时增量写入 MySQL |

**{delta-field} 字段语义详解**：（伪码块）
**关键不变量**：（如 `Redis.{counter-used} = MySQL.SUM + Redis.{delta-field}`）

> 真实业务示例（如某计数类场景的 total/used/expire_at/delta 完整对照）见末尾 §13 参考示例。
```

**必填精度**：
- 字段名一字不差
- 类型 + 单位（"分" vs "元"必须显式）
- 与 MySQL 字段的等式关系（用 SQL 语义表达）
- 关键不变量（让 flush 一致性可机械验证）

---

## 7. §5 与 MySQL 的关系

明确"谁是真相源"：

| 数据类别 | Redis 角色 | MySQL 角色 | 同步机制 |
|----------|-----------|-----------|---------|
| 业务校验热数据（{ns}:info / {ns2}:info） | 主（业务运行时只读 Redis） | 持久化（管理操作双写） | 双写 + 缓存预热 |
| {核心数值状态数据} | 主（实时增减） | flush 目标 | 定时 flush 增量 |
| 业务事件流水（{example_log}） | 不缓存 | 主 | MQ 异步 INSERT |
| 限流数据（rl:*） | 唯一存储 | 不持久化 | 自然过期 |

---

## 8. §6 GORM Model 层设计规范

### 8.1 包命名

```markdown
| 数据库 | Go 包名 | 目录 |
|--------|---------|------|
| db_{example_core} | `db_{example_core}` | `models/db_{example_core}/` |
| db_{example_trade} | `db_{example_trade}` | `models/db_{example_trade}/` |
| ... |
```

### 8.2 编码约定（必填）

```markdown
- NOT NULL 字段 → 值类型（string, int8, int, int64, float64, time.Time）
- 可空字段（DEFAULT NULL / 无 NOT NULL）→ 指针类型（*string, *int, *time.Time）
- 软删除字段：`Deleted int8`（0=正常, 1=已删除），不使用 gorm.DeletedAt
- 敏感字段（凡含密钥 / 哈希 / token 类——具体字段名见末尾 §13 参考示例）使用 `json:"-"`
- 日期字段（仅 YYYY-MM-DD 不带时间）：使用 string 而非 time.Time，避免时区问题
```

### 8.3 类型映射规则（必填）

| MySQL 类型 | Go 类型（NOT NULL） | Go 类型（可空） |
|-----------|---------------------|----------------|
| `bigint unsigned NOT NULL AUTO_INCREMENT` | `uint64` | — |
| `varchar(N) NOT NULL` | `string` | `*string` |
| `tinyint NOT NULL` | `int8` | `*int8` |
| `int NOT NULL` | `int` | `*int` |
| `bigint NOT NULL` | `int64` | `*int64` |
| `decimal(M,N) NOT NULL` | `float64` | `*float64` |
| `datetime NOT NULL` | `time.Time` | `*time.Time` |
| `date NOT NULL` | `string`（YYYY-MM-DD） | `*string` |
| `text NOT NULL` | `string` | `*string` |

### 8.4 GORM tag 规则（必填）

```markdown
- 必须包含 `column:{snake_case列名}`
- 主键加 `primaryKey`
- varchar / decimal / text 类型加 `type:{完整类型}`（如 `type:varchar(32)`）
- int / bigint / tinyint / datetime **不加** `type:`
- NOT NULL 字段加 `not null`
- 有 DEFAULT 值的字段加 `default:{值}`
- 可空字段不加 `not null` 和 `default`
```

### 8.5 动态分表（如有）

```markdown
对按月分片的表（如 {example_table_log}），生成额外的 `ShardTableName(t time.Time)` 函数：

\`\`\`go
func ShardTableName(t time.Time) string {
    return fmt.Sprintf("{example_table_log}_%s", t.Format("200601"))
}
\`\`\`

物理表名按业务时间字段（如 `called_at`）的月份决定。
```

### 8.6 §6.6 对外 ID 模型登记（表示记录槽位）

`§6.6` 是工程填空记录（非方法论）：登记真实主键不外泄、对外 ID 加密形态，以及"用于定位 shard 的字段 vs 用于唯一标识的字段"的区分（混用会算错 shard、查错物理表）：

```markdown
| 登记项 | 取值 |
|--------|------|
| 真实主键 | `{真实主键字段}`（仅内部用，禁止出现在对外响应） |
| 对外 ID 形态 | {确定性加密密文 / —}（解密失败统一降级"查无结果"） |
| 用于定位 shard 的字段 | `{shard-key 字段}`（参与 `ModTable` 取模） |
| 用于唯一标识的字段 | `{identity 字段}`（业务唯一键 / 主键） |
```

> 决策方法见 [`design-spec/02_data_model_design.md §6`](../design-spec/02_data_model_design.md)；确定性加密对外 ID 的加密细节见 [`design-spec/05_caching_design.md §9.5`](../design-spec/05_caching_design.md)。本节只登记字段归属，不复制加密方法论。

---

## 9. §7 索引设计

每张表的索引必须显式列出 + 说明：

```markdown
### {table} 索引清单

| 索引名 | 字段 | 类型 | 用途 |
|--------|------|------|------|
| PRIMARY | id | 主键 | 主键查询 |
| `uk_{business-key}` | `{business-key}` | 唯一 | 业务唯一约束 |
| `idx_{actor-id-field}` | `{actor-id-field}` | 普通 | {核心实体}维度的相关资源列表查询 |
| `idx_{status-field}` | `{status-field}` | 普通 | 状态过滤 |
```

不允许的写法：
- 索引名是默认生成的（如自动后缀 `_2` 等无语义命名）
- 不写索引用途
- 复合索引不写字段顺序

### 9.1 §7.1 高频查询索引支撑登记（表示记录槽位）

`§7.1` 是工程填空记录（非方法论）：为每条高频查询登记支撑索引及其优化形态。SQL 关键字（覆盖索引 / ICP / 前缀索引）为技术术语可直接用；具体 `CREATE INDEX` 示例归末尾 §13 参考示例：

```markdown
| 高频查询（语义） | 支撑索引 | 索引列序（最左前缀） | 覆盖索引 | ICP | 前缀索引 |
|------------------|----------|----------------------|----------|-----|----------|
| `{查询语义-1}` | `{idx_name}` | `{(等值列, 范围列)}` | {是 index-only / 否} | {是 / 否} | `{col}({N}) / —` |
```

- **覆盖索引**：查询所需列全进索引 → index-only scan **免回表**（登记"是 index-only / 否"）。
- **ICP（索引下推）**：WHERE 非前缀条件在索引层先过滤，减少回表行数。
- **前缀索引**：长列只索前 N 字符 `{col}({N})`，以选择性换索引体积。
- **复合索引列序**：最左前缀——等值列在前、范围列在后。

### 9.2 §7.2 不过度索引登记（写放大约束）

`§7.2` 登记每表二级索引数与写放大取舍：每个二级索引 = 写放大 + 额外空间，按查询路径建、定期审无用索引。

```markdown
| 表名 | 二级索引数 | 写放大评估 | 无用索引审计周期 | 备注 |
|------|-----------|-----------|------------------|------|
| `{table_1}` | {N} | {可接受 / 需精简} | {周期} | {取舍一句话 / —} |
```

> 决策方法（覆盖索引 / ICP / 前缀索引 / 复合列序 / 不过度索引）见 [`design-spec/02_data_model_design.md §3.5`](../design-spec/02_data_model_design.md)；读路径"消灭往返"延伸见 [`design-spec/01_io_design.md §3.1`](../design-spec/01_io_design.md)。本节只登记选择结果，不复制决策树。

---

## 10. 与缓存设计的交叉引用

`database_design.md` §4 是 Redis Key **总览注册表**；具体的策略（三级缓存、Flush 时序、限流算法）放在 `cache/cache_design.md`。两者**不重复，互引用**。

---

## 11. §8 字段演进历史（新增 / 推荐）

> 解决痛点 #3 数据模型历史包袱。schema 演进过程中，被废弃的字段、变更过语义的字段、跨版本的兼容期 —— 这些信息一旦丢失，AI 在 B1 增量同步时容易把"看起来未使用"的字段当成可删字段，引入兼容性故障。

### 11.1 §8.1 废弃字段注册表

```markdown
| 表名 | 字段 | 废弃日期 | 原因 | 迁移到 | 清理计划 |
|------|------|---------|------|--------|---------|
| `{example_table-A}` | `{deprecated-field}` | {YYYY-MM-DD} | {废弃原因} | `{new-field}` 或外部存储 | `{清理日期}` 后物理删除 |
```

字段废弃 ≠ 物理删除。废弃后必须经过 **至少 1 个发版周期**（推荐 2-4 周）的"软兼容期"，旧字段保留但不再写入，确认无任何读取依赖后再 DROP。

### 11.2 §8.2 字段语义变更记录

```markdown
| 表名 | 字段 | 变更日期 | 旧语义 | 新语义 | 影响代码 |
|------|------|---------|--------|--------|---------|
| `{example_table-B}` | `{field-X}` | {YYYY-MM-DD} | `{旧含义}` | `{新含义}` | `service/{module}/{file}.go` 等 |
```

**典型场景**：

- 字段单位变更（s → ms，元 → 分）
- 状态值集合扩展（枚举新增值）
- 字段含义从"绝对值"改为"差值"

### 11.3 §8.3 Schema 兼容性规则（强制）

```markdown
- 新增字段必须有默认值（INSERT 旧代码不传该字段时仍可成功）
- 字段类型变更必须兼容旧数据（如 INT → BIGINT 可；BIGINT → INT 不可）
- 删除字段必须先标记废弃（§8.1 登记），经过至少一个发版周期后再物理删除
- 修改字段语义（§8.2 范畴）必须同步：
  - 所有读写代码同时升级
  - 数据迁移脚本（如 UPDATE 既有行）
  - service docs 中字段说明
  - 缓存设计 §4.6 Hash 字段映射（如关联）
```

### 11.4 与 BUILD_STATUS §11 约束清单状态轨道的关系

§8.1 / §8.2 每新增一行 → BUILD_STATUS §11 "Schema 演进约束"类目"已设计条目数" +1。详见 [`docs-spec/02 §11`](02_build_status_md_spec.md)。

### 11.5 B1 反向同步规则（强制）

| 代码迹象（git diff） | 反向同步动作 |
| --- | --- |
| GORM struct 字段 tag 加 `column:"-"` 或注释为 deprecated | §8.1 废弃字段注册表新增一行 |
| 字段类型变更（如 INT → BIGINT，VARCHAR(N) → VARCHAR(M)） | §3 DDL 同步 + §8.2 评估是否构成语义变更 |
| migration 脚本含 `UPDATE` 类型回填 | §8.2 字段语义变更记录登记 |
| 字段被删除（DROP COLUMN） | §8.1 检查是否已经过软兼容期；未过 → 拒绝合并 |

---

## 12. 维护

| 触发 | 动作 |
|------|------|
| 新增表 | §1 表清单 + §3 完整 DDL + §4 Redis Key（如有缓存）+ §7 索引 |
| 修改字段 | §3 DDL + §4.6 Hash 字段映射（如关联）+ 同步通知 service / cache 文档；如涉及语义变更 → §8.2 登记 |
| 删除表 | §1 / §3 / §4 / §7 全部清理 + 通知 service / api 文档 |
| 调整索引 | §7 索引清单 + §7.1 高频查询支撑登记（覆盖/ICP/前缀/列序）+ §7.2 写放大登记 + 性能影响说明 |
| 新增 / 调整批量写方法（`{...UpsertList}` / `LOAD DATA` / `COPY`） | §3.0.3 批量写方法登记 |
| 引入 / 调整读副本路由（只读 DSN / read-your-writes 回主 / 副本延迟阈值） | §2.5 读副本路由登记更新一行 |
| 废弃字段 | §8.1 登记 + 设置清理计划日期 + 软兼容期监控 |
| 物理删除字段（DROP COLUMN） | §8.1 检查软兼容期是否已过 + §3 DDL 同步移除 |

---

## 13. 参考示例（仅示意，落地按业务替换）

> ⚠️ 以下为示意，规范本体（§1-§12）已用占位符表达；落地工程时按业务语义替换占位符，不得直接照搬本节具体业务名作为规范。
>
> 示例服从 [`PRINCIPLES.md §12 占位符规则`](../PRINCIPLES.md#12-占位符规则强制--双轨结构) 的"参考示例段豁免"。

### 13.1 §6.3 §4.6 Hash 字段完整映射（示例：某计数类业务）

| Field | 类型 | 单位 | 说明 | 写入时机 | 读取时机 | 与 MySQL 的关系 |
|-------|------|------|------|---------|---------|---------------|
| total | int64 | 次 | 总额度 | 状态上报时聚合写入 | 校验步骤 ⑪ 判定剩余额度 | = `SUM(quota.total_amount) WHERE expire_at > NOW()` |
| used | int64 | 次 | 已使用量 | 状态上报 HINCRBY +1 | 校验步骤 ⑪ | ≈ `SUM(quota.used_amount)` + delta |
| expire_at | string | datetime | 最早到期时间 | 状态上报时取最早 | 校验步骤 ⑪ | = `MIN(quota.expire_at)` |
| delta | int64 | 次 | 未提交增量 | 状态上报 HINCRBY +1 | flush 任务 HGET → HDEL | flush 时增量写入 MySQL |

**关键不变量**：`Redis.used = MySQL.SUM(used_amount) + Redis.delta`

### 13.2 §8.2 编码约定 — 敏感字段示例

落地工程的典型敏感字段：

- `password_hash`（bcrypt 哈希后的密码）
- `secret_key`（API 调用密钥）
- `access_token` / `refresh_token`（OAuth token）
- `bank_card_no` / `id_card_no`（个人金融 / 证件号）

这些字段在 GORM struct 上**必须**带 `json:"-"`，防止意外通过 JSON 序列化返回给上游。

### 13.3 占位符 → 业务名 对照（示例）

| 占位符 | 本节示例值 |
| --- | --- |
| `{counter-total}` / `{counter-used}` | total / used |
| `{delta-field}` | delta |
| `{time-field}` | expire_at |
| `{amount-field}` / `{used-field}` | total_amount / used_amount |
| `{example_table_*}` | quota |
| `{核心写入动作}` | 状态上报 |
| `{校验步骤}` | 校验步骤 ⑪ |
| `{时间字段语义}` | 最早到期时间 |


---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
