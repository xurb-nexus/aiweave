# 23 - 可观测性规范（Metrics 限服务级）

> 规定 `docs/architecture/observability.md` 的内容结构与维护规则。
>
> 本规范是 AIWeave 的 P1 规范之一，对应痛点 #12 运维可观测性。

---

## 1. 定位

`observability.md` 是 **AI 在生成涉及 Metric 注册、告警规则、运行时观测代码之前必须读的文档**。它回答三个问题：

> 1. 工程允许采集哪些指标？哪些是禁止的？
> 2. Metric 的内存占用上限是多少？怎么防止 cardinality 爆炸？
> 3. 告警分级 / 触发条件是什么？

没有这篇文档：
- AI 把业务数据当 Metric 采集（如 `{业务实体}_count`），导致 cardinality 爆炸
- AI 用 user_id / IP / 动态路径作为 label，把 Metric 库内存打爆
- AI 把日志与 Metric 职责混淆，业务事件用 Metric 实现"埋点"

---

## 2. 核心原则（强制）

- **只采集服务级基础指标，不采集业务数据**
- Tracing 通过结构化日志（`trace_id`/`request_id` 串联）构建，不引入独立 Tracing 组件
- 严格控制内存：总 cardinality 上限 ≤ 500（**默认**，可在 ai_dev_guide.md 登记理由后突破）

---

## 3. 与 13_logging 的边界划分（强制 / 双源真相规避）

| 内容 | 13_logging.md | 23_observability.md |
| --- | --- | --- |
| 结构化日志 API（`tlog.*` / 字段约定 / 级别判定） | ✅ 真相源 | 不写 |
| `trace_id` / `request_id` 串联（构成 Tracing） | ✅ 真相源 | 引用 |
| 敏感数据脱敏 | ✅ 真相源 | 不写 |
| Metric 注册 / cardinality / label 约束 | 不写 | ✅ 真相源 |
| 服务级告警规则（指标级） | 不写 | ✅ 真相源 |
| 日志级告警（关键错误日志触发） | ✅ 真相源 | 不写 |

**23 强制策略**：**禁止用 Metric 替代日志做"{业务事件}埋点"**——业务事件走 13_logging 的结构化日志 + 离线分析；Metric 仅承载"服务级运行状况"。

---

## 4. 顶层结构（强制）

`observability.md` 必须按以下章节顺序组织：

- `## 1.` 可观测性定位（Logging 引用 / Metrics 限定）
- `## 2.` Metrics 采集范围（强制 / 限定）
- `## 3.` 内存控制规范
- `## 4.` 命名约定
- `## 5.` Alerting 规范
- `## 6.` 与代码的关系（helpers/metrics.go 集中注册）
- `## 7.` 维护流程（含 B1 反向同步规则）

> **完整章节骨架见** [`templates/docs/architecture/observability.md`](../templates/docs/architecture/observability.md)。

---

## 5. §1 可观测性定位

```markdown
### 1.1 Logging（已有）
- 真相源：[`logging.md`](logging.md)
- Tracing 通过 `trace_id` / `request_id` 结构化日志串联，不引入独立 Tracing 组件

### 1.2 Metrics（仅服务级基础指标）
- 真相源：本文档
- 仅采集 HTTP 统计 / 服务健康 / Go runtime / 连接池状态
- 严格禁止业务数据 Metric
```

---

## 6. §2 Metrics 采集范围（强制 / 限定）

### 6.1 HTTP 请求统计（中间件自动采集）

```markdown
| Metric 名 | 类型 | Labels | 说明 |
|----------|------|--------|------|
| `http_request` | Counter | `url`、`code` | 请求计数 |
| `http_request_cost` | Counter | `url`、`code` | 请求耗时累计（ms） |
```

### 6.2 服务健康状态

```markdown
| Metric 名 | 类型 | Labels | 说明 |
|----------|------|--------|------|
| `health` | Gauge | `module` | 模块健康状态（1=健康 / 0=异常） |
```

### 6.3 运行时指标（独立 Registry / `/runtime-metrics` 端点）

```markdown
- Go runtime（GC、goroutine、内存）—— 由 `GoCollector` 自动采集
- 连接池状态（MySQL / Redis active / idle）—— 按需注册 `Collector`
- 生产持续 profiling（pprof / 火焰图）与运行时指标的联动见 [`docs-spec/22 §7.4`](22_performance_contract_spec.md)：定位"分配 / CPU 去哪了"，闭合离线 bench 之外的在线证据
```

### 6.4 禁止扩展到业务数据

```markdown
| ❌ 反例 Metric | 为什么禁止 | 替代方案 |
|--------------|----------|---------|
| `{业务实体}_count` | label cardinality 不可控 | 日志 + 离线分析 |
| `{核心数值字段}_sum`（聚合金额） | 业务数据 | 日志 + 离线分析 |
| `{状态字段}_distribution` | 状态值多变 | 日志 + 离线分析 |
```

