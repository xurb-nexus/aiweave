# 可观测性（Metrics 限服务级）

> 版本：1.0 | 日期：{YYYY-MM-DD}
>
> 规范来源：[`aiweave/docs-spec/23_observability_spec.md`](../../../docs-spec/23_observability_spec.md)
>
> 本文档是 **AI 在生成涉及 Metric 注册、告警规则、运行时观测代码之前必须读的文档**。

---

## 1. 可观测性定位

### 1.1 Logging
- 真相源：[`logging.md`](logging.md)
- Tracing 通过 `trace_id` / `request_id` 结构化日志串联，不引入独立 Tracing 组件

### 1.2 Metrics
- 真相源：本文档；**决策方法论**（该观测什么 / 控制环怎么暴露 / SLO 反推面板）见 [`design-spec/13`](../../../design-spec/13_observability_design.md)
- 仅采集 HTTP 统计 / 服务健康 / Go runtime / 连接池状态 + **控制面内部状态**（控制环饱和度 / 触发态，见 §2.4）
- 按 **RED**（请求面）+ **USE**（资源面）+ **控制面**三面组织；严格禁止业务数据 Metric

---

## 2. Metrics 采集范围

### 2.1 HTTP 请求统计（中间件自动采集）

| Metric 名 | 类型 | Labels | 说明 |
|----------|------|--------|------|
| `http_request` | Counter | `url`、`code` | 请求量 + 错误率（按 `code`） |
| `http_request_duration` | **Histogram** | `url`、`method` | 请求延迟分布，桶按 §3.3 SLO，取 P50/P90/P99（**延迟必走 Histogram，不用累计 Counter 求均值**） |

### 2.2 服务健康状态

| Metric 名 | 类型 | Labels | 说明 |
|----------|------|--------|------|
| `health` | Gauge | `module` | 模块健康状态（1=健康 / 0=异常） |

### 2.3 运行时指标（独立 Registry / `/runtime-metrics` 端点）

- Go runtime（GC、goroutine、内存）—— 由 `GoCollector` 自动采集
- 连接池状态（MySQL / Redis active / idle）—— 按需注册 `Collector`
- 生产持续 profiling（pprof / 火焰图）与运行时指标的联动见 [`performance_contract.md §7.4`](performance_contract.md)：定位"分配 / CPU 去哪了"，闭合离线 bench 之外的在线证据
- **collector 工程纪律**：即时态用拉取式即读（采集时现读、无后台定时 Set）；框架已注册的（连接池）禁重复注册（panic）；注册在实例初始化之后；仅 HTTP 路径暴露；双 Registry 分离。详见 [`docs-spec/23 §6.3`](../../../docs-spec/23_observability_spec.md)

### 2.4 控制面 / 饱和度指标（控制环可观测性）

> 凡 design-spec 各 lens 开了**控制环**，其内部状态必须暴露成指标（黑箱 = 故障看不见拐点，`R-OBS-CONTROL-LOOP-BLIND`）。决策见 [`design-spec/13`](../../../design-spec/13_observability_design.md)。

| 控制环（来源 lens） | 指标 | label（有界） | collector 形态 |
|--------------------|------|--------------|----------------|
| `{worker-pool}`（并发 Little 池 03） | 运行 / 空闲 / 容量 → 饱和度 | `{pool}` | 拉取式即读 |
| `{breaker}`（熔断器 06 / 12） | 窗口请求·接受数 + 拒绝率（0~1） | `{breaker}` | 拉取式即读 |
| `{limiter}`（自适应限制 / shed / 对冲 / 重试 07） | 限额 / 触顶 / 丢弃 / 对冲量 / 预算消耗 | 有界 | 拉取式即读 + 计数器 |

> 饱和阈值见 [`performance_contract.md §6.4`](performance_contract.md)；面板编排见 [`observability_dashboard.md`](observability_dashboard.md)（每指标必有面板格，否则 `R-OBS-DASH-MISSING`）。

### 2.5 禁止扩展到业务数据

| ❌ 反例 Metric | 为什么禁止 | 替代方案 |
|--------------|----------|---------|
| `{业务实体}_count` | label cardinality 不可控 | 日志 + 离线分析 |
| `{核心数值字段}_sum` | 业务数据 | 日志 + 离线分析 |
| `{状态字段}_distribution` | 状态值多变 | 日志 + 离线分析 |

---

## 3. 内存控制规范

### 3.1 Cardinality 默认上限（可登记突破）

- 单个 Metric 的 label 组合数 ≤ 100（默认）
- 全服务所有 Metric 的 label 组合总数 ≤ 500（默认）
- 突破需在 [`ai_dev_guide.md §9 约束突破登记表`](ai_dev_guide.md)登记

### 3.2 Label 禁止清单（强制 / 不可登记突破）

