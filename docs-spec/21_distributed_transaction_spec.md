# 21 - 分布式事务与补偿设计规范

> 规定 `docs/service/transaction_design.md` 的内容结构与维护规则。
>
> 本规范是 AIWeave 的 P0 规范之一，对应痛点 #4 事务边界与补偿、#2 跨服务调用链。

---

## 1. 定位

`transaction_design.md` 是 **AI 在生成涉及多数据源写入、跨服务调用、需要一致性保证的代码之前必须读的文档**。它回答三个问题：

> 1. 工程内每个"写入类业务操作"对应的事务模型是什么？
> 2. 失败时怎么补偿？补偿动作幂等吗？
> 3. 最终一致性的窗口是多长？读取策略是什么？

没有这篇文档：
- AI 把跨数据源写入当作两次单独写入，不处理失败补偿
- AI 不知道哪些接口需要幂等保证，遗漏 idempotent key 设计
- AI 把"最终一致性"理解为"立即一致"，读取策略错误

---

## 2. 顶层结构（强制）

`transaction_design.md` 必须按以下章节顺序组织：

- `## 1.` 事务策略总览（事务模型选型 + 选型理由）
- `## 2.` 事务边界清单（核心 / 强制）
- `## 3.` 补偿逻辑设计（每个 Saga 步骤）
- `## 4.` 幂等性设计（含 `§4.4` 读写分离消幂等登记 = 表示记录槽位）
- `## 5.` 最终一致性窗口（`§5.1` 窗口承诺 + `§5.2` 弱一致边界兜底登记 + `§5.3` delta flush 不变量登记 + `§5.4` 批量提交 / group-commit 登记 = 表示记录槽位）
- `## 6.` 失败路径全景图（强制）
- `## 7.` 与 Service 伪代码的关系（伪码标记语法）
- `## 8.` 维护流程（含 B1 反向同步规则）

> **表示记录槽位 vs 决策方法**：`§4.4` / `§5.2` / `§5.3` / `§5.4` 是**工程填空"选了什么"的记录槽**（待填表 + `{占位符}`），不承载"何时该这么选"的决策树——决策方法论的真相源在 [`design-spec/04 §3`](../design-spec/04_transaction_design.md)，槽位仅一行反向引用回链，不复制。

> **完整章节骨架见** [`templates/docs/service/transaction_design.md`](../templates/docs/service/transaction_design.md)。

---

## 3. §1 事务策略总览

```markdown
## 1. 事务策略总览

### 1.1 本工程的事务模型选型

| 业务类别 | 事务模型 | 适用范围 |
|---------|---------|---------|
| {单数据源写入类} | 本地事务（GORM Transaction） | {Module-A} / {Module-B} |
| {跨数据源 / 强补偿类} | Saga（正向 + 补偿） | {Module-C} 的 {state-change-method} |
| {高吞吐异步类} | MQ 投递 + 最终一致 | {flush-task} / {audit-stream} |
| {跨服务远程调用类} | 本地消息表 + 重试 | 远程 `{external-api}` 调用 |

### 1.2 选型理由

| 维度 | 本地事务 | Saga | MQ 异步 |
|------|---------|------|--------|
| 一致性 | 强 | 最终 | 最终（窗口 {N}s） |
| 性能 | 单库写入快 | 多步骤序列化 | 异步，无写入路径阻塞 |
| 运维复杂度 | 低 | 高（需要补偿逻辑 + 监控） | 中（依赖 MQ 可用性） |
```

---

## 4. §2 事务边界清单（核心 / 强制）

### 4.1 表格模板

```markdown
| 业务操作 | 事务范围 | 涉及数据源 | 一致性级别 | 失败策略 |
|---------|---------|-----------|-----------|---------|
| `{Module-A}.{create-method}` | 本地事务 | MySQL `{db-core}` | 强一致 | 回滚 |
| `{Module-B}.{state-change-method}` | Saga 3 步 | MySQL `{db-trade}` + Redis `{cluster}` | 最终一致（窗口 {N}s） | 补偿（详见 §3）|
| `{Module-C}.{flush-method}` | MQ 投递 | Kafka `{topic}` + MySQL `{db-log}` | 最终一致 | 重试 + 死信 |
```

### 4.2 必填字段

