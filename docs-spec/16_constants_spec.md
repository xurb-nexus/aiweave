# 16 - 业务常量 / 魔法数字唯一真相源规范

> 规定 `docs/architecture/constants.md` 的内容结构。

---

## 1. 定位

业务常量文档是**魔法数字唯一真相源**。所有跨模块共享的"数值约定"都集中在这里：

- 业务校验窗口 ±300s
- 验证码 TTL 5min
- bcrypt cost = 10
- Pipeline 批大小 100
- Redis Key 模板字符串
- 分布式锁 TTL
- 三层频率限制阈值的默认值
- 攒批参数（200 条 / 5s）
- 各种超时

**为什么必须集中**：散布在各 service 的"60 * 60 * 24"、"5 * time.Minute"等数字，会导致：

- 同一概念在不同位置数值不同（测试 / 生产环境差异）
- 调整时需要全局搜索替换，容易遗漏
- 文档与代码漂移（md 写 ±300s，代码写 ±200s）

---

## 2. 顶层结构

```markdown
# 业务常量

## 1. 时间窗口
   1.1 业务校验 / 签名相关
   1.2 验证码 / 会话相关
   1.3 缓存 TTL

## 2. 加密参数
   2.1 bcrypt cost
   2.2 EntityID / EntitySecret 长度

## 3. 批量与攒批
   3.1 Pipeline 批大小
   3.2 Kafka 攒批参数
   3.3 flush 批量大小

## 4. 限流阈值（默认值）
   4.1 QPS 限流默认
   4.2 日调用量默认
   4.3 接口级日限默认

## 5. 分布式锁 TTL

## 6. ID 生成规则
   6.1 各业务对象的 ID 格式
   6.2 占位符规则

## 7. Redis Key 模板（如有）
   集中列出所有 Key 字符串模板（与 cache_design 互引用）

## 8. 各业务对象的状态值（如未在 status_codes.md 集中定义，则放这里）

## 9. 配置中默认值的来源（与 conf/ 的关系）

## 10. components/constants.go 目标代码（强制：完整 Go 文件内容）
```

---

## 3. §1-§9 常量分组（必填示例）

### 3.1 时间窗口

```markdown
## 1. 时间窗口

### 1.1 业务校验 / 签名相关

| 常量 | 值 | 用途 | 出处 |
|------|-----|------|------|
| `SignatureWindow` | `300 * time.Second` | 签名校验中间件时间戳允许偏移 | docs/architecture/{核心校验链}_flow.md §1 |
| `BatchVerifyMax` | 100 | 批量业务校验单次最多多少个 Key | audience_a_interfaces.md §2.2 |

### 1.2 验证码 / 会话

| 常量 | 值 | 用途 |
|------|-----|------|
| `VerifyCodeTTL` | `5 * time.Minute` | 验证码 Redis Key 过期时间 |
| `VerifyCodeLength` | 6 | 验证码位数 |
| `UserSessionTTL` | `2 * time.Hour` | 前端 BFF Session 有效期 |

### 1.3 缓存 TTL

| 常量 | 值 | 用途 |
|------|-----|------|
| `LocalCacheActiveTTL` | `15 * time.Minute` | gcache 活跃 TTL |
| `LocalCacheMaxTTL` | `30 * time.Minute` | gcache 最大 TTL |
| `DailyStatsTTL` | `48 * time.Hour` | daily_stats Hash TTL |
```

### 3.2 加密参数

```markdown
## 2. 加密参数

| 常量 | 值 | 用途 |
|------|-----|------|
| `BcryptCost` | 10 | bcrypt 哈希成本（约 80ms / 次） |
| `EntityIDPrefix` | `"ak_"` | EntityID 前缀 |
| `EntityIDLength` | 19 | EntityID 总长度（含前缀） |
| `EntitySecretLength` | 32 | EntitySecret 长度 |
```

### 3.3 批量与攒批

```markdown
## 3. 批量与攒批

| 常量 | 值 | 用途 |
|------|-----|------|
| `PipelineBatchSize` | 100 | Redis Pipeline 一批最多发送的命令数 |
| `KafkaBatchSize` | 200 | Kafka Consumer 攒批阈值 |
| `KafkaBatchTimeout` | `5 * time.Second` | Kafka Consumer 攒批超时 |
| `FlushBatchSize` | 500 | flush_state_to_db 单次处理 entry 数 |
```

### 3.4 限流阈值（默认值）

```markdown
## 4. 限流阈值（默认值，可被 {ns}:info / {ns4}:info 覆盖）

| 常量 | 值 | 用途 |
|------|-----|------|
| `DefaultEntityQPSLimit` | 100 | 实体级 QPS 默认 |
| `DefaultEntityDailyLimit` | 100000 | 实体级日调用量默认 |
| `DefaultActionDailyLimit` | 1000000 | 动作级日调用量默认 |
```

