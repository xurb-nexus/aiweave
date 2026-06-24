# 13 - 日志规范文档规范

> 规定 `docs/architecture/logging.md` 的内容结构。日志规范是跨整个工程的基础约束。

---

## 1. 定位

日志文档要让 AI 能回答：

1. 用什么日志库（如 zap、tlog 封装）
2. Sugared API（Printf 风格）vs Structured API（field 风格）的取舍
3. 日志级别使用规则（DEBUG / INFO / WARN / ERROR / FATAL）
4. 通用字段约定（哪些字段必须出现、key 名一致性）
5. 敏感数据脱敏规则
6. 各模块的日志规范（业务校验 / 中间件 / 定时任务 / Service）

---

## 2. 顶层结构

```markdown
# 日志规范

## 1. 日志框架概述
   1.1 框架选型
   1.2 与 zap / logrus 的关系

## 2. 日志级别使用规范（强制）
   2.1 DEBUG / INFO / WARN / ERROR / FATAL 各自的语义
   2.2 自愈 vs 需人工介入的判定

## 3. 两种 API 风格
   3.1 Sugared API（禁用）
   3.2 Structured API（强制）
   3.3 迁移指南

## 4. 上下文字段与 Notice 机制
   4.1 ctx 上下文字段（traceId、{actor-id} 等）
   4.2 Notice 字段（额外信息）

## 5. Field 构造器完整列表
   tlog.String / Int / Int64 / Float / Bool / Time / Error / ...

## 6. 各模块日志规范
   6.1 定时任务日志
   6.2 核心业务链路日志
   6.3 中间件日志
   6.4 Service 层日志
   6.5 Redis / MySQL 故障日志

## 7. 敏感数据脱敏（强制）

## 8. 错误字段固定 key

## 9. 日志输出文件
```

---

## 3. §2 日志级别（强制）

```markdown
## 2. 日志级别使用规范

| 级别 | 语义 | 触发示例 | 后果 |
|------|------|---------|------|
| DEBUG | 调试细节 | 步骤进入/退出 | 仅开发环境开启 |
| INFO | 业务正常事件 | "task completed"、"cache warmed up" | 生产环境保留 |
| WARN | 可恢复异常 | Redis 写失败但 MySQL 已成功；本地缓存 miss 回源 Redis 也 miss | 不影响主流程 |
| ERROR | 不可恢复失败 | MySQL 写失败导致业务返回错误；外部 API 熔断回退 | 影响主流程，需人工介入 |
| FATAL | 进程必须退出 | 启动时初始化资源失败 | log.Fatal 触发 os.Exit(1) |

**判定规则**：

- 能自愈 → WARN
- 需人工介入 → ERROR

**核心热路径不打日志**（每秒上万次{核心校验类调用}，打日志会成为瓶颈）。
```

---

## 4. §3 两种 API 风格（强制）

### 4.1 必须使用 Structured API

```markdown
### 3.2 Structured API（强制）

\`\`\`go
// 正确 ✅
tlog.InfoLogger(ctx, "task completed",
    tlog.String("task", taskName),
    tlog.Int("processed", count))

tlog.WarnLogger(ctx, "redis write failed after mysql success",
    tlog.String("redisKey", key),
    tlog.String("error", err.Error()))

tlog.ErrorLogger(ctx, "db select failed",
    tlog.String("table", "{example_table_user}"),
    tlog.String("error", err.Error()))
\`\`\`
```

### 4.2 必须禁用 Sugared API

```markdown
### 3.1 Sugared API（禁用）

\`\`\`go
// 错误 ✗ — 字段嵌在字符串里，无法结构化检索
tlog.Infof(ctx, "task completed: %s, processed=%d", taskName, count)
tlog.Warnf(ctx, "redis write failed: key=%s, err=%v", key, err)
\`\`\`

**原因**：
- 字段无法被 ELK / Grafana Loki 等结构化检索
- 关键值（如 traceId、{actor-id}）混在字符串里，告警规则难以匹配
- 性能：fmt.Sprintf 比直接 append field 慢
```

---

## 5. §6 各模块日志规范（强制示例）