| 字段 | 说明 |
|------|------|
| 业务操作 | Service 方法的完整路径（`{Module}.{Method}`），与 [`docs-spec/09 §10 §4.X.8`](09_service_design_spec.md) 的"事务与一致性"子节双向对应 |
| 事务范围 | "本地事务" / "Saga N 步" / "MQ 投递" / "无事务（只读）" |
| 涉及数据源 | 列出所有读写的数据源，含 MySQL 库名 / Redis 集群 / Kafka topic |
| 一致性级别 | 强一致 / 最终一致（必须标窗口时长） |
| 失败策略 | 回滚 / 补偿 / 重试 / 告警人工介入 |

---

## 5. §3 补偿逻辑设计

每个 Saga 步骤必须有"正向 + 补偿"对照：

```markdown
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
```

---

## 6. §4 幂等性设计

```markdown
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
```

### 6.1 §4.4 读写分离消幂等登记（表示记录槽位）

`§4.4` 是**表示记录槽位**：登记本工程哪些操作把"读判定（只读幂等）+ 写执行"拆成两接口，从而在架构层省掉"预扣 + 回退 + 定时扫描"。槽位**只记选了什么**（读接口 / 写接口 / 省掉的重机制 / 漏写损耗承诺），决策方法（何时该这么拆）不复制，反向引用 [`design-spec/04 §3.2`](../design-spec/04_transaction_design.md)。

```markdown
| 原合一操作 | 拆出的读接口（只读幂等） | 拆出的写接口 | 省掉的重机制 | 漏写损耗承诺 |
|-----------|----------------------|------------|------------|------------|
| `{Module}.{combined-method}` | `{Module}.{read-judge-method}` | `{Module}.{write-exec-method}` | 预扣 + 回退 + `{scan-task}` 定时扫描 | {漏写=可接受损耗，{监控指标} 兜底} |
```

---

## 7. §5 最终一致性窗口

`§5` 含四个子节：`§5.1` 一致性窗口承诺（窗口时长 + 读取策略），`§5.2` / `§5.3` 为**高频数值变更**的两个**表示记录槽位**（见 §7.1 / §7.2），`§5.4` 为**批量提交 / group-commit**的持久性窗口登记（见 §7.3）。

```markdown
### 5.1 一致性窗口承诺

| 业务操作 | 一致性窗口承诺 | 窗口内读取策略 |
|---------|--------------|---------------|
| `{Module-B}.{state-change-method}` | 5s 内 Redis 与 MySQL 一致 | 优先读 Redis（带版本号），跨窗口校对走 MySQL |
| `{Module-C}.{flush-method}` | 1min 内 MySQL log 与 Kafka 一致 | 不对外暴露窗口内查询 |
```

### 7.1 §5.2 弱一致 + 边界兜底登记（表示记录槽位）

登记哪些高频 `{核心数值字段}` 变更采用"热路径不拦越界 + 持久化边界 `GREATEST`/`LEAST` 夹断 + 记 WARN 当损耗"。槽位**只记选了什么 + 夹断表达式 + 损耗指标**，决策方法（何时用弱一致换零锁）不复制，反向引用 [`design-spec/04 §3.2`](../design-spec/04_transaction_design.md)。

```markdown
| 数值变更操作 | 热路径原子命令（不拦越界） | 持久化边界夹断表达式 | 越界损耗记录 |
|------------|------------------------|------------------|------------|
| `{Module}.{numeric-change-method}` | `{atomic-cmd}`（O(1) 无锁） | `GREATEST({核心数值字段}+{delta}, {下界})` / `LEAST({核心数值字段}+{alloc}, {上界})` | 越界夹断记 WARN + `{损耗指标}` |
```

### 7.2 §5.3 delta 增量 flush 一致性不变量登记（表示记录槽位）

登记"内存态高频写 + 定时增量 flush 落库"路径的一致性不变量式与校验窗口处理。槽位**只记不变量与校验窗口**，削峰编排决策反向引用 [`design-spec/04 §3.1`](../design-spec/04_transaction_design.md)（数值不变量）与 [`design-spec/01 §3.2`](../design-spec/01_io_design.md)（flush 编排）。

```markdown
| flush 路径 | 一致性不变量式 | 校验窗口处理 | 脏标记消费原子性 |
|-----------|--------------|------------|---------------|
| `{Module}.{flush-method}` | `{内存值}` = `{DB聚合}` + `{delta}` | `expected = {db_agg} + {pending_delta}`（吸收 flush 窗口并发新增量） | `{pop-cmd}` 弹出即消费 → 多实例并行不重复、无需分布式锁 |
```

### 7.3 §5.4 批量提交 / group-commit 登记（表示记录槽位）

