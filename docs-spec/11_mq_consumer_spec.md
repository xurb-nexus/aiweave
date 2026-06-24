# 11 - MQ 消费者设计文档规范

> 规定 `docs/service/scheduled_tasks_design.md` 中"MQ 消费者"部分的内容结构（通常与定时任务合写在同一篇 md，但视角独立）。

> 适用 Kafka、RocketMQ、Pulsar 等。本规范以 Kafka 为主线，其他 MQ 类似适配。

---

## 1. 定位

MQ 消费者文档要让 AI 能回答：

1. 这个工程从哪些 topic 消费、生产到哪些 topic
2. 每条消息的字段结构（含类型、单位、约束）
3. 消费端的攒批策略、提交语义、失败处理
4. 与 Redis / MySQL 的协同（如 HINCRBY + 写 MySQL 的双更新）
5. 死信队列与重试

精度要求：**消息 struct 字段级 + 消费流程步骤级**。AI 仅凭这一篇就能生成 `controllers/mq/{topic}_consumer.go`。

---

## 2. 顶层结构

```markdown
## 3. MQ 消费者（如有）

### 3.1 总览
   - 消费的 topic 清单
   - 生产的 topic 清单（用于 Producer 侧）
   - 与定时任务的协同

### 3.2 每个 Consumer 的详细设计
   3.2.1 消息结构（Go struct + JSON 字段表）
   3.2.2 消费流程
       - 攒批策略
       - 落库
       - Redis 联动
       - offset 提交
   3.2.3 失败处理
       - 重试策略
       - 死信队列
       - 告警
   3.2.4 性能与并发
       - 单 Consumer 吞吐
       - 分区数与并发
       - 反压机制

### 3.3 Producer 侧
   3.3.1 消息构造
   3.3.2 调用方
   3.3.3 失败处理（同步/异步、降级）

### 3.4 helpers 集成
   3.4.1 PubClient / SubClient 全局变量
   3.4.2 InitKafka{Producer/Consumer} 时机

### 3.5 配置
   resource.yaml 节点：brokers / topic / group / balance / offset
```

---

## 3. §3.1 总览（必填）

```markdown
### 3.1 总览

**消费的 topic**：

| Topic | 消费者文件 | 消费 group | 用途 |
|-------|----------|-----------|------|
| `{example-topic}` | `controllers/mq/{example-topic}_consumer.go` | `{example-consumer-group}` | 事件流水批量落库 |

**生产的 topic**：

| Topic | 生产者文件 | 调用方 | 用途 |
|-------|----------|--------|------|
| `{example-topic}` | `service/{module-N}/{producer-action}.go` | `{module}.{producer-method}` | {核心动作}后异步落库 |
| `{example-topic}_dlq` | `controllers/mq/{example-topic}_consumer.go` | Consumer 失败时 | 死信队列 |
```

---

## 4. §3.2.1 消息结构（强制）

### 4.1 Go struct + JSON 字段表（双重定义）

```markdown
#### 3.2.1 消息结构

\`\`\`go
// 与 service/{module-N}/{producer-action}.go 中 Producer 侧 struct 共享
type {Example}Msg struct {
    {IdempotentKey}  string  `json:"{idempotent-key}"`  // 自生成 ULID/UUID
    RequestId        string  `json:"requestId"`         // 网关传入
    EntityID         string  `json:"entityId"`
    {ActorRef}       string  `json:"{actor-ref}"`
    ActionID         string  `json:"actionId"`
    StatusCode       int     `json:"statusCode"`        // HTTP 状态码
    {BilledFlag}     int8    `json:"{billed-flag}"`     // 1=计入状态 0=不计入
    Cost             int64   `json:"cost"`              // 单位：分
    Latency          int     `json:"latency"`           // 单位：毫秒
    CalledAt         int64   `json:"calledAt"`          // Unix timestamp，秒
    ResponseSize     int     `json:"responseSize"`      // 字节
    CreatedAt        int64   `json:"createdAt"`         // Unix timestamp，秒
}
\`\`\`

| 字段 | 类型 | 必填 | 单位 | 来源 | 说明 |
|------|------|------|------|------|------|
| {idempotent-key} | string | 是 | — | `{module}.{producer-method}` 自生成 | 唯一标识，幂等用 |
| requestId | string | 否 | — | 网关传入 | 链路追踪 |
| entityId | string | 是 | — | 上游中间件后写入 | |
| {actor-ref} | string | 是 | — | 上游中间件后写入 | |
| cost | int64 | 是 | 分 | {计费模块}计算 | 实际数值变更金额 |
| ... |
```

