# 业务常量

> 规范来源：`aiweave/docs-spec/16_constants_spec.md`

## 1. 时间窗口

### 1.1 业务校验 / 签名相关
| 常量 | 值 | 用途 | 出处 |
|------|-----|------|------|
| ... | ... | ... | ... |

### 1.2 验证码 / 会话
### 1.3 缓存 TTL

## 2. 加密参数

| 常量 | 值 | 用途 |
|------|-----|------|
| `BcryptCost` | 10 | bcrypt 哈希成本 |
| `EntityIDPrefix` | `"ak_"` | EntityID 前缀 |
| ... | ... | ... |

## 3. 批量与攒批

| 常量 | 值 | 用途 |
|------|-----|------|
| `PipelineBatchSize` | 100 | Redis Pipeline 一批最多命令数 |
| `KafkaBatchSize` | 200 | Kafka Consumer 攒批阈值 |
| `KafkaBatchTimeout` | 5*time.Second | Kafka Consumer 攒批超时 |
| `FlushBatchSize` | 500 | flush 任务单次处理数 |

## 4. 限流阈值（默认值）

| 常量 | 值 | 用途 |
|------|-----|------|
| ... | ... | ... |

## 5. 分布式锁 TTL

| 常量 | 值 | 用途 |
|------|-----|------|
| ... | ... | ... |

## 6. ID 生成规则

### 6.1 各业务对象的 ID 格式
| 对象 | 格式 | 示例 |
|------|------|------|
| ... | ... | ... |

### 6.2 占位符规则

## 7. Redis Key 模板

```go
const (
    KeyTplEntityInfo  = "{ns}:info:%s"
    // ... 完整列表
)

const (
    SetXxxDirty = "xxx:dirty"
    // ... 完整列表
)
```

## 8. 业务对象的状态值（如未在 status_codes.md 集中定义）

## 9. 配置中默认值的来源

## 10. components/constants.go 目标代码

```go
package components

import "time"

// 时间窗口
const (
    // ...
)

// 加密参数
const (
    BcryptCost = 10
    EntityIDPrefix = "ak_"
    // ...
)

// 批量与攒批
const (
    PipelineBatchSize = 100
    KafkaBatchSize    = 200
    KafkaBatchTimeout = 5 * time.Second
    FlushBatchSize    = 500
)

// Redis Key 模板
const (
    KeyTplEntityInfo = "{ns}:info:%s"
    // ...
)
```


---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