登记哪些高频写走 group-commit（N 条逻辑写攒进一个事务 / 一次提交，把 N 次 fsync 摊成 1 次）+ 批大小 / 提交间隔 / 刷盘级别（`innodb_flush_log_at_trx_commit` 等）+ 崩溃可丢失窗口。槽位**只记选了什么 + 旋钮 + 可丢失窗口**，何时该摊批 / 何时放宽持久性的决策方法不复制，反向引用 [`design-spec/04 §3.3`](../design-spec/04_transaction_design.md)。**取舍本质：持久性窗口换吞吐**——批越大吞吐越高，代价是崩溃可能丢"已 ack 未落盘"的一批；强一致核心库不放宽刷盘级别（见 [`design-spec/02 §9.1`](../design-spec/02_data_model_design.md)）。

```markdown
| 写路径 / 目标库 | 批大小 | 提交间隔 | 刷盘级别（`innodb_flush_log_at_trx_commit` 等） | 崩溃可丢失窗口 |
|----------------|--------|---------|----------------------------------------------|---------------|
| `{Module}.{batch-commit-method}` / `{db-write-heavy}` | `{batch-size}` | `{commit-interval}` | `{2=允许丢 ~1s / 1=每事务落盘}` | `{已 ack 未落盘的一批 ≈ {window}}` |
```

> 决策方法（小事务高频 → 攒批摊 fsync、持久性可放宽 → 组提交 + 刷盘分级、纯追加极高写 → 出 DB 走 MQ 攒批）见 [`design-spec/04 §3.3`](../design-spec/04_transaction_design.md)；与 [`design-spec/01 §3.2`](../design-spec/01_io_design.md) 写路径削峰互补（前者摊"必须进 DB 的写"，后者把写移出 DB 关键路径）。

---

## 8. §6 失败路径全景图（强制）

每个关键写入操作必须有 ASCII 失败路径图：

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

## 9. §7 与 Service 伪代码的关系（伪码标记语法）

涉及一致性的步骤伪码必须标注以下标记之一（统一标记表见 [`docs-spec/09 §10 §4.X.3`](09_service_design_spec.md)）：

| 标记 | 含义 |
|------|------|
| `[TXN-START]` / `[TXN-COMMIT]` / `[TXN-ROLLBACK]` | 本地事务边界 |
| `[SAGA-STEP-N]` / `[COMPENSATE-N]` | Saga 正向 / 补偿步骤 |
| `[IDEMPOTENT-CHECK: key={...}]` | 幂等校验入口 |
| `[INVARIANT-CHECK: I-N]` | 此步必须保护领域不变量（详见 [`docs-spec/09 §7`](09_service_design_spec.md)） |
| `[LOCK-ACQUIRE: {lock-name}]` / `[LOCK-RELEASE: {lock-name}]` | 加锁 / 释放（详见 [`docs-spec/20 §3`](20_concurrency_safety_spec.md)） |
| `[HOT-PATH]` | 此方法属热路径（详见 22_performance_contract） |
| `[METRIC-EMIT: name{labels}]` | 此步发出指标（详见 23_observability） |

> **价值**：AI 写伪码、读伪码、生成代码、审计代码共享同一组机械锚点。

---

## 10. §8 维护流程（含 B1 反向同步规则）

### 10.1 B1 反向同步规则（强制）

| 代码迹象（git diff） | 反向同步动作 |
| --- | --- |
| 新增 `BeginTx` / `db.Transaction(...)` / `tx.Commit` / `tx.Rollback` | §2 事务边界清单新增一行；同步 `docs-spec/09 §4.X.8 事务与一致性` |
| 新增 Saga 步骤标记（伪码 `[SAGA-STEP-N]`） | §3 补偿逻辑设计新增正向/补偿对照行；§6 失败路径全景图扩展分支 |
| 新增 `idempotent` / 唯一索引 / SETNX 去重类代码 | §4 幂等性设计新增 Key 生成规则与适用接口 |
| 把合一操作拆成"只读判定 + 写执行"两接口（架构层去幂等） | §4.4 读写分离消幂等登记新增一行 |
| 热路径原子 ± `{核心数值字段}` + 落库 `GREATEST`/`LEAST` 夹断 | §5.2 弱一致 + 边界兜底登记新增一行 |
| 新增"内存态原子写 + 脏标记 + 定时增量 flush"路径 | §5.3 delta 增量 flush 不变量登记新增一行（不变量 + 校验窗口） |
| 把小事务高频提交改为攒批一次提交 / 调 `innodb_flush_log_at_trx_commit` 等刷盘级别 | §5.4 批量提交 / group-commit 登记新增一行（批大小 + 提交间隔 + 刷盘级别 + 可丢失窗口） |
| 修改一致性等级（强一致 ↔ 最终一致） | §1.1 事务模型选型变更日志 + §5.1 一致性窗口承诺更新 |
| 新增跨数据源写入（MySQL + Redis / MySQL + MQ） | §2 必须新增一行；如未在 §1.1 列出对应模型 → 先补 §1.1 |

