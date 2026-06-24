# 07 - API 接口详细设计与实现规范

> 规定 `docs/api/{audience}_interfaces.md` 的内容结构。每个调用方一份独立 md。

---

## 1. 定位

API 接口文档是 Controller / Service 入参出参 / 请求校验 / 错误码映射的**唯一真相源**。

精度要求：**字段级 + binding tag 级**。AI 仅凭这一篇就能：

1. 生成 Controller handler（含参数绑定 + 校验 + service 调用）
2. 生成对应的 Service 入参/出参 struct（不含 binding tag）
3. 生成测试用例的请求构造和断言

---

## 2. 文档拆分原则

按**调用方**（audience）拆分，**不按业务模块**：

| 调用方 | 典型路由前缀 | 文档名 |
|--------|------------|-------|
| 内部网关 / 服务间调用 | `/{prefix}/internal/*` | `audience_a_interfaces.md` |
| 前端 BFF / 调用方自助 | `/{prefix}/operator/*` | `audience_b_interfaces.md` |
| 内部管理后台 BFF | `/{prefix}/admin/*` | `audience_c_interfaces.md` |

每份 md 独立，**不互相引用接口规格**。

---

## 3. 顶层结构（强制）

每个 audience 的接口规格 md 必须按以下章节顺序组织：

- `## 1.` 概述（调用场景 / 路由前缀 / 返回格式 / 业务校验方式 / 上游集成设计如有）
- `## 2..N.` 各业务模块章节，每个接口下含 10 个子节：
  - `2.X.1` 请求方法 + 路径
  - `2.X.2` 请求参数说明与校验规则
  - `2.X.3` Go 请求结构体（Controller 层 / Service 层）
  - `2.X.4` 响应结构（成功 / 失败示例）
  - `2.X.5` Go 响应结构体
  - `2.X.6` 错误码映射
  - `2.X.7` Controller 伪代码（关键步骤）
  - `2.X.8` Service 调用关系
  - `2.X.9` 业务规则（如适用）
  - `2.X.10` 边界情况
- `## N+1.` 安全注意事项 / `## N+2.` 性能注意事项

> **完整章节骨架见** [`templates/docs/api/audience_a_interfaces.md`](../templates/docs/api/audience_a_interfaces.md) / [`audience_b_interfaces.md`](../templates/docs/api/audience_b_interfaces.md) / [`audience_c_interfaces.md`](../templates/docs/api/audience_c_interfaces.md)。下面 §4-§7 给出每个子节的填充精度与必填要素。

---

## 4. §1 概述（强制必填）

### 4.1 调用场景

```markdown
## 1. 概述

Internal 接口供 上游网关在处理每个 API 请求时调用，核心场景：

1. **业务校验验证**：每次 API 请求前调用，判断是否放行（只检测，不扣费）
2. **{核心写入动作}**：API 请求完成后调用，执行实际数值变更并记录业务事件流水
3. **产品查询**：网关启动时或定期同步产品目录信息

所有接口通过签名认证（`{sign_mw}` 中间件），不走调用方身份会话态。

**路由前缀**：`/{prefix}/internal`
```

### 4.2 返回格式（强制）

显式给出 JSON 结构 + 错误格式：

```markdown
**返回格式**：所有 Internal 接口统一使用 `{status, message, data}` 三字段结构（与 Operator/Admin 的 `{errNo, errStr, data}` 不同），Controller 层通过自定义 `renderInternalResponse` 方法输出。

\`\`\`json
{
  "status": 200,
  "message": "success",
  "data": {}
}
\`\`\`

**错误返回**：service 层返回 `*ServiceError`，Controller 层提取 `StatusCode → status`、`Message → message`，`data` 置为 `null`。
```

### 4.3 业务校验方式（强制）

签名算法、必填参数都要写到。一个例子：

