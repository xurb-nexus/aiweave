# 10 - 定时任务设计文档规范

> 规定 `docs/service/scheduled_tasks_design.md` 中"定时任务"部分的内容结构。本规范覆盖 cobra（外部调度）、cycle（内部协程间隔）、crontab（内部协程定点）三种执行模式。

> MQ 消费者部分有专门的规范见 `11_mq_consumer_spec.md`，但通常与定时任务**合写在同一篇 md** 中。

---

## 1. 定位

定时任务文档要让 AI 能回答：

1. 这个工程有哪些定时任务、各自的触发方式、间隔
2. 每个任务调用哪个 Service 方法、为什么这样设计
3. 多进程并行时如何避免重复处理（SPOP / 分布式锁 / 单进程外部调度）
4. 任务失败时如何恢复
5. 任务在 cobra 命令、cycle 协程、crontab 定点中的注册代码

精度要求：**Service 方法签名 + 注册代码片段 + 并行策略**。AI 仅凭这一篇就能生成 `controllers/command/` 和 `router/command.go` 中的注册代码。

---

## 2. 顶层结构（强制）

scheduled_tasks_design.md 必须按以下章节顺序组织：

- `## 1.` 任务总览（三种执行模式对比 + **任务清单总表** + 触发频率与时序图）
- `## 2.` cobra 任务（外部调度）（命名约定 / 注册代码模板 / 每个任务详细设计）
- `## 3.` cycle 任务（内部协程间隔执行）（cycle vs crontab 选择规则 / 注册模板 / 每个任务详细）
- `## 4.` crontab 任务（内部协程定点执行）
- `## 5.` 异常恢复与容错（崩溃恢复 / 锁未释放兜底 / 数据不一致修复 / 监控告警阈值）
- `## 6.` Controller 入口与 Service 方法签名（每个任务对应：controller 函数 + service 方法 + 参数）
- `## 7.` 并行策略详细（SPOP 模式 / 分布式锁模式 / 单进程外部调度）
- `## 8.` router/command.go 完整结构（含三种注册点样板）

> **完整章节骨架见** [`templates/docs/service/scheduled_tasks_design.md`](../templates/docs/service/scheduled_tasks_design.md)。下面各节给出每个章节的填充精度与必填要素。

---

## 3. §1.1 三种执行模式对比（必填）

```markdown
| 模式 | 触发方式 | 适用场景 | 并行特点 |
|------|---------|---------|---------|
| cobra | `go run main.go {task}` | 间隔 ≥ 30min 的外部调度（k8s CronJob / 云任务平台） | 由调度平台控制单进程执行 |
| cycle | 随 HTTP 服务启动，每实例都启 | 间隔 < 30min，"上次完成后再过 N 时间执行"（保证不重叠） | 多实例并行，需要 SPOP 或分布式锁 |
| crontab | 随 HTTP 服务启动，每实例都启 | 必须在固定时刻触发（如月底跨天对账） | 多实例并行，需要分布式锁 |
```

**判断流程**：

```
间隔 ≥ 30min → cobra
间隔 < 30min ┬ 强制定点 → crontab + 分布式锁
              └ 不重叠语义 → cycle + SPOP / 分布式锁
```

---

## 4. §1.2 任务清单总表（必填）

```markdown
| 任务名 | 类型 | Service 方法 | 触发频率 | 并行策略 | 文档章节 |
|-------|------|-------------|---------|---------|---------|
| `{flush-state-task}` | cycle | `{module-N}.{FlushStateMethod}` | 每 5min | SPOP（多进程并行安全） | §3.1 |
| `{integrity-check-task}` | cycle | `{auth-module}.{IntegrityCheckMethod}` | 每 1h | 分布式锁互斥 | §3.2 |
| `{periodic-aggregate-task-A}` | cobra | `{module-N}.{AggregateMethod-A}` | 每日 {time} | 单进程外部调度 | §2.1 |
| `{periodic-aggregate-task-B}` | cobra | `{module-N}.{AggregateMethod-B}` | 每月 1 日 {time} | 单进程外部调度 | §2.2 |
| {clean-expired-task} | cobra | `{module-N}.{cleanup-method}` | 每日 04:00 | 单进程外部调度 | §2.3 |
| {topic}-consumer | cobra（长驻） | Kafka Consumer | 长驻 | 单进程外部调度（独立进程） | §2.X |
```

**任务命名规范**：

