# IO 契约（IO 极致与聚合并行）

> 规范来源：`aiweave/docs-spec/25_io_aggregation_spec.md`
>
> 本文是 AI 生成"涉及多次数据访问 / 跨实例读取 / 批量处理 / 多依赖编排"代码前的必读约束。
> 版本：{vX.Y} | 日期：{YYYY-MM-DD}

---

## 1. IO 极致目标（每请求 / 每批往返预算）

| 维度 | 目标值 | 测量方式 | 超标动作 |
|------|--------|---------|---------|
| 每请求 DB 往返 | ≤ {DB-RTT-budget}（聚合后） | 链路 span 计数 / mock 计数断言 | 审查漏聚合点 |
| 每请求跨实例缓存往返 | ≤ {Cache-RTT-budget}（按实例分组后） | 同上 | 合并为批量读 |
| 每批处理任务批量写次数 | 每批 ≤ {Batch-write-N} | 任务日志计数 | 攒批 |
| 独立 IO 并行化 | 无依赖的 IO 必须并行 | 代码审查 + /io-review | 改聚合 / 扇出 |

---

## 2. 三条 IO 铁律落地（豁免登记）

> 铁律零：批量优先（数据层缺批量方法先补 `{...List}`，违规表现即 N+1，共享 §2.1 登记）。铁律一：禁止循环内单条查询（N+1）。铁律二：禁止独立串行查询编排。

### 2.1 铁律一豁免登记（N+1）

| 位置 | 豁免理由 | 复核人 | 复核日期 |
|------|---------|-------|---------|
| {file}:{func} | 循环次数硬编码 ≤ {Loop-safe-N} | {actor} | {YYYY-MM-DD} |

### 2.2 铁律二豁免登记（串行编排 / 真实依赖链）

| 位置 | 数据依赖链说明 | 复核人 |
|------|--------------|-------|
| {file}:{func} | {后查询} 入参取自 {前查询}.{field}，真实依赖 | {actor} |

---

## 3. 聚合器模式（收集 → 批量读 → 并行回源 → 单飞 → 异步写回）

### 3.1 五阶段编排

```
① 收集：登记 N 个 cache-backed 读 + M 个 direct 任务，各返回延迟结果槽 Slot
② 批量读：按后端实例分组，每组一次批量读（一次 RTT）
③ 并行回源：未命中项 + direct 全部并行（worker pool / WaitGroup），收首错
④ 单飞合并：同 key 并发回源塌缩为一次底层查询，共享结果指针
⑤ 异步写回：回源结果 fire-and-forget 批量写回，脱离响应关键路径
```

### 3.2 延迟结果槽（Slot）契约

- nil-safe：对 nil Slot 取值返回零值。
- **共享只读铁律**：单飞使并发请求共享同一指针；下游必须只读，禁止就地改字段（如需改先深拷贝）。

### 3.3 本工程聚合器原语

| 原语 | 位置 | 用途 |
|------|------|------|
| {Aggregator} | helpers/{aggregator}/ | {多 cache-backed 读 + direct 编排} |

---

## 4. 并行编排原语清单

| 原语 | 适用场景 | 强制约束 |
|------|---------|---------|
| 聚合器 | 多缓存兜底读 + 少量直接任务 | 批量读按实例分组；回源并行；Slot 只读 |
| 扇出并行 | N 个无缓存、互不依赖的 IO | worker pool 上限；首错聚合；ctx 透传 |
| 流水线 | 多阶段、阶段间无 barrier 批处理 | 每项独立流过各阶段 |

- 并发上限：受 worker pool / 信号量约束，禁止裸 `go func()` 无界扇出（生命周期见 concurrency_safety.md §1.1）。
- 错误聚合：收集首错返回，其余 best-effort。
- ctx 透传：子任务透传 ctx；异步写回 detach。

---

## 5. 数据访问聚合约束

### 5.1 批量读优先
循环外收集 key → 一次 `IN(...)` / 批量缓存读 → `map[{key}]→{entity}` 装配。跨实例按实例分组。

### 5.2 批量写优先
N 次同类写 → 一次批量写 / pipeline。攒批阈值：{Batch-threshold}，flush 周期：{Flush-interval}。

