# 响应渲染函数设计（如有多种响应格式）

> 规范来源：`aiweave/docs-spec/04_architecture_overview_spec.md` §8

## 1. 响应模式对比

| 模式 | JSON 结构 | 错误处理 |
|------|-----------|---------|
| {Audience1}（Internal） | `{"status":...,"message":...,"data":...}` | `*ServiceError` |
| {Audience2/3}（Operator/Admin） | `{"errNo":...,"errStr":...,"data":...}` | `base.Error` |

## 2. renderInternalResponse 完整定义

```go
func renderInternalResponse(ctx *gin.Context, err error, data interface{}) {
    // 完整实现
}
```

行为规则：
- err == nil → ctx.JSON(200, {"status":200,"message":"success","data":data})
- err 是 *ServiceError → ctx.JSON(200, {"status":se.StatusCode,"message":se.Message,"data":nil})
- err 其他 → ctx.JSON(200, {"status":199,"message":"系统错误","data":nil})

## 3. CustomRender 注册机制

（如适用，描述自定义 Render 的注册时机和优先级）

## 4. base.RenderJson 系列方法

| 函数 | 用途 |
|------|------|
| `base.RenderJsonSucc(ctx, data)` | 成功响应 |
| `base.RenderJsonFail(ctx, err)` | 失败响应（自动提取 base.Error） |
| `base.RenderJsonAbort(ctx, err)` | 中间件中止响应 |

## 5. 设计决策

为什么有多种格式（历史原因 / 网关协议）。


---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
