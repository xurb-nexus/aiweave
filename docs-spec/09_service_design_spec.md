# 09 - Service 设计文档规范

> 规定 `docs/service/service_design.md` 总览 + 各 `{module}_service.md` 详情的内容结构。

---

## 1. 定位

Service 文档是 **AI 生成 service 层代码的唯一真相源**。精度上限：**方法签名 + 步骤级伪码 + 错误类型选择**。

每个 Service 方法必须有：

- 签名（参数名、类型、错误返回类型）
- 入参/出参 struct 定义
- 步骤伪码（每一步对应一个明确的代码动作）
- 数据访问说明（哪些 Redis Key、哪个 MySQL 库的哪张表）
- 错误码列举
- 与其他 service 的调用关系

---

## 2. 文档拆分

| 文档 | 内容 |
|------|------|
| `service_design.md` | **总览**：设计原则 / 错误处理双类型 / 模块依赖图 / 模块导航 / 调用矩阵 / 定时任务-service 映射 / 审计日志辅助函数 |
| `{module}_service.md` | **详情**：单个 module 的所有方法签名 + 步骤伪码 + 入参出参 |

模块清单按业务域划分（{module-A} / {module-B} / {module-C} / {状态变更} / {核心校验} / {受限模块} / {审计} 等，具体落地示例见末尾 §16）。

---

## 3. service_design.md 顶层结构（强制）

service_design.md（总览）必须按以下章节顺序组织：

- `## 1.` 设计原则（总则 + 数据访问规则）
- `## 2.` 统一错误处理（两套错误类型 / ServiceError / base.Error / Controller 处理模式）
- `## 3.` Service 模块总览（目录结构 / 依赖关系 DAG / 调用规则）
- `## 4.` 模块详细设计导航（每个模块 → 文档 → 方法数 → 状态）
- `## 5.` 审计日志辅助函数（如有）
- `## 6.` 定时任务 Service 方法（每个任务 → 调用的 service 方法 → 并行策略）
- `## 7.` Service 间调用矩阵（合法 / 禁止调用方向）
- `## 8.` 组装层 Assembler 登记（如有 DB→API 组装：哪些响应走 map 入参纯函数）

> **完整章节骨架见** [`templates/docs/service/service_design.md`](../templates/docs/service/service_design.md)。

---

## 4. §1.2 数据访问规则（必填表格）

```markdown
| 数据源 | 使用场景 | 访问方式 |
|--------|---------|---------|
| Redis | 核心业务链路、状态变更操作、限流 | `helpers.{ProjectName}CacheClient`（rediscache.Client） |
| MySQL core | {核心实体类}、权限、{核心数值类} | `helpers.MysqlClientCore`（GORM） |
| MySQL trade | {业务对象类}、状态变化、报表 | `helpers.MysqlClientTrade`（GORM） |
| MySQL log | 日统计、{受限模块}事件、审计 | `helpers.MysqlClientLog`（GORM） |
| MySQL {example_log} | 业务事件流水 | `helpers.MysqlClientCallLog`（GORM） |
| Kafka | 业务事件流水异步落库 | `helpers.{Example}PubClient` |
| 本地缓存 | 业务校验热数据 | `helpers.{LocalCache1}` 等具名实例 |
```

每行 3 字段：数据源 / 使用场景 / 访问方式（含 Go 全局变量名）。

---

## 5. §2 统一错误处理（强制）

### 5.1 错误类型映射表

```markdown
### 2.1 两套错误类型

| Go 类型 | 定义文件 | 使用接口 | 返回格式 |
|---------|---------|---------|----------|
| `*ServiceError` | `components/service_error.go` | Internal 接口（业务状态码） | `{"status": 107, "message": "...", "data": null}` |
| `base.Error` | `components/error.go` | Operator + Admin 接口 | `{"errNo": {N}, "errStr": "...", "data": {}}` |

**核心规则**：
- {auth-module} service 的方法 → 只返回 `*ServiceError`
- 服务于 Internal audience 的{业务上报类方法}（如 `{module}.{Method-N}`） → 只返回 `*ServiceError`
- 其他所有 service 方法 → 只返回 `base.Error`
```