- 禁止使用：`{actor-id}` / `request_id` / IP / `trace_id` / 任何无界枚举值
- url label 必须为路由模板（如 `/{prefix}/{audience}/{module}/:{param}`）

### 3.3 Histogram bucket 设计（桶按 SLO 设）

> 选型方法论 / 真相源见 [`aiweave/design-spec/06_resilience_design.md §3.3 / §7`](../../../design-spec/06_resilience_design.md)；表示规范见 `aiweave/docs-spec/23_observability_spec.md §3.3`。SLA 数值真相源见 [`performance_contract.md §1`](performance_contract.md)。

- **桶边界对齐 SLA 分位（桶按 SLO 设）**：bucket 边界围绕本路径 SLA / SLO 分位点（P50 / P90 / P99 目标值）布桶，使 `histogram_quantile` 误差最小，不套用通用等距 / 指数桶
- **label 带 status/mode**：关键路径 histogram 必须带 `status`（结果维度）与 `mode`（正常 / 降级运行模式维度）label，均为有界枚举（受 §3.2 禁止清单约束）
- bucket 数量 ≤ 15

| 关键路径 histogram | SLO 分位点（布桶依据） | label |
|------|------|------|
| `{path-histogram}` | {P50/P90/P99 目标值} | `status`、`mode` |

### 3.4 Metric 总数约束

- 服务自定义 Metric 总数 ≤ 10 个（默认）
- 必须在 helpers/metrics.go 集中注册

---

## 4. 命名约定

- 前缀：`service_{namespace}:{service_name}_{metric_name}`（由库自动拼接）
- metric_name 格式：`{模块}_{动作}[_{单位}]`
- 示例：`http_request` / `http_request_cost_ms` / `health`

---

## 5. Alerting 规范

### 5.1 告警层级

| 层级 | 含义 | 响应时长 |
|------|------|---------|
| P0 | 服务级不可用 | < 5 min |
| P1 | 显著劣化 | < 30 min |
| P2 | 异常但暂未影响业务 | 工作时段 |

### 5.2 基础告警规则

| 告警名 | 条件 | 持续时间 | 层级 |
|--------|------|---------|------|
| 服务不健康 | `health == 0` | 1 min | P0 |
| 错误率突增 | `5xx rate > {Error-rate-threshold}%` | 2 min | P1 |
| 延迟劣化 | `P99 > SLA × 2` | 3 min | P1 |
| Goroutine 泄漏 | `goroutine 数 > {Goroutine-leak-threshold}` 持续增长 | 10 min | P2 |
| GC 停顿过高 | `GC pause P99 > {GC-pause-threshold}` 持续 | 5 min | P2 |

SLA 数值的真相源在 [`performance_contract.md §1`](performance_contract.md)。

---

## 6. 与代码的关系

### 6.1 Metric 集中注册

所有 Metric 统一在 `helpers/metrics.go` 中初始化。禁止业务代码直接调用 `prometheus.MustRegister`。

### 6.2 HTTP 指标通过中间件自动采集

业务代码不感知 HTTP Metric——由 `{metric-middleware}` 中间件统一处理。

### 6.3 两个独立 Registry

- `/{prefix}/metrics` → 服务指标
- `/runtime-metrics` → 运行时指标

### 6.4 与 service docs 伪码标记

涉及发出指标的步骤必须使用 `[METRIC-EMIT: name{labels}]` 标记。

---

## 7. 维护流程

### 7.1 B1 反向同步规则

| 代码迹象（git diff） | 反向同步动作 |
| --- | --- |
| `helpers/metrics.go` 新增 `prometheus.MustRegister` | §2 对应分类新增一行；评估 cardinality |
| 新增 label 维度 | 检查是否命中 §3.2 禁止清单 |
| 新增 Histogram / bucket | §3.3 检查 bucket 数 ≤ 15；校验桶边界对齐 SLA 分位（桶按 SLO 设）+ 关键路径 label 带 `status` / `mode` |
| 新增告警规则 | §5.2 基础告警规则表新增一行 |
| 新增控制环（池 / 熔断 / 限制器 / shed） | §2.4 控制面指标登记 + 注册 collector + `observability_dashboard.md` 控制环行加格（否则 `R-OBS-CONTROL-LOOP-BLIND` / `R-OBS-DASH-MISSING`） |
| 新增延迟观测点 | 必为 Histogram（非累计 Counter 均值），§2.1 + §3.3 桶对齐 SLO（否则 `R-OBS-LATENCY-COUNTER-ONLY`） |

### 7.2 与 BUILD_STATUS §11 约束清单状态轨道的关系

每个 Metric / 告警规则对应 BUILD_STATUS.md §11 "可观测性约束"类目的"已设计 / 已启用"计数。


---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