- 全小写 + 连字符（kebab-case）
- 动词在前（`flush-`、`generate-`、`clean-`、`create-`、`cache-`）
- 与 cobra 子命令名一字不差

---

## 5. §2 cobra 任务的详细设计

### 5.1 §2.2 注册代码模板

```markdown
### 2.2 注册代码模板

\`\`\`go
// router/command.go
func Commands(rootCmd *cobra.Command, engine *gin.Engine) {
    rootCmd.AddCommand(&cobra.Command{
        Use: "{periodic-aggregate-task-A}",
        Run: func(cmd *cobra.Command, args []string) {
            run(engine, command_ctrl.{AggregateMethod-A}, args...)
        },
    })
    rootCmd.AddCommand(&cobra.Command{
        Use: "{periodic-aggregate-task}",
        Run: func(cmd *cobra.Command, args []string) {
            run(engine, command_ctrl.{AggregateMethod}, args...)
        },
    })
    // 新增任务在此追加
}

func run(engine *gin.Engine, f func(ctx *gin.Context, args ...string) error, args ...string) error {
    helpers.InitResourceForCron(engine)
    return helpers.Job.RunWithRecoveryE(f, args...)
}
\`\`\`
```

### 5.2 单个任务的完整规格

```markdown
### 2.X `{periodic-aggregate-task-A}`（每日 {time} 聚合）

**目标**：将 Redis 中的 `{stats-key}:*` 聚合数据写入 MySQL `{example_table_stats}` 表。

**触发**：cobra 子命令，外部调度平台每日 {time} 执行 `go run main.go {periodic-aggregate-task-A}`。

**并行策略**：单进程外部调度。如多进程并发，依赖 {stats-key}:dirty Set 的 SPOP 弹出避免重复。

**Service 方法**：

\`\`\`go
// service/{module-N}/{aggregate-file}.go
func {AggregateMethod-A}(ctx *gin.Context) error
\`\`\`

**执行流程**（步骤伪码）：

1. SMEMBERS `{stats-key}:dirty` → 取得待 flush 的 (entityId, actionId, date) 列表
2. 对每个 entry：
   - HGETALL `{stats-key}:{entityId}:{actionId}:{date}` → 取得 {核心统计字段集（如 _count / _amount）}
   - INSERT ... ON DUPLICATE KEY UPDATE 写入 MySQL
   - SREM {stats-key}:dirty {entry}
3. 失败的 entry 不 SREM，保留在 dirty Set 中，下次重试

**Controller 入口**：

\`\`\`go
// controllers/command/{periodic_aggregate_a}.go
package command

func {AggregateMethod-A}(ctx *gin.Context, args ...string) error {
    return {module-N}.{AggregateMethod-A}(ctx)
}
\`\`\`

**异常处理**：
- MySQL 写入失败：tlog.ErrorLogger 记录，不 SREM dirty 标记，下次任务重试
- Redis 读取失败：tlog.WarnLogger，跳过该 entry

**性能预期**：每次 flush 处理约 1000-5000 entries，总耗时 < 30s。

**监控**：
- entries 处理数：通过 metrics 暴露
- flush 耗时：直方图
- 错误率：> 5% 触发告警
```

---

## 6. §3 cycle 任务的详细设计

### 6.1 §3.2 注册代码模板

```markdown
### 3.2 注册代码模板

\`\`\`go
// router/command.go
func Tasks(engine *gin.Engine) {
    startCycle(engine)
    startCrontab(engine)
}

func startCycle(engine *gin.Engine) {
    c := command.InitCycle(engine)

    // 每 5 分钟一轮
    c.AddFunc(5*time.Minute, func(ctx *gin.Context) error {
        return {module}.FlushStateToDb(ctx)
    })
    // 每 1 小时一轮
    c.AddFunc(1*time.Hour, func(ctx *gin.Context) error {
        return {module}.CacheIntegrityCheck(ctx)
    })

    c.Start()  // 注意：c.Start() 只调用一次
}
\`\`\`

⚠️ 注意：
- `c.AddFunc` 必须在 `command.InitCycle(engine)` 之后、`c.Start()` 之前
- `c.Start()` 在函数末尾**只调用一次**——新增任务时不要再加一次 Start()
```

### 6.2 §3.X 单个 cycle 任务的完整规格

格式与 cobra 任务类似，但**额外说明**：

