# AIWeave 核心原则

> 本文是 AIWeave 规范的最高优先级纪律。任何 docs-spec/ 与 skills-spec/ 中的具体规则，都从本文的原则推导而来。

---

## 1. 第一性目标

```
仅凭 docs/ + .claude/skills/ 即可重建整个工程。
```

这一目标推导出全部其他规则。

---

## 1.5 两大使用模式

本规范服务两种使用模式，第一性目标在两种模式下都成立。

### 模式 A：建设模式（一次性事件）
- A0 — design（上游 / 可选）：用 design-spec 十三视角对需求做架构决策，产出技术方案(TRD)，再喂给 A1 / B1（详见 `design-spec/00`；执行器 `/design-solution`）
- A1 — forward：新工程从零启动，按规范填充 docs/ → 按 Stage 生成代码
- A2 — rebuild：从已有 docs/ 重建代码（代码丢失/换栈）

### 模式 B：增量同步模式（持续常态）
- B1 — sync-feature：每次需求迭代后，把"需求技术文档 + 代码改动 + 本规范"喂给 AI，AI 反向同步全链路文档
- B2 — refactor：跨模块重构时，按 mvp_rebuild_path 切片心智推进

> **本节核心断言**：本规范的所有规则，都必须能在 B1 增量同步场景下按"反向"应用。换言之——任何"建设时怎么写"的规则，都应当伴随一条"维护时如何反向更新"的镜像规则。这是规范保持长效有用的关键。

详细流程见 `aiweave/docs-spec/19_incremental_sync_spec.md`。

---

## 2. 文档与代码的关系

### 2.1 双向同步（强制）

无论变更从哪个方向发起，都必须保证另一侧同步更新。**没有例外。**

| 变更方向 | 触发场景 | 强制要求 |
|---------|---------|---------|
| **md → 代码** | 用户提供技术方案/md，要求据此写代码 | 严格按 md 实现；实现中发现 md 需调整，必须同步更新 md |
| **代码 → md** | 修改已有代码、修复 bug、重构 | 找到关联 md 并必须同步更新；如果没有对应 md，必须新建 |
| **新增代码** | 添加新模块、新接口、新基础设施 | 必须新建/更新对应 md，并在 INDEX.md 登记 |
| **删除代码** | 移除模块、接口、文件 | 必须同步删除/更新对应 md，并更新 INDEX.md |

### 2.2 不可绕过

下列情况都视为"未交付"，不允许合并/上线：

- 代码改了，没改文档
- 文档改了，没改代码（除非该 md 描述的是未来设计，对应代码尚未生成）
- 新增 md 没登记到 INDEX.md
- 业务代码生成了但测试没同步
- 删除代码后 md 仍残留对该代码的引用

---

## 3. 文档的精度要求

文档必须精准到 **可重建代码** 的级别。这条原则的具体形态因视角而异：

| 视角 | 精度要求 |
|------|----------|
| 数据库 DDL | 字段名、类型、长度、nullable、默认值、索引、字符集都要写到 md |
| 缓存 Key | Key 模式、值类型、Hash 每个 field 名/类型/单位/写入时机/读取时机/与 MySQL 的映射关系 |
| API 接口 | 请求/响应 JSON、Go struct（含 binding tag）、字段校验规则、错误码映射、Controller 伪代码 |
| Service 方法 | 函数签名（参数名、类型、错误返回）+ 步骤级伪代码（每一步对应一行业务动作） |
| 中间件 | 完整执行步骤、ctx.Set 注入字段、错误响应格式 |
| 定时任务 | cobra 注册代码片段、并行策略、异常容错、与 Service 方法的映射 |
| 错误码 | 错误码常量名、数值范围、HTTP 状态码映射 |
| 工具函数 | 函数签名、入参/出参类型、调用方清单 |

判定标准：**让从未见过这个工程的 AI 仅凭这一段 md 能否生成与原代码语义等价的实现？** 能 → 合格；不能 → 补到能。

---

## 4. 文档优先级（默认）

> 默认以文档为真相源，代码服从文档。

当 md 描述与代码不一致时，**默认结论是"代码错了，按 md 改代码"**。

这与传统工程"代码即真相"的范式相反，是 AI 原生工程的根本前提：如果代码可以随意偏离文档，那么"仅凭 md 重建工程"就不再可能。

---

## 5. 签名冲突裁决（默认规则的唯一例外）

### 5.1 例外条款

