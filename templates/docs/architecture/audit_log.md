# 审计日志设计（如有审计需求）

> 规范来源：`aiweave/docs-spec/04_architecture_overview_spec.md` §7

## 1. insertOperationLog 函数

```go
func insertOperationLog(ctx *gin.Context, action, targetType, targetId, detail string) {
    // 完整实现
}
```

### 字段映射规则

| DDL 字段 | 值来源 | 说明 |
|----------|--------|------|
| operator_id | ctx.GetString("actorId") | {admin_mw} 注入 |
| operator_type | "admin" / "system" | |
| target_type | 调用方传入 | |
| target_id | 调用方传入 | |
| action | 调用方传入 | |
| detail | 调用方传入 | |
| ip_address | ctx.ClientIP() | |
| created_at | DEFAULT | |

## 2. 触发表

| # | 端点 | action | targetType | targetId | detail |
|---|------|--------|------------|----------|--------|
| 1 | ... | ... | ... | ... | ... |

## 3. 数据流

```
端点 → service → insertOperationLog → MySQL log 库 / `{example_table_audit_log}`
```

## 4. DDL 引用

详见 `docs/schema/database_design.md` §3.X `{example_table_audit_log}`。

## 5. 查询接口

`GET /admin/log/list` 调用 `auditlog.QueryOperationLogs`。


---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
