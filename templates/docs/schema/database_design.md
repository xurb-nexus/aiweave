# {ProjectName} — 数据库设计

> 规范来源：`aiweave/docs-spec/05_schema_design_spec.md`

## 1. 数据库概览

### 设计哲学
- {例：Redis-first / MQ 异步落库 / 分库设计}

### 表清单

| #  | 表名 | 说明 | 数据量预估 | 所属库 |
|----|------|------|-----------|--------|
| 1  | `{table_1}` | ... | 万级 | db_xxx_core |
| 2  | `{table_2}` | ... | 十万级 | db_xxx_core |
| ... | ... | ... | ... | ... |

## 2. 分库设计

### 2.1 分库策略
按业务域 + 数据量级 + 读写模式划分。

### 2.2 各库清单与特点

```
┌──────────────────────────────┐
│  MySQL 实例 1                 │
│  db_xxx_core（核心库）        │
│                              │
│  table_1   万级               │
│  table_2   十万级             │
│                              │
│  特点：读多写少 / Redis 缓存   │
└──────────────────────────────┘
```

### 2.3 容量与性能预估

### 2.4 各库读写模式 / 持久化级别登记

> 工程填空：每个物理库登记读写模式 + 持久化 / 复制级别（把写风暴库与强一致核心库的故障域分开）。
> 决策方法见 [design-spec/02_data_model_design.md §3.2](../../../design-spec/02_data_model_design.md)。

| 库名 | 量级 | 读写模式 | `innodb_flush_log_at_trx_commit` | 复制级别 |
|------|------|----------|----------------------------------|----------|
| `db_{example_core}` | {量级} | {读为主 / 读写均衡 / 写为主} | {1=强一致 / 2=允许丢 1s} | {半同步 / 异步} |
| `db_{example_log}` | {量级} | {读写模式} | {持久化级别} | {复制级别} |

### 2.5 读副本路由登记

> 工程填空记录（非方法论），与 §2.4 复制级别相邻互补（§2.4 登记"每库复制级别"，本节登记"读流量怎么路由到副本"）：登记复制拓扑 / 只读连接 / read-your-writes 必走主库的接口清单 / 副本延迟阈值（超阈值摘副本）。
> 决策方法见 [design-spec/02_data_model_design.md §3.7](../../../design-spec/02_data_model_design.md)（热点读先走缓存、海量非热点读才上副本、read-your-writes 回主、副本延迟取舍）。本节只登记选择结果，不复制决策树。

| 登记项 | 取值（工程填空） |
|--------|------------------|
| 复制拓扑 | `{主 → N 副本 / 级联}`（哪几个物理库有读副本） |
| 只读连接 | `{只读 DSN / 连接池}`（路由读流量的只读句柄） |
| read-your-writes 必走主库的接口清单 | `{接口/方法清单}`（刚写就读 → 该请求回主 / 会话粘主 / 等版本；其余默认路由副本） |
| 副本延迟阈值（超阈值摘副本） | `{lag-threshold}`（延迟超阈值 → 该副本摘出读池，退主库 / 他副本，不读过旧数据） |

## 3. 表结构 DDL

### 3.0 数据访问层范式 / 分表路由登记

> 工程填空记录（非方法论）：登记每表 Repository 四件套的实现情况、分表算法收敛函数、shard 分组并行约定。
> 决策方法见 [design-spec/02_data_model_design.md §3.1 / §3.2 / §3.3](../../../design-spec/02_data_model_design.md)。

#### 3.0.1 Repository 四件套登记

每表 Repository 标准形态 = 私有纯查询 `getFromDB`（无副作用真相源）+ 公有缓存编排 `GetBy{X}` + 预取注册 `Preload` + 成对批量 `GetBy{X}List`。登记每表已实现哪几件：

| 表名 | `getFromDB`（私有纯查询） | `GetBy{X}`（缓存编排） | `Preload`（预取注册） | `GetBy{X}List`（批量） |
|------|--------------------------|------------------------|----------------------|------------------------|
| `{table_1}` | {已实现 / —} | {已实现 / —} | {已实现 / —} | {已实现 / —} |
| `{table_2}` | {已实现 / —} | {已实现 / —} | {已实现 / —} | {已实现 / —} |

#### 3.0.2 分表路由收敛 + shard 分组并行登记

