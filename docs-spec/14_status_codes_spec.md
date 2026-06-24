# 14 - 错误码 / 状态码体系规范

> 规定 `docs/architecture/status_codes.md` 的内容结构。

---

## 1. 定位

错误码文档要让 AI 能回答：

1. 项目有几套错误类型，各自用在哪些接口
2. 每套错误类型的范围划分
3. 每个错误码的常量名、数值、消息文案
4. 业务状态枚举（{module-A} / {module-B} / {module-C} 等的状态值集中定义）
5. Controller 层如何处理两类错误

---

## 2. 顶层结构

```markdown
# 状态码与错误码设计

## 1. 双错误类型体系（如有）
   1.1 ServiceError vs base.Error
   1.2 适用接口对照表
   1.3 选择规则

## 2. ServiceError 完整定义
   2.1 Go 类型定义
   2.2 状态码范围与含义
   2.3 全部状态码清单（强制）
   2.4 预定义实例

## 3. base.Error 完整定义
   3.1 Go 类型定义
   3.2 错误码范围与含义
   3.3 全部错误码清单（强制）
   3.4 错误链与格式化

## 4. 业务状态枚举（强制）
   4.1 {module-A}状态
   4.2 Key 状态
   4.3 ...（每类业务对象的状态值集中）

## 5. Controller 层处理模式

## 6. 与日志的协同

## 7. 错误码新增规范
```

---

## 3. §1 双错误类型体系

```markdown
## 1. 双错误类型体系

| Go 类型 | 定义文件 | 使用接口 | 返回格式 | 数值范围 |
|---------|---------|---------|----------|---------|
| `*ServiceError` | `components/service_error.go` | Internal（与外部调用方协议对齐） | `{"status": 107, "message": "...", "data": null}` | {业务状态码范围} |
| `base.Error` | `components/error.go` | Operator + Admin（内部） | `{"errNo": {N}, "errStr": "...", "data": {}}` | {内部错误码范围} |

**选择规则**：

- **协议对齐**优先：当下游/上游已存在协议（如 上游网关）时，按协议定义状态码，使用 ServiceError
- **内部接口**统一用 base.Error，便于查错
```

---

## 4. §2.3 全部 ServiceError 清单（强制）

```markdown
### 2.3 全部状态码清单

#### 2.3.1 成功（2xx）
| 数值 | 含义 |
|------|------|
| 200 | 请求成功 |

#### 2.3.2 Key 状态异常（10x）
| 数值 | 常量 | 消息文案 | 触发条件 |
|------|------|---------|---------|
| {N} | ErrEntityInvalid | 资源/凭证无效或未生效 | 实体不存在 / 状态异常 |
| {N} | Err{Module}Disabled | {module-A}已禁用 | {module-A}.status=禁用 |
| {N} | ErrEntitySuspended | 资源/凭证已暂停 | 实体.status=异常 |
| {N} | ErrEntityDeleted | 资源/凭证已删除 | 实体.status=已删除 |
| {N} | ErrActionOffline | 功能已下线 | 资源.status=下线 |

#### 2.3.3 签名/时间戳错误（10x）
| 数值 | 常量 | 消息文案 |
|------|------|---------|
| 106 | ErrRiskBlocked | {受限模块}暂时拦截 |
| {N} | ErrSignatureFail | 签名错误或 IP 不允许 |
| {N} | ErrSignatureExpired | 身份/签名校验失败（时间戳超过容差窗口） |
| {N} | ErrParamInvalid | 请求参数错误 |

#### 2.3.4 资源 / 接口异常（11x）
| 数值 | 常量 | 触发条件 |
|------|------|---------|
| {N} | ErrStateInsufficient | {核心数值字段}不足且无可用补充资源 |
| {N} | ErrActionNotGranted | 未授权访问该功能 |
| {N} | ErrDailyLimitExceeded | 功能日调用上限达到 |
| {N} | ErrConcurrencyLimit | QPS 并发上限 |

#### 2.3.5 内部错误（199-224）
| 数值 | 常量 | 触发条件 |
|------|------|---------|
| {N} | ErrSystemError | 系统错误 |
```

