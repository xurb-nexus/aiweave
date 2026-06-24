# 24 - 跨服务合约规范

> 规定 `docs/architecture/cross_service_contract.md` 的内容结构与维护规则。
>
> 对应痛点 #2 跨服务调用链理解不足。

---

## 1. 定位

`cross_service_contract.md` 是 **AI 在生成涉及调用外部服务、被外部服务调用、修改对外接口契约之前必须读的文档**。它回答四个问题：

> 1. 本服务在整体架构中的位置是什么？上下游有哪些？
> 2. 上游 / 下游各自的 SLA / 超时 / 重试约束是什么？
> 3. 下游故障如何传播到本服务？本服务如何应对？又如何传播回上游？
> 4. 接口版本如何管理？什么时候必须升版？

没有这篇文档：
- AI 给下游设置不合理的超时（如远大于上游超时，导致级联阻塞）
- AI 修改对外接口签名而不感知"已有外部调用方"
- AI 不知道哪些下游故障应"立即返回"、哪些应"重试"、哪些应"降级"

---

## 2. 顶层结构（强制）

`cross_service_contract.md` 必须按以下章节顺序组织：

- `## 1.` 上下游依赖图（强制）
- `## 2.` 上游合约（谁调我）
- `## 3.` 下游合约（我调谁）
- `## 4.` 故障传播矩阵（核心）
- `## 5.` 接口版本管理
- `## 6.` 维护流程（含 B1 反向同步规则）

> **完整章节骨架见** [`templates/docs/architecture/cross_service_contract.md`](../templates/docs/architecture/cross_service_contract.md)。

---

## 3. §1 上下游依赖图（强制 ASCII 图）

```
                ┌──────────────┐
                │ {Upstream-A} │
                └──────┬───────┘
                       │
                       ▼
              ┌──────────────────┐
              │  本服务 {Self}    │
              └──────┬───────────┘
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
  ┌──────────┐ ┌──────────┐ ┌──────────┐
  │ Down-A   │ │ Down-B   │ │ Down-C   │
  │ (MySQL)  │ │ (Redis)  │ │ (RPC svc)│
  └──────────┘ └──────────┘ └──────────┘
```

图中节点说明：

- **Upstream** = 显式调用本服务的外部系统（含其他微服务 / 网关 / 客户端）
- **Down** 含三类：基础设施（MySQL/Redis/Kafka）、其他微服务、第三方 API

> 基础设施类下游（MySQL/Redis 等）的详细约束在 `docs/architecture/infrastructure.md` 与 `docs/circuit_breaker/circuit_breaker_design.md` 已有承载；本文档**只标存在性**，不重复参数。

---

## 4. §2 上游合约（谁调我）

```markdown
| 调用方 | 接口 | SLA 约束 | 超时设置 | 重试策略 | 备注 |
|--------|------|---------|---------|---------|------|
| `{Upstream-A}` | `POST /{prefix}/internal/{module}/{action}` | P99 < `{P99-ms}` ms | 调用方设 `{T-up-ms}` ms | 失败重试 N 次（间隔指数退避） | 业务幂等保证由本服务 §4 提供 |
| `{Upstream-B}` | `GET /{prefix}/operator/{module}/{action}` | P99 < `{P99-ms}` ms | 调用方设 `{T-up-ms}` ms | 不重试（查询类）| — |
```

**规则**：

- SLA 数值与 [`docs-spec/22 §1`](22_performance_contract_spec.md) 全局性能目标一致
- 接口路径、JSON 结构详细规格在 [`docs-spec/07 api_interfaces`](07_api_interfaces_spec.md)
- 本表只承载"对外契约维度"——调用方期望什么、本服务承诺什么

---

## 5. §3 下游合约（我调谁）

```markdown
| 被调方 | 接口 | 预期延迟 | 超时设置 | 重试策略 | 熔断配置 | 降级策略 |
|--------|------|---------|---------|---------|---------|---------|
| `{Down-RPC-A}` | `POST /{remote-path}` | < `{T-down-ms}` ms | `{T-self-set-ms}` ms | 失败重试 N 次 | 详见 [`circuit_breaker_design.md §2`](../circuit_breaker/circuit_breaker_design.md) | 详见 [`performance_contract.md §6.3`](22_performance_contract_spec.md) L2 |
| `{Down-DB-A}` | MySQL `{db}` | < 5 ms | 1 s | GORM 内置 | GORM Plugin（详见 12） | 默认报错向上传 |
| `{Down-Cache-A}` | Redis `{cluster}` | < 1 ms | 100 ms | 不重试 | 详见 12 §2 | 缓存 miss → 降级到源 |
```

**约束**：