| 登记项 | 取值（工程填空） |
|--------|------------------|
| 分表算法唯一函数 | `{ModTable(base, shardKey, count, width)}`（分表逻辑只此一处；缓存 key 用逻辑表名，物理名只查询期算） |
| 已收敛到唯一函数的表 | {表清单 / —} |
| 批量查询 shard 分组约定 | 按 shard 分组、只查有请求的物理表、单 shard 直查 / 多 shard 并行 IN（不广播全表） |
| 已落地分组并行的批量方法 | {方法清单 / —} |

#### 3.0.3 批量写方法登记

> 工程填空记录（非方法论）：登记数据访问层的批量写方法，取代"select 判有无再分别 insert/update"与"循环单条写"。批量 Upsert 用 `ON DUPLICATE KEY UPDATE`（MySQL）/ `ON CONFLICT DO UPDATE`（PG）一次往返；大批量装载用 `LOAD DATA` / `COPY`。
> 决策方法见 [design-spec/02_data_model_design.md §3.1](../../../design-spec/02_data_model_design.md)（数据访问层范式末项）；写路径极致编排见 [design-spec/01_io_design.md §3.2](../../../design-spec/01_io_design.md)。

| 表名 | 批量 Upsert 方法 | 冲突子句 | 大批量装载 | 落地状态 |
|------|------------------|----------|------------|----------|
| `{table_1}` | `{...UpsertList}` | `{ON DUPLICATE KEY UPDATE / ON CONFLICT DO UPDATE / —}` | `{LOAD DATA / COPY / —}` | {已实现 / —} |
| `{table_2}` | `{...UpsertList}` | `{冲突子句 / —}` | `{装载方式 / —}` | {已实现 / —} |

### 3.1 {table_1}

**用途**：

**所属库**：db_xxx_core

**预估数据量**：万级

**写入特征**：