```markdown
**签名方式**：

| 参数 | 位置 | 类型 | 必填 | 说明 |
|------|------|------|------|------|
| key | GET query | String | 是 | EntityID |
| Token | Header | String | 是 | `{SignatureFormula}` 32 位大写 hex |
| Timestamp | Header | String | 是 | Unix 时间戳，±300s |

`EntitySecret` 存储在 `{example_table_entity}` 表中。中间件校验时通过 `key` 从本地缓存/Redis（`{ns}:info:{entityId}` Hash）查得 `EntitySecret`。
```

### 4.4 上游集成设计（如有 BFF）

涉及 Cookie/Session 时，必须显式说明：

- BFF 与 目标服务的职责边界
- Cookie 字段（名字、有效期、scope）
- Session 存储位置
- {身份生命周期}流程
- 超时与续签

---

## 5. §N 单个接口的完整规格

> ⚠️ §5 的子节内容（路径、字段表、Go struct、JSON 取值、错误码描述等）均为**结构性占位示意**——`{action}` / `{Method-N}Req` / `{Field-N}` / `{Status-N}` 等是占位符，可由具体业务名替换。规范本体的**强制要求**是子节顺序、表格列、Go struct 必带 JSON tag 这些**结构性约束**。具体业务示例（含真实字段名 / 错误码 / JSON 取值）统一收录在末尾 §10 参考示例。

### 5.1 N.1.1 请求方法 + 路径

```markdown
### 2.1 请求校验（核心接口，高频调用）

\`\`\`
GET /{prefix}/internal/{module}/{action}
\`\`\`

> ⚠️ 修订标注（如有）：从 POST + JSON 改为 GET + Query。原因：...
```

### 5.2 N.1.2 请求参数说明与校验规则（强制）

```markdown
#### 2.1.2 请求参数说明与校验规则

| 字段 | 位置 | 类型 | 必填 | 校验规则 | 说明 |
|------|------|------|------|---------|------|
| key | Query | string | 是 | `^ak_[0-9a-f]{16}$`，长度固定 19 | 客户端传的 EntityID |
| Token | Header | string | 是 | `^[0-9A-F]{32}$`，长度固定 32，大写 hex | 签名 Token |
| Timestamp | Header | string | 是 | 纯数字，10 位 Unix 时间戳，`|now-ts| ≤ 300s` | 时间戳 |
| routePath | Query | string | 是 | 以 `/` 开头，长度 ≤ 200 | 被调用接口的路由路径 |
| clientIp | Query | string | 是 | 合法 IPv4 或 IPv6 | 客户端真实 IP |
| requestMethod | Query | string | 否 | 枚举：`"GET"`/`"POST"`，默认 `"GET"` | 调用方式 |
| partnerId | Query | int64 | 否 | `≥ 0`，0 或不传表示无企业维度 | 业务用 |

**校验失败统一返回 `status: {N}`（请求参数错误）**。中间件层先校验 `key/Token/Timestamp`，service 层 `checkParams` 校验剩余字段。
```

**必填列**：

- 字段名（一字不差，含大小写）
- 位置（Query / Header / Body）
- 类型（string / int / int64 / bool）
- 必填（是 / 否 / 条件）
- 校验规则（正则 / 长度 / 范围 / 枚举）
- 说明（业务含义）

### 5.3 N.1.3 Go 请求结构体（强制）

```markdown
#### 2.1.3 Go 请求结构体

\`\`\`go
// service/{auth-module}/types.go
type {Method-N}Req struct {
    {QueryField}     string `json:"{json-key}"`        // 来自 ctx.Query("...")
    {HeaderField}    string `json:"{json-key}"`        // 来自 Header
    {TimestampField} string `json:"{json-key}"`        // 来自 Header
    // ... 其余字段
}
\`\`\`

> Controller 从 `ctx.Query(...)` 和 `ctx.GetHeader(...)` 直接构造结构体；不使用 `ShouldBindJSON`（GET 无 body）。

> 真实业务示例（按具体 Method 替换 `{Method-N}Req`）见末尾 §10 参考示例。
```