- 每实例都会启动 → 必须有并行安全策略（SPOP 或分布式锁）
- 任务执行完才计时（cycle 语义） → 不会出现两个相同任务并发执行同一实例

---

## 7. §4 crontab 任务的详细设计

### 7.1 注册代码模板

```markdown
### 4.1 注册代码模板

\`\`\`go
func startCrontab(engine *gin.Engine) {
    c := command.InitCrontab(engine)
    c.AddFunc("0 0 2 * * *", func(ctx *gin.Context) error {  // 6 位表达式：秒 分 时 日 月 周
        return {module-N}.{AggregateMethod-A}(ctx)
    })
    // InitCrontab 内部已调用 c.Start()，**不要**再调用 c.Start()
}
\`\`\`

> 当前项目优先选 cycle，仅"必须在固定时刻触发"才用 crontab。
```

### 7.2 cron 表达式规范

- **6 位表达式**：秒 分 时 日 月 周
- 使用 0-59 / 0-23 / 1-31 / 1-12 / 0-6 (周日=0)
- 时区：默认 UTC，按业务需求显式注明 +08:00 / Asia/Shanghai

---

## 8. §5 异常恢复与容错（强制）

```markdown
## 5. 异常恢复与容错

### 5.1 任务崩溃时的恢复
- cobra 任务崩溃 → 调度平台下次重试 + helpers.Job.RunWithRecoveryE 兜底 panic
- cycle / crontab 任务崩溃 → 协程被 recovery 中间件捕获，下次间隔到达后重启

### 5.2 锁未释放时的兜底
- 分布式锁必有 TTL（默认 5min - 30min）
- 即使进程崩溃未 DEL 锁，TTL 到期后自动释放
- 锁竞争失败时立即返回（不阻塞等待）

### 5.3 数据不一致时的修复
- flush 失败的 entry 保留在 dirty Set
- 下次任务自动重试
- 持续失败超过 N 次 → 告警人工介入
- cache_integrity_check 任务作为兜底层

### 5.4 监控告警触发条件
- entries 数量异常下降 → 上游消费者可能停了
- flush 耗时 P99 > 1min → 数据量异常或 MySQL 慢
- 错误率 > 5% → 立即告警
```

---

## 9. §7 并行策略详细

### 9.1 SPOP 模式

```markdown
### 7.1 SPOP 模式（推荐用于 flush 类任务）

**适用**：脏标记驱动的增量任务，每个 entry 独立处理。

**实现**：

\`\`\`go
for {
    member, err := conn.Do("SPOP", "{state-1}:dirty")
    if member == nil { break }   // dirty Set 已空
    
    // 处理 member（如 flush 该 entry 到 MySQL）
    if err := flushOne(member); err != nil {
        // 失败 → SADD 回去重试
        conn.Do("SADD", "{state-1}:dirty", member)
    }
}
\`\`\`

**优势**：多进程同时跑该任务时，SPOP 原子保证每个 entry 只被一个进程处理。
```

### 9.2 分布式锁模式

```markdown
### 7.2 分布式锁模式

**适用**：必须由单进程执行的任务（如全量遍历 / 跨 entry 聚合）。

**实现**：

\`\`\`go
const lockKey = "lock:{integrity_task}"
const lockTTL = 5 * time.Minute

reply, err := conn.Do("SET", lockKey, "1", "NX", "EX", int64(lockTTL.Seconds()))
if err != nil || reply == nil {
    return nil // 已被其他进程持有，本次跳过
}
defer conn.Do("DEL", lockKey)

// 执行任务...
\`\`\`

**关键**：
- 锁 TTL 必须长于任务最大耗时
- 用 defer 释放
- 崩溃时 TTL 兜底
```

### 9.3 单进程外部调度

```markdown
### 7.3 单进程外部调度

**适用**：cobra 任务，由 k8s CronJob / 任务平台保证只起一个进程。

**注意**：即使外部保证单进程，仍建议在 service 内增加分布式锁（防止人为多次启动）。
```

---

## 10. §8 router/command.go 完整结构

提供**可直接 copy-paste 的完整代码骨架**：

