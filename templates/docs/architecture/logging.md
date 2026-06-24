# 日志规范

> 规范来源：`aiweave/docs-spec/13_logging_spec.md`

## 1. 日志框架概述

基于 zap，封装为 `pkg/golib/v2/tlog`。

## 2. 日志级别使用规范

| 级别 | 语义 | 触发示例 |
|------|------|---------|
| DEBUG | 调试细节 | 步骤进入 / 退出 |
| INFO | 业务正常事件 | task completed |
| WARN | 可恢复异常 | redis 写失败但 mysql 已成功 |
| ERROR | 不可恢复失败 | mysql 写失败 |
| FATAL | 进程必须退出 | 启动初始化失败 |

判定规则：
- 能自愈 → WARN
- 需人工介入 → ERROR

热路径不打日志。

## 3. 两种 API 风格

### 3.1 Sugared API（禁用）
```go
tlog.Infof(ctx, "...")  // ✗
```

### 3.2 Structured API（强制）
```go
tlog.InfoLogger(ctx, "task completed",
    tlog.String("task", taskName),
    tlog.Int("processed", count))
```

## 4. 上下文字段与 Notice 机制

### 4.1 自动注入字段
- traceId、`{actor-id}`、entityId 等由中间件注入

### 4.2 Notice 字段
- ctx.Set("notice.X", val) → tlog 自动展开

## 5. Field 构造器完整列表

详见 `pkg_api.md` §2.2。

## 6. 各模块日志规范

### 6.1 定时任务
### 6.2 核心业务链路（不打详细日志）
### 6.3 中间件（仅异常打 WARN）
### 6.4 Service 层
### 6.5 Redis / MySQL 故障

### 6.6 分层日志纪律（数据层禁日志）

> 决策/理由见 [design-spec/06_resilience_design.md §3.3 / §9.3](../../../design-spec/06_resilience_design.md)。与「热路径不打日志」（性能理由）并列且不同：本条是分层归属理由。本节只登记本工程"哪些层不打、谁定级"，不复制方法论。

| 层 | 是否打日志 | 约定 |
|----|-----------|------|
| `controller`（请求边界） | ✅ 唯一记日志层 | 收到下层 `return error` 后按"自愈 → WARN / 需人工 → ERROR"定级记一次 |
| `service` | ✗ 禁打日志 | 只 `return error` 上传 |
| `data` | ✗ 禁打日志 | 只 `return error` 上传 |

要点：`service` / `data` 一律 `return error` 上传，由 `controller` 统一定级（`ErrRecordNotFound` 在 `data` 层不记，在 `controller` 层才判定是否正常业务）。

## 7. 敏感数据脱敏

禁止打印：secret / password / token / verifyCode / 完整身份证号 / 银行卡号 / 手机号。

## 8. 错误字段固定 key

错误信息**必须**用 `error` 作为 key，不用 err / reason / msg。

## 9. 日志输出文件

| 文件 | 内容 | 切割 |
|------|------|------|
| log/{project}.log | INFO+ | 按日 |
| log/{project}.log.wf | WARN/ERROR | 按日 |
| log/{project}.log.access | accessLog | 按日 |


---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