**必填**：

- Go struct 完整定义（含 json tag、注释）
- 字段表（**含单位**——"分"和"元"必须明确）
- 来源（哪个上游字段或自生成）

### 4.2 协议版本兼容（如有）

如消息可能被多版本生产者/消费者共享，明确版本兼容策略：

- 新增字段 → 可空，老消费者忽略
- 删除字段 → 不允许，先标记 deprecated
- 字段类型变更 → 引入新字段，老字段保留

---

## 5. §3.2.2 消费流程（强制）

### 5.1 攒批策略

```markdown
**攒批策略**：

- 攒满 200 条 OR 等待 5 秒（先到先触发）
- 数值在 `docs/architecture/constants.md` §7 中定义
- 可通过 batchSize / batchTimeout 配置

**心智模型**：
\`\`\`
buffer = []{Example}Msg
loop:
    msg = consume(timeout=5s)
    if msg:
        buffer.append(msg)
    if len(buffer) >= 200 || elapsed >= 5s:
        flushBatch(buffer)
        buffer = []
        commitOffsets()
\`\`\`
```

### 5.2 落库流程（步骤伪码）

```markdown
**落库流程**：

1. 按 `calledAt` 月份分组：`2026-04-XX → {example_table_log}_202604`
2. 对每个分组：
   a. 检查分表是否存在（不存在则触发自动建表）
   b. 批量 INSERT INTO {tableName}
   c. 失败 → 重试（见 §3.2.3）
3. 同步更新 Redis 日统计：
   \`\`\`
   for each msg:
     HINCRBY {stats-key}:{entityId}:{actionId}:{date} {counter-total} 1
     if HINCRBY 返回 1: EXPIRE 172800 (48h)
     if msg.{biz-flag-1}:
       HINCRBY {stats-key}:* {counter-N} 1
       HINCRBY {stats-key}:* {amount-field} {msg.{amount-source}}
     if msg.{status-field} == 200:
       HINCRBY {stats-key}:* {counter-success} 1
     SADD {stats-key}:dirty {entityId}:{actionId}:{date}
   \`\`\`
4. 提交 Kafka offset（手动模式）
```

### 5.3 offset 提交语义

```markdown
**offset 提交策略**：手动提交（at-least-once）

- 批量落库**全部成功**才提交 offset
- 部分失败 → 失败的消息送 DLQ + 提交剩余 offset（避免无限重试阻塞）
- 全部失败 → 不提交 offset，下次重新消费同一批

**幂等设计**：消费端幂等基于 `{idempotent-key}`（唯一索引）。重复消费同一 key → MySQL 触发 duplicate key → 跳过。
```

---

## 6. §3.2.3 失败处理（强制）

```markdown
#### 3.2.3 失败处理

| 失败类型 | 处理策略 |
|---------|---------|
| MySQL INSERT 失败 | 重试 3 次（间隔 1s/2s/4s），3 次仍失败 → DLQ |
| Redis HINCRBY 失败 | tlog.WarnLogger，不影响 MySQL 写入（Redis 故障时通过 daily_stats 补偿任务兜底） |
| 分表不存在 | 触发自动建表 + 重试当前批次 |
| 消息 JSON 反序列化失败 | 直接 DLQ（脏数据，不重试） |
| Consumer 进程崩溃 | sarama 自动重平衡，未提交 offset 的消息会被重新分配给其他 Consumer |

**死信队列（DLQ）**：

- topic：`{原 topic}_dlq`
- 消息体：原消息 + 错误信息 + 重试次数
- 处理：人工介入或独立 DLQ Consumer
```

