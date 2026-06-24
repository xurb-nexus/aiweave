# 08 - API 逻辑描述与流程串联规范

> 规定 `docs/architecture/{xxx}_flow.md` 的内容结构。**这是与 07 不同的视角**：07 是单接口契约，08 是跨接口/跨模块的业务流程。

---

## 1. 定位

API 接口文档（`docs/api/*.md`）回答 **"接口长什么样"**；
流程文档（`docs/architecture/*_flow.md`）回答 **"接口之间如何协作完成业务"**。

两者都是必要视角，缺一不可。

精度要求：**步骤级**，每一步对应一个明确的代码动作（中间件、Service 方法、Redis 操作）。

---

## 2. 何时需要专门写流程文档

满足以下任一条件时，单独成篇：

1. 单一业务链路涉及 **≥ 2 个接口**（如 `{接口-A}` → `{接口-B}`）
2. 单一业务链路涉及 **≥ 5 步内部校验**（如 N 步校验链）
3. 多个 Service 之间存在**强时序依赖**（如 `{module-A} 写库` → `{module-B} 状态变更` → `{module-C} 权限写入`）
4. 异步链路（如 `{Producer 方法}` → Kafka → Consumer → MySQL flush）

---

## 3. 标准命名

```markdown
| 流程类型 | 文件名 |
|---------|--------|
| {核心业务链路} | `{核心业务链路}_flow.md` |
| {业务闭环} | `{业务闭环}_flow.md` |
| {示例流程} | `{example_flow}.md` |
| {数据同步流程} | `sync_flow.md` |
```

放在 `docs/architecture/` 下。具体业务命名（按工程业务流程主线命名为 `{核心校验链}_flow.md` / `{核心写入链}_flow.md` 等）见末尾 §14 参考示例。

---

## 4. 顶层结构（推荐）

```markdown
# {流程名} 流程

## 1. 流程定位
   1.1 这个流程要解决什么业务问题
   1.2 涉及的接口清单（链接到 api/*.md）
   1.3 涉及的 Service 模块清单（链接到 service/*.md）
   1.4 性能 / 一致性 / 可用性 目标

## 2. 总览图（强制：ASCII 流程图）
   分步骤连线，覆盖：调用方 → 中间件 → 接口 → service → 数据访问

## 3. 详细步骤
   3.1 第 1 步：{动作}
       - 输入：{参数}
       - 处理：{Service / Redis / MySQL 操作}
       - 输出：{ctx.Set 字段 / 返回值}
       - 错误码：{失败时返回}
       - 性能：{耗时上限 / 缓存命中}
   3.2 第 2 步
   ...

## 4. 边界情况与失败处理
   4.1 每个失败点的恢复策略
   4.2 幂等性（如适用）
   4.3 并发安全

## 5. 性能优化策略（如有）
   5.1 Pipeline 优化
   5.2 短路条件
   5.3 本地缓存命中

## 6. 关联代码定位
   每个步骤对应的代码文件:行号（首次实现后填回）

## 7. 反向对照
   流程出问题时，从症状反查到代码的指南
```

---

## 5. §2 总览图（强制）

ASCII 流程图必须覆盖：

- 调用方
- 中间件链
- 路由
- Service 入口
- Service 内部步骤（如 N 步校验链）
- 数据访问（Redis / MySQL）
- 异步链路（如 Kafka）
- 返回路径

例（占位符示意）：

```
{调用方}
  │
  ▼  GET /{prefix}/{audience}/{module}/{action}?key=...
[{上游网关}]
  │
  ▼  GET (透传 query + Header)
[{目标服务}]
  │
  ▼ recovery → accessLog → Cors → Metrics
[{sign_mw} 签名校验中间件]
  │
  ├── ① 提取 key / Token / Timestamp
  ├── ② 查 EntitySecret（L1 → L2）
  ├── ③ 时间窗口校验
  ├── ④ 签名匹配
  └── ctx.Set("entityId", ...) ctx.Set("{actor-id}", ...)
  │
  ▼
[{Method-A} Controller]
  │
  ▼  调 {module}.{Method-A}(ctx, req)
[{module}.{Method-A}]
  │
  ├── ① checkParams
  ├── ② checkToken（已由中间件做，跳过）
  ├── ③ load{Entity}Info（L1 → L2）
  ├── ④ check{Entity}Status
  ├── ...
  └── ⑪ check{核心业务约束}（仅校验，不执行变更）
  │
  ▼  返回 *{Method-A}Resp 或 *ServiceError
[renderInternalResponse]
  │
  ▼  {"status": 200, "message": "success", "data": {...}}
返回
```

> 真实业务的完整流程图（按具体 Controller `{Module}.{Method}` → 下游 service 调用链替换）见末尾 §14 参考示例。

---

## 6. §3 详细步骤（核心章节）

每一步严格按照以下结构：