### 3.5 分布式锁 TTL

```markdown
## 5. 分布式锁 TTL

| 常量 | 值 | 用途 |
|------|-----|------|
| `LockCacheIntegrityCheck` | `5 * time.Minute` | 缓存巡检任务锁 |
| `Lock{PeriodicAggregateTask}` | `30 * time.Minute` | `{periodic-aggregate-task}` 任务锁（如多进程触发） |
```

### 3.6 ID 生成规则

```markdown
## 6. ID 生成规则

### 6.1 各业务对象的 ID 格式（占位符示意）

| 对象 | 格式 | 示例 |
|------|------|------|
| `{entity-A-id}` | `{prefix-A}{yyyyMMdd}{seqN位}` | `{prefix-A}{date-day}{seq}` |
| `{entity-B-id}` | `{prefix-B}{yyyyMMddHHmmss}{seqN位}` | `{prefix-B}{date-second}{seq}` |
| `{entity-C-id}` | `{prefix-C}{yyyyMMddHHmmss}{randN位}` | `{prefix-C}{date-second}{rand-hex}` |
| `{entity-D-id}` | `{prefix-D}{yyyyMMddHHmmss}{seqN位}` | `{prefix-D}{date-second}{seq}` |

> 真实业务的 ID 命名（按业务实体替换 `{entity-id}`）见末尾 §11 参考示例。

### 6.2 占位符规则

- {seqN位} = 进程内 atomic 自增计数 % 10^N，零填充
- {randN位} = crypto/rand 生成的 N 位 hex
- 时间戳精度：按业务需要选 day / minute / second
```

### 3.7 Redis Key 模板

```markdown
## 7. Redis Key 模板

集中列出（详细字段映射见 cache_design.md §2.X）：

\`\`\`go
const (
    KeyTplEntityInfo        = "{ns}:info:%s"           // {entityId}
    KeyTplEntityIPs      = "{ns}:ips:%s"            // {entityId}
    KeyTplAccountInfo    = "{ns2}:info:%s"       // {entity-A-id}
    KeyTplPerm           = "{ns3}:%s:%s"            // {entityId}:{actionId}
    KeyTplProductInfo    = "{ns4}:info:%s"       // {actionId}
    KeyTplApiPath        = "{ns5}:%s"            // {urlPath}
    KeyTplDiscount       = "{ns6}:%s:%s"        // {entityId}:{actionId}
    KeyTplState1         = "{state-1}:%s:%s"           // {entityId}:{actionId}
    KeyTplState2         = "{state-2}:%s"            // {entity-A-id}
    KeyTplState3        = "{state-3}:%s:%s"         // {entityId}:{actionId}
    KeyTplDailyStats     = "{stats-key}:%s:%s:%s"  // {entityId}:{actionId}:{date}
    KeyTplRLQPS          = "{rl}:qps:%s:%s"          // {entityId}:{actionId}
    KeyTplRLDaily        = "{rl}:daily:%s:%s:%s"     // {entityId}:{actionId}:{date}
    KeyTplRLDailyTotal   = "{rl}:daily:total:%s:%s"  // {entityId}:{date}
    KeyTplLock           = "lock:%s"               // {taskName}
    KeyTpl{OneTimeAuth}     = "{ns2}:{otp-namespace}:%s"// {actor-id}
)

const (
    SetState1Dirty       = "{state-1}:dirty"
    SetState2Dirty     = "{state-2}:dirty"
    SetDailyStatsDirty  = "{stats-key}:dirty"
)
\`\`\`
```

---

## 4. §10 components/constants.go 目标代码（强制）

> ⚠️ 下方代码块的常量名 / 取值 / 命名分组方式均为示意，不是规范本体。规范本体的强制要求是：**docs/architecture/constants.md §10 必须以一段完整的 Go 文件内容呈现，与 components/constants.go 1:1 对应**。具体常量名请按业务命名规范定义。

