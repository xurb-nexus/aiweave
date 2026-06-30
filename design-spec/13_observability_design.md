# 13 - 可观测性与服务可视化设计决策规范

> 回答"面对一个需求 / 子系统，怎么决定**该观测什么、暴露哪些内部状态、怎么把 SLO 反推成指标 + 告警 + 面板**"。是 design-spec 的第 13 个决策透镜——补上可观测性这条**此前只有表示规范（[`docs-spec/23`](../docs-spec/23_observability_spec.md)）、没有决策方法论**的支柱。
>
> **核心命题**：前 12 视角一路开了**控制环**——并发的 Little 池（[`03`](03_concurrency_design.md)）、稳定性的熔断器（[`06`](06_resilience_design.md)）、尾延迟的自适应限制 / load shedding / 对冲 / 重试预算（[`07`](07_tail_latency_design.md)）、容量的 ρ≈0.8 膝盖（[`12`](12_capacity_scaling_design.md)）。但没有任何规范要求"把每个控制环的内部状态暴露成饱和度指标、并在面板上对着阈值 / 膝盖看"。**lens 开了控制环，观测侧却是黑箱——故障时看不见拐点。** 本视角补这座桥。
>
> **边界先声明**：Metric 注册 / cardinality / label 约束 / 告警规则表示（[`docs-spec/23`](../docs-spec/23_observability_spec.md)）、熔断面板路径 / 差异化参数（[`docs-spec/12`](../docs-spec/12_circuit_breaker_spec.md)）、SLA 数值 / 控制环参数表示（[`docs-spec/22`](../docs-spec/22_performance_contract_spec.md)）、结构化日志 / `trace_id` 串联（[`docs-spec/13`](../docs-spec/13_logging_spec.md)）、控制环本身的选型（[`03`](03_concurrency_design.md) / [`06`](06_resilience_design.md) / [`07`](07_tail_latency_design.md) / [`12`](12_capacity_scaling_design.md)）等**真相源在对应规范**，本篇只引用不复制。本篇新增决策资产：**请求面 RED / 资源面 USE / 控制面内部状态三面识别、控制环可观测性铁律、SLO 反推链、collector 形态选型、服务标准面板四行结构**。

---

## 1. 设计目标

- **该观测的可观测、该可视的成面板**：本视角优化"观测覆盖"——尤其 design-spec 各 lens 开的**控制环**（Little 池 [`03`](03_concurrency_design.md) / 熔断 [`06`](06_resilience_design.md) / 自适应限制·shed·对冲·重试 [`07`](07_tail_latency_design.md) / ρ 膝盖 [`12`](12_capacity_scaling_design.md)）不能是黑箱。
- **第一性约束（不可妥协）**：
  - **控制环必可观测**：凡前序 lens 开了控制环，其**饱和度 / 触发态**必须有指标；黑箱控制环 = 故障时看不见拐点。
  - **只观测服务级 + 控制面，不把业务数据当指标**（引 [`23 §2`](../docs-spec/23_observability_spec.md)）：业务事件走结构化日志（[`docs-spec/13`](../docs-spec/13_logging_spec.md)）。
  - **测分位不测均值**：关键路径延迟必走直方图，对齐尾延迟 lens（[`07`](07_tail_latency_design.md)）对 P99 / P999 的依赖。
  - **观测不得反噬热路径**：遥测构造（日志 / 指标 field）本身是热路径分配源，受 [`22 §3 / §11`](../docs-spec/22_performance_contract_spec.md) 的热路径分配约束（级别门控 / 采样）。**观测覆盖全 ∧ 采集开销省，二者同时成立。**

---

## 2. 输入信号识别

| 迹象 | 典型描述 | 落入本视角的哪一面 |
| --- | --- | --- |
| 开了控制环 | "我加了 worker 池 / 熔断器 / 自适应限制 / load shedding / 对冲" | **控制面**：必出饱和度 / 触发态指标（§3.1） |
| 对外接口 / RPC | "有对外接口，要看 QPS / 错误率 / 延迟" | **请求面 RED**（§3.1） |
| 有端到端 SLO | "有 P99 目标 / SLA" | **SLO 反推**指标 + 告警 + 面板（§3.2） |
| 要判断扩缩容 | "什么时候扩容 / 缩容" | **资源面 USE 饱和信号**（接 [`12 §3.4`](12_capacity_scaling_design.md)） |
| 要知道依赖健康 | "某下游挂没挂 / 降级没" | 熔断态 + 健康态指标 |
| 业务事件埋点 | "想统计某业务实体的量 / 分布" | **不落本视角**：走日志 + 离线（引 [`23 §2`](../docs-spec/23_observability_spec.md)，反模式） |

