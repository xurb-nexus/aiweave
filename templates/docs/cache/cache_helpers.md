# 缓存辅助函数设计

> 规范来源：`aiweave/docs-spec/06_cache_design_spec.md` §12

## 1. 三级缓存访问标准模板（L1 → L2 → 拒绝）

```go
// 标准模板：先查 L1，miss 时查 L2，再 miss 时返回 not found（不回源 MySQL）
func LoadXxx(ctx *gin.Context, key string) (*XxxCache, bool) {
    if v, ok := helpers.{Cache}.Get(key).(*XxxCache); ok {
        return v, true
    }

    fields, err := helpers.{Project}CacheClient.HGetAll(ctx, fmt.Sprintf("xxx:info:%s", key))
    if err != nil || len(fields) == 0 {
        return nil, false
    }

    v := &XxxCache{}
    parseFields(v, fields)
    helpers.{Cache}.Set(key, v)
    return v, true
}
```

## 2. gcache.BucketCache API 规范

| 方法 | 用途 |
|------|------|
| `Get(key string) interface{}` | 取缓存 |
| `Set(key string, value interface{})` | 写缓存 |
| `Delete(key string)` | 删缓存 |
| `Has(key string) bool` | 是否存在 |

## 3. 5 个辅助函数完整签名与伪代码

### 3.1 LoadEntityInfo
```go
func LoadEntityInfo(ctx *gin.Context, entityId string) (*EntityInfoCache, bool)
```

### 3.2 LoadUserInfo
### 3.3 LoadResourceInfo
### 3.4 LoadIPWhitelist
### 3.5 LoadPartnerState

## 4. 缓存 Struct 定义

```go
type EntityInfoCache struct {
    Status       int8
    ActorID      string
    EntitySecret    string
    DailyLimit   int64
    QpsLimit     int
    Rate float64
    CreatedAt    time.Time
}

type UserInfoCache struct { ... }
type ResourceInfoCache struct { ... }
type PartnerStateCache struct { ... }
```

## 5. 类型转换工具（Redis Hash → Go struct）

```go
func parseEntityInfo(fields map[string]string) *EntityInfoCache { ... }
```

## 6. 调用关系总览

```
service/{auth-module}/checker.go::checkXxx
  ↓
service/{auth-module}/cache_loaders.go::LoadXxx
  ↓
helpers.{LocalCache1}（L1）
  ↓
helpers.{Project}CacheClient（L2）
```

## 7. 缓存失效策略

| 场景 | 动作 |
|------|------|
| 数据变更（管理操作） | 主动 helpers.{LocalCache1}.Delete |
| TTL 到期 | 自动失效 |
| Redis 故障 | 本地缓存兜底 |

## 8. 批量自动分批登记

> 登记访问层 MGET / MSET 的自动分批阈值：单批超过 `{batch-threshold}` 个 key 时，访问层自动按 `{batch-threshold}` 切批走 Pipeline 合并（对调用方透明，调用方一次传全量 keys）。未命中 key 不进结果 map，调用方按 `_, ok` 分流。

| 方法 | 自动分批阈值 `{batch-threshold}` | 分批方式 | 未命中处理 |
|------|--------------------------------|---------|-----------|
| `MGet(ctx, keys)` | `{batch-threshold}` | 超阈值按 `{batch-threshold}` 切批 Pipeline 合并 | 未命中 key 不进结果 map |
| `MSet(ctx, items)` | `{batch-threshold}` | 超阈值按 `{batch-threshold}` 切批 Pipeline 合并 | — |

> 决策方法（为何超阈值才分批、Pipeline 合并的取舍）见 [design-spec/05_caching_design.md §3.3](../../../design-spec/05_caching_design.md)。


---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
