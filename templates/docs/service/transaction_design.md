# 分布式事务与补偿设计

> 版本：1.0 | 日期：{YYYY-MM-DD}
>
> 规范来源：[`aiweave/docs-spec/21_distributed_transaction_spec.md`](../../../docs-spec/21_distributed_transaction_spec.md)
>
> 本文档是 **AI 在生成涉及多数据源写入、跨服务调用、需要一致性保证的代码之前必须读的文档**。

---

## 1. 事务策略总览

### 1.1 本工程的事务模型选型

| 业务类别 | 事务模型 | 适用范围 |
|---------|---------|---------|
| {单数据源写入类} | 本地事务（GORM Transaction） | `{Module-A}` / `{Module-B}` |
| {跨数据源 / 强补偿类} | Saga（正向 + 补偿） | `{Module-C}` 的 `{state-change-method}` |
| {高吞吐异步类} | MQ 投递 + 最终一致 | `{flush-task}` / `{audit-stream}` |
| {跨服务远程调用类} | 本地消息表 + 重试 | 远程 `{external-api}` 调用 |

### 1.2 选型理由

| 维度 | 本地事务 | Saga | MQ 异步 |
|------|---------|------|--------|
| 一致性 | 强 | 最终 | 最终（窗口 {N}s） |
| 性能 | 单库写入快 | 多步骤序列化 | 异步，无写入路径阻塞 |
| 运维复杂度 | 低 | 高（需要补偿逻辑 + 监控） | 中（依赖 MQ 可用性） |

---

## 2. 事务边界清单（核心）

| 业务操作 | 事务范围 | 涉及数据源 | 一致性级别 | 失败策略 |
|---------|---------|-----------|-----------|---------|
| `{Module-A}.{create-method}` | 本地事务 | MySQL `{db-core}` | 强一致 | 回滚 |
| `{Module-B}.{state-change-method}` | Saga 3 步 | MySQL `{db-trade}` + Redis `{cluster}` | 最终一致（窗口 {N}s） | 补偿（详见 §3）|
| `{Module-C}.{flush-method}` | MQ 投递 | Kafka `{topic}` + MySQL `{db-log}` | 最终一致 | 重试 + 死信 |

> **维护规则**：每新增一个写入类 service 方法，本表必须新增一行；同步在 `docs/service/{module}_service.md §4.N.8 事务与一致性`引用本行。

---

## 3. 补偿逻辑设计

### 3.1 {Module-B}.{state-change-method} 的 Saga 步骤

| Step | 正向操作 | 补偿操作 | 触发条件 |
|------|---------|---------|---------|
| 1 | INSERT `{example_table_trade}` (状态=PENDING) | UPDATE 状态=CANCELLED | Step 2 / 3 失败 |
| 2 | DECR Redis `{ns}:{state-key}:%s` | INCR Redis `{ns}:{state-key}:%s` | Step 3 失败 |
| 3 | UPDATE `{example_table_trade}` (状态=DONE) | — | — |

### 3.2 补偿幂等性保证

- 补偿操作必须可重复执行而不产生副作用
- 通过 `{idempotent-key}` 去重；窗口 {N}min，存于 Redis `{ns}:compensation_lock:%s`
- 补偿失败 N 次 → 写入 `{audit-table}` 并告警，转人工

### 3.3 补偿超时

- Saga 启动后 {N}s 内未走到终态 → 自动触发补偿链
- 由 `{cleanup-task}` 定时扫描中间态记录

---

## 4. 幂等性设计

### 4.1 幂等 Key 生成规则

| 场景 | Key 模式 | 来源 |
|------|---------|------|
| {外部回调类} | `{ns}:idemp:{action}:%s` | 调用方在 header / body 内传 |
| {内部重试类} | `{ns}:idemp:{action}:%s` | service 内基于业务字段生成 |

### 4.2 去重窗口

| 场景 | 窗口 | 实现 |
|------|------|------|
| 短期回调去重 | {N}min | Redis SETNX + EXPIRE |
| 跨天去重 | 永久 | MySQL 唯一索引 |

### 4.3 幂等适用的接口清单

| 接口 | 幂等 Key | 窗口 |
|------|---------|------|
| `POST /{prefix}/{audience}/{module}/{action-1}` | `{ns}:idemp:{action-1}:%s` | 5min |
| `POST /{prefix}/{audience}/{module}/{action-2}` | `{ns}:idemp:{action-2}:%s` | 24h |

### 4.4 读写分离消幂等登记（架构层规避幂等）

