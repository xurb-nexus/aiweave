# 服务可视化 / 标准面板（observability_dashboard.md）

> **这是什么**：本服务的**标准监控面板**结构——把 [`observability.md`](observability.md) 登记的指标，按"看什么 / 红线画哪"组织成可视化。
>
> **决策真相源**：该观测什么 / 控制环怎么暴露 / 面板看什么，方法论见 [`design-spec/13_observability_design.md`](../../../design-spec/13_observability_design.md)（本工程不含 design-spec 时，见 AIWeave 第 13 视角）。
> **表示真相源**：指标登记 / cardinality / 告警在 [`observability.md`](observability.md)；SLA 数值在 [`performance_contract.md §1`](performance_contract.md)；熔断面板路径在 [`circuit_breaker_design.md`](../circuit_breaker/circuit_breaker_design.md)。本文档只**编排**这些指标成面板，不重复登记。
>
> **占位符纪律**：面板配置（具体可视化工具的 JSON / DSL）不进本文档，只进末尾"参考示例"或外部面板仓库；本文档只写"哪一行 / 看什么 / 红线画哪"。

---

## 1. 标准面板 = 四行（强制）

一个服务的面板按下列**四行**组织。每行的指标必须已在 [`observability.md`](observability.md) 登记：

| 面板行 | 看什么 | 红线画在哪 | 指标来源 |
| --- | --- | --- | --- |
| **RED 行**（请求面） | QPS、错误率（按状态码）、延迟 P50 / P90 / P99 | 对着 SLA（[`performance_contract.md §1`](performance_contract.md)） | `observability.md §2` HTTP 指标 |
| **USE 行**（资源面） | 各池利用率 / 饱和度、连接池在用 / 空闲、CPU / 内存 / GC | 饱和度对着**排队膝盖 ρ≈`{ρ-knee}`（默认 0.8）**，非 100% | `observability.md §2` 运行时 + 控制面指标 |
| **控制环行** | 熔断器拒绝率、池运行数 / 容量、自适应限额、shed 速率、对冲量、重试预算消耗 | 各自的触发阈值（[`performance_contract.md §6.4`](performance_contract.md) / [`circuit_breaker_design.md`](../circuit_breaker/circuit_breaker_design.md)） | `observability.md §2` 控制面指标类别 |
| **SLO 行** | SLO 燃尽 / 错误预算 | 错误预算耗尽线 | SLA 见 `performance_contract.md §1` |

> **铁律（控制环可观测性）**：[`observability.md`](observability.md) 控制面指标类别里**每一个控制环指标，必有本面板里对应的一格**——否则即"有指标无面板"（盲采：采了不看 = 没采，`R-OBS-DASH-MISSING`）。
> **铁律（红线画在膝盖）**：饱和度类面板的告警红线画在**排队膝盖 ρ≈`{ρ-knee}`** 而非满载（100%）——满载时排队时延已发散，红线画在 100% 等于没有早期预警。

---

## 2. 各行面板登记（按服务填）

```markdown
### 2.1 RED 行
| 面板格 | 指标 | 红线 |
|--------|------|------|
| QPS | `{http-request-metric}` rate | — |
| 错误率 | `{http-request-metric}` 中 code 5xx 占比 | > `{Error-rate-threshold}` 告警 |
| 延迟分位 | `{http-latency-histogram}` P50 / P90 / P99 | P99 > `{P99-SLA}` 告警 |

### 2.2 USE 行
| 面板格 | 指标 | 红线 |
|--------|------|------|
| `{pool-name}` 饱和度 | 运行数 / 容量 | ρ > `{ρ-knee}`（默认 0.8） |
| 连接池 | 在用 / 空闲 | 在用 ≈ max 持续 → 告警 |
| 运行时 | GC pause / goroutine / RSS | 见 `observability.md §5` |

### 2.3 控制环行（每个前序 lens 开的控制环一格）
| 面板格 | 指标 | 红线 |
|--------|------|------|
| 熔断器 `{breaker}` | 拒绝率（0~1） | > 0 即开始丢弃 |
| `{pool}` | 运行 / 空闲 / 容量 | 空闲长期 0 + 运行顶容量 = 排队 |
| 自适应限制 | 当前限额 + 触顶计数 | 触顶频繁 → 下游漂移 |
| load shedding | 丢弃速率（按优先级 / 原因） | 持续 > 0 = 系统性过载 |

### 2.4 SLO 行
| 面板格 | 指标 | 红线 |
|--------|------|------|
| 错误预算燃尽 | SLO 达成率 | 错误预算耗尽线 |
```

---

## 3. 维护流程（含 B1 反向同步规则）

| 代码 / 文档迹象 | 反向同步动作 |
| --- | --- |
| `observability.md` 控制面指标类别**新增一个控制环指标** | 本面板"控制环行"**必须**加对应一格（否则 `R-OBS-DASH-MISSING`） |
| 新增对外接口 / 延迟点 | RED 行补格，延迟必走直方图分位（非累计均值） |
| 修改 SLA / `{ρ-knee}` | RED / USE 行红线同步 |
| 新增熔断器 | 控制环行加格 + 接 [`circuit_breaker_design.md`](../circuit_breaker/circuit_breaker_design.md) 的 `{Path to dashboard}` |

---

## 4. 参考示例（仅示意，落地按业务替换）

> ⚠️ 占位符豁免段：以下面板配置 / 工具名 / 阈值为某真实工程示意，落地按业务替换。服从 [`PRINCIPLES.md §12`](../../../PRINCIPLES.md) 参考示例段豁免。

- 面板工具：Grafana / 内置看板；面板定义存外部面板仓库或 `{Path to dashboard}`。
- USE 行饱和度红线：ρ = 0.8（排队膝盖）；延迟桶：1ms ~ 5s。
- 控制环行示例格：`circuit_breaker_refuse_ratio{breaker}`、`gopool_running_workers / _capacity{pool}`。

---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