### 5.3 JOIN vs 多查询
强依赖 / 同库 / 可索引 → 单条 JOIN（≤ {JOIN-limit}，见 performance_contract.md §4.2）；跨库 / 弱依赖 → 聚合器并行。

### 5.4 读写路径分治登记

> 读和写是相反的 IO 模式，本工程用相反的手段优化。本表登记本服务**读路径手段**（消灭往返）与**写路径手段**（削峰）各自选了什么。**只登记选了什么；读/写两棵决策树（怎么选）见下方引用。**

| 路径 | 形态 | 本服务选定手段（登记） |
|------|------|---------------------|
| 读 | {单实体读 / 集合 enrich / 多依赖编排} | {本地命中(0 网络)>单次往返 / 消重复读：一个 key 全链路只读一次派生值透传 / 分支裁剪跳过无关查询} |
| 写 | {高频状态写 / 纯追加} | {Redis-first 原子命令 + 脏标记 + 定时增量 flush（delta）/ 投 MQ 按 `{key}` 保序攒批 INSERT} |

> 决策方法见 [`design-spec/01 §3.1`](../../../design-spec/01_io_design.md)（读路径树）与 [`design-spec/01 §3.2`](../../../design-spec/01_io_design.md)（写路径树）。
>
> **DB 读补充**：缓存 miss 回源 / 直查 DB 的读，走**覆盖索引**（index-only scan 免回表，见 [`database_design.md §7.1`](../schema/database_design.md)）——"消灭往返"在 DB 层的延伸。

### 5.5 shard 分组并行编排登记

> 登记本工程批量查询遇到分表时的编排约定：**按 shard 分组 + 只查有请求落点的物理表 + 多 shard 并行**（即铁律零"补 `{...List}` 方法"的分表落地形态）。**只登记选了什么；批量优先的决策见铁律零。**

| 批量方法 | 分组键 | 只查规则 | 并行约束 |
|---------|-------|---------|---------|
| `{Module}.{Get-method}List` | `{shard-key}`（如 `{key} -> shard(N)`） | 仅查"有请求落点"的物理表，空 shard 不发查询 | 多 shard 并行（worker pool 上限见 §6，与 concurrency_safety.md §1.1 对齐） |

> 决策方法见 [`design-spec/01 §3.1`](../../../design-spec/01_io_design.md)（集合 enrich：循环外收集 key → 一次批量读）及铁律零批量优先（§2 / docs-spec/25 §2.0）。

### 5.6 读路径命中不合并登记

> 反向纪律登记：**已被本地缓存命中的读，禁止再合并进 Pipeline**（合并会绕过本地缓存 0 往返、反而劣化为一次 RTT）。本表登记本服务哪些读走"命中即返回、不进 Pipeline"。**只登记约定；"何时不合并"的反直觉原则见下方引用。**

| 本地缓存层 | 命中即返回的读 | 禁止合并进 Pipeline 的原因 |
|-----------|-------------|----------------------|
| `{L1-local-cache}`（如进程内 struct，0 网络 0 序列化） | `{Module}.{hot-read-method}` | 本地命中已是 0 往返，合并 Pipeline 反劣化为 1 次 RTT |

> 决策方法见 [`design-spec/01 §3.1`](../../../design-spec/01_io_design.md)（读路径反直觉点：消灭往返 > 合并往返；本地命中禁再合并 Pipeline）。

### 5.7 批量写极致登记（Upsert / 批量装载）

> 写路径极致登记：**插入/更新混合 → 批量 Upsert**（`ON DUPLICATE KEY`/`ON CONFLICT`）一次往返免 select-then-write；**大批量装载**（冷启 / 大迁移 / 离线导入）→ `LOAD DATA`/`COPY`。本表登记本工程哪些写走 Upsert、哪些走 bulk load。**只登记选了什么；"插入/更新混合 / 大批量装载怎么选"的写路径决策见下方引用。**