错误类型选择是 AI 写代码时最容易出错的点之一，必须显式映射。

---

## 6. §3.2 依赖关系图（强制）

ASCII DAG（占位符示意）：

```
              ┌──────────┐
              │ {auth-module} │ ← 直接读 Redis（不依赖其他 service）
              └──────────┘
                    │
             ({module-A}.{state-report-method} 内部调用)
                    ▼
┌──────────┐  ┌──────────┐  ┌──────────┐
│ {module-A} │  │ {module-B} │  │ {module-C} │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │             │
     └──────┬──────┘             │
            ▼                    │
      ┌──────────┐              │
      │ {module-D} │◄─────────────┘
      └──────────┘
```

附调用规则：

```markdown
**调用规则**：
- {auth-module} 不调用任何 service，直接读写 Redis
- {module-A}.{state-report-method} 内部执行实际数值变更操作（Redis 原子操作），与 {auth-module} 共享 Redis Key
- {module-C}（如产品 / 资源类）变更时需要写 {module-A} 的权限（同库事务）
- service 之间严禁循环依赖
```

---

## 7. §6 定时任务 Service 方法映射（强制）

```markdown
| 任务 | Service 方法 | 并行策略 | 说明 |
|------|-------------|---------|------|
| `{flush-state-task}` | `{module}.{FlushStateMethod}(ctx)` | 多进程并行安全（SPOP） | Redis → MySQL 增量 flush |
| `{integrity-check-task}` | `{module}.{IntegrityCheckMethod}(ctx)` | 分布式锁互斥 | 缓存完整性巡检 |
| `{periodic-aggregate-task}` | `{module}.{AggregateMethod}(ctx)` | 单进程外部调度 | 周期性聚合任务 |
```

---

## 7.5 §8 组装层 Assembler 登记（如有 DB→API 组装）

凡"DB 模型 → API 响应"的组装走 Assembler 纯函数 + 已查好的 `map` 入参的，必须在 service_design.md §8 登记一张表，记录"选了什么"——哪些响应用此模式、关联数据 map 入参有哪些、是否纯函数。这是**表示记录槽位**（工程填空），不是决策方法论。

```markdown
| # | 响应类型 | 组装方法 | 关联数据 map 入参（`map[{id}]→{Model}`） | 纯函数（无 IO） |
|---|---------|---------|------------------------------------|----------------|
| 1 | `{Method-N}Resp` | `{Assembler}.Fill(mainList, total, {assoc-A}Map, ...).Build()` | `{assoc-A}Map` / `{assoc-B}Map` | 是 |
```

随表附结构性纪律（占位符化）：关联数据**强制 map 入参**、Assembler 内**禁止 DB/缓存/RPC 调用**、查询编排集中在上层、`Fill` 逐行从 map 取值做纯转换、主表空 → 统一空响应、map 入参从类型签名上强制"先批量查再传入"故可单测。

> **决策方法论不在此复制**：组装层 map 入参纪律（结构上杜绝 N+1、可单测的推导）的真相源在 [`design-spec/02 §3.4`](../design-spec/02_data_model_design.md)，§8 槽位必须反向引用它，本规范只规定"该有此登记表"。
>
> 若工程无 DB→API 组装场景，§8 写"无组装层，N/A"。

---

## 8. {module}_service.md 顶层结构（强制）

每个 `{module}_service.md` 必须按以下章节顺序组织：

- `## 1.` 模块定位（职责 / 与其他模块的关系 / 关联的接口）
- `## 2.` 数据访问（涉及的 MySQL 表 / Redis Key）
- `## 3.` 方法清单（**强制：所有方法的总览表**）
- `## 4.` 方法详细设计（每个方法 **8 个子节**：签名 / 入参出参 / 处理步骤 / 数据访问明细 / 错误码 / 调用关系 / 边界情况 / **事务与一致性（§4.N.8）**）
- `## 5.` ID 生成策略（如有自定义 ID 格式）
- `## 6.` 接口映射表（每个方法 → 哪个 audience 的哪个接口）
- `## 7.` 领域不变量（详见本规范 §13.5）