每个错误码必填：**数值 + 常量名 + 消息文案 + 触发条件**。

---

## 5. §3 base.Error 清单（强制）

```markdown
## 3. base.Error 完整定义

### 3.2 错误码范围与含义

| 范围 | 含义 |
|------|------|
| {范围1} | 通用参数错误 |
| {范围2} | {module-A}业务错误 |
| {范围3} | 资源/凭证业务错误 |
| {范围4} | 业务对象 A 错误 |
| {范围5} | 业务对象 B 错误 |
| {范围6} | {受限模块}业务错误 |
| {范围7} | 第三方/外部服务错误 |
| {范围8} | 通用系统错误 |
| {范围9} | 数据库错误 |
| {范围10} | Redis 错误 |
| {范围11} | MQ 错误 |
| {范围12} | 内部 RPC 错误 |

### 3.3 全部错误码清单

| 数值 | 常量 | 消息文案 | 业务含义 |
|------|------|---------|---------|
| {N} | ErrorParamInvalid | 参数错误 | 通用参数错误（具体原因看 ErrMsg） |
| {N} | ErrorParamRequired | required param missing: %s | 参数缺失（含格式化模板） |
| {N} | Error{Module}NotFound | {module} not found | {核心实体}不存在 |
| {N} | Error{Module}AlreadyExists | {module} already exists | 唯一约束冲突 |
| ... |
| {N} | ErrorSystemError | 系统错误 | 兜底未分类错误 |
| {N} | ErrorDbInsert | db insert failed | INSERT 失败 |
| ... |
```

---

## 6. §3.4 错误链与格式化（必填）

```markdown
### 3.4 错误链与格式化

\`\`\`go
// 格式化（带占位符）
components.ErrorParamRequired.Sprintf("entityId")
// → ErrMsg = "required param missing: entityId"

// 错误链（保留原始错误）
components.ErrorDbInsert.Wrap(dbErr)
// → 内部保留 dbErr 的错误栈，便于排查

// 同时支持链式调用
components.ErrorParamInvalid.Sprintf("phone").Wrap(parseErr)
\`\`\`

**Sprintf 模板规则**：

- 占位符使用 `%s`、`%d` 等标准 Go 格式
- 在 ErrMsg 中**显式标注**（如 `"required param missing: %s"` 而不是 `"required param missing"`）
- 调用 .Sprintf 时**必须**提供对应数量的参数
```

---

## 7. §4 业务状态枚举（强制）

```markdown
## 4. 业务状态枚举

集中定义所有业务对象的状态值，避免魔法数字散布。

### 4.1 {module-A}状态（`{example_table_actor}.status`）
| 值 | 含义 | 影响 |
|----|------|------|
| 0 | 禁用 | 拒绝所有请求（status=102） |
| 1 | 正常 | 允许 |

### 4.2 {module-A}认证状态（`{example_table_actor}.verified`）
| 值 | 含义 |
|----|------|
| 0 | 未认证 |
| 1 | 已认证 |
| 2 | 认证审核中 |
| 3 | 认证拒绝 |

### 4.3 资源/凭证状态（`{example_table_entity}.status`）
| 值 | 含义 | 影响 |
|----|------|------|
| 1 | 正常 | 允许 |
| 2 | 暂停 | 业务校验返回 103 |
| 3 | 删除 | 业务校验返回 104 |

### 4.4 资源 A 状态（`{example_table_product}.status`）
| 值 | 含义 |
|----|------|
| 1 | 正常 |
| 2 | 下线 |

### 4.5 业务对象 A 状态 / 业务对象 B 状态 / {受限模块}状态 / ...

每类业务对象的所有取值都在这里集中。
```

---

## 8. §5 Controller 层处理模式