```markdown
### 3.N 第 N 步：{动作名}

**目标**：（一句话说明这一步在干什么）

**输入**：
- `{字段}`：{来源、类型}
- `{字段}`：{来源、类型}

**处理**：
1. {子步骤 1}
2. {子步骤 2}
3. ...

**Redis / MySQL 操作**（如有）：
\`\`\`
HGET {ns}:info:{entityId} secret_key
\`\`\`

**输出**：
- 成功：{ctx.Set / 返回值}
- 失败：{错误码 + 触发条件}

**错误码**：
| status | 触发条件 |
|--------|---------|
| 101 | Key 不存在 |
| 103 | Key 暂停 |

**性能**：
- 缓存命中：< 0.1ms
- 缓存 miss：< 2ms（Redis 单查）
- 不访问 MySQL

**短路条件**：（如适用）
当 X 时直接通过，不进入下一步
```

---

## 7. §4 边界情况与失败处理

所有"非愉快路径"都要列：

```markdown
## 4. 边界情况

### 4.1 时钟漂移
- 现象：网关与 本服务时钟差 > 60s
- 影响：签名 Timestamp 校验失败
- 缓解：NTP 同步、时间窗口 ±300s 比 60s 大
- 兜底：返回 status={N}

### 4.2 Redis 故障
- 现象：Redis 不可达
- 影响：核心业务链路不可用
- 缓解：本地缓存兜底（TTL 内仍可用）+ 熔断器降级
- 兜底：返回 status={N}（系统错误）

### 4.3 缓存与 MySQL 不一致
- 现象：cache_warmup 漏写 / flush 失败
- 缓解：cache_integrity_check 定时巡检
- 兜底：参见 cache_design.md §7

### 4.4 高并发竞争（数值变更类）
- 现象：N 个并发 `{Method}` 都通过 `{核心数值字段}` 检查，但实际可用量只够 M 个（M < N）
- 缓解：HINCRBY 原子变更，可能短期出现负值
- 后处理：<{核心数值字段}不足后续处理机制>，下次状态补充优先{冲销/补扣}
```

> 真实业务示例（按具体业务的核心约束违反场景替换）见末尾 §14 参考示例。

---

## 8. §5 性能优化策略

### 8.1 Pipeline 优化（如适用）

```markdown
**优化前**：
\`\`\`
HGET {ns}:info:{entityId} status
HGET {ns}:info:{entityId} secret_key
HGET {ns}:info:{entityId} qps_limit
\`\`\`
3 次 RTT。

**优化后**：
\`\`\`
HGETALL {ns}:info:{entityId}
\`\`\`
1 次 RTT。本地缓存命中后 0 次 RTT。

**收益量化**：单次业务校验 RTT 从 N 次降至 N 次，P99 减少约 3ms。
```

### 8.2 短路条件

```markdown
- IP 白名单为空 → 跳过 ⑥ 步直接通过
- partnerId=0 → 跳过分支
- mode=none → 跳过状态变更校验
```

---

## 9. §6 关联代码定位

实现完成后填回（设计阶段为空）：

```markdown
## 6. 关联代码定位

| 步骤 | 文件 | 函数 |
|------|------|------|
| 中间件 {sign_mw} | `middleware/{sign_mw}.go` | `{SignMw}()` |
| ① checkParams | `service/{auth-module}/checker.go:23` | `checkParams()` |
| ② checkToken | （由中间件做） | — |
| ③ loadEntityInfo | `service/{auth-module}/cache_loaders.go:15` | `LoadEntityInfo()` |
| ... |
```

这一节让"流程图 → 代码"的反查变得机械化。

---

## 10. §7 反向对照（FAQ 式）

```markdown
## 7. 反向对照（出问题时怎么查）

| 症状 | 查哪一节 | 查哪段代码 |
|------|---------|-----------|
| 签名总是失败 | §3.2 第 2 步 + §4.1 时钟漂移 | `middleware/{sign_mw}.go::checkSignature` |
| 状态显示 `{某错误码}` 但 Key 实际有效 | §3.3 第 3 步缓存加载 | `service/{auth-module}/cache_loaders.go::Load{Entity}Info` |
| QPS 限流总误触发 | §3.10 第 ⑩ 步 + cache_design.md §2.3 | `service/{auth-module}/checker.go::checkRateLimit` |
| {核心数值字段}已变更但 {辅助字段}还有 | §3.N 业务决策步骤 | `service/{module-N}/{action}.go::{决策函数}` |
```

> 真实业务的反向对照（含 `{核心写入动作}已完成` / `{模式枚举字段}` 等）见末尾 §14 参考示例。

---

## 11. 流程文档与其他文档的关系