> **完整章节骨架**：service_design.md 总览参考 [`templates/docs/service/service_design.md`](../templates/docs/service/service_design.md)；每个 `{module}_service.md` 由用户按本节结构填充（templates 中未预生成空白骨架以避免与具体业务名绑死）。

---

## 9. §3 方法清单总览表

```markdown
## 3. 方法清单

| # | 方法 | 入参 | 出参 | 错误类型 | 调用方 |
|---|------|------|------|----------|--------|
| 1 | `{Method-1}` | `*{Method-1}Req` | `*{Method-1}Resp, base.Error` | base.Error | {Audience}: {action-1} |
| 2 | `{Method-2}` | `*{Method-2}Req` | `*{Method-2}Resp, base.Error` | base.Error | {Audience}: {action-2} |
| ... |
```

> 表格行的具体填法见末尾 §16 参考示例。

---

## 10. §4.N 单个方法的完整规格

每个方法详细设计章节按以下子节模板组织（**主体仅给占位符结构**，具体业务示例见末尾 §16 参考示例）：

```markdown
### 4.N {Method-N}（{Module} 中一个具体动作的语义说明）

#### 4.N.1 方法签名

\`\`\`go
func (s *{Module}Service) {Method-N}(ctx *gin.Context, req *{Method-N}Req) (*{Method-N}Resp, error)
\`\`\`

#### 4.N.2 入参/出参 struct

\`\`\`go
type {Method-N}Req struct {
    {Field-1}   {Type}  // 业务语义注释
    {Field-2}   {Type}
    // ...
}

type {Method-N}Resp struct {
    {Field-1}   {Type}
    {Field-2}   {Type}
}
\`\`\`

> service struct **不含** binding tag，binding 只在 Controller 层的 struct 上做。

#### 4.N.3 处理步骤

按"步骤伪码 + 数据访问明确"的精度写每一步：
1. 校验入参（checkParams）
2. {数据源操作}（HGET/SELECT/...，含完整 Key 模板或表名占位符）
3. {业务条件分支}（显式 if / else）
4. {写入操作}（含事务 / Pipeline / 锁 / 不变量校验等伪码标记，详见 §10 §4.X.3 标记语法）
5. 返回 *{Method-N}Resp

#### 4.N.4 数据访问明细

| 操作 | 类型 | Key / 表 |
|------|------|---------|
| {Op-1} | Redis | `{ns}:{...}:%s` |
| {Op-2} | MySQL {db} | `{example_table_*}` |
| ... | ... | ... |

#### 4.N.5 错误码

| 错误 | 触发条件 |
|------|---------|
| `Error{Reason-1}` | {触发场景} |
| `Error{Reason-2}.Wrap(err)` | {外部依赖失败} |
| ... | ... |

#### 4.N.6 调用关系

- 被调用：{Audience}: `{HTTP-Method} /{prefix}/{audience}/{module}/{action}`
- 调用：{下游 Service 方法清单 | 无（叶子方法）}

#### 4.N.7 边界情况

- {高并发场景} → {处理方式（唯一索引兜底 / 分布式锁 / 重试）}
- {性能 / 资源敏感点} → {缓解措施}

#### 4.N.8 事务与一致性（如涉及写入 / 跨数据源）

按 [`docs-spec/21 §2`](21_distributed_transaction_spec.md) 事务边界清单填写本方法的事务模型：

- **事务范围**：本地事务 / Saga 第 N 步 / MQ 投递 / 无事务（只读）
- **失败策略**：回滚 / 补偿 / 重试 / 告警人工介入
- **幂等保证**：是（Key=`{...}`，窗口=`{N}`）/ 否（原因：{...}）
- **一致性窗口**：`{N}ms` / 强一致

> 纯读方法可省略本子节，但 service 文档中应显式声明"本方法只读，无 §4.N.8"。
>
> 涉及多数据源写入但未在 [`docs-spec/21 §2`](21_distributed_transaction_spec.md) 中登记 → AI 必须先补 21 再写本子节。
```

---

## 10.5 §4.N.3 处理步骤的伪码标记语法（强制）