---

## 7. §3.2.4 性能与并发

```markdown
#### 3.2.4 性能与并发

- **单 Consumer 吞吐**：~5000 msg/s（取决于 MySQL INSERT 性能）
- **分区数**：建议 = Consumer 实例数 × 2-3
- **并发模型**：sarama Consumer Group + Partition Worker Pool
- **反压**：buffer 满时阻塞拉取（不丢消息）
- **冷启动**：从 latest offset 开始消费（避免回放历史）
```

---

## 8. §3.3 Producer 侧（如本工程也是生产者）

```markdown
### 3.3 Producer 侧

**消息构造**（`service/{module-N}/{producer-action}.go`）：

\`\`\`go
msg := &{Example}Msg{
    {IdempotentKey}: generate{Key}(),
    EntityID:        req.EntityID,
    ActionID:        req.ActionID,
    Cost:            cost,        // 单位：分
    CalledAt:        time.Now().Unix(),
    ...
}
helpers.{Example}PubClient.Pub(ctx, "{example-topic}", msg)
\`\`\`

**调用方**：`{module}.{producer-method}`

**失败处理**：
- Pub 失败 → tlog.WarnLogger，不影响 `{producer-method}` 主流程返回（核心数值变更已在 Redis 完成）
- 异步降级：失败的消息暂存内存 buffer，定时重发（避免阻塞主流程）
```

> 真实业务示例（按具体 `{Module}` 替换 `{id-field}` / `{核心写入动作}` / `{主流程方法}`）见末尾 §14 参考示例。

---

## 9. §3.4 helpers 集成

```markdown
### 3.4 helpers 集成

**全局变量**：

\`\`\`go
// helpers/kafka.go
var {Example}PubClient *kafka.PubClient
var {Example}SubClient *kafka.SubClient
\`\`\`

**初始化时机**：

- HTTP 服务模式：`InitResource()` 中调用 `InitKafkaProducer(engine)`
- Consumer 模式：cobra 子命令 `{topic}-consumer` 的 Run 回调中调用 `InitResourceForCron(engine); InitKafkaConsumer(engine)`

**关键设计**：Producer 与 Consumer **不同进程**——避免资源竞争与退出语义冲突。
```

---

## 10. §3.5 配置（强制）

```markdown
### 3.5 配置

**resource.yaml**：

\`\`\`yaml
kafkaPub:
  {example_log}:
    brokers: [...]
    topic: {example-topic}
    timeout: 5s
    asyncBufSize: 1000

kafkaSub:
  {example_log}:
    brokers: [...]
    topic: {example-topic}
    group: {example-consumer-group}
    balance: range  # range / roundrobin
    offset: latest  # latest / earliest
    sessionTimeout: 30s
\`\`\`

**Go struct 映射**：

\`\`\`go
type TKafkaSub struct {
    Brokers        []string
    Topic          string
    Group          string
    Balance        string
    Offset         string
    SessionTimeout time.Duration
}
\`\`\`
```

---

## 11. 失败案例与排查

```markdown
| 症状 | 可能原因 | 排查 |
|------|---------|------|
| Consumer 消费不到消息 | offset 已 commit / group rebalance / 分区分配 | 查 Kafka 控制台 group 状态 |
| 消息丢失 | offset 提前 commit / DLQ 失败也提交了 | 检查 §3.2.3 提交语义 |
| 重复落库（{idempotent-key} 冲突） | offset 未 commit + 重启 / DLQ 回放 | 检查 §3.2.3 幂等设计 |
| MySQL 慢，消费阻塞 | 批量 size 过大 / 索引缺失 | 调整 batchSize + 加索引 |
| Redis HINCRBY 缺失 | Redis 故障 + 没补偿任务 | 启用 daily_stats 补偿任务 |
```

---

## 12. 与其他文档的关系

