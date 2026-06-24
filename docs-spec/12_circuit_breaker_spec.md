# 12 - 熔断 / 限流 / 降级文档规范

> 规定 `docs/circuit_breaker/circuit_breaker_design.md` 的内容结构。

---

## 1. 定位

故障保护文档要让 AI 能回答：

1. 哪些资源被熔断器保护（Redis / MySQL / 外部 API / Kafka）
2. 各资源的熔断参数差异化原因
3. 接入方式（自动 vs 手动）
4. 熔断触发后的降级策略
5. 与 helpers 的集成（initialize 顺序）

---

## 2. 顶层结构

```markdown
# 熔断器配置设计

## 1. 算法选型
   1.1 算法名称（如 Google SRE 自适应算法）
   1.2 与传统二态开关的差异
   1.3 关键参数（K、窗口期、采样数）

## 2. 资源差异化参数（强制：所有受保护资源的参数表）

## 3. 接入方式
   3.1 Redis（rediscache.Client 内置）
   3.2 MySQL（GORM Plugin）
   3.3 外部 API（cb.Do() 手动包裹）
   3.4 Kafka（如有保护）

## 4. helpers 集成顺序
   4.1 InitCircuitBreaker 必须最先
   4.2 后续 InitRedis / InitMysql 注册到 CB

## 5. 降级策略
   5.1 按依赖性质的降级三类法（只读 / 关键 / best-effort）
   5.2 熔断时的回退响应（按资源，与 5.1 分类对齐）
   5.3 ErrNotAllowed 的判定与处理

## 6. 监控与告警
   6.1 熔断状态指标
   6.2 错误率告警阈值
   6.3 实时降级率监控

## 7. 配置文件位置
```

---

## 3. §1 算法选型（必填）

```markdown
## 1. 算法选型

采用 **Google SRE 自适应熔断算法**（非传统二态开关），按实时错误率概率性丢弃请求。

**核心公式**：

\`\`\`
丢弃概率 = max(0, (requests - K * accepts) / (requests + 1))
\`\`\`

- requests：滑动窗口内总请求数
- accepts：滑动窗口内成功数
- K：敏感度系数，越小越激进（典型 1.0 - 1.5）

**与传统二态开关的差异**：

| 维度 | 二态开关 | 自适应（SRE） |
|------|----------|--------------|
| 开关切换 | 全开 / 全关 | 概率性丢弃 |
| 半开探测 | 需要 | 不需要（错误率自然下降时丢弃率降至 0） |
| 突发恢复 | 一次放过大量请求容易再次崩溃 | 平滑过渡 |
| 参数调优 | error_threshold + recovery_time | K + 窗口期 |
```

> **客户端自适应限流（client-side throttling）语义 + K 选型权衡**（节流启动阈值 = 实际失败率超过 `1−1/K`；作用于**依赖侧**自我节流 vs [`design-spec/07 §3.4`](../design-spec/07_tail_latency_design.md) 作用于**入口侧**的 load shedding 的边界）见 [`design-spec/06 §9.1`](../design-spec/06_resilience_design.md)。本节只登记每个资源**选定**的 K / 窗口 / 采样数，不复制选型方法论（双源真相）。

---

## 4. §2 资源差异化参数（强制）

```markdown
## 2. 资源差异化参数

| 资源 | 实例数 | K 值 | 滑动窗口 | 最小采样数 | 接入方式 | 配置位置 |
|------|--------|------|---------|-----------|---------|---------|
| Redis | 1 | 1.04 | 5s | 50 | rediscache.Client 内置 | conf/app/app.yaml |
| MySQL core | 1 | 1.09 | 10s | 100 | GORM Plugin | conf/app/app.yaml |
| MySQL trade | 1 | 1.09 | 10s | 100 | GORM Plugin | conf/app/app.yaml |
| MySQL log | 1 | 1.09 | 10s | 100 | GORM Plugin | conf/app/app.yaml |
| MySQL {example_log} | 1 | 1.09 | 10s | 100 | GORM Plugin | conf/app/app.yaml |
| 外部 API X | 1 | 1.20 | 30s | 30 | cb.Do() 手动 | conf/app/app.yaml |
```

**参数选择原则**：

- **K 值越小越激进**：内部强依赖（Redis）K 偏小，外部弱依赖 K 偏大
- **窗口期越短反应越快**：高频依赖（Redis）短窗口，低频依赖（外部 API）长窗口
- **最小采样数避免抖动**：QPS 越高的依赖采样数越大

---

## 5. §3 接入方式（强制：每个资源都给出代码模板）

### 5.1 Redis（自动接入）

