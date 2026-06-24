# 熔断器配置设计

> 规范来源：`aiweave/docs-spec/12_circuit_breaker_spec.md`

## 1. 算法选型

采用 **Google SRE 自适应熔断算法**，按实时错误率概率性丢弃请求。

```
丢弃概率 = max(0, (requests - K * accepts) / (requests + 1))
```

## 2. 资源差异化参数

| 资源 | 实例数 | K 值 | 滑动窗口 | 最小采样数 | 接入方式 | 配置位置 |
|------|--------|------|---------|-----------|---------|---------|
| Redis | 1 | 1.04 | 5s | 50 | rediscache.Client 内置 | conf/app/app.yaml |
| MySQL core | 1 | 1.09 | 10s | 100 | GORM Plugin | conf/app/app.yaml |
| 外部 API X | 1 | 1.20 | 30s | 30 | cb.Do() 手动 | conf/app/app.yaml |

## 3. 接入方式

### 3.1 Redis（自动接入）
### 3.2 MySQL（GORM Plugin）
### 3.3 外部 API（手动接入）

```go
cb := helpers.CircuitBreakerInstance("demo")
err := cb.Do(func() error {
    // 调用代码
    return nil
})
if errors.Is(err, circuitbreaker.ErrNotAllowed) {
    // 降级
}
```

## 4. helpers 集成顺序

```
helpers.PreInit()
helpers.InitResource()
  ├── InitCircuitBreaker()    ← 必须最先
  ├── InitRedis()
  ├── InitMysql()
  └── ...
```

## 5. 降级策略

### 5.1 按依赖性质的降级三类法

> 每个受保护依赖先归入下表三类之一，回退方向随分类确定。选型方法论 / 真相源见 [`aiweave/design-spec/06_resilience_design.md §3.1 / §5`](../../../design-spec/06_resilience_design.md)；表示规范见 `aiweave/docs-spec/12_circuit_breaker_spec.md §5`。

| 依赖性质 | 判定标准 | 降级方向 |
|---------|---------|---------|
| 只读依赖 | 拿不到不影响主流程正确性 | 返空 / 默认值 |
| 关键依赖 | 拿不到则结果不可信，放行会污染数据 | 拒绝（返业务错误码，不放行） |
| best-effort 依赖 | 旁路副作用，失败不阻塞主流程 | 静默跳过（记 WARN + 继续） |

### 5.2 熔断时的回退（按资源，登记依赖性质）

| 资源 | 依赖性质（§5.1） | 回退 |
|------|----------------|------|
| Redis | {只读 / 关键} | 本地缓存兜底 |
| MySQL | 关键依赖 | 直接返回错误 |
| 外部 API | {只读 / 关键 / best-effort} | 静态默认值 / 拒绝 / 静默跳过 |

### 5.3 ErrNotAllowed 处理

## 6. 监控与告警

### 6.1 指标
- `cb_dropped_total{resource}`
- `cb_dropped_rate{resource}`

### 6.2 告警阈值
- 丢弃率 > 5% 持续 1min → 告警

## 7. 配置文件位置

`conf/app/app.yaml`：

```yaml
circuitBreaker:
  redis:
    k: 1.04
    window: 5s
  mysql_core:
    k: 1.09
    window: 10s
```


---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