| 内容 | flow.md | api/*.md | service/*_service.md | cache_design.md |
|------|---------|----------|----------------------|------------------|
| 业务问题描述 | ✅ 完整 | 简略 | 不写 | 不写 |
| 流程图 | ✅ 完整 | 不写 | 简略 | 不写 |
| 接口字段定义 | 不写 | ✅ 完整 | 不写 | 不写 |
| Service 方法签名 | 引用 | 引用 | ✅ 完整 | 不写 |
| Redis Key 定义 | 引用 | 不写 | 引用 | ✅ 完整 |
| 步骤伪码 | ✅ 完整 | 不写 | ✅ 完整（同步） | 不写 |

**步骤伪码** 在 flow.md 和 service_design.md **都写**，但视角不同：

- flow.md 视角：从输入到输出的完整跨模块串联
- service_design.md 视角：单个 service 模块内部的所有方法

两者**相互呼应**——任何一侧改了，另一侧必须同步。

---

## 12. 维护

| 触发 | 动作 |
|------|------|
| 新增链路步骤 | §3 新增子节 + §2 总览图更新 + §6 关联代码 |
| 修改步骤实现 | §3 对应子节 + 受影响的 service / cache 文档 |
| 改变性能目标 | §1.4 + §5 优化策略 |

---

## 13. 当前阶段简化（如有）

如果某条流程在当前阶段做了简化（如 某些工程的 `check{DisabledModule}` 空壳实现），必须在该步骤显式标注：

```markdown
### 3.8 第 ⑧ 步：check{DisabledModule}

> ⛔ **当前阶段简化**：本步骤实现为**空壳直接 return nil**，不做{受限模块}判定（{disabled_module} 模块整体不实现，见 [`BUILD_STATUS.md`](../BUILD_STATUS.md) §0）。完整设计见 §3.8.2，启用时按完整设计恢复。

#### 3.8.1 当前实现（空壳）
\`\`\`go
func (c *checker) check{DisabledModule}(ctx *gin.Context, ...) *ServiceError {
    return nil
}
\`\`\`

#### 3.8.2 启用后的完整实现（设计保留）
（完整伪码）
```

让 AI 在不同阶段都能正确选择实现方式。

---

## 14. 参考示例（仅示意，落地按业务替换）

> ⚠️ 以下为示意，规范本体（§1-§13）已用占位符表达；本节给出具体业务（业务校验链 / 状态变更闭环）的示例，便于读者建立对应直觉。落地工程时按业务语义替换占位符，不得直接照搬本节具体业务名作为规范。
>
> 示例服从 [`PRINCIPLES.md §12 占位符规则`](../PRINCIPLES.md#12-占位符规则强制--双轨结构) 的"参考示例段豁免"。

### 14.1 §3 标准命名（示例）

| 流程类型 | 文件名 |
|---------|--------|
| 核心业务链路 | `auth_flow.md` |
| 业务闭环 | `billing_flow.md` |
| 注册/登录 | `account_flow.md` |

### 14.2 §5 总览图（示例：业务校验链 Verify Controller）

```
外部用户
  │
  ▼  GET /api/internal/auth/verify?key=...
[上游网关]
  │
  ▼
[签名校验中间件 sign_mw]
  │
  ├── ① 提取 key / Token / Timestamp
  ├── ② 查 EntitySecret（L1 → L2）
  ├── ③ 时间窗口校验
  ├── ④ 签名匹配
  └── ctx.Set("entityId", ...) ctx.Set("userId", ...)
  │
  ▼
[Verify Controller]
  │
  ▼  调 auth.Verify(ctx, req)
[auth.Verify]
  │
  ├── ① checkParams
  ├── ② checkToken（中间件已做）
  ├── ③ loadEntityInfo（L1 → L2）
  ├── ④ checkKeyStatus
  ├── ...
  └── ⑪ checkBilling（只检测余额，不扣费）
  │
  ▼  返回 *VerifyResp 或 *ServiceError
[renderInternalResponse]
  │
  ▼  {"status": 200, "message": "success", "data": {...}}
返回
```

### 14.3 §7.4 高并发超扣（示例：业务校验链中的余额扣减）

- 现象：100 个并发 Verify 都通过余额检查，但实际只够 50 个
- 缓解：HINCRBY 原子扣减，可能短期负余额
- 后处理：<余额不足后续处理机制>，下次状态补充优先扣

### 14.4 §10 反向对照（示例）

| 症状 | 查哪一节 | 查哪段代码 |
|------|---------|-----------|
| 余额已扣但额度还有 | §3.N 业务决策步骤 | `service/billing/report.go::pickMode` |

### 14.5 占位符 → 业务名 对照表（示例）

| 占位符 | 本节示例值 |
| --- | --- |
| `{核心业务链路}_flow.md` | `auth_flow.md` |
| `{业务闭环}_flow.md` | `billing_flow.md` |
| `{Method-A}` | Verify |
| `{Method-A}Resp` | VerifyResp |
| `{module}.{Method-A}` | auth.Verify |
| `{actor-id}` / `{userId}` | userId |
| `{核心数值字段}` | balance |
| `{决策函数}` | pickMode |
| `{module-N}/{action}.go` | billing/report.go |



---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