**对于 POST + JSON 请求**，binding tag 必须显式（占位符示意）：

```go
type {Method-N}Req struct {
    {Field-1}     string `json:"{field-1}"     binding:"required,{rule-1}"`
    {Field-2}     string `json:"{field-2}"     binding:"required,min=N,max=M"`
    {Field-3}     string `json:"{field-3}"     binding:"required,len=N"`
}
```

> 真实业务的 binding tag 完整示例见末尾 §10 参考示例（按具体 `{Module}` 的 `{Method-N}` 替换 `{Module}{Method-N}Req`）。

### 5.4 N.1.4 响应结构（强制：成功 + 失败两个 JSON 示例）

```markdown
#### 2.1.4 响应结构

**业务校验通过（status = 200）**：

\`\`\`json
{
  "status": 200,
  "message": "success",
  "data": {
    "{entity-id}": "{prefix-A}{date}{seq}",
    "entityId": "{ak-prefix}{hex16}",
    "mode": "{enum-value}",
    "price": 0.00,
    "effective_price": 0.00,
    "remainAmount": 0
  }
}
\`\`\`

**业务校验失败（status ≠ 200）**：

\`\`\`json
{
  "status": 101,
  "message": "资源/凭证无效或未生效",
  "data": null
}
\`\`\`

**特殊状态（如 status = 105，接口已下线）**：
不变更状态。`data` 为 `null`。
```

### 5.5 N.1.5 Go 响应结构体（强制）

占位符示意：

```go
type {Method-N}Resp struct {
    {Field-1}  string  `json:"{field-1}"`
    {Field-2}  string  `json:"{field-2}"`
    {Field-3}  string  `json:"{field-3}"`  // 枚举：{enum-A} / {enum-B} / {enum-C}
    {Field-4}  float64 `json:"{field-4}"`
    {Field-5}  int64   `json:"{field-5}"`
}
```

每个字段必有 JSON tag。**字段顺序与 JSON 示例一致**，方便对照。

> 真实业务的完整字段示例（按具体 Method 替换 `{Method-N}Resp`）见末尾 §10.6 参考示例。

### 5.6 N.1.6 错误码映射（强制）

```markdown
#### 2.1.6 错误码映射

| status | 含义 | 触发条件 |
|--------|------|---------|
| 101 | Key 无效 | Key 不存在或状态=2/3 |
| 103 | Key 暂停 | Key 状态=2 |
| 107 | 签名错误 | Token 不匹配 |
| 109 | {核心数值字段}不足 | {amount-field} < {单次操作开销} 且 {辅助配额} 不足 |
| 115 | 时间戳过期 | `|now-ts| > 300s` |
| 116 | 功能日调用上限达到 | {rl}:daily 计数超阈值 |
| 122 | QPS 超限 | {rl}:qps 计数超阈值 |
| 125 | 参数错误 | 任一必填字段缺失或格式错 |
```

每个错误码对应**触发条件**，AI 写代码时直接对应到 if 分支。

### 5.7 N.1.7 Controller 伪代码（强制）

```markdown
#### 2.1.7 Controller 伪代码

\`\`\`go
// controllers/http/{audience}/{module}/{action}.go
func {Method-N}(ctx *gin.Context) {
    // 1. 从 query / header / body 构造 req
    req := &{module}.{Method-N}Req{
        {QueryField}:     ctx.Query("{json-key}"),
        {HeaderField}:    ctx.GetHeader("{Header-Name}"),
        {TimestampField}: ctx.GetHeader("{Header-Name}"),
        // ...
    }

    // 2. 调 service
    resp, err := {module}Svc.{Method-N}(ctx, req)

    // 3. 渲染响应（按 audience 选择）
    renderInternalResponse(ctx, err, resp)
}
\`\`\`
```

> 真实业务的 Controller 伪代码（按具体 Controller `{Module}.{Method-N}` 替换）见末尾 §10 参考示例。