为让 AI 写伪码、读伪码、生成代码、审计代码共享同一组机械锚点，§4.N.3 处理步骤伪码中**必须**使用以下统一标记（命中相应场景时）：

| 标记 | 含义 | 关联规范 |
|------|------|---------|
| `[TXN-START]` / `[TXN-COMMIT]` / `[TXN-ROLLBACK]` | 本地事务边界 | [`docs-spec/21`](21_distributed_transaction_spec.md) |
| `[SAGA-STEP-N]` / `[COMPENSATE-N]` | Saga 正向 / 补偿 | [`docs-spec/21`](21_distributed_transaction_spec.md) |
| `[IDEMPOTENT-CHECK: key={...}]` | 幂等校验 | [`docs-spec/21`](21_distributed_transaction_spec.md) §4 |
| `[INVARIANT-CHECK: I-N]` | 此步必须保护领域不变量 N | 本文档 §7 |
| `[LOCK-ACQUIRE: {lock-name}]` / `[LOCK-RELEASE: {lock-name}]` | 加锁 / 释放（涉及多锁时强制） | [`docs-spec/20 §3`](20_concurrency_safety_spec.md) |
| `[HOT-PATH]` | 此方法属热路径，需应用性能约束 | 22_performance_contract |
| `[METRIC-EMIT: name{labels}]` | 此步发出指标 | 23_observability |
| `[BATCH]` | 此步为循环外批量读 / 写（消灭 N+1，铁律一） | [`docs-spec/25 §5`](25_io_aggregation_spec.md) |
| `[PARALLEL]` | 此步为聚合器 / 扇出并行编排（消灭串行，铁律二） | [`docs-spec/25 §3`](25_io_aggregation_spec.md) §4 |

### 10.5.1 标记使用规则

- 标记单独占一行（不与代码动作同行），置于该步骤代码动作之前
- 命中多个标记的步骤，按"事务/锁 → 不变量 → 指标"顺序列出
- 不命中任何标记的纯计算 / 校验步骤无需打标记
- AI 写代码时**严禁**自行省略标记——审计 Skill 会按标记机械匹配

### 10.5.2 占位符化的标记写法

伪码主体使用占位符，标记内的具体名（如 `{lock-name}` / `{idempotent-key}` / `I-N`）也用占位符；具体业务示例统一移到末尾 §16 参考示例段。

---

## 11. 步骤伪码的精度要求

### 11.1 必须做到

- 每一步对应一个明确的代码动作
- 数据访问操作显式（HGET / SELECT / INSERT 等）+ 完整 Key 名
- 调用其他 service 的方法名一字不差
- 条件分支（if / else）显式

### 11.2 反例

```
1. 校验参数
2. 查数据库
3. 处理业务
4. 返回结果
```

这种伪码是无价值的，AI 无法据此生成代码。

### 11.3 正确写法（占位符示意）

```
1. checkParams(req)
2. {row}, err := MysqlClient{Db}.Where("{key-field} = ? AND deleted = 0", req.{Field}).First(...)
3. if err != nil { return nil, Error{NotFound} }
4. if !{verify-fn}({row}.{secret-field}, req.{Input}) { return nil, Error{Mismatch} }
5. ...
```

> 真实业务示例（含具体字段名、bcrypt 等）见末尾 §16 参考示例。

---

## 12. §5 ID 生成策略（如有自定义格式）

```markdown
### 5.1 {entity-id} 格式：`{prefix-X}{时间精度}{seqN位}` 或 `{prefix-X}{时间精度}{randN位}`

\`\`\`go
func generate{Entity}Id() string {
    seq := atomic.AddInt32(&{entity}Seq, 1) % {seq-mod}
    return fmt.Sprintf("{Prefix}%s%0{N}d", time.Now().Format("{layout}"), seq)
}
\`\`\`

冲突处理：依赖唯一索引 + 重试 N 次。
```

> 实际工程的 ID 格式（按业务实体替换 `{entity-id}`）见 [`docs-spec/16_constants_spec.md`](16_constants_spec.md) §6 与末尾 §16 参考示例。