| 写形态 | 选定手段（登记） | 落点（哪些写） |
|-------|---------------|--------------|
| {插入/更新混合} | {批量 Upsert：`ON DUPLICATE KEY`/`ON CONFLICT`，一次往返} | `{file}:{func}` / `{Module}.{Upsert-method}List` |
| {大批量装载（冷启 / 迁移 / 离线导入）} | {`LOAD DATA`/`COPY` 批量装载} | `{file}:{func}` / `{bulk-load-entry}` |

> 决策方法见 [`design-spec/01 §3.2`](../../../design-spec/01_io_design.md)（写路径树：批量插入/更新混合 → Upsert；大批量装载 → `LOAD DATA`/`COPY`）。具体 SQL 见末尾「参考示例」段。

---

## 6. IO 预算与背压（引用 22）

| 维度 | 预算 | 引用 |
|------|------|------|
| 攒批 channel 容量 / 满时策略 | {N} | performance_contract.md §6.1 |
| 回源 worker pool 并发上限 | {W} | concurrency_safety.md §1.1 |
| 单批最大条数 | {Batch-max} | 本节 |

---

## 7. IO 回归测试

| 测试 | 构造 | 断言 |
|------|------|------|
| Test{Method}_NoNPlusOne | N 条关联数据 | 查询次数 == {常量}，与 N 无关 |
| Test{Method}_Parallel | 多个独立读 | 总耗时 ≈ max(单次) 而非 sum |

mock 后端 framework API 见 testing_design.md §4。

---

## 8. 维护流程

### 8.1 B1 反向同步规则

| 代码迹象 | 反向同步动作 |
|---------|------------|
| 循环内新增单条读（key 来自循环变量） | 命中铁律一；改批量 + §1 复核 |
| ≥ 2 个独立读串行 | 命中铁律二；改聚合 / 扇出，或 §2.2 登记依赖链 |
| 新增聚合器 / 批量读 / 扇出 | §3 / §4 登记 + §1 复核 |
| 新增读路径手段（本地缓存命中 / 消重复读 / 分支裁剪）或写路径削峰（Redis-first / MQ 攒批） | §5.4 读写路径分治登记新增 / 更新对应行 |
| 批量方法新增分表分组（按 shard 分组 / 只查有请求物理表 / 多 shard 并行） | §5.5 shard 分组并行编排登记新增一行 |
| 把已被本地缓存命中的读改为"命中即返回、不进 Pipeline" | §5.6 读路径命中不合并登记新增一行 |
| 新增批量 Upsert（`ON DUPLICATE KEY`/`ON CONFLICT`）/ 大批量装载（`LOAD DATA`/`COPY`） | §5.7 批量写极致登记新增 / 更新对应行 |
| 新增 worker pool / 信号量 | §6 登记 + 与 concurrency_safety.md §1.1 对齐 |
| 新增 IO 计数断言测试 | §7 同步 |

### 8.2 维护触发

| 触发 | 动作 |
|------|------|
| 新增集合访问资源方法 | §5 评估聚合 + 伪码标 `[BATCH]` / `[PARALLEL]` |
| 引入新数据源 / 缓存实例 | §1 预算 + §3 分组评估 |

---

## 9. 参考示例（仅示意，落地按业务替换）

> ⚠️ 以下为示意：本文主体（§1-§8）已用占位符表达；本节的表名、列名、SQL 均为具体示意，落地工程时按业务语义替换。本节服从 `aiweave/PRINCIPLES.md §12` 参考示例段豁免。

### 9.1 §5.7 批量 Upsert / 批量装载 SQL（示意）

```sql
-- 批量 Upsert：一次往返完成"有则更新无则插"，免 select-then-write（登记在 §5.7）
INSERT INTO t (k, v) VALUES (?,?),(?,?),…
  ON DUPLICATE KEY UPDATE v = VALUES(v);            -- MySQL
  -- ON CONFLICT (k) DO UPDATE SET v = EXCLUDED.v;  -- PostgreSQL

-- 大批量装载：冷启 / 迁移 / 离线导入，比多行 INSERT 快约 10-100×（登记在 §5.7）
LOAD DATA LOCAL INFILE 'f.csv' INTO TABLE t …;      -- MySQL
-- COPY t FROM STDIN …;                              -- PostgreSQL
```


---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