当文档给出的函数/类型签名与"已在代码中存在、可编译、被使用的实现"不一致时，**以代码实现为真相源**，并反向修正文档。

### 5.2 判定标准

| 情况 | 真相源 |
|------|--------|
| 文档描述"未来设计"，代码不存在/不可编译 | **以文档为准**，按 Skill 生成代码 |
| 文档描述"既有契约"，代码存在且被外部调用 | **以代码为准**，修正文档签名 |

### 5.3 典型场景

- 内部框架/工具库的公开类型与函数（如 `pkg/golib`、`helpers/rediscache.Client`）—— 一旦投产，业务代码大量依赖，签名不能因为某一处文档伪码不符就反过来改库；应改 md。
- 业务 Service 方法（设计阶段）—— 签名以 md 为准，代码必须服从。

### 5.4 触发后的动作

1. 报告差异给用户：「md 写 `X(a, b)`，代码是 `X(a, b, c)`」
2. 由用户裁决以哪一侧为准
3. 默认导向"代码已在生产 → 改 md" / "代码未投产 → 改代码"
4. 修正后必须更新 BUILD_STATUS.md / INDEX.md（如果涉及）

---

## 6. BUILD_STATUS 先查

### 6.1 规则

生成任何"已有文档"的代码前，**必须先读 `docs/BUILD_STATUS.md`**，区分：

| 标记 | 含义 | AI 行为 |
|------|------|---------|
| 🟢 | 已实现且在运行时装配 | 按既有签名调用，不得重复创建 |
| 🟡 | 代码已落盘但未装配 | 启用前先读取代码确认签名 |
| ⬜ | 设计已定稿，代码尚未创建 | 按 Skill 生成 |
| 🚫 | 设计已定稿，但当前阶段明确不实现 | **拒绝生成**，回复"暂不实现"原因 |
| ❌ | 未设计且未实现 | 停止，先补 md |

### 6.2 反例（不应该的行为）

- 文档描述了 `{auth-module}.{Method}` 方法 → AI 立即生成 `service/{module}/{module}.go::{Method}` —— **错**，应先查 BUILD_STATUS，可能已是 🟢
- 任务文档把 `{disabled-task}` 列为 N 个定时任务之一 → AI 自动生成 cobra 命令 —— **错**，可能 🚫 暂不实现

---

## 7. 暂不实现模块（项目期决策建模）

### 7.1 概念

实际工程里经常出现：**某模块设计已完成，但因业务/合规/优先级原因，当前阶段不投入实现**。

如果不显式登记这种状态，AI 容易：
1. 看到完整 md → 自动生成代码 → 引入未规划的依赖
2. 把"设计完整"误等同为"应该实现"

### 7.2 标准做法

在 `BUILD_STATUS.md` 的顶部专设 §0 节登记所有"暂不实现模块"，包括：

- 涉及代码路径（明确"哪些不生成"）
- 涉及文档（明确"完整保留不动"）
- 业务校验链/调用链中"该模块原本承担的职责"如何被替代（通常是空壳实现/直接放行）
- Skill 检查规则（命中时立即拒绝）
- 解除条件（什么时候可以删除本节、恢复实现）

### 7.3 Skill 联动

所有 Skill 在第 0 步必须执行 BUILD_STATUS 拒绝规则匹配。命中 🚫 时直接拒绝，回复 §0 的拒绝理由。

### 7.4 参考实现

"暂不实现模块"机制的完整模板（涉及代码路径 / 关联文档 / 调用链替代实现 / Skill 黑名单 / 解除条件）见 [`aiweave/docs-spec/02_build_status_md_spec.md`](docs-spec/02_build_status_md_spec.md) §3。

---

## 8. 测试是验收的唯一标准

### 8.1 铁律

任何 Skill 在生成业务代码后，**必须**同步新增/扩展对应测试用例。

- 代码 + 测试 + 文档同步是一次完整交付
- 测试红了 = 未交付
- 漏写测试视为未交付

### 8.2 用例覆盖底线

每个新接口至少 4 类用例：

| 类别 | 含义 |
|------|------|
| 正向 | 合法输入 → 期望成功 |
| 参数校验 | 缺字段/格式错/超范围 → 对应错误码 |
| 业务错误 | 状态分支（{状态枚举值}/{资源不足类错误}/权限不足等）→ 对应错误码 |
| 副作用 | MySQL/Redis/MQ 是否被正确写入 |

### 8.3 例外

