# {Audience} 接口规格

> 规范来源：`aiweave/docs-spec/07_api_interfaces_spec.md`

## 1. 概述

### 1.1 调用场景
{描述本路由组的 N 个接口的核心用途}

### 1.2 路由前缀
`/{prefix}/{audience}`

### 1.3 返回格式

```json
{
  "errNo": 0,        // 或 "status": 200
  "errStr": "",      // 或 "message": "success"
  "data": {}
}
```

### 1.4 业务校验方式

| 参数 | 位置 | 类型 | 必填 | 说明 |
|------|------|------|------|------|
| ... | ... | ... | ... | ... |

### 1.5 上游集成设计（如有 BFF/Cookie/Session）

---

## 2. {模块 1}（N 个接口）

### 2.1 {接口 1}

#### 2.1.1 请求方法 + 路径

```
POST /{prefix}/{audience}/{module}/{action}
```

#### 2.1.2 请求参数说明与校验规则

| 字段 | 位置 | 类型 | 必填 | 校验规则 | 说明 |
|------|------|------|------|---------|------|
| ... | ... | ... | ... | ... | ... |

#### 2.1.3 Go 请求结构体

```go
type {Action}Req struct {
    Field1 string `json:"field1" binding:"required,max=64"`
    // ...
}
```

#### 2.1.4 响应结构

**成功**：

```json
{ "errNo": 0, "errStr": "", "data": { ... } }
```

**失败**：

```json
{ "errNo": {N}, "errStr": "...", "data": {} }
```

#### 2.1.5 Go 响应结构体

```go
type {Action}Resp struct {
    Field1 string `json:"field1"`
    // ...
}
```

#### 2.1.6 错误码映射

| errNo | 含义 | 触发条件 |
|-------|------|---------|
| ... | ... | ... |

#### 2.1.7 Controller 伪代码

```go
func {Action}(ctx *gin.Context) {
    var req {Action}Req
    if err := ctx.ShouldBindJSON(&req); err != nil {
        base.RenderJsonFail(ctx, components.ErrorParamInvalid)
        return
    }
    resp, err := svc.{Method}(ctx, &req)
    if err != nil {
        base.RenderJsonFail(ctx, err)
        return
    }
    base.RenderJsonSucc(ctx, resp)
}
```

#### 2.1.8 Service 调用关系

- **Service 方法**：`{module}.{Method}(ctx, req)`
- **设计文档**：[`docs/service/{module}_service.md`](../service/{module}_service.md)

#### 2.1.9 业务规则（如适用）

#### 2.1.10 边界情况

### 2.2 {接口 2}
（同上结构）

---

## 3. {模块 2}
...

---

## N. 安全注意事项

## N+1. 性能注意事项


---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