> 登记本工程哪些操作把"读判定（只读幂等）+ 写执行"拆成两个接口，从而在架构层省掉"预扣 + 回退 + 定时扫描"整套幂等机制（重复读天然幂等、漏写=可接受损耗）。**只登记选了什么；何时该这么拆的决策方法见下方引用。**

| 原合一操作 | 拆出的读接口（只读幂等） | 拆出的写接口 | 省掉的重机制 | 漏写损耗承诺 |
|-----------|----------------------|------------|------------|------------|
| `{Module}.{combined-method}` | `{Module}.{read-judge-method}` | `{Module}.{write-exec-method}` | 预扣 + 回退 + `{scan-task}` 定时扫描 | {漏写=可接受损耗，由 {监控指标} 兜底} |

> 决策方法见 [`design-spec/04 §3.2`](../../../design-spec/04_transaction_design.md)（弱一致 + 边界兜底 + 监控的架构层取舍）。

---

## 5. 最终一致性窗口

### 5.1 一致性窗口承诺

| 业务操作 | 一致性窗口承诺 | 窗口内读取策略 |
|---------|--------------|---------------|
| `{Module-B}.{state-change-method}` | {N}s 内 Redis 与 MySQL 一致 | 优先读 Redis（带版本号），跨窗口校对走 MySQL |
| `{Module-C}.{flush-method}` | {N}min 内 MySQL log 与 Kafka 一致 | 不对外暴露窗口内查询 |

### 5.2 弱一致 + 边界兜底登记（高频数值变更）

> 登记本工程哪些高频 `{核心数值字段}` 变更采用"热路径不拦越界（原子但允许瞬时越界）+ 在持久化边界用 `GREATEST` / `LEAST` 夹断 + 记 WARN 当系统损耗"。**只登记选了什么 + 夹断表达式与损耗指标；何时用弱一致换零锁的决策方法见下方引用。**

| 数值变更操作 | 热路径原子命令（不拦越界） | 持久化边界夹断表达式 | 越界损耗记录 |
|------------|------------------------|------------------|------------|
| `{Module}.{numeric-change-method}` | `{atomic-cmd}`（如 `HINCRBY {ns}:{state-field}:%s` ±1，O(1) 无锁） | `GREATEST({核心数值字段}+{delta}, {下界})` / `LEAST({核心数值字段}+{alloc}, {上界})` | 越界夹断记 WARN + `{损耗指标}` |

> 决策方法见 [`design-spec/04 §3.2`](../../../design-spec/04_transaction_design.md)（强一致复杂度过高时优先评估弱一致 + 边界兜底）。

### 5.3 delta 增量 flush 一致性不变量登记

> 登记本工程"内存态高频写 + 定时增量 flush 落库"路径的一致性不变量式，以及 flush 窗口内并发新增量如何被校验吸收（避免误判账不平）。**只登记不变量与校验窗口处理；削峰编排本身的决策方法见下方引用。**

| flush 路径 | 一致性不变量式 | 校验窗口处理 | 脏标记消费原子性 |
|-----------|--------------|------------|---------------|
| `{Module}.{flush-method}` | `{内存值}` = `{DB聚合}` + `{delta}`（如 `Redis.{state-field}` = `DB.SUM({col})` + `Redis.{delta-field}`） | `expected = {db_agg} + {pending_delta}`（吸收 flush 窗口内并发新增量，避免误判） | `{pop-cmd}`（如 `SPOP` 弹出即消费）→ 多实例并行 flush 不重复、无需分布式锁 |

> 决策方法见 [`design-spec/04 §3.1`](../../../design-spec/04_transaction_design.md)（高频数值变更：单命令原子 + 持久化边界兜底）；削峰 flush 编排见 [`design-spec/01 §3.2`](../../../design-spec/01_io_design.md)。

### 5.4 批量提交 / group-commit 登记（持久性窗口换吞吐）

> 登记哪些高频写走 group-commit（N 条逻辑写攒进一个事务 / 一次提交，把 N 次 fsync 摊成 1 次）+ 批大小 / 提交间隔 / 刷盘级别 + 崩溃可丢失窗口。**只登记选了什么 + 旋钮 + 可丢失窗口；何时该摊批 / 何时放宽持久性的决策方法见下方引用。取舍本质：批越大吞吐越高，代价是崩溃丢"已 ack 未落盘"的一批；强一致核心库不放宽刷盘级别。**