```go
package router

import (
    "time"

    command_ctrl "{project_module}/controllers/command"
    "{project_module}/helpers"
    "{project_module}/service/{module-3}"
    "{project_module}/service/{auth-module}"

    "github.com/spf13/cobra"

    "{project_module}/pkg/gin"
    "{project_module}/pkg/gin/command"
)

func Commands(rootCmd *cobra.Command, engine *gin.Engine) {
    // ... AddCommand 块
}

func run(engine *gin.Engine, f func(ctx *gin.Context, args ...string) error, args ...string) error {
    helpers.InitResourceForCron(engine)
    return helpers.Job.RunWithRecoveryE(f, args...)
}

func Tasks(engine *gin.Engine) {
    startCycle(engine)
    startCrontab(engine)
}

func startCycle(engine *gin.Engine) {
    c := command.InitCycle(engine)
    // ... AddFunc 块
    c.Start()
}

func startCrontab(engine *gin.Engine) {
    c := command.InitCrontab(engine)
    // ... AddFunc 块（如有）
}
```

新增任务时，**严格只在对应函数体内追加注册行**，不改变函数签名或拆文件。

---

## 11. 当前阶段简化（如有）

如果某些任务在当前阶段不实现，必须显式标注：

```markdown
### N.X {disabled-task}（每日 03:00 {受限模块}分析）

> ⛔ **当前阶段不实现**：{disabled_module} 模块整体不进入代码实现，详见 [`BUILD_STATUS.md`](../BUILD_STATUS.md) §0。设计完整保留如下，启用时按本节伪码生成。

（完整设计内容...）
```

---

## 12. 与其他文档的关系

| 内容 | scheduled_tasks_design.md | service/{module}_service.md | cache_design.md |
|------|---------------------------|----------------------------|------------------|
| 任务名 / 触发频率 / 并行策略 | ✅ 完整 | 引用 | 不写 |
| Service 方法签名 | 引用 | ✅ 完整 | 不写 |
| Service 方法步骤伪码 | 简略 | ✅ 完整 | 不写 |
| 注册代码 | ✅ 完整 | 不写 | 不写 |
| Redis Key 操作 | 简略 | 详细 | ✅ 注册表 |

---

## 13. 维护

| 触发 | 动作 |
|------|------|
| 新增任务 | §1.2 总表 + §2/§3/§4 详细 + §6 入口 + 同步 BUILD_STATUS §6 |
| 修改触发频率 | §1.2 + 对应任务详细 + 同步监控告警阈值 |
| 删除任务 | 整节移除 + BUILD_STATUS / router 同步清理 |
| 改变并行策略 | §7 详细 + §1.2 总表 |

---

## 14. 参考示例（仅示意，落地按业务替换）

> ⚠️ 以下为示意，规范本体（§1-§13）已用占位符表达；落地工程时按业务语义替换占位符，不得直接照搬本节具体业务名作为规范。
>
> 示例服从 [`PRINCIPLES.md §12 占位符规则`](../PRINCIPLES.md#12-占位符规则强制--双轨结构) 的"参考示例段豁免"。

### 14.1 §1.2 任务清单（示例：某计数 / 计费类业务）

| 任务名 | 类型 | Service 方法 | 触发频率 | 并行策略 | 详细章节 |
|-------|------|-------------|---------|---------|---------|
| `flush-state-to-db` | cycle | `BillingService.FlushStateToDb` | 每 5min | SPOP（多进程并行安全） | §3.1 |
| `cache-integrity-check` | cycle | `AuthService.CacheIntegrityCheck` | 每 1h | 分布式锁互斥 | §3.2 |
| `flush-daily-stats` | cobra | `BillingService.FlushDailyStats` | 每日 00:10 | 单进程外部调度 | §2.1 |
| `clean-expired-quota` | cobra | `BillingService.CleanExpiredQuota` | 每日 04:00 | 单进程外部调度 | §2.3 |

### 14.2 §3.x flush 流程伪码（示例）

```
1. SMEMBERS stats:dirty → 取得待 flush 的 (entityId, actionId, date) 列表
2. 对每个 entry：
   - HGETALL stats:{entityId}:{actionId}:{date} → 取得 total_count / counted_count / success_count / total_amount
   - INSERT ... ON DUPLICATE KEY UPDATE 写入 MySQL
   - SREM stats:dirty {entry}
3. 失败的 entry 不 SREM，保留在 dirty Set 中，下次重试
```

### 14.3 占位符 → 业务名 对照表（示例）

| 占位符 | 本节示例值 |
| --- | --- |
| `{clean-expired-task}` | clean-expired-quota |
| `{cleanup-method}` | CleanExpiredQuota |
| `{module-N}` | BillingService |
| `{核心统计字段集}` | total_count / counted_count / success_count / total_amount |


---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