### 10.2 维护触发

| 触发 | 动作 |
|------|------|
| 新增写入类 Service 方法 | §2 必须新增一行；同步 `docs-spec/09 §4.X.8` |
| 修改事务模型选型 | §1.1 变更 + 影响清单走对账 |
| 引入新数据源 | §2 / §5 评估对一致性窗口的影响 |

---

## 11. 与其他文档的关系

| 内容 | transaction_design.md | service docs §4.X.8 | service_design.md |
|------|----------------------|--------------------|--------------------|
| 事务策略总览 / 选型 | ✅ 真相源 | 不写 | 引用 |
| 事务边界清单 | ✅ 真相源 | 单方法粒度引用 | 不重复 |
| 补偿逻辑表 | ✅ 真相源 | 单方法粒度引用 | 不写 |
| 幂等 Key 模板 | ✅ 真相源 | 单方法粒度引用 | 不写 |
| 一致性窗口 | ✅ 真相源 | 单方法粒度引用 | 不写 |
| 批量提交 / group-commit 登记（§5.4） | ✅ 真相源 | 单方法粒度引用 | 不写 |
| 失败路径全景图 | ✅ 真相源 | 不重复 | 不写 |

---

## 12. 与 Skill 的联动

- **`/new-service`** 生成涉及多数据源写入的方法时：先检查 §2 是否已有对应条目；未登记 → 拒绝生成，要求先补本文档
- **`/sync-feature-to-docs`**：扫描 git diff 中的 `BeginTx` / SETNX / 唯一索引迹象 → 反向同步 §2 / §4
- **`/doc-sync-check`** 维度扩展：检查 service 方法签名是否在 §2 出现矛盾

---

## 13. 参考示例（仅示意，落地按业务替换）

> ⚠️ 以下为示意，规范本体（§1-§12）已用占位符表达；落地工程时按业务语义替换占位符，不得直接照搬本节具体业务名作为规范。
>
> 示例服从 [`PRINCIPLES.md §12 占位符规则`](../PRINCIPLES.md#12-占位符规则强制--双轨结构) 的"参考示例段豁免"。

### 13.1 §2 事务边界清单（示例）

| 业务操作 | 事务范围 | 涉及数据源 | 一致性级别 | 失败策略 |
|---------|---------|-----------|-----------|---------|
| `AccountService.Register` | 本地事务 | MySQL core | 强一致 | 回滚 |
| `BillingService.Deduct` | Saga 3 步 | MySQL trade + Redis | 最终一致（5s） | 补偿 |
| `CallLogConsumer.Flush` | MQ 投递 | Kafka + MySQL log | 最终一致（1min） | 重试 + 死信 |

### 13.2 §3 补偿逻辑（示例：BillingService.Deduct）

| Step | 正向操作 | 补偿操作 | 触发条件 |
|------|---------|---------|---------|
| 1 | INSERT billing_log (状态=PENDING) | UPDATE 状态=CANCELLED | Step 2 / 3 失败 |
| 2 | DECR Redis `acc:balance:%s` | INCR Redis `acc:balance:%s` | Step 3 失败 |
| 3 | UPDATE billing_log (状态=DONE) | — | — |

### 13.3 §7 伪码标记（示例：扣减余额）

```
1. checkParams(req)
2. [IDEMPOTENT-CHECK: key=acc:idemp:deduct:{orderId}]
3. [INVARIANT-CHECK: I-1]  # balance 非负
4. [LOCK-ACQUIRE: account-{userId}]
5. [TXN-START]
6. SELECT balance FROM account WHERE id=? FOR UPDATE
7. balance -= req.Amount
8. UPDATE account SET balance=? WHERE id=?
9. [SAGA-STEP-2]  DECR acc:balance:{userId}
10. [METRIC-EMIT: billing_deduct_total{result="ok"}]
11. [TXN-COMMIT]
12. [LOCK-RELEASE: account-{userId}]
```


---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
