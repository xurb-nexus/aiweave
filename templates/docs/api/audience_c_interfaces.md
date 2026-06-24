# Admin 接口规格 — 面向内部管理后台

> 按 `aiweave/docs-spec/07_api_interfaces_spec.md` 规范填充。结构同 `audience_b_interfaces.md`。

## 1. 概述

### 1.1 调用场景
- {核心实体管理}
- Key 状态变更
- 业务调整
- ...

### 1.2 路由前缀
`/{prefix}/admin`

### 1.3 返回格式
（同 operator）

### 1.4 业务校验方式
- IP 白名单
- X-{Actor}-Id / X-{Actor}-Role 多角色

### 1.5 上游集成（SSO Cookie / Session / 角色权限）

---

## 2. {模块 1}
（按 audience_b_interfaces.md 结构）

> ⛔ **{受限模块}相关接口**（如有 🚫 标注）：当前阶段不实现，详见 `BUILD_STATUS.md` §0。设计完整保留如下：

## N. {受限模块}管理（🚫 当前阶段不实现）

### N.1 ...
### N.2 ...


---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