```markdown
## 5. Controller 层处理模式

### 5.1 Internal controller（ServiceError）

\`\`\`go
result, err := {module}Svc.{Method}(ctx, req)
if err != nil {
    if se, ok := err.(*components.ServiceError); ok {
        ctx.JSON(200, gin.H{
            "status":  se.StatusCode,
            "message": se.Message,
            "data":    nil,
        })
        return
    }
    ctx.JSON(200, gin.H{"status": 199, "message": "系统错误", "data": nil})
    return
}
ctx.JSON(200, gin.H{"status": 200, "message": "success", "data": result})
\`\`\`

### 5.2 Operator/Admin controller（base.Error）

\`\`\`go
result, err := svc.Method(ctx, req)
if err != nil {
    base.RenderJsonFail(ctx, err)
    return
}
base.RenderJsonSucc(ctx, result)
\`\`\`

`base.RenderJsonFail` 自动提取 base.Error 的 ErrNo / ErrMsg。
```

---

## 9. §7 错误码新增规范

```markdown
## 7. 错误码新增规范

### 7.1 选择类型
1. 该错误是否对外（Internal）？是 → ServiceError；否 → base.Error
2. 已有相似错误？尽量复用，不要新增近义码

### 7.2 选择数值
1. 在对应业务范围内（如 {module-A} 错误用 {范围2}）
2. 在该范围内取下一个未占用的数值
3. 不要"留空位"——按顺序密集分配

### 7.3 命名
- 常量名以 `Error` / `Err` 开头
- CamelCase（base.Error 用 `ErrorParamInvalid`，ServiceError 用 `ErrParamInvalid`）
- 名字反映业务含义，不反映数值

### 7.4 流程
1. 在 components/error.go 或 components/service_error.go 添加常量
2. 在 status_codes.md 对应清单表新增行
3. 在受影响的 service / api 文档同步引用
```

### 9.5 对外 ID / token 解密失败的错误码纪律（强制 / 安全）

> §7 错误码新增规范的一条**安全特例**，写入 `status_codes.md` 错误码新增规范一节（骨架 §6.5）。

对外暴露的主键 / ID 走确定性加密（决策见 [`design-spec/05 §9.5`](../design-spec/05_caching_design.md)、登记见 [`05 §6.6`](05_schema_design_spec.md)）。其**入参侧解密失败的错误处理**有一条反预言机硬纪律：

- **禁止**为"token 非法 / 解密失败 / 解密后格式不对"新增任何专用错误码（如 `ErrInvalidKeyNo`）。
- 一律**统一降级为"查无结果"**（与"真无数据"返回完全相同的 status / message），响应**不得**携带任何"解密失败"细节。
- 理由：响应差分（哪些 token 有效 / 哪些伪造）会让加密层退化为**预言机**，攻击者可批量探测有效 token 空间。统一"查无结果"把信号噪声化。

> grep 锚 `R-SEC-DECRYPT-ORACLE`（解密失败给差异化错误 / 4xx / 携带失败细节）、`R-SEC-ID-PLAINTEXT`（响应直接返回未加密真实主键）；完整定义见 `ai_dev_guide.md §10.11`。本节是**表示纪律**——确定性加密算法（PKCS7 / AES-CBC / 确定性 IV / KDF 隔离 / 版本前缀轮换）真相源在 design-spec/05 §9.5，不复制。

---

## 10. 与其他文档的关系

| 内容 | status_codes.md | service/*.md | api/*.md |
|------|-----------------|---------------|----------|
| 错误码常量 + 数值 + 文案 | ✅ 完整 | 引用常量名 | 引用数值 |
| 触发条件 | 简略 | ✅ 详细 | ✅ 在错误码映射表 |
| 业务状态枚举 | ✅ 集中 | 引用 | 引用 |

---

## 11. 维护

| 触发 | 动作 |
|------|------|
| 新增错误码 | §2.3 / §3.3 清单 + components/error.go + 受影响的 service/api 文档 |
| 修改文案 | §2.3 / §3.3 + 同步代码 + i18n（如有） |
| 新增业务对象状态 | §4 新增子节 + database_design.md / cache_design.md 同步 |
| 调整错误码范围划分 | §3.2 + 影响评估 |


---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