```markdown
## 6. 各模块日志规范

### 6.1 定时任务

\`\`\`go
tlog.InfoLogger(ctx, "flush_state_to_db start")

tlog.InfoLogger(ctx, "flush_state_to_db completed",
    tlog.Int("processed", n),
    tlog.String("duration", elapsed.String()))

tlog.WarnLogger(ctx, "flush entry failed, will retry next round",
    tlog.String("entry", entry),
    tlog.String("error", err.Error()))
\`\`\`

### 6.2 核心业务链路（性能敏感，谨慎打日志）

- ✅ 业务校验失败 → INFO 一行（含 status 码 + 失败原因）
- ✗ 业务校验成功 → 不打日志（依赖 accessLog 中间件）
- ✗ 不要在核心热路径调 tlog.Debugf

### 6.3 中间件

\`\`\`go
// 中间件本身不打详细日志，依赖 accessLog 中间件
// 仅在异常分支打 WARN：
tlog.WarnLogger(ctx, "{user_mw}: client ip not internal",
    tlog.String("clientIp", ctx.ClientIP()))
\`\`\`

### 6.4 Service 层

\`\`\`go
tlog.InfoLogger(ctx, "{Method} {module}",
    tlog.String("{module}", req.{Field}))

tlog.ErrorLogger(ctx, "{Method} failed: db insert",
    tlog.String("{module}", req.{Field}),
    tlog.String("error", err.Error()))
\`\`\`

### 6.5 Redis / MySQL 故障

\`\`\`go
// Redis 写失败但 MySQL 已成功
tlog.WarnLogger(ctx, "redis write failed after mysql success",
    tlog.String("redisKey", key),
    tlog.String("error", err.Error()))

// MySQL 写失败
tlog.ErrorLogger(ctx, "db update failed",
    tlog.String("table", "{example_table_user}"),
    tlog.String("error", err.Error()))
\`\`\`
```

---

## 6. §7 敏感数据脱敏（强制）

```markdown
## 7. 敏感数据脱敏

**禁止打印的字段**（示例清单，落地按业务补充）：

- `secret` / `password` / `passwordHash`
- `token`（除非明确为公开 token）
- 一次性验证码字段（如 `otp` / `2fa-code`）
- 完整身份证号 / 银行卡号 / 手机号

**正确做法**：

\`\`\`go
// 错误 ✗
tlog.InfoLogger(ctx, "{Method}",
    tlog.String("password", req.Password))

// 正确 ✅
tlog.InfoLogger(ctx, "{Method}",
    tlog.String("{module}", req.{Field}))
\`\`\`

如必须输出标识用途，做掩码处理：

\`\`\`go
tlog.InfoLogger(ctx, "send sms",
    tlog.String("phoneMasked", maskPhone(req.Phone)))  // 138****1234
\`\`\`
```

---

## 7. §8 错误字段固定 key（强制）

```markdown
## 8. 错误字段固定 key

错误信息**必须**用 `error` 作为 key：

\`\`\`go
// ✅
tlog.ErrorLogger(ctx, "x failed", tlog.String("error", err.Error()))

// ✗ 不要用 "err" / "reason" / "msg" 等变体
tlog.ErrorLogger(ctx, "x failed", tlog.String("err", err.Error()))
\`\`\`

**原因**：监控告警规则按 key 匹配。统一 key 名才能写一条规则覆盖所有错误。
```

---

## 8. §9 日志输出文件

```markdown
## 9. 日志输出文件

| 文件 | 内容 | 切割策略 |
|------|------|---------|
| log/{project}.log | INFO 及以上正常日志 | 按日切割，保留 7 天 |
| log/{project}.log.wf | WARN / ERROR 日志 | 按日切割，保留 30 天 |
| log/{project}.log.access | accessLog 中间件输出 | 按日切割，保留 7 天 |

**.wf 后缀约定**：业内常见的"warning/fatal"分流约定，便于告警系统单独抓取异常流。
```

---

## 9. §4 上下文字段（强制）

```markdown
## 4. 上下文字段与 Notice 机制

### 4.1 自动注入字段

`ctx *gin.Context` 中如有以下字段，tlog 自动注入到 log 行：

- `traceId` —— 由 metrics / accessLog 中间件生成
- `{actor-id}` —— 由 {actor_mw} / 签名校验中间件注入
- `entityId` —— 由 {sign_mw} 注入

业务代码**不需要**手动添加这些字段。

### 4.2 Notice 字段

如需在某次请求的整个生命周期中累积上下文：

\`\`\`go
// 中间件或 service 入口
ctx.Set("notice.opId", req.ActorId)
ctx.Set("notice.module", "{module}")

// 后续 tlog.* 自动包含 opId、module 字段
\`\`\`

约定：`notice.` 前缀的 ctx 值会被 tlog 自动展开为 field。
```

---

## 10. §5 Field 构造器完整列表（必填）

```markdown
## 5. Field 构造器完整列表

| 函数 | 入参 | 用途 |
|------|------|------|
| `tlog.String(key, val string)` | 字符串 | 大多数场景 |
| `tlog.Int(key, val int)` | int | 计数 / 状态码 |
| `tlog.Int64(key, val int64)` | int64 | timestamp / id |
| `tlog.Float(key, val float64)` | float64 | 金额 / 比率 |
| `tlog.Bool(key, val bool)` | bool | 布尔 |
| `tlog.Time(key, val time.Time)` | time.Time | 时间字段 |
| `tlog.Duration(key, val time.Duration)` | duration | 耗时 |
| `tlog.Error(err error)` | error | 自动转为 String("error", err.Error()) |
| `tlog.Any(key, val interface{})` | 任意类型 | 复杂结构（避免在生产用） |

**Field 顺序**：与 message 强相关的字段在前，元信息字段在后。
```

---

## 11. 与其他文档的关系