详见 `aiweave/docs-spec/17_testing_design_spec.md` §8。常见例外：纯数据结构、framework 内部、🚫 暂不实现模块。

---

## 9. 自检清单（每次交付前过一遍）

无论建设模式还是增量同步模式，每次交付都按此清单自检：

- [ ] 本次新增了代码文件？→ 是否有对应 md？
- [ ] 本次修改了已有代码？→ 是否更新了关联 md？
- [ ] 本次新增/修改了 md？→ 是否更新了 INDEX.md？
- [ ] 本次删除了代码或 md？→ 是否同步清理了另一侧？
- [ ] 本次新增的代码是否同步了测试用例？
- [ ] 本次修改是否触碰了 🚫 暂不实现模块？（应拒绝）
- [ ] 本次修改是否更新了 BUILD_STATUS.md 的状态列？
- [ ] **B1 增量同步专属**：是否按 docs-spec/19 的"全链路文档变更清单 + 自检结果 + 文档同步声明"格式输出？

每次交付的回复末尾必须声明文档同步状态：

- 有变更：「文档同步：更新了 `docs/xxx.md`、新建了 `docs/yyy.md`」
- 无变更：「文档同步：本次修改不涉及文档变更」

---

## 10. 设计先行流程

```
⓪  用 design-spec 十三视角做架构决策 → 技术方案(TRD)（可选；复杂需求强烈推荐，执行器 /design-solution）
        ↓
①  写 / 改 md（按 TRD 落地，技术方案文档）
        ↓
②  AI 按 md 生成代码（调用 Skill）
        ↓
③  go build / go test 验证 + 反向同步 md
```

即使用户跳过 ⓪（简单需求）或跳过 ① 直接让 AI 改代码，③ 仍然是强制的。

---

## 11. AI 与人的分工

### 人（设计者）应做的

- **写好设计文档** —— 这是核心产出
- **审查 AI 输出** —— 尤其是 Service 业务逻辑、Redis 操作、错误码选择
- **维护接口契约** —— API 规格文档是接口定义的权威来源
- **定期审计** —— 每次发布前跑一次 `/doc-sync-check all`

### 人不需要做的

- 手写 GORM model（AI 从 DDL 生成）
- 手写 Controller boilerplate（AI 从接口规格生成）
- 手写路由注册（AI 自动追加）
- 手写定时任务框架（AI 从任务设计生成）
- 维护 INDEX.md（`/update-index` 自动更新）

### AI 不应做的

- 不查 BUILD_STATUS 就生成代码
- 跳过文档同步直接交付
- 在 🚫 模块上生成代码
- 修改 md 中"既有契约"的签名（应改代码服从 md，除非命中 §5 例外）
- 一次性 `/rebuild-from-docs all`（应分 Stage 推进）

---

## 12. 占位符规则（强制 / 双轨结构）

### 12.1 核心断言

AIWeave 自身不耦合任何具体业务领域。`docs-spec/*` 与 `skills-spec/*` 中**主体规范内容必须使用占位符**表达，不得出现具体业务名（如 `Register` / `RegisterReq` / `userId` / `verify_code` / `balance` / 订单 / 用户 / 余额 / 注册 / 登录 / 扣减 等）。

这是 §1 第一性目标的直接推论：AIWeave 是要让 AI 能在**任何业务工程**中重建代码的规范框架，规范本体一旦绑死某个业务，就会污染读者建立"规范结构 vs 业务内容"的心智模型。

### 12.2 双轨结构（强制）

为兼顾"零业务耦合"和"AI 能直观学习规范结构"两个目标，每篇 `docs-spec/*` 采用双轨结构：

| 轨道 | 章节范围 | 内容要求 |
| --- | --- | --- |
| **规范本体** | `§1 ~ §N`（含主体内的所有示例代码块） | 仅使用 `{Module}` / `{Method-N}` / `{ModuleReq}` 等占位符；所有路径、Key、字段名、ID 命名都占位化；路径参数使用 `{prefix}` / `{audience}` / `{module}` / `{action}` |
| **参考示例** | 末尾独立小节，统一标题"参考示例（仅示意，落地按业务替换）" | 可使用具体业务名做示意，但必须在小节首行用 `> ⚠️` 显式标注"以下为示例，规范本体已用占位符；落地工程时按业务语义替换" |

### 12.3 占位符约定（最小集）