| 写路径 / 目标库 | 批大小 | 提交间隔 | 刷盘级别（`innodb_flush_log_at_trx_commit` 等） | 崩溃可丢失窗口 |
|----------------|--------|---------|----------------------------------------------|---------------|
| `{Module}.{batch-commit-method}` / `{db-write-heavy}` | `{batch-size}` | `{commit-interval}` | `{2=允许丢 ~1s / 1=每事务落盘}` | `{已 ack 未落盘的一批 ≈ {window}}` |

> 决策方法见 [`design-spec/04 §3.3`](../../../design-spec/04_transaction_design.md)（小事务高频 → 攒批摊 fsync / 持久性可放宽 → 组提交 + 刷盘分级 / 纯追加极高写 → 出 DB 走 MQ 攒批）；与 [`design-spec/01 §3.2`](../../../design-spec/01_io_design.md) 写路径削峰互补；刷盘分级与分库见 [`design-spec/02 §9.1`](../../../design-spec/02_data_model_design.md)。

---

## 6. 失败路径全景图

> 每个关键写入操作必须有 ASCII 失败路径图，覆盖 happy path + 所有失败分支 + 补偿路径。

```
                {Module}.{Method}
                       │
                       ▼
                 [checkParams]
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
         参数错误             参数 OK
                                 │
                                 ▼
                         [TXN-START]
                                 │
               ┌─────────────────┼─────────────────┐
               ▼                 ▼                 ▼
         INSERT 失败       INSERT 成功         INSERT 唯一索引冲突
               │                 │                 │
               ▼                 ▼                 ▼
         [TXN-ROLLBACK]    [写 Redis]      [转更新或返回已存在]
         返回 ErrorDb            │
                                 ▼
                       ┌─────────┴─────────┐
                       ▼                   ▼
                   Redis 失败           Redis 成功
                       │                   │
                       ▼                   ▼
               ⚠️ 异步对账修正        [TXN-COMMIT]
               （不阻塞主路径）             │
                                           ▼
                                       返回 Resp
```

---

## 7. 与 Service 伪代码的关系

涉及一致性的步骤必须在 `docs/service/{module}_service.md §4.N.3 处理步骤` 伪码中使用统一标记（详见 [`aiweave/docs-spec/09 §10.5`](../../../docs-spec/09_service_design_spec.md)）：

| 标记 | 含义 |
|------|------|
| `[TXN-START]` / `[TXN-COMMIT]` / `[TXN-ROLLBACK]` | 本地事务边界 |
| `[SAGA-STEP-N]` / `[COMPENSATE-N]` | Saga 正向 / 补偿步骤 |
| `[IDEMPOTENT-CHECK: key={...}]` | 幂等校验入口 |
| `[INVARIANT-CHECK: I-N]` | 此步必须保护领域不变量 |
| `[LOCK-ACQUIRE: {lock-name}]` / `[LOCK-RELEASE: {lock-name}]` | 加锁 / 释放 |

---

## 8. 维护流程

### 8.1 B1 反向同步规则

| 代码迹象（git diff） | 反向同步动作 |
| --- | --- |
| 新增 `BeginTx` / `db.Transaction(...)` | §2 新增一行；同步 `docs/service/{module}_service.md §4.N.8` |
| 新增 Saga 步骤标记 | §3 新增正向/补偿对照行；§6 失败路径全景图扩展分支 |
| 新增唯一索引 / SETNX 去重 | §4 新增 Key 生成规则与适用接口 |
| 把合一操作拆成"只读判定 + 写执行"两接口（架构层去幂等） | §4.4 读写分离消幂等登记新增一行 |
| 热路径原子 ± `{核心数值字段}` + 落库 `GREATEST`/`LEAST` 夹断 | §5.2 弱一致 + 边界兜底登记新增一行 |
| 新增"内存态原子写 + 脏标记 + 定时增量 flush"路径 | §5.3 delta 增量 flush 不变量登记新增一行（不变量 + 校验窗口） |
| 把小事务高频提交改为攒批一次提交 / 调 `innodb_flush_log_at_trx_commit` 等刷盘级别 | §5.4 批量提交 / group-commit 登记新增一行（批大小 + 提交间隔 + 刷盘级别 + 可丢失窗口） |
| 修改一致性等级 | §1.1 变更日志 + §5.1 一致性窗口承诺更新 |

### 8.2 与 BUILD_STATUS §11 约束清单状态轨道的关系

每条事务边界条目（§2 一行）对应 BUILD_STATUS.md §11 的"已设计 / 已启用"计数。新增事务边界 → BUILD_STATUS §11 对应类目"已设计条目数"+1。


---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