- 超时数值必须与 [`docs-spec/22 §6.2`](22_performance_contract_spec.md) 链路超时协调表一致
- 熔断器参数真相源在 [`docs-spec/12 §2`](12_circuit_breaker_spec.md)，本表只引用不重复
- L2/L3 业务降级行为真相源在 [`docs-spec/22 §6.3`](22_performance_contract_spec.md)
- **剩余截止传播**：下游调用应传"端到端剩余预算"而非各自固定超时（剩余预算 < 本跳成本 → fast-fail / 跳过，决策见 [`design-spec/01 §3.3`](../design-spec/01_io_design.md)）
- **重试预算 + 对冲**（尾延迟治理，决策见 [`design-spec/07`](../design-spec/07_tail_latency_design.md)，参数登记 [`22 §6.4`](22_performance_contract_spec.md)）：
  - 重试只对**幂等**下游；全局重试占比设上限（如 ≤10%）+ backoff + jitter，防重试风暴放大故障（§4 原则 2"不可放大"）
  - 对冲 / 备份请求仅对**幂等 + 下游有余量**的尾敏感调用启用；触发分位（如 P95）+ 对冲量上限（如 ≤5%）；**下游饱和时禁用**（会翻倍打）

---

## 6. §4 故障传播矩阵（核心）

> 这是本文档最关键的章节。它显式回答："下游 X 故障时，本服务做什么？上游会看到什么？"

```markdown
| 下游故障类型 | 触发条件 | 对本服务的影响 | 本服务的应对 | 对上游的传播 |
|-------------|---------|---------------|------------|------------|
| `{Down-RPC-A}` 超时 | `{T-self-set-ms}` ms 无响应 | 本接口无法完成 | 重试 N 次；仍失败 → 返回 `{Error-Reason}` | 5xx + `errStr={Error-Reason}` |
| `{Down-RPC-A}` 熔断 | 连续失败 K 次 | 本接口走降级路径 | 返回缓存数据（如有）或固定降级响应 | 2xx + `data=degraded` 标记 |
| `{Down-DB-A}` 故障 | MySQL 连接失败 / 超时 | 本接口写入失败 | 立即返回错误（写入操作必须强一致） | 5xx + `errNo=DB_ERROR` |
| `{Down-Cache-A}` miss | Redis 无对应 key | 本接口可能慢一些 | fallback 到 MySQL；同时异步预热 | 2xx 正常 |
| `{Down-Cache-A}` 故障 | Redis 连接失败 / 熔断 | 缓存读路径不可用 | 直接走 MySQL；写路径转 [`transaction_design.md §6 失败路径`](../service/transaction_design.md) | 视具体接口 |
```

**故障传播必须满足的原则**：

1. **不可静默吞**：下游错误必须有显式应对（重试 / 降级 / 报错）；不允许 `_ = downstream.Call()` 忽略错误
2. **不可放大**：单个下游故障不应让多个上游接口同时报错——必须有熔断或降级；**重试必须带预算**（占比上限 + backoff + jitter），无预算重试是级联失败的头号放大器（详见 [`design-spec/07 §3.5`](../design-spec/07_tail_latency_design.md)）
3. **不可级联**：本服务的超时设置必须 < 上游超时（详见 [`22 §6.2`](22_performance_contract_spec.md)）
4. **必有可观测**：每条传播路径必须对应一条 [`logging.md`](logging.md) 的告警日志或 [`observability.md §5.2`](observability.md) 的告警规则

---

## 7. §5 接口版本管理

```markdown
### 5.1 版本命名规范

| 维度 | 规则 |
|------|------|
| URL 路径版本 | `/{prefix}/v{N}/{audience}/{module}/{action}`（仅在主版本不兼容时升） |
| 字段级演进 | 用 `Deprecated` 注释 + 软兼容期 |
| Breaking change | 必须升主版本号；旧版本继续运行至少 1 个发版周期 |

### 5.2 向后兼容规则

- 新增字段必须有默认值（请求体 / 响应体均适用）
- 字段类型变更必须兼容（如 `int` → `int64` 可；反之不可）
- 删除字段必须先标记 `Deprecated`，经过软兼容期后再移除
- 修改字段语义（如单位变更）必须升主版本号

### 5.3 废弃接口生命周期

| 阶段 | 行为 | 时长 |
|------|------|------|
| Deprecated 标注 | 接口仍可用，响应头加 `Deprecation: true` | 至少 1 个发版周期 |
| Sunset 通告 | 通知所有上游调用方 | 至少 2 周 |
| 下线 | 路由移除；旧版本服务保留可选 | — |
```

---

## 8. §6 维护流程（含 B1 反向同步规则）

### 8.1 B1 反向同步规则（强制）

| 代码迹象（git diff） | 反向同步动作 |
| --- | --- |
| `api/` 目录新增 client / `pkg/` 引入新 RPC 包 | §3 下游合约新增一行（被调方 / SLA / 超时 / 熔断 / 降级） |
| 新增 server 端路由被外部调用方接入 | §2 上游合约新增一行 |
| 修改超时 / 重试参数 | §2 或 §3 对应行参数同步；如形成新故障路径 → §4 故障传播矩阵更新 |
| 新增对冲 / 重试预算 / 剩余截止传播 | §3 约束补登记；参数登记 [`22 §6.4`](22_performance_contract_spec.md)（决策见 design-spec/07） |
| 接口签名变更（请求 / 响应字段） | §5 版本管理评估是否需要发新版本号 |
| 引入新的 5xx / 业务错误分支 | §4 故障传播矩阵新增一行 |