| 内容 | scheduled_tasks_design.md §3 | cache_design.md | service/{module-N}_service.md |
|------|------------------------------|------------------|----------------------------|
| 消息 struct 定义 | ✅ 完整 | 不写 | 引用 |
| 消费流程 | ✅ 完整 | 不写 | 不写 |
| Redis 联动操作 | 简略（步骤伪码） | ✅ 详细（HINCRBY 字段映射） | 不写 |
| 调用 Producer 的位置 | 简略 | 不写 | ✅ 完整 |

---

## 13. 维护

| 触发 | 动作 |
|------|------|
| 新增 topic | §3.1 总览表 + §3.2 详细 + §3.5 配置 |
| 修改消息字段 | §3.2.1 双重定义 + 同步 Producer / Consumer 代码 + 协议兼容评估 |
| 调整攒批策略 | §3.2.2 + §3.2.4 + 同步 constants.md |
| 改变 offset 语义 | §3.2.2 提交语义 + 评估幂等性影响 |

---

## 14. 参考示例（仅示意，落地按业务替换）

> ⚠️ 以下为示意，规范本体（§1-§13）已用占位符表达；本节给出具体业务（业务校验链 + 状态上报 Report → call_log 异步落库）的示例，便于读者建立对应直觉。落地工程时按业务语义替换占位符，不得直接照搬本节具体业务名作为规范。
>
> 示例服从 [`PRINCIPLES.md §12 占位符规则`](../PRINCIPLES.md#12-占位符规则强制--双轨结构) 的"参考示例段豁免"。

### 14.1 §3.1 生产 topic（示例：状态上报）

| Topic | 生产者文件 | 调用方 | 用途 |
|-------|----------|--------|------|
| `call_log` | `service/billing/report.go` | `billing.Report` | 状态上报后异步落库 |

### 14.2 §3.2.1 消息结构（示例：CallLogMsg）

```go
// 与 service/billing/report.go 中 Producer 侧 struct 共享
type CallLogMsg struct {
    LogId        string  `json:"logId"`        // 自生成 UUID
    RequestId    string  `json:"requestId"`    // 网关传入
    EntityID     string  `json:"entityId"`
    UserId       string  `json:"userId"`
    ActionID     string  `json:"actionId"`
    StatusCode   int     `json:"statusCode"`
    Billed       int8    `json:"billed"`       // 1=计入扣减 0=不计入
    Cost         int64   `json:"cost"`         // 单位：分
    Latency      int     `json:"latency"`
    CalledAt     int64   `json:"calledAt"`
    ResponseSize int     `json:"responseSize"`
    CreatedAt    int64   `json:"createdAt"`
}
```

| 字段 | 类型 | 必填 | 单位 | 来源 | 说明 |
|------|------|------|------|------|------|
| logId | string | 是 | — | service.Report 自生成 | 唯一标识，幂等用 |
| userId | string | 是 | — | 业务校验后写入 | |
| cost | int64 | 是 | 分 | billing 计算 | 实际扣减金额 |

### 14.3 §3.3 Producer 侧（示例：billing.Report）

```go
msg := &CallLogMsg{
    LogId:    generateLogId(),
    EntityID: req.EntityID,
    ActionID: req.ActionID,
    Cost:     cost,
    CalledAt: time.Now().Unix(),
}
helpers.CallLogPubClient.Pub(ctx, "call_log", msg)
```

**调用方**：`billing.Report`
**失败处理**：Pub 失败不影响 Report 主流程返回（余额扣减已在 Redis 完成）

### 14.4 占位符 → 业务名 对照表（示例）

| 占位符 | 本节示例值 |
| --- | --- |
| `{Example}Msg` | CallLogMsg |
| `{idempotent-key}` / `{IdempotentKey}` | logId / LogId |
| `{actor-ref}` / `{ActorRef}` | userId / UserId |
| `{billed-flag}` / `{BilledFlag}` | billed / Billed |
| `{example-topic}` | call_log |
| `service/{module-N}/{producer-action}.go` | service/billing/report.go |
| `{module}.{producer-method}` | billing.Report |
| `{计费模块}` | billing |
| `{核心动作}` | 状态上报 |
| `generate{Key}()` | generateLogId() |



---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