```markdown
### 3.1 Redis

`rediscache.Client` 内置 CB，业务代码无需手动调用：

\`\`\`go
val, err := helpers.{Project}CacheClient.HGet(ctx, "{ns}:info:xxx", "secret_key")
// 熔断时 err 是 circuitbreaker.ErrNotAllowed
if errors.Is(err, circuitbreaker.ErrNotAllowed) {
    // 降级处理（如返回本地缓存的数据）
}
\`\`\`
```

### 5.2 MySQL（GORM Plugin 自动接入）

```markdown
### 3.2 MySQL

通过 `helpers/mysql_cb_plugin.go` 注册 GORM Plugin，所有 GORM 调用自动经熔断器：

\`\`\`go
db.Where(...).First(&user)
// 熔断时 err 是 circuitbreaker.ErrNotAllowed
\`\`\`

业务代码无需手动调用。
```

### 5.3 外部 API（手动接入）

```markdown
### 3.3 外部 API

业务代码用 `cb.Do(func)` 包裹：

\`\`\`go
cb := helpers.CircuitBreakerInstance("demo")
err := cb.Do(func() error {
    result, httpErr := conf.API.Demo.HttpGet(ctx, "/path", options)
    if httpErr != nil { return httpErr }
    // ...
    return nil
})
if errors.Is(err, circuitbreaker.ErrNotAllowed) {
    // 返回降级数据
    return getFallbackResult()
}
\`\`\`
```

---

## 6. §4 helpers 集成顺序（强制）

```markdown
## 4. helpers 集成顺序

\`\`\`
helpers.PreInit()
helpers.InitResource()
  ├── InitCircuitBreaker()       ← 必须最先
  ├── InitRedis()                ← 内部注册到 CB registry
  ├── InitMysql()                ← Plugin 注册到 CB registry
  ├── InitGPool()
  ├── InitGCache()
  └── InitKafkaProducer()
\`\`\`

**InitCircuitBreaker 必须最先**：后续资源初始化时会向 CB registry 注册各自的实例。如果顺序错了，注册时找不到 registry 会 panic。
```

---

## 7. §5 降级策略（强制）

```markdown
## 5. 降级策略

### 5.1 按依赖性质的降级三类法（强制：每个受保护依赖先归类，再定回退）

降级回退的方向由**依赖在业务中的性质**决定，而非由它是哪种基础设施决定。每个受保护依赖必须先归入下表三类之一，回退行为随分类确定（真相源 / 选型方法论见 [`design-spec/06 §3.1 / §5`](../design-spec/06_resilience_design.md)）：

| 依赖性质 | 判定标准 | 降级方向 | 落地形态（示意） |
|---------|---------|---------|----------------|
| **只读依赖** | 拿不到不影响主流程正确性，缺值可用空 / 默认值替代 | 返空 / 默认值 | 返回空集合 / 零值 / 兜底快照 |
| **关键依赖** | 拿不到则结果不可信，放行会污染数据 / 误导调用方 | 拒绝（返错，不放行） | 返业务错误码，请求整体失败 |
| **best-effort 依赖** | 旁路副作用，失败不阻塞主流程 | 静默跳过 | 记 WARN + 继续；可暂存补偿 |

> **归类即决策**：同一基础设施（如外部 API）在不同业务里可能落不同类——`{只读查询类}` 调用是只读依赖（返空），`{核心校验类}` 调用是关键依赖（拒绝），`{旁路上报类}` 调用是 best-effort（静默）。先归类再查 §5.2 回退矩阵，不可跳过归类直接照搬资源默认。
>
> **降级（本节）vs load shedding（07）的分界**：本节降级应对的是"**依赖故障**了返什么"（按依赖性质）；当问题是"**系统自身过载**"（队列积压 / CPU 满 / 自适应 limit 触顶）时，治理手段是**优先级 / 截止时间感知 load shedding**（过载时丢谁先），决策见 [`design-spec/07 §3.4`](../design-spec/07_tail_latency_design.md)、档位登记 [`22 §6.4`](22_performance_contract_spec.md)。二者正交叠加。

### 5.2 熔断时的回退响应（按资源，与 §5.1 分类对齐）

| 资源 | 依赖性质（§5.1） | 熔断时的回退 |
|------|----------------|-------------|
| {只读依赖资源} | 只读依赖 | 返空 / 默认值 / 本地缓存兜底（TTL 内仍有效）；兜底也 miss 时返回业务错误码 |
| {关键依赖资源} | 关键依赖 | 直接返回错误，不做兜底（数据一致性优先） |
| {best-effort 资源} | best-effort 依赖 | tlog.WarnLogger + 内存 buffer 暂存，定时重发 / 静默跳过 |

> 每个受保护资源在本表的"依赖性质"列必须显式登记其 §5.1 分类；新增受保护资源时同步补本列。

### 5.3 ErrNotAllowed 的判定与处理

\`\`\`go
import "github.com/your-org/circuitbreaker"

if errors.Is(err, circuitbreaker.ErrNotAllowed) {
    // 熔断触发，进入降级
}
\`\`\`

业务代码必须显式处理 `ErrNotAllowed`，否则它会被当作普通错误返回 5xx。
```