**必填**：
- 参数绑定方式（`ShouldBindJSON` / `ShouldBindQuery` / 手动构造）
- 调 service 的方法名
- 渲染函数

### 5.8 N.1.8 Service 调用关系（强制）

```markdown
#### 2.1.8 Service 调用关系

- **Service 方法**：`{module}.{Method}(ctx, req) (*{Method}Resp, *ServiceError)`
- **设计文档**：[`docs/service/{auth-module}_service.md`](../service/{auth-module}_service.md) §3
```

### 5.9 N.1.9 业务规则（如适用）

```markdown
#### 2.1.9 业务规则

- 业务校验通过 status=200 → 进入业务链路
- 业务校验失败 status≠200 → **不变更状态**（{核心写入动作}不会被网关调用）
- 105 接口下线、115 时间戳过期等也不变更状态
- 122 QPS 超限**不变更状态**
```

### 5.10 N.1.10 边界情况

```markdown
#### 2.1.10 边界情况

- 时钟漂移：网关与 目标服务时钟差异 > 60s 时签名失败 → 必须保证 NTP 同步
- 高并发{操作}超出预期：{核心数值字段}变更用 HINCRBY 原子操作，可能短期出现负值 → 后续通过<{核心数值字段}不足后续处理机制>补
- 缓存与 MySQL 不一致：详见 cache_design.md §7 巡检任务
```

---

## 6. 性能 / 安全章节

文末添加：

```markdown
## N. 安全注意事项

- 签名时间窗口 ±300s，避免重放
- EntityID + EntitySecret 不可日志输出（参考 logging.md §敏感数据）
- 内网 IP 校验由中间件兜底
- 频率限流由 L1/L2/L3 三层联防

## N+1. 性能注意事项

- 核心业务链路 P99 < {N}ms 目标
- 业务校验 0 次 MySQL 查询，全走 Redis + 本地缓存
- Pipeline 优化（如适用）
```

---

## 7. 不允许的写法

| 反例 | 修正 |
|------|------|
| "请求参数：调用方传 {field-1} 和 {field-2}" | 用表格列出，含校验规则 |
| 响应结构只画 success 不画 fail | 必须画两个 JSON 示例 |
| Go struct 没有 JSON tag | 每个字段必有 `json:"..."` |
| 校验规则写"邮箱格式" | 写 `binding:"required,email"` 或正则 |
| 错误码"自行根据业务判断" | 必须列出每个错误码的触发条件 |
| Controller 伪代码省略调 service 步骤 | 必须显式 |

---

## 8. 与 service_design.md 的关系