---

## 13. §6 接口映射表（强制）

```markdown
| Service 方法 | Audience | 接口路径 |
|-------------|----------|----------|
| {Method} | UserSelf | POST /{prefix}/operator/{module}/{action} |
| {Method} | UserSelf | POST /{prefix}/{audience}/{module}/{action} |
| {Method} | UserSelf | GET /{prefix}/operator/{module}/{action} |
| {AdminMethod} | BackOffice | GET /{prefix}/admin/{module}/{action} |
```

---

## 13.5 §7 领域不变量（强制）

每个 `{module}_service.md` 必须有 §7，列出本模块的领域不变量（隐式业务约束）。

### 13.5.1 §7.1 业务规则（必须始终成立）

```markdown
| # | 不变量 | 校验时机 | 违反后果 | 代码位置 |
|---|--------|---------|---------|---------|
| 1 | `{核心数值字段} ≥ 0` | 每次{扣减/变更}前 | 拒绝操作 | `service/{module-A}/{action-1}.go` |
| 2 | `{状态字段}`只能单向流转 | 状态变更前 | 拒绝操作 | `service/{module-B}/{action-2}.go` |
| 3 | `{唯一性约束}`（如同一主体不能同时持有两个活跃记录） | {开通/激活}前 | 先关旧再开新 | `service/{module-C}/{action-3}.go` |
```

每条不变量都必须有对应的 `[INVARIANT-CHECK: I-N]` 标记出现在伪码 §4.N.3 中。

### 13.5.2 §7.2 隐式约束（代码中不明显但必须遵守）

```markdown
| # | 约束 | 原因 | 违反后果 |
|---|------|------|---------|
| 1 | `{核心校验接口}`不能写 MySQL | 性能 SLA 要求 | P99 飙升 |
| 2 | `{去重队列消费任务}`必须用 SPOP 而非 SMEMBERS | 多实例并行安全 | 数据重复处理 |
| 3 | `{外部 API 字段语义}`不同于内部同名字段 | 历史遗留 | 状态映射错误 |
```

### 13.5.3 §7.3 业务状态机（如有状态字段）

```markdown
\`\`\`
[PENDING] ──→ [ACTIVE] ──→ [SUSPENDED] ──→ [TERMINATED]
    │                           │
    └──→ [REJECTED]             └──→ [ACTIVE]（恢复）
\`\`\`

转换条件表：

| 当前状态 | 目标状态 | 触发条件 | 触发方 |
|---------|---------|---------|--------|
| PENDING | ACTIVE | `{audit-pass-condition}` | `{audit-actor}` |
| ACTIVE | SUSPENDED | `{suspend-condition}` | `{suspend-actor}` |
```

### 13.5.4 B1 反向同步规则（强制）

| 代码迹象（git diff） | 反向同步动作 |
| --- | --- |
| 新增 / 修改业务规则校验代码（`if {amount-field} < 0` 等） | §7.1 业务规则表新增一行 + 伪码添加 `[INVARIANT-CHECK]` 标记 |
| 修改状态字段流转条件 | §7.3 业务状态机转换条件表同步 |
| 新增看似无关但关键的隐式约束（如"`{核心校验接口}`禁止写库"） | §7.2 隐式约束表新增一行 |
| 删除某条不变量的校验代码 | §7.1 / §7.2 删除对应行；如需保留 → 拒绝合并 |

### 13.5.5 与 BUILD_STATUS 约束清单状态轨道的关系

§7 的每条不变量条目对应 BUILD_STATUS.md 约束清单状态轨道的一行：

- ⬜ 已设计未启用（如设计完成但对应校验代码未生成）
- 🟢 启用（校验代码已生成并装配）
- 🚫 暂不实现（与 §0 联动）

详见 [`docs-spec/02 §11 约束清单状态轨道`](02_build_status_md_spec.md)。

---

## 14. 与其他文档的关系