> 识别口诀：**"我开了一个会饱和 / 会触发的东西"→ 必须能在指标里看见它饱和 / 触发；"我有 SLO"→ 指标必是能算分位的直方图。**

---

## 3. 决策树（强制 / 本篇核心）

### 3.0 先分流：metric vs log（引 23 边界，不复制）

```
要观测的是什么？
├─ 服务级运行状况 / 控制面内部状态（可数、维度有界） ──► Metric（本视角 §3.1）
└─ 业务事件 / 高基数实体 / 需逐条追溯 ──► 结构化日志（docs-spec/13）+ 离线分析；trace_id 串联（23 §1.1）
```

### 3.1 观测什么：请求面 RED · 资源面 USE · 控制面内部状态（本篇核心）

```
一个子系统，观测哪几面？（三面正交，按形态命中）
│
├─ 请求面（对外接口 / RPC） ──► RED
│     ├─ Rate ───── 请求量计数（按路由模板 + 状态码维度）
│     ├─ Errors ─── 错误率（按状态码维度切）
│     └─ Duration ─ 延迟 **直方图**（桶按 SLO，P50/P90/P99；禁只用累计计数求均值）
│
├─ 资源面（池 / 连接 / 队列 / CPU / 内存） ──► USE
│     ├─ Utilization ─ 用了多少（在用数 / 容量）
│     ├─ Saturation ── 排了多少（运行数≈容量 且 空闲≈0 = 排队；队列深；ρ）
│     └─ Errors ────── 资源级错误（获取失败 / 超时 / 拒绝）
│
└─ 控制面（本支柱各 lens 开的控制环——黑箱化是头号反模式） ──► 暴露内部状态
      ├─ 熔断器（06 / docs-spec 12） ──► 窗口请求 / 接受数 + 拒绝率（0~1）
      ├─ worker 池（03 Little） ──────► 运行 / 空闲 / 容量 → 饱和度（IO 瓶颈信号）
      ├─ 自适应并发限制（07 §3.3） ───► 当前限额值 + 触顶计数
      ├─ load shedding（07 §3.4） ───► 丢弃计数（按优先级 / 原因切）
      ├─ 对冲（07 §3.2） ────────────► 对冲量（占比）+ 命中率（校 ≤X% 预算）
      └─ 重试预算（07 §3.5） ────────► 预算消耗率（校 ≤X% 护栏）
```

### 3.2 SLO 反推（每个 SLO 一条链）

```
SLO（如 P99 < {X}ms） ──► 指标（直方图分位） ──► 告警（分位 > 阈值，引 23 §5） ──► 面板（对着阈值 / 膝盖画，§7 落 observability_dashboard.md）
对应 07 §9.6 的 ρ≈0.8 膝盖：饱和度面板红线画在膝盖，不画在 100%。
```

### 3.3 collector 形态选型

```
这个指标怎么采？
├─ 即时态（当前运行数 / 限额 / 拒绝率） ──► 拉取式 collector：采集时现读实例状态，无后台定时写值
│     ↳ 去重：连接池等若框架已自动注册，禁重复注册（否则框架 panic）
│     ↳ 时序：collector 注册须在被观测实例初始化之后；仅 HTTP 服务路径暴露（cron 不挂端点）
├─ 累计态（请求量 / 错误数 / 耗时分布） ──► 计数器 / 直方图，在中间件或埋点处累加
└─ 双 Registry 分离：运行时 + 基础设施一个端点、业务 / HTTP 自定义一个端点（便于按需采集）
```

---

## 4. 默认选型与升级路径（强制）