| 类型 | 推荐占位符 | 反例（出现在主体即违规） |
| --- | --- | --- |
| 模块名 / 业务领域 | `{Module}` / `{Module-N}` / `{module}` / `{module-N}` | account / order / product / billing |
| 方法名 / 动作 | `{Method}` / `{Method-N}` / `{create-method}` / `{state-change-method}` | Register / Login / Deduct / Activate |
| 入参出参类型 | `{Module}{Method}Req` / `{Method-N}Req` / `{Method-N}Resp` | RegisterReq / LoginResp / VerifyReq |
| 业务实体 ID | `{entity-A-id}` / `{prefix-A}{date}{seq}` | userId / orderId / logId / rechargeId |
| 数据表 | `{example_table_*}` / `{table-name}` | account / order / billing_log |
| Redis 命名空间 | `{ns}` / `{ns-N}` / `{state-N}` | acc / verify_code / balance |
| 数值字段 | `{amount-field}` / `{核心数值字段}` | balance / amount / quota |
| 状态字段 | `{status-field}` / `{状态字段}` | order_status / user_status |
| 接口路径 | `/{prefix}/{audience}/{module}/{action}` | /api/operator/account/register |
| 调用方 | `{Audience}` / `Operator` / `Admin`（限定通用值） | UserSelf 之类业务化命名 |

> 占位符不要求穷尽，原则是：**任何一个占位符替换为具体业务名后是合法的，但反向不行**——主体里出现具体业务名就是泄漏。

### 12.4 触发审计（强制 / doc-sync-check 子规则）

`doc-sync-check` 在扫描 `docs-spec/` 与 `skills-spec/` 时，应执行"业务名泄漏"专项检查：

| 步骤 | 动作 |
| --- | --- |
| 1 | 锁定文件主体范围（从 `# 标题` 到最后一节 "维护"，但**不含**任何标题为"参考示例"或"参考实现"或带 `示例` / `仅示意` 字样的小节） |
| 2 | 在主体范围内 grep 业务名关键词清单（不区分大小写）：<br>**英文动作 / 类型**：Register / Login / Logout / Verify / Report / Recharge / Deduct / Activate / Refund / Withdraw<br>**英文实体 ID**：userId / orderId / accountId / rechargeId / logId<br>**英文 struct**：RegisterReq / LoginReq / VerifyReq / VerifyResp / ReportReq / AccountService / BillingService / UserService / OrderService<br>**英文字段 / 概念**：verify_code / balance / password_hash / access_token / refresh_token / app_secret<br>**英文业务模块名**（与 §12.3 反例对照表对齐）：<br>&nbsp;&nbsp;- 典型业务模块：account / order / billing / product / merchant / payment / recharge / customer / cart / checkout<br>&nbsp;&nbsp;- 作 `package` / 路径 `service/` / `notice.module` 标签值 / Redis 命名空间前缀等出现皆视为泄漏<br>**英文业务字段 / 业务量化**：<br>&nbsp;&nbsp;- 业务标识字段：app_key / api_code / account_id / order_id / product_id / merchant_id<br>&nbsp;&nbsp;- 业务量化字段：total_amount / used_amount / remaining / quota<br>&nbsp;&nbsp;- 业务计数字段：total_count / success_count / fail_count / counted_count<br>&nbsp;&nbsp;- 业务消息字段：msg.cost / msg.billed / msg.statusCode（业务语境而非 HTTP）<br>&nbsp;&nbsp;- 业务过期时间：expire_at（业务过期，vs 通用 TTL）<br>**英文文件 / 函数 / 变量名**：account_flow / auth_flow / billing_flow / pickMode / generateUserId / userSeq / monthly-report / daily-report / call_log_consumer / CleanExpiredQuota / FlushDailyStats（任务名 / 方法名含业务词）<br>**中文业务概念**：用户 / 订单 / 余额 / 注册 / 登录 / 扣减 / 充值 / 开户 / 套餐 / 状态扣减 / 状态上报 / 余额检查 / 负余额 / 余额已扣 / 余额不足 / 用户禁用 / 用户态 / 额度 / 已使用量 / 总额度 / 剩余额度 / 计费 / 结算 |
| 3 | 命中 → 标 🔴 严重；要求迁移到末尾"参考示例"段或改占位符 |
| 4 | 末尾"参考示例"段不参与扫描，但段首必须有 `> ⚠️` 警告行（缺失 → 标 🟡 中度） |

误报抑制：在主体单行末尾追加 `<!-- aiweave:allow=domain-leak reason=... -->` 注解可豁免该行。豁免必须填写理由，且每个文件不超过 3 处。

### 12.5 反例 / 正例对照