```markdown
## 10. components/constants.go 目标代码

\`\`\`go
package components

import "time"

// 时间窗口
const (
    SignatureWindow = 300 * time.Second
    BatchVerifyMax  = 100

    VerifyCodeTTL      = 5 * time.Minute
    VerifyCodeLength   = 6
    UserSessionTTL = 2 * time.Hour

    LocalCacheActiveTTL = 15 * time.Minute
    LocalCacheMaxTTL    = 30 * time.Minute
    DailyStatsTTL       = 48 * time.Hour
)

// 加密参数
const (
    BcryptCost      = 10
    EntityIDPrefix    = "ak_"
    EntityIDLength    = 19
    EntitySecretLength = 32
)

// 批量与攒批
const (
    PipelineBatchSize  = 100
    KafkaBatchSize     = 200
    KafkaBatchTimeout  = 5 * time.Second
    FlushBatchSize     = 500
)

// 限流默认
const (
    DefaultEntityQPSLimit       = 100
    DefaultEntityDailyLimit     = 100000
    DefaultActionDailyLimit = 1000000
)

// 分布式锁 TTL
const (
    LockCacheIntegrityCheck    = 5 * time.Minute
    Lock{PeriodicAggregateTask} = 30 * time.Minute
)

// Redis Key 模板
const (
    KeyTplEntityInfo  = "{ns}:info:%s"
    // ... 完整列表
)
\`\`\`

> 这个 §10 是 AI 直接生成 components/constants.go 文件的依据。**Go 文件 = §10 内容 1:1**。
```

---

## 5. 与其他文档的关系（重复就改）

| 概念 | 真相源 | 其他位置 |
|------|--------|----------|
| 业务校验时间窗口数值 | constants.md §1.1 | {核心校验链}_flow.md / api/audience_a_interfaces.md / middleware.md **引用** |
| Pipeline 批大小 | constants.md §3 | service/{auth-module}_service.md **引用** |
| Redis Key 模板 | constants.md §7 | cache_design.md **引用**（cache_design 给字段映射） |
| ID 格式 | constants.md §6 | service/{module}_service.md **引用**（service 文档给生成函数） |

**禁止**：在多处分别写 "300s" 这个数字。所有引用统一指向 `constants.md` 或代码常量。

---

## 6. AI 行为约束

> 所有 Skill 在生成业务代码时，**禁止**直接使用魔法数字，必须使用常量名（即使该常量当前还未在 constants.go 中定义，也必须先补常量再用）。

如发现某个数值在 md 中没有对应常量：

1. 停止生成代码
2. 提醒用户：xxx 这个数字应该作为常量
3. 由用户裁决：在 constants.md / constants.go 中补 + 同步引用

---

## 7. 维护

| 触发 | 动作 |
|------|------|
| 新增魔法数字 | §X 表格新增 + §10 Go 文件 + 同步引用方 |
| 调整常量值 | 评估对运行时的影响 + 更新所有引用方文档（数值如出现在多处描述里需要全部改） |
| 删除常量 | §X / §10 + 检查无引用 |
| 重构常量名 | §10 + 全量代码搜索替换 |

---

## 8. 不允许的写法

```markdown
❌ "时间窗口约 300 秒"          ← 模糊
❌ "bcrypt cost 用一个合适的值" ← 没有数值
❌ "Pipeline 一次发若干个命令"  ← 缺数值
❌ 文档写 300s，代码写 200s     ← 不一致（应以本文档为准修代码）
```

---

## 11. 参考示例（仅示意，落地按业务替换）

> ⚠️ 以下为示意，规范本体（§1-§8）已用占位符表达；本节给出具体业务（账号 / 订单 / 日志 / 充值等）的 ID 命名示例，便于读者建立对应直觉。落地工程时按业务语义替换占位符，不得直接照搬本节具体业务名作为规范。
>
> 示例服从 [`PRINCIPLES.md §12 占位符规则`](../PRINCIPLES.md#12-占位符规则强制--双轨结构) 的"参考示例段豁免"。

### 11.1 §6.1 ID 格式（示例：账号 / 订单 / 日志 / 充值业务）

| 对象 | 格式 | 示例 |
|------|------|------|
| userId | `ACC{yyyyMMdd}{seq3位}` | `ACC20260407001` |
| orderId | `ORD{yyyyMMddHHmmss}{seq3位}` | `ORD20260407103015001` |
| logId | `LOG{yyyyMMddHHmmss}{rand5位}` | `LOG20260407103015A1B2C` |
| rechargeId | `RCG{yyyyMMddHHmmss}{seq3位}` | `RCG20260407103015001` |

> 时间戳精度：userId 用日，其他用秒。

### 11.2 占位符 → 业务名 对照表（示例）

| 占位符 | 本节示例值 |
| --- | --- |
| `{entity-A-id}` / `{prefix-A}` | userId / `ACC` |
| `{entity-B-id}` / `{prefix-B}` | orderId / `ORD` |
| `{entity-C-id}` / `{prefix-C}` | logId / `LOG` |
| `{entity-D-id}` / `{prefix-D}` | rechargeId / `RCG` |
| `{date-day}` / `{date-second}` | `20260407` / `20260407103015` |
| `{seq}` / `{rand-hex}` | `001` / `A1B2C` |



---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
