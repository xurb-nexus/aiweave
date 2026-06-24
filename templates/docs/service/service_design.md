# Service 层设计 — 总览

> 规范来源：`aiweave/docs-spec/09_service_design_spec.md`

## 1. 设计原则

### 1.1 总则
- Service 层是唯一业务逻辑层
- Service 之间可互相调用，但避免循环依赖
- {核心 service}（如 auth）直接读 Redis
- 统一错误处理（双类型）

### 1.2 数据访问规则

| 数据源 | 使用场景 | 访问方式 |
|--------|---------|---------|
| Redis | 业务校验 / 状态变更 / 限流 | `helpers.{Project}CacheClient` |
| MySQL core | ... | `helpers.MysqlClientCore` |
| MySQL trade | ... | `helpers.MysqlClientTrade` |
| MySQL log | ... | `helpers.MysqlClientLog` |
| MySQL {example_log} | ... | `helpers.MysqlClientCallLog` |
| Kafka | ... | `helpers.{Topic}PubClient` |
| 本地缓存 | ... | `helpers.{LocalCache1}` 等 |

## 2. 统一错误处理

### 2.1 两套错误类型
| Go 类型 | 定义文件 | 使用接口 | 返回格式 |
|---------|---------|---------|----------|
| `*ServiceError` | ... | ... | ... |
| `base.Error` | ... | ... | ... |

**核心规则**：
- {模块 X} 全部方法 → `*ServiceError`
- 其他 → `base.Error`

### 2.2 ServiceError 定义
### 2.3 base.Error 定义
### 2.4 Controller 层处理模式

## 3. Service 模块总览

```
service/
├── {module_1}/
├── {module_2}/
└── ...
```

### 依赖关系图

```
{ASCII DAG}
```

**调用规则**：
- {module_1} 不调用任何 service
- ...

## 4. 模块详细设计导航

| 模块 | 文件 | 方法数 | 说明 |
|------|------|--------|------|
| {module_1} | [{module_1}_service.md](...) | N | ... |

## 5. 审计日志辅助函数（如有）

### 5.1 insertOperationLog
### 5.2 全部 Admin POST 端点的审计日志调用

## 6. 定时任务 Service 方法

| 任务 | Service 方法 | 并行策略 | 说明 |
|------|-------------|---------|------|
| ... | ... | ... | ... |

## 7. Service 间调用矩阵

| 调用方 → 被调用方 | 场景 |
|-------------------|------|
| ... | ... |

**禁止的调用方向**：
- 循环依赖
- service 调 controller

## 8. 组装层 Assembler 登记

> 登记"DB 模型 → API 响应"用 Assembler 纯函数 + 已查好的 `map` 入参组装的响应。结构上杜绝 Assembler 内 N+1、可单测。
>
> 决策方法见 [design-spec/02 §3.4 组装层 map 入参纪律](../../../design-spec/02_data_model_design.md)。

| # | 响应类型 | 组装方法 | 关联数据 map 入参（`map[{id}]→{Model}`） | 纯函数（无 IO） |
|---|---------|---------|------------------------------------|----------------|
| 1 | `{Method-N}Resp` | `{Assembler}.Fill(mainList, total, {assoc-A}Map, {assoc-B}Map, page, size).Build()` | `{assoc-A}Map` / `{assoc-B}Map` | 是 |
| 2 | ... | ... | ... | 是 |

**纪律（结构性，非可选）**：
- 关联数据**必须**以 `map[{id}]→{Model}` 入参传入；查询编排集中在上层（Service / 编排器），Assembler 内**禁止**任何 DB / 缓存 / RPC 调用。
- `Fill` 逐行从 map 取值（`{assoc-A}Map[{id}]`）→ 纯转换 → 输出（含加密 ID）；主表空 → 统一空响应形态。
- map 入参从类型签名上强制"先批量查再传入"，使 Assembler 可被单测覆盖（造假 map → 断言输出）。


---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