---

## 7. §3 内存控制规范

### 7.1 Cardinality 默认上限（可登记突破）

```markdown
- 单个 Metric 的 label 组合数 ≤ 100（默认）
- 全服务所有 Metric 的 label 组合总数 ≤ 500（默认）
- 超出即视为设计缺陷；如确有业务理由必须突破，需在
  [`ai_dev_guide.md §9 约束突破登记表`](ai_dev_guide.md)登记
  理由 + 实际值 + 替代方案评估 + 责任人
```

### 7.2 Label 禁止清单（强制 / 不可登记突破）

```markdown
- 禁止使用：`{actor-id}` / `request_id` / IP / `trace_id` / 任何无界枚举值
- url label 必须为路由模板（如 `/{prefix}/{audience}/{module}/:{param}`），
  禁止使用实际请求路径
```

### 7.3 Histogram bucket 设计（强制 / 桶按 SLO 设）

```markdown
- **桶边界对齐 SLA 分位**：histogram 的 bucket 边界必须围绕本路径的 SLA / SLO 分位点（如 P50 / P90 / P99 目标值）布桶，使关键分位落在桶边界附近，
  `histogram_quantile` 估算误差最小——桶按 SLO 设，而非套用通用等距/指数桶。SLA 数值真相源见 [`docs-spec/22 §1`](22_performance_contract_spec.md)；选型方法论见 [`design-spec/06 §3.3 / §7`](../design-spec/06_resilience_design.md)。
- **label 带 status/mode**：关键路径 histogram 必须带 `status`（成功 / 失败等结果维度）与 `mode`（正常 / 降级等运行模式维度）label，
  以便按结果 / 模式切片分位；两者均为**有界枚举**，不得退化为无界值（仍受 §3.2 禁止清单约束）。
- bucket 数量 ≤ 15（库层面已限制 ≤ 20）；在 SLO 分位点之外的区间用尽量少的桶覆盖，避免指数级 bucket 浪费。
```

### 7.4 Metric 总数约束（默认上限）

```markdown
- 服务自定义 Metric 总数 ≤ 10 个（默认）
- 新增 Metric 必须在 helpers/metrics.go 集中注册，禁止分散注册
- 突破默认值同样走 ai_dev_guide.md 登记流程
```

---

## 8. §4 命名约定

```markdown
- 前缀：`service_{namespace}:{service_name}_{metric_name}`（由库自动拼接）
- metric_name 格式：`{模块}_{动作}[_{单位}]`
- 示例：`http_request` / `http_request_cost_ms` / `health`
- 禁止业务领域词作为前缀（如 `{business-name}_request`）
```

---

## 9. §5 Alerting 规范

### 9.1 告警层级（P0 / P1 / P2）

```markdown
| 层级 | 含义 | 响应时长 |
|------|------|---------|
| P0 | 服务级不可用 | < 5 min |
| P1 | 显著劣化 | < 30 min |
| P2 | 异常但暂未影响业务 | 工作时段 |
```

### 9.2 基础告警规则

```markdown
| 告警名 | 条件 | 持续时间 | 层级 |
|--------|------|---------|------|
| 服务不健康 | `health == 0` | 1 min | P0 |
| 错误率突增 | `5xx rate > {Error-rate-threshold}%`（推荐 5%） | 2 min | P1 |
| 延迟劣化 | `P99 > SLA × 2` | 3 min | P1 |
| Goroutine 泄漏 | `goroutine 数 > {Goroutine-leak-threshold}` 持续增长 | 10 min | P2 |
| GC 停顿过高 | `GC pause P99 > {GC-pause-threshold}`（推荐 10ms）持续 | 5 min | P2 |
```

SLA 数值的真相源在 [`docs-spec/22 §1`](22_performance_contract_spec.md)，本节仅引用阈值条件。

---

## 10. §6 与代码的关系

```markdown
### 6.1 Metric 集中注册

所有 Metric 统一在 `helpers/metrics.go` 中初始化。**禁止**业务代码直接调用 `prometheus.MustRegister`。

### 6.2 HTTP 指标通过中间件自动采集

业务代码**不感知** HTTP Metric——由 `{metric-middleware}` 中间件统一处理。

### 6.3 两个独立 Registry

- `/{prefix}/metrics` → 服务指标（自定义 Registry，不含 Go runtime）
- `/runtime-metrics` → 运行时指标（GoCollector）

两个端点物理隔离，方便 Prometheus 按需采集。

### 6.4 与 service docs 伪码标记的关系

service 方法伪码涉及发出指标的步骤必须使用 `[METRIC-EMIT: name{labels}]` 标记
（详见 [`docs-spec/09 §10.5`](09_service_design_spec.md)）。
```