```markdown
❌ 反例（主体段）：
   func (s *AccountService) Register(req *RegisterReq) (*RegisterResp, error)

✅ 正例（主体段）：
   func (s *{Module}Service) {Method}(req *{Module}{Method}Req) (*{Module}{Method}Resp, error)
   并在末尾"参考示例"段附 Register 具体例子。

❌ 反例（不变量表）：
   | 1 | 用户余额 ≥ 0 | 每次扣减前 | service/billing/deduct.go |

✅ 正例（不变量表）：
   | 1 | {核心数值字段} ≥ 0 | 每次{扣减/变更}前 | service/{module}/{action}.go |

❌ 反例（路径）：
   GET /api/internal/verify

✅ 正例（路径）：
   GET /{prefix}/internal/{action}
```

### 12.6 例外

下列情形不视为业务名泄漏：

1. **AIWeave 框架自身的固定术语**：`AIWeave` / `BUILD_STATUS` / `INDEX.md` / `OPERATIONS.md` / `PRINCIPLES.md` / `docs-spec` / `skills-spec` / Skill 名（如 `doc-sync-check` / `new-service`）等。
2. **Go / Gin / GORM / Redis / Kafka 等技术栈固定 API**：`gin.Context` / `*ServiceError` / `base.Error` / `ShouldBindJSON` / `HGET` / `SPOP` 等。
3. **HTTP / 协议层固定术语**：`Header` / `Query` / `Body` / `Cookie` / `JSON` / `binding:"required"` 等。
4. **占位符内部包含的语义提示词**：`{state-report-method}` / `{actor-id}` / `{state-cleanup-task}` / `{verify-fn}` 等大括号包裹的 kebab/snake 标识符——花括号本身是占位符语法的清晰标识，内部的 hint 词不视为泄漏。审计时应剔除所有 `{...}` 占位符内的内容再做关键词匹配。
5. **中文"用户"指代开发者/操作者/调用方时**：在 docs-spec / skills-spec / OPERATIONS / templates/skills 等"指导文档"中，"用户"常指阅读规范的开发者或调用 Skill 的操作者（如"提示用户"、"用户裁决"、"用户提供需求文档"），而非业务领域中的 end-user，这类用法不视为泄漏。判定标准：上下文是"AI ↔ 开发者"协作时为 OK；上下文是"系统 ↔ end-user 业务"时为泄漏。模糊时优先改写为"开发者"/"操作者"/"调用方"以消歧义。
6. **§12 自身定义规则的关键词清单与对照表**：本节 §12.3 对照表的"反例值"列、§12.4 关键词清单本身——它们必须列出业务名才能定义规则；这是"meta 内容"，不参与审计。
7. **AIWeave 规范本体引用具体 template 文件名**：当规范本体需要链接到 `templates/docs/architecture/auth_flow.md` 等具体 template 文件时，因为 template 文件名是物理实体（git 中的真实文件路径），可以原样引用，但应同时注明"template 文件实际名为 X，落地按业务重命名"。
8. **末尾"参考示例"段内**：可任意使用具体业务名，但段首必须有警告行。

### 12.7 与 §1.5 B1 反向同步的关系

B1 要求"建设规则伴随维护规则"。§12 在此基础上加一层"建设规则与维护规则都必须用占位符表达"——任何 B1 反向同步规则的迹象列、动作列若需举例，应使用占位符或显式归入"参考示例"段。

---

## 13. IO 极致（三条铁律 / 读写分治 / 聚合优先）

### 13.1 第一性推论

"AI 可重建工程"不仅要求行为等价，也要求**重建出的代码默认就是 IO 高效的**。一个把请求放大成 O(N) 次往返的实现，即便行为正确也不可接受。因此 IO 聚合是 AIWeave 的一等纪律。IO 的核心问题可拆成三个可度量维度：**次数（消灭 N+1）· 串/并行（消灭独立串行）· 往返（消灭远端往返）**——把它们压到最小，工程的技术天花板自然抬高。

### 13.2 三条铁律（强制 / 不可绕过）

一条正向总则统摄两条负向强制：