### 8.2 维护触发

| 触发 | 动作 |
|------|------|
| 新增上游调用方 | §2 必须新增一行 + 评估对 SLA 的影响 |
| 新增下游依赖 | §3 必须新增一行 + 同步 [`infrastructure.md`](../templates/docs/architecture/infrastructure.md) 与 [`circuit_breaker_design.md §2`](12_circuit_breaker_spec.md) |
| 修改接口版本 | §5 + 同步 [`api/*.md`](07_api_interfaces_spec.md) + 通知所有上游 |
| 引入故障路径 | §4 故障传播矩阵新增一行 + 同步 [`observability.md §5.2`](23_observability_spec.md) 告警规则 |

---

## 9. 与其他文档的关系

| 内容 | cross_service_contract.md | 07 api_interfaces.md | 12 circuit_breaker.md | 22 performance_contract.md | 23 observability.md |
|------|--------------------------|---------------------|----------------------|---------------------------|--------------------|
| 上下游依赖图 | ✅ 真相源 | 不写 | 不写 | 不写 | 不写 |
| 上游 / 下游合约（SLA / 超时 / 重试） | ✅ 真相源 | 引用 | 不写 | 引用（链路超时协调） | 不写 |
| 接口 JSON 详细规格 | 不写 | ✅ 真相源 | 不写 | 不写 | 不写 |
| 熔断器参数 | 引用 | 不写 | ✅ 真相源 | 不写 | 不写 |
| 故障传播矩阵 | ✅ 真相源 | 不写 | 不写 | 不写 | 引用（告警规则） |
| 接口版本管理 | ✅ 真相源 | 不重复 | 不写 | 不写 | 不写 |

---

## 10. 与 Skill 的联动

- **`/new-controller`**：生成 controller 时，如涉及对外接口，提示先读 §2 / §5
- **`/new-service`** / **`/new-mq-consumer`**：涉及调用下游时，提示先读 §3 / §4
- **`/failure-path-review`**：扫描 service 方法是否覆盖了 §4 故障传播矩阵的所有下游故障路径
- **`/doc-sync-check`**：扫描 `api/` client 是否在 §3 登记；扫描对外路由是否在 §2 登记

---

## 11. 参考示例（仅示意，落地按业务替换）

> ⚠️ 以下为示意，规范本体（§1-§10）已用占位符表达；落地工程时按业务语义替换占位符，不得直接照搬本节具体业务名作为规范。
>
> 示例服从 [`PRINCIPLES.md §12 占位符规则`](../PRINCIPLES.md#12-占位符规则强制--双轨结构) 的"参考示例段豁免"。

### 11.1 §1 上下游依赖图（示例：鉴权 + 计费服务）

```
                ┌──────────────┐
                │ gateway-svc  │
                └──────┬───────┘
                       │
                       ▼
              ┌──────────────────┐
              │  auth-billing    │  ← 本服务
              └──────┬───────────┘
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
  ┌──────────┐ ┌──────────┐ ┌──────────┐
  │ MySQL    │ │ Redis    │ │ user-svc │
  │ core+log │ │ cluster1 │ │ (RPC)    │
  └──────────┘ └──────────┘ └──────────┘
```

### 11.2 §3 下游合约（示例）

| 被调方 | 接口 | 预期延迟 | 超时设置 | 重试策略 | 熔断配置 | 降级策略 |
|--------|------|---------|---------|---------|---------|---------|
| user-svc | `POST /api/internal/user/profile` | < 30 ms | 100 ms | 失败重试 2 次 | 详见 circuit_breaker §2 | 缓存最近一次响应 |
| MySQL core | — | < 5 ms | 1 s | GORM 内置 | GORM Plugin | 报错向上传 |
| Redis cluster1 | — | < 1 ms | 100 ms | 不重试 | 详见 12 §2 | miss → MySQL fallback |

### 11.3 §4 故障传播矩阵（示例）

| 下游故障 | 触发 | 影响 | 应对 | 对上游 |
|---------|------|------|------|--------|
| user-svc 超时 | 100 ms 无响应 | verify 接口无法完成 | 重试 2 次；仍失败 → 返回 `ErrorUserServiceTimeout` | 5xx + 错误描述 |
| user-svc 熔断 | 连续失败 K 次 | verify 走降级 | 返回缓存 user 数据 | 2xx + `data.from_cache=true` |
| MySQL core 故障 | 连接失败 | register 写入失败 | 立即返回 `ErrorDbInsert` | 5xx |
| Redis miss | acc:info:userId 不存在 | verify 慢 | fallback MySQL；异步预热 Redis | 2xx 正常 |


---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