| 内容 | logging.md | helpers_api.md | constants.md |
|------|-----------|----------------|--------------|
| 日志级别规范 | ✅ 完整 | 不写 | 不写 |
| Field 构造器签名 | ✅ 完整 | 引用 | 不写 |
| ctx 字段约定 | ✅ 完整 | 不写 | 不写 |
| 日志切割阈值 | 简略 | 不写 | ✅（如 logBufferSize 等） |

---

## 12. 维护

| 触发 | 动作 |
|------|------|
| 新增日志风格 | §3 规范 + 各模块示例同步 |
| 新增受保护字段（敏感） | §7 列表 |
| 调整级别判定规则 | §2 表 + §6 各模块同步 |
| 修改 ctx 自动注入字段 | §4 + 中间件文档同步 |
| `service` / `data` 层出现自打日志（代码 → md 反向同步）| 按 §14 改为 `return error` 上传、`controller` 定级；§14 登记槽位同步，回链 `design-spec/06 §3.3 / §9.3` |

---

## 13. 与 23_observability 的边界划分

13 与 23 都涉及"可观测性"，但承载不同维度。本节明确切分，避免双源真相：

| 内容 | 13_logging.md（本文档） | 23_observability.md |
| --- | --- | --- |
| 结构化日志 API（`tlog.*` / 字段约定 / 级别判定） | ✅ 真相源 | 不写 |
| `trace_id` / `request_id` 串联（构成 Tracing） | ✅ 真相源 | 引用 |
| 敏感数据脱敏 | ✅ 真相源 | 不写 |
| Metric 注册 / cardinality / label 约束 | 不写 | ✅ 真相源 |
| 服务级告警规则（指标级） | 不写 | ✅ 真相源 |
| 日志级告警（关键错误日志触发） | ✅ 真相源 | 不写 |

**23 强制策略**：**禁止用 Metric 替代日志做"{业务事件}埋点"**——{业务事件}走本文档的结构化日志 + 离线分析；Metric 仅承载"服务级运行状况"。

本文档中涉及"业务事件埋点"的描述时，必须显式指明"用 `tlog.*` 结构化日志"，不要引导读者去用 Metric 实现。

---

## 14. §N 分层日志纪律 / 数据层禁日志（强制）

> 本节与 §2「核心热路径不打日志」**并列且理由不同**：§2 是**性能**理由（高频调用，打日志成瓶颈）；本节是**分层与上下文归属**理由（日志是横切关注点，归请求边界）。两条都成立，互不替代，落地按业务分别套用。
>
> 本节是**表示记录槽位**——只规定"该怎么把这条纪律写进 `logging.md`"。纪律本身的方法论真相源在 [`design-spec/06 §3.3`](../design-spec/06_resilience_design.md)（分层依赖单向 + 数据层禁日志）与 [`design-spec/06 §9.3`](../design-spec/06_resilience_design.md)（数据层禁日志的理由），本节只回链不复制（双源真相规避）。

### 14.1 纪律表示（该怎么写下来）

`logging.md` 必须用如下结构登记这条分层纪律（占位符化主体，落地按业务替换层与方法名）：

```markdown
## N. 分层日志纪律（数据层禁日志）

| 层 | 是否打日志 | 职责 |
|----|-----------|------|
| `controller`（请求边界） | ✅ 唯一记日志层 | 收到下层 `return error` 后，按"自愈 → WARN / 需人工 → ERROR"定级别记一次 |
| `service` | ✗ 禁打日志 | 只 `return error` 上传，不自记 |
| `data` | ✗ 禁打日志 | 只 `return error` 上传，不自记 |

判定规则：底层（`service` / `data`）一律 `return error` 向上传递，由 `controller` 在请求边界**统一**决定记 WARN（可自愈）还是 ERROR（需人工介入）。
```

### 14.2 这条纪律的理由（写进 md 时一并落一句回链即可，方法论见 design-spec/06）

| # | 理由 | 一句话 |
|---|------|--------|
| ① | 横切关注点归请求边界 | 只有请求边界才有完整上下文（`{request-id}` / `{api-code}` 等），底层缺上下文记的日志价值低 |
| ② | 多入口复用防重复 + 防误判 | 同一 `{Repo}` 方法被 `HTTP` / `cron` / `job` 多入口复用，自打日志会**重复打点 + 级别误判**（"未找到"在 `data` 层是 `ErrRecordNotFound`，在 `controller` 层才是正常业务） |
| ③ | 纯取数单元 | 禁日志让 `data` 层成为**可无副作用复用的纯取数单元**，支撑 IO 编排在上层统一并行 / 批量 |

> 落地时 `logging.md` 不需复制上表全文，只需写明"`service` / `data` 禁日志、`controller` 统一定级"并附一行回链 `design-spec/06 §3.3 / §9.3`。

### 14.3 与 §6.4 Service 层示例的衔接

§6.4 的 `tlog.*` 示例是**在允许打日志的层**（如 `controller` 调用链上的入口侧）才适用；落地工程套用 §6.4 时，必须先按本节确认该代码所在层**不在** `service` / `data` 禁日志范围内，否则应改为 `return error` 上传、由 `controller` 记。


---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