| 铁律 | 含义 | 正确做法 |
| --- | --- | --- |
| **铁律零：批量优先** | 对"一批 ID / key"的查询必须用批量方法一次完成；数据层若缺批量方法，**先补 `{...List}` 方法**再调用，禁止退化为循环单查 | 先在数据层补批量方法（分表按 shard 分组并行），上层一次批量调用 |
| **铁律一：禁止 N+1** | 禁止在循环 / 递归内对同类资源逐条发起独立查询（DB / 缓存 / RPC / 外部 API） | 循环外收集 key → 一次批量读 → `map` 装配 |
| **铁律二：禁止独立串行编排** | 禁止把多个**互不依赖**的 IO 逐个串行 await | 聚合器或扇出并行；首错聚合，其余 best-effort |

> **唯一例外**：铁律二中，当后一次查询的入参**真实依赖**前一次的结果（数据依赖链）时，串行是必要且合法的。AI 必须能区分"伪串行（可并行）"与"真依赖（必须串行）"。
>
> **反向纪律**：纪律必须同时规定"何时不做"——同阶段只有 1 个查询时禁止套并行编排器；已被本地缓存命中的读禁止再合并 Pipeline（会绕过本地缓存反而劣化）。

### 13.3 读路径 / 写路径分治

读和写是**两种相反的 IO 模式**，必须用相反的手段优化；把它们抹平进统一"数据访问层"是多数性能设计失败的根因。

| 路径 | 目标 | 手段方向 |
| --- | --- | --- |
| **读路径** | 极低延迟 | 消灭往返优先于合并往返：本地命中（0 网络）> 单次往返；消重复读、分支裁剪 |
| **写路径** | 极高吞吐 / 削峰 | 先写内存态原子命令 → 脏标记积攒 → 定时增量批量落库；纯追加走 MQ 批量 |

一个请求里同时存在读写时，分别用两套手段。读写分治的决策方法论见 [`design-spec/01 §3`](design-spec/01_io_design.md)（读/写两棵决策树）。

### 13.4 聚合优先

涉及"对集合逐元素访问资源"或"多个跨实例 / 跨数据源读取"时，**默认走聚合器**（收集 → 按实例分组批量读 → 并行回源 → 单飞合并 → 异步写回），把分散 IO 重写为"批量 + 并行"。聚合器的共享结果指针**只读**，禁止就地改字段。复用本方法论时**借纪律不借实现**（[`design-spec/00 §2.4`](design-spec/00_design_overview.md)）——照搬铁律与判定标准，按自己的数据形态重选原语（同质大对象 MGET、异构小字段 Pipeline）。

完整规则见 [`docs-spec/25_io_aggregation_spec.md`](docs-spec/25_io_aggregation_spec.md)；架构决策方法论见 [`design-spec/01_io_design.md`](design-spec/01_io_design.md)；落地约束清单与 grep 锚见 `io_contract.md`。三条铁律由 `/io-review` 审计（L4）+ Hooks（L0，见 §14）双层兜底。

---

## 14. Hooks 机制（L0 自动化防御 / 强制）

### 14.1 为什么需要 Hooks

L1-L5 防御（文档约束 / Skill 步骤 / 自检清单 / 审计 Skill / 测试）都依赖"AI 自觉执行规范"。Hooks 是**不依赖 AI 自觉**的自动化兜底——由 Claude Code 在工具事件触发时机械执行，构成 **L0 防御层**（事前/事中自动拦截，优先于一切依赖自觉的层）。

### 14.2 两类强制 Hook（规范要求）

| Hook 族 | 事件时机 | 职责 |
| --- | --- | --- |
| **IO 铁律检查 Hook** | 编辑 / 提交 `.go` 文件后 | 机械跑 `R-IO-*` grep 锚，命中三条铁律（批量优先 / N+1 / 串行编排）→ 告警 / 阻断，提示走聚合器 |
| **代码 ↔ 文档双向同步 Hook** | 编辑 `.go` 后 / 会话结束（Stop） | 编辑后按范围判定表提醒同步对应 md；Stop 时校验回复末尾有"文档同步：..."声明 |

> 团队级强制 Hook 放共享配置（入 git）；本机调试 Hook 放本地配置（不入 git）。完整结构、事件、配置见 [`skills-spec/02_settings_local_json_spec.md`](skills-spec/02_settings_local_json_spec.md) §4。

### 14.3 Hooks（L0）与 L1-L5 的关系

Hooks（L0）与 L1-L5 是**叠加非替代**：L0 自动拦截高频低级错误（漏声明、明显 N+1），把人和 AI 的注意力留给 L4 审计需要判断的复杂一致性问题。任一层失效，相邻层兜底。完整对照见 [`OPERATIONS.md`](OPERATIONS.md) 附录"6 层防御 × W 工作流对照表"。



---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