| 场景 | 默认选型 | 升级触发 → 升级到 | 反向纪律（防过度） |
| --- | --- | --- | --- |
| 对外接口 | RED 三件套（请求量计数 + 延迟直方图 + 错误率按状态码） | 多机房 / 多版本要切片 → 加**有界** label（结果 / 模式维度，引 [`23 §7.3`](../docs-spec/23_observability_spec.md)） | 别加无界 label（actor-id / IP / 真实 URL → cardinality 爆炸） |
| 控制环 | 拉取式 collector 暴露饱和度 / 触发态 | — | 别后台定时写值（徒增开销 / 竞态）；别重复注册（框架 panic） |
| 延迟 | 直方图（分位，桶按 SLO） | — | **别用累计耗时计数求均值冒充分位**（看不见尾） |
| 资源池 | USE（利用率 + 饱和度） | 多池 / 多优先级 → 按池 label 切（有界） | 单池别过度切维 |
| 业务统计 | 结构化日志 + 离线 | — | 别当 metric 埋点 |
| 可视化 | 服务标准面板（RED + USE + 控制环 + SLO，落 `observability_dashboard.md`） | 控制环多 / SLO 严 → 拆专题面板 | **别有指标无面板**（盲采：采了不看 = 没采） |

> 反向纪律要点：**"加 label"和"加面板"都不免费**——label 吃 cardinality（[`23 §3`](../docs-spec/23_observability_spec.md) 硬上限）、面板吃维护。但**控制环不可观测是不可接受的**，这条没有"反向纪律"豁免。

---

## 5. 权衡矩阵 + 借纪律不借实现

### 5.1 三类遥测载体权衡

| 载体 | 强项 | 成本 | 适用 |
| --- | --- | --- | --- |
| Metric（本视角） | 低基数聚合、便宜、可告警 | cardinality 受限、无逐条 | 服务级 + 控制面状态 |
| 结构化日志（[`docs-spec/13`](../docs-spec/13_logging_spec.md)） | 逐条可追溯、高维 | 存储 / 检索贵 | 业务事件、错误现场 |
| Trace（经 `trace_id` 串日志，[`23`](../docs-spec/23_observability_spec.md)） | 跨服务因果链 | —（本仓不引独立组件） | 多跳链路定位 |

### 5.2 collector 形态权衡

| 形态 | 实时性 | 开销 | 适用 |
| --- | --- | --- | --- |
| 拉取式即读 | 采集那刻的真值 | 极低（无后台） | 即时态（运行数 / 限额 / 拒绝率） |
| 后台定时写值 | 采样滞后 | 后台协程 + 竞态 | 仅当读实例状态本身昂贵 |
| 累加（计数器 / 直方图） | 累计精确 | 每次调用一次原子 | 请求面累计 |

### 5.3 借纪律不借实现

可迁移的是 **"请求面 RED / 资源面 USE / 控制面内部状态"三面 + SLO 反推 + 拉取式即读** 这套纪律；具体指标名、监控库名、桶边界数值只进 §9 参考示例。把某工程的具体协程池指标名硬搬过来无意义，但"每个 worker 池都暴露运行 / 空闲 / 容量"这条纪律普适。

---

## 6. 反模式速查 + 机械闸门（引用不复制）

### 6.1 反模式速查

| 反模式 | grep 锚 | 正确方向 |
| --- | --- | --- |
| **控制环黑箱**（开了池 / 熔断 / 限制 / shed 却无饱和度 / 触发态指标） | `R-OBS-CONTROL-LOOP-BLIND` | §3.1 控制面 collector 暴露 |
| 关键路径延迟只累计计数求均值 | `R-OBS-LATENCY-COUNTER-ONLY` | 直方图分位（对齐 [`07`](07_tail_latency_design.md)） |
| **有指标无面板**（盲采） | `R-OBS-DASH-MISSING` | 服务标准面板（`observability_dashboard.md`） |
| 业务数据当 metric | （引 [`23 §11`](../docs-spec/23_observability_spec.md)） | 日志 + 离线 |
| 无界 label（cardinality 爆炸） | （引 [`23 §7.2`](../docs-spec/23_observability_spec.md)） | 路由模板 / 有界枚举 |
| 重复注册 collector（框架 panic） | （选型审查） | 查框架已注册，只补未覆盖 |
| 观测反噬热路径（日志 / 指标构造吃分配） | （引 [`22 §11`](../docs-spec/22_performance_contract_spec.md) 热路径约束） | 级别门控 / 采样 |

### 6.2 机械闸门下沉（L0）

可机械判定的：新增 `{池构造}` / `{熔断器构造}` / `{自适应限制器构造}` 但 collector 注册处无对应项（污点：控制环构造无配套 collector）→ L0 提醒（`R-OBS-CONTROL-LOOP-BLIND`）；"该不该出面板""饱和阈值合不合理"走选型审查（软提醒）。