---

## 11. AI 行为约束

```markdown
### Metrics 规则（强制）

- 禁止为业务数据创建 Metric（如 `{业务实体}_count` / `{核心数值字段}_sum` / `{状态字段}_distribution` 等 → 用日志 + 离线分析）
- 禁止在 helpers/metrics.go 之外注册 Metric
- 新增 Metric 前必须评估 cardinality：label 各维度的枚举值相乘 ≤ 100（默认上限，突破需登记）
- 禁止使用无界值作为 label（`{actor-id}` / IP / `request_id` / 动态路径）
- url label 必须使用路由模板，不可使用实际请求 URL
- 新增 Metric 必须出现在 §2 对应子表
```

---

## 12. §7 维护流程（含 B1 反向同步规则）

### 12.1 B1 反向同步规则（强制）

| 代码迹象（git diff） | 反向同步动作 |
| --- | --- |
| `helpers/metrics.go` 新增 `prometheus.MustRegister` / `NewCounterVec` 等 | §2 Metrics 采集范围对应分类新增一行；评估 cardinality 是否超 §3.1 上限 |
| 新增 label 维度 | 检查是否命中 §3.2 禁止清单（`{actor-id}` / IP / ...）；命中即拒绝合并 |
| 新增 Histogram / bucket | §3.3 检查 bucket 数 ≤ 15；校验桶边界对齐 SLA 分位（桶按 SLO 设）；校验关键路径 histogram label 带 `status` / `mode` 且为有界枚举 |
| 新增告警规则（YAML / Grafana） | §5.2 基础告警规则表新增一行 |
| 修改 Metric 命名 | §4 命名约定校对 |

### 12.2 维护触发

| 触发 | 动作 |
|------|------|
| 新增 Metric | §2 + helpers/metrics.go 同步登记 |
| Cardinality 突破默认上限 | ai_dev_guide.md §9 约束突破登记表新增一行 |
| 引入新告警 | §5.2 + Grafana / Alertmanager 同步 |

---

## 13. 与其他文档的关系

| 内容 | observability.md | 13_logging.md | 22_performance_contract.md |
|------|-----------------|----------------|---------------------------|
| 结构化日志 API | 引用 | ✅ 真相源 | 不写 |
| `trace_id` 串联 | 引用 | ✅ 真相源 | 不写 |
| Metric 注册 / cardinality | ✅ 真相源 | 不写 | 不写 |
| 服务级告警规则 | ✅ 真相源 | 不写 | 引用（阈值数值） |
| SLA 数值 | 引用 | 不写 | ✅ 真相源 |

---

## 14. 与 Skill 的联动

- **`/performance-review`**：扫描 helpers/metrics.go diff，对照 §2-§4 约束
- **`/new-middleware`**：生成 metric 类中间件时，先读 §2 / §6.2
- **`/doc-sync-check`**：扫描代码中的 `prometheus.MustRegister` 是否在 §2 登记，且未命中 §3.2 禁止 label

---

## 15. 参考示例（仅示意，落地按业务替换）

> ⚠️ 以下为示意，规范本体（§1-§14）已用占位符表达；落地工程时按业务语义替换占位符，不得直接照搬本节具体业务名作为规范。
>
> 示例服从 [`PRINCIPLES.md §12 占位符规则`](../PRINCIPLES.md#12-占位符规则强制--双轨结构) 的"参考示例段豁免"。

### 15.1 §2 Metrics 清单（示例：典型 Go 微服务）

| Metric 名 | 类型 | Labels | 说明 |
|----------|------|--------|------|
| `http_request` | Counter | `url=/api/internal/auth/verify`、`code=200` | 请求计数 |
| `http_request_cost` | Counter | 同上 | 请求耗时累计（ms） |
| `health` | Gauge | `module=auth` | 鉴权模块健康状态 |

### 15.2 §3.2 禁止 label 反例（示例）

```go
// ❌ 反例
prometheus.NewCounterVec(opts, []string{"user_id", "ip"})

// ✅ 正例
prometheus.NewCounterVec(opts, []string{"module", "result"})
```

### 15.3 §5.2 告警规则（示例）

| 告警名 | 条件 | 持续时间 | 层级 |
|--------|------|---------|------|
| 服务不健康 | `health{module="auth"} == 0` | 1 min | P0 |
| 鉴权错误率突增 | `rate(http_request{url="/api/internal/auth/verify",code=~"5.."}[1m]) > 0.05` | 2 min | P1 |
| 鉴权延迟劣化 | `histogram_quantile(0.99, http_request_cost) > 100` | 3 min | P1 |


---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