| 内容 | api/*.md | service_design.md | {module}_service.md |
|------|----------|-------------------|---------------------|
| 接口路径 + JSON | ✅ 完整 | 引用 | 简略（接口映射表） |
| Controller 伪码 | ✅ | 不写 | 不写 |
| Go 请求 struct（含 binding tag） | ✅ | 不写 | 不写 |
| Service 方法签名 | 引用 | 表格列出 | ✅ 完整 |
| Service 步骤伪码 | 不写 | 不写 | ✅ 完整 |
| 数据访问明细 | 不写 | 总览表 | ✅ 完整 |
| 错误处理双类型 | 不写 | ✅ 详述 | 引用规则 |

---

## 15. 维护

| 触发 | 动作 |
|------|------|
| 新增 service 方法 | §3 方法清单新增行 + §4 新增子节 + §6 接口映射 + 同步 api/*.md 引用 |
| 修改方法签名 | §4.1.1 + §4.1.2 + 同步 service_design.md §3.2 调用矩阵 |
| 删除方法 | §3 + §4 整节移除 + §6 + api/*.md |
| 新增模块 | service_design.md §3 + §4 + 新建 {module}_service.md |
| 调整调用关系 | service_design.md §3.2 依赖图 + §7 调用矩阵 |
| 新增 / 修改 Assembler 纯函数（`{Assembler}.Fill(...).Build()`，map 入参组装） | service_design.md §8 组装层 Assembler 登记新增 / 更新一行 |

---

## 16. 参考示例（仅示意，落地按业务替换）

> ⚠️ 以下为示意，规范本体（§1-§15）已用占位符表达；本节给出一个具体业务（账号注册类操作）的示例，便于读者建立"占位符 → 业务名"的对应直觉。落地工程时，请按业务语义替换占位符，不得直接照搬本节具体业务名作为规范。
>
> 示例服从 [`PRINCIPLES.md §12 占位符规则`](../PRINCIPLES.md#12-占位符规则强制--双轨结构) 的"参考示例段豁免"。

### 16.1 §3 方法清单总览表（示例）

```markdown
| # | 方法 | 入参 | 出参 | 错误类型 | 调用方 |
|---|------|------|------|----------|--------|
| 1 | `Register` | `*RegisterReq` | `*RegisterResp, base.Error` | base.Error | Operator: register |
| 2 | `Login` | `*LoginReq` | `*LoginResp, base.Error` | base.Error | Operator: login |
```

### 16.2 §4.1 单个方法完整规格（示例：账号注册类操作）

```markdown
### 4.1 Register（账号注册）

#### 4.1.1 方法签名

\`\`\`go
func (s *AccountService) Register(ctx *gin.Context, req *RegisterReq) (*RegisterResp, error)
\`\`\`

#### 4.1.2 入参/出参 struct

\`\`\`go
type RegisterReq struct {
    Account     string  // 账号标识（邮箱/手机号等）
    Password    string  // 明文，service 内做 bcrypt
    VerifyCode  string
    Source      string  // 注册来源
}

type RegisterResp struct {
    UserId   string  // 生成的 ACC{yyyyMMdd}{seq3位}
    Status   int8
}
\`\`\`

#### 4.1.3 处理步骤

1. checkParams(req) — 校验账号格式（邮箱）
2. 查询验证码：`HGET {ns2}:verify_code:{account} code`，对比 `req.VerifyCode`
3. 校验密码强度（service 内固定规则）
4. 检查账号是否已存在：`SELECT id FROM account WHERE account = ?`
5. 生成 userId：`ACC{yyyyMMdd}{seq3位}`
6. bcrypt 密码：`password_hash = bcrypt(password, cost=10)`
7. [TXN-START] INSERT account [TXN-COMMIT]
8. 写 Redis：`HSET {ns2}:info:{userId} status 1 verified 0`
9. 删除验证码 Key
10. 返回 *RegisterResp

#### 4.1.4 数据访问明细

| 操作 | 类型 | Key / 表 |
|------|------|---------|
| HGET | Redis | {ns2}:verify_code:{account} |
| SELECT | MySQL core | account |
| INSERT | MySQL core | account |
| HSET | Redis | {ns2}:info:{userId} |
| DEL | Redis | {ns2}:verify_code:{account} |

#### 4.1.5 错误码

| 错误 | 触发条件 |
|------|---------|
| `ErrorParamInvalid` | 邮箱格式错 / 密码弱 |
| `ErrorVerifyCodeExpired` | Redis 中无验证码 |
| `ErrorVerifyCodeMismatch` | 验证码不匹配 |
| `ErrorAccountAlreadyExists` | 账号已存在 |
| `ErrorDbInsert.Wrap(err)` | INSERT 失败 |

#### 4.1.6 调用关系

- 被调用：Operator: `POST /api/operator/account/register`
- 调用：无（叶子方法）

#### 4.1.7 边界情况

- 高并发同账号注册：依赖唯一索引兜底
- bcrypt 性能：cost=10 约 80ms，service 内同步执行
```

### 16.3 §11.3 步骤伪码（示例）

```
1. checkParams(req)
2. accountRow, err := MysqlClientCore.Where("account = ? AND deleted = 0", req.Account).First(...)
3. if err != nil { return nil, ErrorAccountNotFound }
4. if !bcrypt.Compare(accountRow.PasswordHash, req.Password) { return nil, ErrorPasswordMismatch }
5. ...
```

### 16.4 §12 ID 生成策略（示例：userId）

```markdown
### 5.1 userId 格式：`ACC{yyyyMMdd}{seq3位}`，例：ACC20260407001

\`\`\`go
func generateUserId() string {
    seq := atomic.AddInt32(&userSeq, 1) % 1000
    return fmt.Sprintf("ACC%s%03d", time.Now().Format("20060102"), seq)
}
\`\`\`

冲突处理：依赖唯一索引 + 重试 3 次。
```

### 16.5 §7 领域不变量（示例：账户 / 计费）

#### 16.5.1 §7.1 业务规则

| # | 不变量 | 校验时机 | 违反后果 | 代码位置 |
|---|--------|---------|---------|---------|
| 1 | balance ≥ 0 | 每次扣减前 | 拒绝操作 | service/billing/deduct.go |
| 2 | account_status 单向流转（PENDING→ACTIVE→SUSPENDED→TERMINATED） | 状态变更前 | 拒绝操作 | service/account/state.go |
| 3 | 同一 userId 同时只能持有一个活跃订单 | 下单前 | 先关旧再开新 | service/order/create.go |

#### 16.5.2 §7.2 隐式约束

| # | 约束 | 原因 | 违反后果 |
|---|------|------|---------|
| 1 | verify 接口不能写 MySQL | P99 < 50ms SLA | 性能 P99 飙升 |
| 2 | call_log_consumer 必须用 SPOP 而非 SMEMBERS | 多实例并行安全 | 流水重复落库 |

#### 16.5.3 §7.3 业务状态机

```
[PENDING] ──→ [ACTIVE] ──→ [SUSPENDED] ──→ [TERMINATED]
    │                           │
    └──→ [REJECTED]             └──→ [ACTIVE]（恢复）
```

#### 16.5.4 伪码标记示例

```
1. checkParams(req)
2. [IDEMPOTENT-CHECK: key=acc:idemp:deduct:{orderId}]
3. [INVARIANT-CHECK: I-1]  # balance 非负
4. [LOCK-ACQUIRE: account-{userId}]
5. [TXN-START]
6. SELECT balance FROM account WHERE id=? FOR UPDATE
7. balance -= req.Amount
8. UPDATE account SET balance=? WHERE id=?
9. [METRIC-EMIT: billing_deduct_total{result="ok"}]
10. [TXN-COMMIT]
11. [LOCK-RELEASE: account-{userId}]
```

### 16.6 占位符 → 业务名 对照表（示例）

| 占位符 | 本节示例值 |
| --- | --- |
| `{Module}` / `{module}` | Account / account |
| `{Module}Service` | AccountService |
| `{Method-N}` | Register / Login |
| `{Method-N}Req` | RegisterReq |
| `{example_table_*}` | account |
| `{ns2}` | （某 Redis 命名空间） |
| `{entity-id}` / `{prefix-X}{...}` | userId / `ACC{yyyyMMdd}{seq3位}` |
| `{Audience}` | Operator |



---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