---

## 7. 产出与落地

本视角的决策结论**分布落地**到既有 / 新增 docs/（按双源真相挂靠表示真相源）：

| 本视角结论 | 落到哪篇 docs/（按哪条 docs-spec 表示） |
| --- | --- |
| 服务级 + 控制面指标清单 / cardinality / 命名 | `observability.md`（[`23`](../docs-spec/23_observability_spec.md)，**含控制面指标类别**） |
| 服务级告警规则 | `observability.md §5`（[`23`](../docs-spec/23_observability_spec.md)） |
| 熔断面板路径 / 熔断态指标 | `circuit_breaker_design.md`（[`docs-spec/12`](../docs-spec/12_circuit_breaker_spec.md)，接其 `{Path to dashboard}` 占位） |
| 控制环参数与**饱和阈值** | `performance_contract.md §6.4`（[`22`](../docs-spec/22_performance_contract_spec.md)） |
| **服务标准面板** | `observability_dashboard.md`（表示见 [`23`](../docs-spec/23_observability_spec.md)） |
| 延迟直方图桶设计 | `observability.md §3.3`（[`23`](../docs-spec/23_observability_spec.md)，已有） |

---

## 8. 与既有规范的边界（双源真相规避）

| 内容 | 13（本篇） | docs-spec/23 | docs-spec/12 | docs-spec/22 | design-spec/03·06·07·12 |
| --- | --- | --- | --- | --- | --- |
| "该观测什么 / 暴露哪些控制环 / RED·USE·SLO 反推 / collector 形态选型" | ✅ 真相源 | 不写 | 不写 | 不写 | 不写 |
| Metric 注册 / cardinality / label 约束 / 告警规则表示 | 引用 | ✅ 真相源 | 不写 | 不写 | 不写 |
| 熔断面板路径 / 差异化参数 | 引用 | 不写 | ✅ 真相源 | 不写 | 引用（06） |
| SLA 数值 / 控制环参数表示 | 引用 | 引用 | 不写 | ✅ 真相源 | 不写 |
| 控制环本身选型（池容量 / 熔断算法 / 限制 / shed） | 引用（只决定"观测它"） | 不写 | 引用 | 引用 | ✅ 真相源 |
| 结构化日志 / `trace_id` 串联 | 引用 | 引用 | 不写 | 不写 | 不写（[`docs-spec/13`](../docs-spec/13_logging_spec.md) 真相源） |

> 一句话边界：**03 / 06 / 07 / 12 决定"开什么控制环"，本视角决定"怎么把它观测出来、画成面板"，[`23`](../docs-spec/23_observability_spec.md) 决定"指标怎么登记写下来"。** 本视角不碰控制环本身的选型，只决定其可观测性。

---

## 9. 参考示例（仅示意，落地按业务替换）

> ⚠️ 占位符豁免段。以下数值 / 指标名 / 代码取自某真实工程的具体实现，**只借纪律不借实现**——落地按自己的控制环与 SLO 重选指标与面板。服从 [`PRINCIPLES.md §12`](../PRINCIPLES.md) 参考示例段豁免。

### 9.1 控制面 collector（拉取式即读）

- 熔断器：`circuit_breaker_window_requests / _window_accepts / _refuse_ratio{breaker}`——采集时读各 breaker `Info()` 快照。
- 协程池：`gopool_running_workers / _free_workers / _capacity{pool=query|batch_fetch|write_back}`——`Free≈0 且 Running≈Cap` = 并行查询已排队 = IO 瓶颈信号。
- 去重：MySQL `go_sql_*` / Redis `go_redis_*` 由基础库自动注册，业务侧禁重复注册（否则 `duplicate metrics collector` panic）；只补库未覆盖的熔断器 + 协程池两类。

### 9.2 请求面 RED 中间件

- `http_request_total{path,method,code}` 计数器（`path` 用路由模板防 `companyId / searchKey` 高基数）+ `http_request_duration_ms{path,method}` 直方图（桶 1ms~5s，取 P50 / P90 / P99）。

### 9.3 双端点

- `/runtime-metrics`（Go runtime + 连接池 [ 自动 ] + 熔断器 / 协程池 [ 本项目 ]）、`/service/metrics`（HTTP RED + 业务自定义），物理隔离便于按需采集。

---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