```sql
CREATE TABLE `{table_1}` (
    `id` bigint unsigned NOT NULL AUTO_INCREMENT,
    `field_1` varchar(32) NOT NULL DEFAULT '' COMMENT '字段说明',
    -- ...
    `deleted` tinyint NOT NULL DEFAULT '0' COMMENT '0=正常, 1=已删除',
    `created_at` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_field_1` (`field_1`),
    KEY `idx_field_x` (`field_x`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='表注释';
```

**字段说明**：
| 字段 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| ... | ... | ... | ... | ... |

**索引**：
| 索引名 | 字段 | 类型 | 用途 |
|--------|------|------|------|
| ... | ... | ... | ... |

**关联**：
- 上游：
- 下游：

**生命周期**：

**Redis 缓存**（如有）：
- Key 模式：
- 同步方式：

### 3.2 {table_2}

（同上结构）

## 4. Redis Key 设计

### 4.1 热数据缓存
| Key 模式 | 类型 | 说明 | 写入时机 |
|---------|------|------|---------|
| `{ns}:info:{entityId}` | Hash | ... | ... |

### 4.2 状态数据缓存

#### 4.2.1 `{state-1}:{entityId}:{actionId}` Hash 完整字段映射

| Field | 类型 | 单位 | 说明 | 写入时机 | 读取时机 | 与 MySQL 的关系 |
|-------|------|------|------|---------|---------|---------------|
| ... | ... | ... | ... | ... | ... | ... |

### 4.3 频率限制
### 4.4 Flush 脏标记
### 4.5 {受限模块}数据（如有）
### 4.6 完整 Hash 字段映射汇总

## 5. 与 MySQL 的关系

| 数据类别 | Redis 角色 | MySQL 角色 | 同步机制 |
|----------|-----------|-----------|---------|
| ... | ... | ... | ... |

## 6. GORM Model 层设计规范

### 6.1 包命名（按库分包）
| 数据库 | Go 包名 | 目录 |
|--------|---------|------|
| db_xxx_core | `db_xxx_core` | `models/db_xxx_core/` |

### 6.2 编码约定
- NOT NULL → 值类型；可空 → 指针类型
- 软删除：`Deleted int8`（不用 gorm.DeletedAt）
- 敏感字段：`json:"-"`
- date 类型：用 string

### 6.3 类型映射
（按 docs-spec/05 §8.3）

### 6.4 GORM tag 规则
（按 docs-spec/05 §8.4）

### 6.5 动态分表（如有）

### 6.6 对外 ID 模型登记

> 工程填空记录（非方法论）：登记真实主键不外泄、对外 ID 的加密形态，以及"定位 shard 的字段 vs 唯一标识的字段"区分（两者混用会算错 shard、查错物理表）。
> 决策方法见 [design-spec/02_data_model_design.md §6](../../../design-spec/02_data_model_design.md)；确定性加密对外 ID 细节见 [design-spec/05_caching_design.md §9.5](../../../design-spec/05_caching_design.md)。

| 登记项 | 取值（工程填空） |
|--------|------------------|
| 真实主键 | `{真实主键字段}`（仅内部用，**禁止**出现在对外响应） |
| 对外 ID 形态 | {确定性加密密文 / —}（可缓存可比较；解密失败统一降级"查无结果"） |
| 用于定位 shard 的字段 | `{shard-key 字段}`（参与 `ModTable` 取模，决定物理表） |
| 用于唯一标识的字段 | `{identity 字段}`（业务唯一键 / 主键，定位行） |
| 混用风险登记 | {若 shard-key 与 identity 不同源，说明各查询用哪个 / —} |

## 7. 索引设计

每张表的索引清单（详见 §3.X）。

### 7.1 高频查询索引支撑登记

> 工程填空记录（非方法论）：为每条高频查询登记支撑索引及其优化形态——覆盖索引（所需列全进索引，index-only scan **免回表**）/ 索引下推 ICP（WHERE 非前缀条件在索引层先过滤，少回表行数）/ 前缀索引（长列只索前 N 字符）/ 复合索引列序（最左前缀：等值列在前、范围列在后）。
> 决策方法见 [design-spec/02_data_model_design.md §3.5](../../../design-spec/02_data_model_design.md)；读路径"消灭往返"延伸见 [design-spec/01_io_design.md §3.1](../../../design-spec/01_io_design.md)。本节只登记选择结果，不复制决策树。

| 高频查询（语义） | 支撑索引 | 索引列序（最左前缀） | 覆盖索引 | ICP | 前缀索引 |
|------------------|----------|----------------------|----------|-----|----------|
| `{查询语义-1}` | `{idx_name}` | `{(等值列, 范围列)}` | {是 index-only / 否} | {是 / 否} | `{col}({N}) / —` |
| `{查询语义-2}` | `{idx_name}` | `{列序}` | {是 / 否} | {是 / 否} | `{col}({N}) / —` |

### 7.2 不过度索引登记（写放大约束）

> 每个二级索引 = 写放大 + 额外空间。按查询路径建索引，定期审无用索引。登记本表索引总数与一句话取舍理由。

| 表名 | 二级索引数 | 写放大评估 | 无用索引审计周期 | 备注 |
|------|-----------|-----------|------------------|------|
| `{table_1}` | {N} | {可接受 / 需精简} | {周期} | {取舍一句话 / —} |

## 8. 数据生命周期（如有过期/归档）


## 9. 参考示例（仅示意，落地按业务替换）

> ⚠️ 以下为示意：§1-§8 登记槽位已用占位符表达；本节具体 SQL 仅示意 §3.0.3 批量写 / §7.1 覆盖索引的写法，落地按业务表名 / 列名替换，不得照搬作为规范。完整决策示例见 [design-spec/02 §9.6](../../../design-spec/02_data_model_design.md)。

```sql
-- 批量 Upsert（一次往返，取代 select 判有无再 insert/update）
INSERT INTO t (k, v) VALUES (?, ?), (?, ?)
  ON DUPLICATE KEY UPDATE v = VALUES(v);          -- MySQL
-- PostgreSQL: ON CONFLICT (k) DO UPDATE SET v = EXCLUDED.v;

-- 大批量装载
LOAD DATA LOCAL INFILE 'x.csv' INTO TABLE t;      -- MySQL
-- PostgreSQL: COPY t (k, v) FROM STDIN;

-- 覆盖索引：所需列 (b,c) 全在索引 (a,b,c) → EXPLAIN 显示 Using index（不回表）
CREATE INDEX idx_cover ON t(a, b, c);
SELECT b, c FROM t WHERE a = ?;
-- 前缀索引（长列）：CREATE INDEX idx_prefix ON t(longcol(20));
```

---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
