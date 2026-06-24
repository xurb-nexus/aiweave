# 性能合约与热路径

> 版本：1.0 | 日期：{YYYY-MM-DD}
>
> 规范来源：[`aiweave/docs-spec/22_performance_contract_spec.md`](../../../docs-spec/22_performance_contract_spec.md)
>
> 本文档是 **AI 在生成涉及热路径代码、性能敏感操作、背压策略之前必须读的文档**。

---

## 1. 全局性能目标（SLA）

| 指标 | 目标值 | 测量方式 | 降级阈值 |
|------|--------|---------|---------|
| P99 latency | < `{P99-ms}` ms | APM span / Prometheus histogram | > `{P99-degrade-ms}` ms 告警 |
| P999 latency | < `{P999-ms}` ms | APM span / Prometheus histogram | > `{P999-degrade-ms}` ms 告警 |
| QPS | > `{QPS-target}` | Prometheus counter rate | < `{QPS-floor}` 告警 |
| 内存 | < `{Memory-target}` MB RSS | `/proc/meminfo` | > `{Memory-degrade}` MB 告警 |
| CPU | < `{CPU-target}` core | container stats | > `{CPU-degrade}` core 告警 |

---

## 2. 热路径标注（AI 必读）

### 2.1 热路径定义

满足以下任一条件的代码路径：

- QPS > `{HOT-QPS-threshold}`
- P99 影响全局 SLA（P99 > 总 SLA 的 50%）
- 占总 CPU > `{HOT-CPU-threshold}`%

### 2.2 热路径清单

| 路径名 | 入口 | QPS | 允许操作 | 禁止操作 |
|-------|------|-----|---------|---------|
| `{核心校验链}` | `/{prefix}/internal/{action-A}` | `{QPS-A}` | Redis GET/HGET、本地缓存读 | MySQL 查询、反射、fmt |
| `{状态上报链}` | `/{prefix}/internal/{action-B}` | `{QPS-B}` | Redis INCR/HSET | 日志（除异常）、大对象分配 |

### 2.3 热路径标注

热路径方法的 service docs §4.N.3 处理步骤伪码中必须含 `[HOT-PATH]` 标记。

---

## 3. 内存分配预算

### 3.1 每请求允许的堆分配上限

| 接口类别 | 堆分配上限（kB / 请求） | 备注 |
|---------|----------------------|------|
| 热路径 | < `{Alloc-hot-kB}` | — |
| 普通业务接口 | < `{Alloc-normal-kB}` | — |
| 后台批处理任务 | < `{Alloc-batch-kB}` | — |

### 3.2 对象池（sync.Pool）使用清单

| 池名 | 类型 | 用途 | 容量提示 |
|------|------|------|---------|
| `{pool-A}` | `*{Type-A}` | 在 `{module}.{Method}` 热路径中复用 | — |

### 3.3 预分配策略

- slice / map 初始容量必须显式（`make([]T, 0, N)`）
- 已知大小的 slice 必须预分配

### 3.4 Go 运行时旋钮（GC / 调度 / 内存上限）

| 旋钮 | 设定 | 理由 |
|------|------|------|
| `GOMAXPROCS` | = 容器 CPU limit（用 automaxprocs 自动对齐；禁默认=宿主核数） | 容器被 CPU 限流时 P 数远超配额 → 调度争用 + 节流抖动抬高 P99 |
| `GOMEMLIMIT` | = 容器 mem limit × `{0.9}`（软上限，留头部给非堆） | 触顶时 GC 提前更激进，防 OOMKill；与 §3.1 单请求堆预算叠加（预算控单请求，GOMEMLIMIT 控进程总量） |
| `GOGC` | 默认 100；高分配率热路径可调高（如 200）换更少 GC 周期 | GOGC 控频率，GOMEMLIMIT 兜上限，二者必须配对 |

三旋钮的值随容器配额走，必须与 [`deployment.md §7`](deployment.md) 的 CPU / mem limit 对齐（硬规则 `R-DEPLOY-GOMAXPROCS` / `R-RUNTIME-MEMLIMIT`）。

---

## 4. 数据访问性能约束

### 4.1 Redis 命令延迟预算

| 操作 | 单次延迟预算 | 备注 |
|------|------------|------|
| GET / HGET / SET / HSET | < `{Redis-simple-ms}` ms | — |
| HGETALL / SMEMBERS / SUNION | 不允许在热路径中调用 | 用 HGET 字段集或 SPOP |
| Pipeline（≥ 3 命令） | 优于多次 GET | — |

### 4.2 MySQL 查询复杂度限制

- 禁止全表扫描
- JOIN 限制：≤ `{JOIN-limit}`
- 单查询返回行数 ≤ `{Row-limit}`

### 4.3 缓存命中率目标

| Key 类别 | 命中率目标 |
|---------|----------|
| `{ns}:info:*` | > 95% |
| `{ns}:profile:*` | > 80% |

### 4.4 连接池 / 超时基线指针（避免双源真相）

- 各资源连接池与超时取值（`{maxOpen}` / `{conn-timeout}` / `{read-timeout}` / `{write-timeout}` / `{conn-max-lifetime}` / 重试）登记在 [`infrastructure.md §6`](infrastructure.md)（嵌入式库读写双池见 [`infrastructure.md §7`](infrastructure.md)）。本文只引用不复制。
- 自适应熔断参数分级（采样数 / `percent` / 窗口，按组件特性差异化）登记在 [`circuit_breaker_design.md §2`](../circuit_breaker/circuit_breaker_design.md)。本文只引用不复制。

---

