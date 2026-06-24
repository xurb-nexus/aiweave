# 跨服务合约

> 版本：1.0 | 日期：{YYYY-MM-DD}
>
> 规范来源：[`aiweave/docs-spec/24_cross_service_contract_spec.md`](../../../docs-spec/24_cross_service_contract_spec.md)
>
> 本文档是 **AI 在生成涉及调用外部服务、被外部服务调用、修改对外接口契约之前必须读的文档**。

---

## 1. 上下游依赖图

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

> 落地工程时按实际架构填充。基础设施类下游（MySQL/Redis 等）的详细约束在 [`infrastructure.md`](infrastructure.md) 与 [`../circuit_breaker/circuit_breaker_design.md`](../circuit_breaker/circuit_breaker_design.md) 已有承载；本文档只标存在性。

---

## 2. 上游合约（谁调我）

| 调用方 | 接口 | SLA 约束 | 超时设置 | 重试策略 | 备注 |
|--------|------|---------|---------|---------|------|
| `{Upstream-A}` | `POST /{prefix}/internal/{module}/{action}` | P99 < `{P99-ms}` ms | 调用方设 `{T-up-ms}` ms | 失败重试 N 次（指数退避） | 业务幂等保证由本服务 §4 提供 |
| `{Upstream-B}` | `GET /{prefix}/operator/{module}/{action}` | P99 < `{P99-ms}` ms | 调用方设 `{T-up-ms}` ms | 不重试（查询类）| — |

---

## 3. 下游合约（我调谁）

| 被调方 | 接口 | 预期延迟 | 超时设置 | 重试策略 | 熔断配置 | 降级策略 |
|--------|------|---------|---------|---------|---------|---------|
| `{Down-RPC-A}` | `POST /{remote-path}` | < `{T-down-ms}` ms | `{T-self-set-ms}` ms | 失败重试 N 次 | 详见 [`circuit_breaker_design.md §2`](../circuit_breaker/circuit_breaker_design.md) | 详见 [`performance_contract.md §6.3`](performance_contract.md) L2 |
| `{Down-DB-A}` | MySQL `{db}` | < 5 ms | 1 s | GORM 内置 | GORM Plugin | 默认报错向上传 |
| `{Down-Cache-A}` | Redis `{cluster}` | < 1 ms | 100 ms | 不重试 | 详见 circuit_breaker §2 | 缓存 miss → 降级到源 |

**超时关系约束**：本服务对每个下游设置的超时 `{T-self-set-ms}` 必须满足 [`performance_contract.md §6.2`](performance_contract.md) 链路超时协调表。

**约束**：

- **剩余截止传播**：下游调用应传"端到端剩余预算"而非各自固定超时（剩余预算 < 本跳成本 → fast-fail / 跳过，决策见 design-spec/01 §3.3）
- **重试预算 + 对冲**（尾延迟治理，决策见 [`design-spec/07`](../../../design-spec/07_tail_latency_design.md)，参数登记 [`performance_contract.md §6.4`](performance_contract.md)）：
  - 重试只对**幂等**下游；全局重试占比设上限（如 ≤10%）+ backoff + jitter，防重试风暴放大故障（§4 原则 2"不可放大"）
  - 对冲 / 备份请求仅对**幂等 + 下游有余量**的尾敏感调用启用；触发分位（如 P95）+ 对冲量上限（如 ≤5%）；**下游饱和时禁用**（会翻倍打）

---

## 4. 故障传播矩阵

| 下游故障类型 | 触发条件 | 对本服务的影响 | 本服务的应对 | 对上游的传播 |
|-------------|---------|---------------|------------|------------|
| `{Down-RPC-A}` 超时 | `{T-self-set-ms}` ms 无响应 | 本接口无法完成 | 重试 N 次；仍失败 → 返回 `{Error-Reason}` | 5xx + `errStr={Error-Reason}` |
| `{Down-RPC-A}` 熔断 | 连续失败 K 次 | 本接口走降级路径 | 返回缓存数据（如有）或固定降级响应 | 2xx + `data=degraded` 标记 |
| `{Down-DB-A}` 故障 | MySQL 连接失败 / 超时 | 本接口写入失败 | 立即返回错误（写入操作必须强一致） | 5xx + `errNo=DB_ERROR` |
| `{Down-Cache-A}` miss | Redis 无对应 key | 本接口可能慢一些 | fallback 到 MySQL；同时异步预热 | 2xx 正常 |
| `{Down-Cache-A}` 故障 | Redis 连接失败 / 熔断 | 缓存读路径不可用 | 直接走 MySQL；写路径转 [`../service/transaction_design.md §6`](../service/transaction_design.md) | 视具体接口 |

**强制原则**：

1. 不可静默吞下游错误
2. 不可放大单点故障到多接口
3. 不可级联（本服务超时 < 上游超时）
4. 每条传播路径必有可观测（[`logging.md`](logging.md) 告警日志或 [`observability.md §5.2`](observability.md) 告警规则）

---

## 5. 接口版本管理

### 5.1 版本命名规范

| 维度 | 规则 |
|------|------|
| URL 路径版本 | `/{prefix}/v{N}/{audience}/{module}/{action}`（仅在主版本不兼容时升） |
| 字段级演进 | 用 `Deprecated` 注释 + 软兼容期 |
| Breaking change | 必须升主版本号；旧版本继续运行至少 1 个发版周期 |

### 5.2 向后兼容规则

- 新增字段必须有默认值
- 字段类型变更必须兼容（如 `int` → `int64` 可；反之不可）
- 删除字段必须先标记 `Deprecated`，经过软兼容期后再移除
- 修改字段语义（如单位变更）必须升主版本号

### 5.3 废弃接口生命周期

| 阶段 | 行为 | 时长 |
|------|------|------|
| Deprecated 标注 | 接口仍可用，响应头加 `Deprecation: true` | 至少 1 个发版周期 |
| Sunset 通告 | 通知所有上游调用方 | 至少 2 周 |
| 下线 | 路由移除；旧版本服务保留可选 | — |

---

## 6. 维护流程

### 6.1 B1 反向同步规则

| 代码迹象（git diff） | 反向同步动作 |
| --- | --- |
| `api/` 目录新增 client / `pkg/` 引入新 RPC 包 | §3 下游合约新增一行 |
| 新增 server 端路由被外部调用方接入 | §2 上游合约新增一行 |
| 修改超时 / 重试参数 | §2 或 §3 同步；如形成新故障路径 → §4 更新 |
| 接口签名变更（请求 / 响应字段） | §5 评估是否需要升版本号 |
| 引入新的 5xx / 业务错误分支 | §4 故障传播矩阵新增一行 |

### 6.2 与 BUILD_STATUS §11 约束清单状态轨道的关系

每条上游 / 下游 / 故障传播条目对应 BUILD_STATUS.md §11 "跨服务合约约束"类目的"已设计 / 已启用"计数。


---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
