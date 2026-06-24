# Internal 接口规格 — 面向上游网关

> 同样按 `aiweave/docs-spec/07_api_interfaces_spec.md` 规范填充。
> 参见 `audience_b_interfaces.md` 模板的章节结构。

## 1. 概述

### 1.1 调用场景
- 业务校验验证
- {核心写入动作}
- ...

### 1.2 路由前缀
`/{prefix}/internal`

### 1.3 返回格式

```json
{ "status": 200, "message": "success", "data": {} }
```

### 1.4 业务校验方式
（按 docs-spec/07 §4.3）

---

## 2. 业务校验验证

### 2.1 请求校验

#### 2.1.1 请求方法 + 路径

```
GET /{prefix}/internal/{module}/{action}
```

#### 2.1.2 请求参数说明与校验规则
#### 2.1.3 Go 请求结构体
#### 2.1.4 响应结构
#### 2.1.5 Go 响应结构体
#### 2.1.6 错误码映射
#### 2.1.7 Controller 伪代码
#### 2.1.8 Service 调用关系
#### 2.1.9 业务规则
#### 2.1.10 边界情况


---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