## 5. 并发性能与锁等待预算

| 维度 | 预算 | 引用 |
|------|------|------|
| 全局锁持有时间 | < `{Lock-hold-ms}` ms | [`concurrency_safety.md §3`](concurrency_safety.md) |
| Goroutine 数量峰值 | < `{Goroutine-peak}` | [`concurrency_safety.md §1.1`](concurrency_safety.md) |
| Channel 排队时长 | < `{Channel-wait-ms}` ms | [`concurrency_safety.md §4.1`](concurrency_safety.md) |

---

## 6. 背压与容量保护

### 6.1 各队列 / channel 的容量上限

| 队列 / channel | 容量 | 满时策略 | 引用 |
|--------------|------|---------|------|
| `{batch-ch}` | `{N}` | 丢弃尾部 + 计数告警 | [`concurrency_safety.md §4.1`](concurrency_safety.md) |

### 6.2 上游超时 vs 下游处理时间协调

| 链路 | 上游超时 | 本服务处理预算 | 下游超时 |
|------|---------|---------------|---------|
| {链路-A} | `{T-up-ms}` ms | `{T-self-ms}` ms | `{T-down-ms}` ms |

**关系约束**：`{T-self-ms} + {T-down-ms} < {T-up-ms} * 0.8`。

### 6.3 优雅降级层级

| 层级 | 触发条件 | 行为 |
|------|---------|------|
| L1 资源熔断 | 单资源连续失败 K 次 | 详见 [`circuit_breaker_design.md §2`](../circuit_breaker/circuit_breaker_design.md) |
| L2 服务降级 | L1 熔断扩散到多个核心资源 | 关闭非核心读路径 |
| L3 关闭非核心功能 | L2 仍不缓解 | 关闭统计 / 报表 / 异步落库 |

L1 的熔断器参数（K / 窗口 / 采样数）真相源在 `circuit_breaker_design.md §2`。

### 6.4 自适应过载与尾延迟治理

> 决策真相源在 [`design-spec/07`](../../../design-spec/07_tail_latency_design.md)（尾延迟三源 / 对冲 / 自适应限制 / load shedding / 重试预算的选型与决策树）；本节仅承载"参数与档位"的表示槽位，不复制决策树。

| 维度 | 登记内容 | 引用 |
|------|---------|------|
| 自适应并发限制 | 算法（AIMD / 梯度式）+ min-RTT 窗口 + limit 上下界 | design-spec/07 §3.3 |
| 对冲 / 备份请求 | 触发分位（tie-delay 如 P95）+ 对冲量上限（如 ≤5%）+ 仅幂等白名单 | [`cross_service_contract.md §3`](cross_service_contract.md) |
| 重试预算 | 重试占比上限（如 10%）+ backoff 基数 + jitter 比例 | [`cross_service_contract.md §3`](cross_service_contract.md) |
| 优先级 / 截止 load shedding | 优先级分层（interactive/batch/bestEffort）+ 饱和信号（队列/CPU/limit 触顶）+ 丢弃顺序 | 与 §6.3 降级联动 |

**关系约束**：对冲 / 重试在下游饱和时**禁止启用**（会放大过载）——饱和判定与 §6.3 的 L1/L2/L3 降级、熔断联动；对冲量与重试量必须计入 §6.1 的下游并发 / 队列预算。

---

## 7. 性能回归测试

### 7.1 基准测试清单

| 基准测试 | 方法 | 目标 P99 |
|---------|------|---------|
| `Benchmark{Method-A}` | `{Module}.{Method-A}` | `{P99-A-ns}` ns/op |

### 7.2 P99 回归检测规则

- P99 偏差 > 20% → CI 标红
- allocs/op 增长 > 10% → CI 标红

framework API 见 [`docs/testing/testing_design.md §4.9`](../testing/testing_design.md)。

### 7.4 生产持续 profiling

§7.1-§7.2 是**离线**回归；本节补**在线**观测——没有它，"IO 是第一性瓶颈"在生产只是假设而非证据。

| 维度 | 要求 |
|------|------|
| pprof 端点 | 暴露 `/debug/pprof/*`（heap / allocs / profile(CPU) / goroutine / mutex / block），仅内网或鉴权后可达，**禁公网裸暴露** |
| 采集方式 | 持续采样上报（如 Pyroscope / Parca）或按需抓取；热路径 P99 回归时优先看 alloc_space 火焰图定位分配源 |
| 运行时指标 | `go_gc_duration_seconds` / `go_memstats_*` / `go_goroutines` 接入 [`observability.md §3.3`](observability.md) |
| 告警联动 | GC pause P99、分配速率、goroutine 数突增 → 告警阈值登记 [`observability.md §5.2`](observability.md) |

---

## 8. 维护流程

### 8.1 B1 反向同步规则

| 代码迹象（git diff） | 反向同步动作 |
| --- | --- |
| 热路径文件新增 `reflect.*` / `fmt.Sprintf` | 拒绝合并 |
| 新增 `sync.Pool` / 显式预分配 | §3.2 / §3.3 登记 |
| 新增 channel 容量字段（N 与 §6.1 不一致） | §6.1 满时策略校对 |
| 新增 `*_bench_test.go` 文件 | §7.1 基准测试清单同步 |
| 修改 SLA 数值 | §1 全局性能目标表同步 |

### 8.2 与 BUILD_STATUS §11 约束清单状态轨道的关系

每条热路径条目（§2 一行）/ 每个基准测试（§7.1 一行）对应 BUILD_STATUS.md §11 "性能合约约束"类目的"已设计 / 已启用"计数。


---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