| 内容 | api/*.md | service/*_service.md |
|------|----------|---------------------|
| 请求/响应 JSON | 完整 | 不写 |
| Go 请求 struct（含 binding tag） | 完整 | 不写 |
| Go 响应 struct | 完整 | 引用 |
| Controller 伪代码 | 完整 | 不写 |
| Service 方法签名 | 引用（一行） | 完整 |
| Service 步骤伪代码 | 不写 | 完整 |

**禁止重复**。同一字段 struct 只在 api 文档中定义，service 文档引用。

---

## 9. 维护

| 触发 | 动作 |
|------|------|
| 新增接口 | 新增 §N 子节，含全部 8-10 个子项 |
| 修改字段 | §N.1.2 校验规则表 + §N.1.3 Go struct + 受影响的 Controller / Service |
| 删除接口 | 整节移除 + 通知 routing.md / service_design.md |
| 修改错误码 | §N.1.6 错误码映射表 + status_codes.md 同步 |

---

## 10. 参考示例（仅示意，落地按业务替换）

> ⚠️ 以下为示意，规范本体（§1-§9）已用占位符表达；本节给出具体业务（账号注册类操作 / 业务校验签名链）的示例，便于读者建立对应直觉。落地工程时按业务语义替换，不得直接照搬本节具体业务名作为规范。
>
> 示例服从 [`PRINCIPLES.md §12 占位符规则`](../PRINCIPLES.md#12-占位符规则强制--双轨结构) 的"参考示例段豁免"。

### 10.1 §5.3 POST + JSON 请求 binding tag（示例：账号注册类）

```go
type RegisterReq struct {
    Account     string `json:"account"     binding:"required,email,max=64"`
    Password    string `json:"password"    binding:"required,min=8,max=64"`
    VerifyCode  string `json:"verifyCode"  binding:"required,len=6"`
}
```

### 10.2 §5.4 响应结构 JSON（示例：业务校验通过）

```json
{
  "status": 200,
  "message": "success",
  "data": {
    "userId": "ACC20260407001",
    "entityId": "ak_a1b2c3d4e5f6g7h8",
    "mode": "balance",
    "price": 0.00,
    "effective_price": 0.00,
    "remainAmount": 0
  }
}
```

### 10.3 §5.6 错误码映射（示例：业务校验链）

| status | 含义 | 触发条件 |
|--------|------|---------|
| 109 | 余额不足 | balance < 单次扣减 且 quota 不足 |

### 10.4 §5.10 边界情况（示例：余额扣减）

- 高并发超扣：余额扣减用 HINCRBY 原子操作，可能短期出现负余额 → 后续通过<余额不足后续处理机制>机制补

### 10.7 §5.3 Go 请求结构体（示例：业务校验链 VerifyReq）

```go
// service/auth/types.go
type VerifyReq struct {
    Key       string `json:"key"`        // 来自 ctx.Query("key")
    Token     string `json:"token"`      // 来自 Header
    Timestamp string `json:"timespan"`   // 来自 Header
    ApiPath   string `json:"routePath"`
    ClientIP  string `json:"clientIP"`
    Method    string `json:"method"`
    RequestId string `json:"requestId"`
    CompanyId int64  `json:"partnerId"`
}
```

### 10.8 §5.7 Controller 伪代码（示例：业务校验链 Verify）

```go
// controllers/http/internal/auth/verify.go
func Verify(ctx *gin.Context) {
    req := &auth.VerifyReq{
        Key:       ctx.Query("key"),
        Token:     ctx.GetHeader("Token"),
        Timestamp: ctx.GetHeader("Timestamp"),
        ApiPath:   ctx.Query("routePath"),
        ClientIP:  ctx.Query("clientIp"),
        Method:    ctx.DefaultQuery("requestMethod", "GET"),
        RequestId: ctx.Query("requestId"),
    }
    if companyIdStr := ctx.Query("partnerId"); companyIdStr != "" {
        req.CompanyId, _ = strconv.ParseInt(companyIdStr, 10, 64)
    }
    resp, err := authSvc.Verify(ctx, req)
    renderInternalResponse(ctx, err, resp)
}
```

### 10.5 占位符 → 业务名 对照表（示例）

| 占位符 | 本节示例值 |
| --- | --- |
| `{Method-N}Req` | RegisterReq |
| `{Field-N}` | Account / Password / VerifyCode |
| `{entity-id}` | userId |
| `{prefix-A}{date}{seq}` | `ACC20260407001` |
| `{ak-prefix}{hex16}` | `ak_a1b2c3d4e5f6g7h8` |
| `{核心数值字段}` | balance |
| `{amount-field}` | balance |
| `{单次操作开销}` | 单次扣减 |
| `{辅助配额}` | quota |

### 10.6 §5.5 Go 响应结构体（示例：业务校验链 VerifyResp）

```go
type VerifyResp struct {
    UserId         string  `json:"userId"`
    EntityID       string  `json:"entityId"`
    Mode           string  `json:"mode"`  // "quota" / "balance" / "company"
    Price          float64 `json:"price"`
    Rate           float64 `json:"rate"`
    EffectivePrice float64 `json:"effective_price"`
    RemainAmount   int64   `json:"remainAmount"`
    DailyRemain    int64   `json:"dailyRemain"`
}
```




---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