---

## 8. §6 监控与告警

```markdown
## 6. 监控与告警

### 6.1 熔断状态指标
- `cb_dropped_total{resource}` —— 因熔断丢弃的请求计数
- `cb_dropped_rate{resource}` —— 实时丢弃率（0-1）
- `cb_window_error_rate{resource}` —— 滑动窗口错误率

### 6.2 告警阈值
- 丢弃率 > 5% 持续 1min → 告警
- 错误率 > 30% 持续 30s → 严重告警
- 任意资源连续 5min 处于熔断 → 触发故障预案

### 6.3 实时降级率监控
- Grafana 面板路径：`{Path to dashboard}`
- 各资源面板独立
```

---

## 9. §7 配置文件位置

```markdown
## 7. 配置文件位置

熔断器参数在 `conf/app/app.yaml` 中定义（**业务配置，随代码发布**），通过 `helpers.CircuitBreakerInstance(name)` 获取实例。

\`\`\`yaml
# conf/app/app.yaml
circuitBreaker:
  redis:
    k: 1.04
    window: 5s
    minSamples: 50
  mysql_core:
    k: 1.09
    window: 10s
    minSamples: 100
  # ...
\`\`\`
```

---

## 10. 限流补充（如有）

如果限流也归入本文档（应用层限流，非中间件层），单独成节：

```markdown
## 8. 应用层限流（如适用）

### 8.1 与 cache_design.md §2.3 的关系
- 应用层限流（QPS / 日调用量）见 cache_design.md
- 本节仅描述熔断保护的限流（如令牌桶）
```

通常 本工程 类工程的限流在 cache_design.md 中描述（基于 Redis INCR），熔断器文档不重复。

---

## 11. 与其他文档的关系

| 内容 | circuit_breaker_design.md | infrastructure.md | helpers_api.md |
|------|---------------------------|-------------------|----------------|
| 算法选型 | ✅ 完整 | 引用 | 不写 |
| 参数差异化 | ✅ 完整 | 不写 | 不写 |
| 接入方式 | ✅ 完整 | 引用 | ✅ CircuitBreakerInstance 函数签名 |
| 资源初始化顺序 | 简略 | ✅ 完整 | 不写 |

---

## 12. 维护

| 触发 | 动作 |
|------|------|
| 新增受保护资源 | §2 参数表 + §3 接入方式 + §5.1 归入降级三类 + §5.2 登记"依赖性质"列 + 同步 helpers / 配置 |
| 调整 K 值 / 窗口 | §2 + 评估对告警阈值的影响 |
| 改变降级策略 | §5.1 重新归类 + §5.2 回退矩阵 + 同步 service 层文档（业务代码改动） |
| 启用/禁用熔断 | BUILD_STATUS 标记 + §2 表格更新 |

---

## 13. 与 22_performance_contract 的边界划分

12 与 22 在"背压 / 降级"上有重叠，本节明确切分，避免双源真相：

| 内容 | 12_circuit_breaker.md（本文档） | 22_performance_contract.md |
| --- | --- | --- |
| 资源差异化熔断参数（K / 窗口 / 采样数） | ✅ 真相源 | 不写 |
| GORM Plugin / `cb.Do` 接入方式 | ✅ 真相源 | 不写 |
| 熔断时回退响应矩阵 | ✅ 真相源 | 不写 |
| 队列 / channel 容量上限 | 不写 | ✅ 真相源（§6.1） |
| 上游超时 vs 下游处理时间协调 | 不写 | ✅ 真相源（§6.2） |
| 优雅降级层级（L1 资源熔断 / L2 服务降级 / L3 关闭非核心） | L1 ✅；L2/L3 引用 22 §6.3 | L2/L3 ✅ 真相源；L1 引用本文档 |

**约束**：本文档中"L2 服务降级"/"L3 关闭非核心功能"等业务降级层级的具体行为**不写**——必须显式写"详见 22_performance_contract §6.3"，由 22 承载真相。本文档仅描述 L1 资源熔断的算法 / 参数 / 接入。


---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
