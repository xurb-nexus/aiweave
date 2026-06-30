# 01 - Skill 编写指南（含公共章节真相源）

> 规定 `.claude/skills/{name}/SKILL.md` 的标准结构、frontmatter 格式、步骤化要求、文档同步契约、测试同步契约。
>
> **本文同时是 19 个 Skill 的"公共章节真相源"**——所有 SKILL.md 中的公共部分（第 0 步前置 / 第 1 步必读 / 第 4 步文档同步 / 第 5 步测试同步 / 第 6 步验证）都从下方 §A-§E 抽取，每个 SKILL.md 只保留 Skill 特定内容。

---

## 第一部分：Skill 编写规范

### 1. SKILL.md 标准结构（强制）

```markdown
---
name: {skill-name}
description: {一句话描述本 Skill 的用途}
disable-model-invocation: true
argument-hint: <参数> 例如 ...
---

根据 `$ARGUMENTS` 生成 / 同步 ...。严格按以下步骤执行。

> 公共流程模板见 [skills-spec/01_skill_authoring_guide.md](path) §A-§E；本 Skill 特定内容如下。

## 第 0 步：拒绝规则与依赖前置（公共模板见 §A，本 Skill 特定填充）
（Skill 特定的拒绝模式 + 依赖前置文件）

## 第 1 步：读取设计文档（公共必读见 §B）
- 主文档：{对应 docs/ 路径}
- 额外补充：{Skill 专属的额外必读}

## 第 2 步：编码 / 同步规则（Skill 特定）
（错误类型选择 / Client 选择 / 双写模式 / 路径规则等）

## 第 3 步：生成 / 修改文件（Skill 特定）
（代码模板 / 文件路径 / 命名规则）

## 第 4 步：文档同步（公共项见 §C）
- {Skill 特定要更新的 md}

## 第 5 步：测试同步（4 类用例标准见 §D）
- 测试位置：{Skill 特定路径表}

## 第 6 步：验证（公共格式见 §E）
（具体的 go build / go test 路径）
```

### 2. frontmatter 字段约定

| 字段 | 必填 | 含义 |
|------|------|------|
| `name` | 是 | 与目录名一字不差 |
| `description` | 是 | 一句话描述（< 100 字），用于 AI 判断何时调用 |
| `disable-model-invocation` | 是 | 设为 `true`：禁止 AI 在不调用 / 命令时自动触发本 Skill |
| `argument-hint` | 推荐 | 参数提示（如 `<service模块/方法名>`），用户调用时显示为 hint |

### 3. 步骤间强制顺序

```
第 0 步 → 第 1 步 → 第 2 步 → 第 3 步 → 第 4 步 → 第 5 步 → 第 6 步
                                              ▲
                                              │
                              第 4 / 5 / 6 步必须全部执行才算交付完成
```

**禁止跳步**。如果某步骤在特定 Skill 下不适用（如 update-index 不生成代码无需测试），SKILL.md 必须**显式说明跳过原因**，不能默认略过。

### 4. 代码模板的精度要求

每个代码模板必须做到：

- 包名一字不差（`package {module}`）
- 全部 import（不能用 `...`）
- struct 字段顺序与 docs 定义一致
- tag（json / binding / gorm）完整
- 错误类型显式
- 调用其他 service / helpers 时签名一字不差

**反例**：

```markdown
❌ "// 这里调用 service 方法" 
❌ "// 处理错误"
❌ struct 字段省略 tag
❌ import 省略
```

**正确**：

```go
result, err := svc.Method(ctx, &req)
if err != nil {
    base.RenderJsonFail(ctx, err)
    return
}
base.RenderJsonSucc(ctx, result)
```

### 5. SKILL.md 自检（写完一个 Skill 后过一遍）

- [ ] frontmatter 4 字段完整
- [ ] 第 0 步含拒绝规则 + 状态检查 + 依赖前置（已引用 §A）
- [ ] 第 1 步列出主 md + 引用 §B 公共必读
- [ ] 第 3 步代码模板可 copy-paste（无 `...`）
- [ ] 第 4 步列出 Skill 特定要同步的 md（公共项引用 §C）
- [ ] 第 5 步含测试位置（4 类用例标准引用 §D）
- [ ] 第 6 步含验证命令
- [ ] BUILD_STATUS.md §9 已新增 Skill 登记
- [ ] aiweave/INDEX.md 已新增 Skill 引用

### 6. Skill 与 docs-spec/ 的对应关系

每个 Skill 都有对应的"上游 docs-spec/"——Skill 是这条规范的**可机械执行版本**：

| Skill | 对应 docs-spec/ |
|-------|----------------|
| new-model | docs-spec/05_schema_design_spec.md |
| new-service | docs-spec/09_service_design_spec.md |
| new-controller | docs-spec/07_api_interfaces_spec.md |
| new-middleware | docs-spec/04_architecture_overview_spec.md §5 |
| new-router | docs-spec/04_architecture_overview_spec.md §4 |
| new-scheduled-task | docs-spec/10_scheduled_tasks_spec.md |
| new-mq-consumer | docs-spec/11_mq_consumer_spec.md |
| new-test | docs-spec/17_testing_design_spec.md |
| doc-sync-check | docs-spec/01_index_md_spec.md + 02 + 03 + 范围判定表 |
| update-index | docs-spec/01_index_md_spec.md |
| rebuild-from-docs | docs-spec/18_mvp_rebuild_path_spec.md |
| sync-feature-to-docs | docs-spec/19_incremental_sync_spec.md |
| new-saga-step | docs-spec/21_distributed_transaction_spec.md |
| concurrency-review | docs-spec/20_concurrency_safety_spec.md |
| performance-review | docs-spec/22_performance_contract_spec.md |
| io-review | docs-spec/25_io_aggregation_spec.md |
| domain-invariant-check | docs-spec/09_service_design_spec.md §7 |
| failure-path-review | docs-spec/21_distributed_transaction_spec.md §6 + 24_cross_service_contract_spec.md §4 |
| design-solution | design-spec/00_design_overview.md + 01..13（十三视角决策方法论，唯一对应 design-spec 而非 docs-spec 的 Skill） |

### 7. 不允许的写法

```markdown
❌ Skill 没有"第 0 步前置检查"
❌ Skill 没有"文档同步"步骤
❌ Skill 跳过测试同步（即使是 update-index 这种无业务代码的 Skill，也要明确说明"无测试要求"）
❌ Skill 描述含糊（"按需要生成"、"根据情况调整"）
❌ Skill 步骤顺序乱（先生成代码再读文档）
❌ Skill 不提及 BUILD_STATUS / 范围判定表 / pkg_api.md
```

---

## 第二部分：公共章节真相源

> 18 个 SKILL.md 中的**第 0 / 1 / 4 / 5 / 6 步**的公共部分都从下方 §A-§E 抽取。SKILL.md 中通过 `（公共模板见 §A）` 等方式引用，只填充 Skill 特定内容。

### §A 公共第 0 步：BUILD_STATUS 与拒绝规则

每个 Skill 在第 0 步必须做以下 3 类检查（顺序固定）：

**A.1 拒绝规则匹配**

读 `docs/BUILD_STATUS.md` §0「暂不实现的模块」，如果 `$ARGUMENTS` 命中 §0 中列出的任一 🚫 模块的代码路径模式（如 `{disabled_module}/*` / `admin/{disabled-module}/*` / 任务名 `{disabled-task}` 等），**立即停止**生成，并回复 §0 中给出的拒绝理由。

每个 Skill 应在其"第 0 步"中列出本 Skill 视角下需要拦截的具体模式。

**A.2 BUILD_STATUS 状态检查**

读 `docs/BUILD_STATUS.md` 对应层级的明细（如 Service 看 §4、Controller 看 §5、定时任务看 §6 等），按以下表决定如何继续：

| 状态 | 处理方式 |
|------|---------|
| 🟢 已实现 | 停止并提示用户（避免重复创建） |
| 🟡 部分实现 | 继续，仅补缺失部分 |
| ⬜ 待实现 | 完整生成 |
| 🚫 暂不实现 | 同 A.1 → 立即拒绝 |
| ❌ 未设计 | 停止，先补 md |

**A.3 依赖前置检查**

检查本 Skill 依赖的"helper / cache_loader / render 函数 / 共享文件 / 上游 service"等是否就绪。缺失则提示用户先生成依赖（或本 Skill 自动先生成依赖再继续）。

每个 Skill 应在其"第 0 步"中列出本 Skill 的依赖清单（如 controller 依赖 service、Skill 依赖 components/constants.go 等）。

---

### §B 公共第 1 步：读取设计文档（必读列表）

每个 Skill 的第 1 步在读取主设计文档之外，**必须**额外读取以下 4 篇通用真相源（顺序决定不出错）：

| # | 必读文档 | 作用 |
|---|---------|------|
| 1 | `docs/architecture/pkg_api.md` | 内部框架 pkg/ 的类型与函数签名（用错全盘崩溃，最重要） |
| 2 | `docs/architecture/helpers_api.md` | helpers/ 工具函数与全局变量（业务代码大量引用） |
| 3 | `docs/architecture/constants.md` | 业务常量 / 魔法数字唯一真相源 |
| 4 | `docs/architecture/logging.md` | 日志规范（Structured API / 级别判定 / 敏感字段） |

每个 Skill 应在其"第 1 步"中**额外**列出 Skill 专属的必读 md（如 new-service 需读 service_design + 对应 module 的 service.md + cache_design 等）。

---

### §C 公共第 4 步：文档同步检查清单

每个 Skill 在生成代码后，**必须**逐项检查并更新以下文档：

- [ ] 生成代码对应的**主 md** 与代码一致
- [ ] 实现中如发现 md 需调整 → **必须同步更新 md**
- [ ] 如新增了导出函数 / 全局变量 → 同步 `docs/architecture/helpers_api.md`
- [ ] 如新增了错误码 → 同步 `docs/architecture/status_codes.md` 和 `components/error.go`
- [ ] 如新增了魔法数字 → 同步 `docs/architecture/constants.md` 和 `components/constants.go`
- [ ] 如新增 / 重命名 / 删除 md → 同步 `docs/INDEX.md`（或调用 /update-index）
- [ ] 更新 `docs/BUILD_STATUS.md` 对应行的状态列（⬜ → 🟢 / 🟡）

每个 Skill 应在其"第 4 步"中**额外**列出 Skill 特定要更新的 md。

---

### §D 公共第 5 步：测试同步标准

**铁律**：任何 Skill 在生成业务代码后，必须同步新增/扩展 `test/cases/` 下对应测试用例。**漏写测试视为未交付**。

**用例覆盖底线**：每个新接口 / 方法至少 **4 类**用例：

| 类别 | 含义 | 常用断言 |
|------|------|---------|
| 1. 正向成功 | 合法输入 → 期望返回 | `AssertSucc` / `AssertInternalSucc` + `AssertJSONField` |
| 2. 参数校验 | 缺字段 / 格式错 / 越界 → 错误码 | `AssertErrNo(t, r, {N})` / `AssertInternalStatus(t, r, {N})` |
| 3. 业务分支 | 状态分支（{状态枚举值}/{资源不足类错误}/权限不足等）→ 错误码 | 同上 |
| 4. 副作用 | MySQL / Redis / Kafka 是否被正确写入 | `AssertMysqlExists` / `AssertRedisHashField` / `framework.Kafka.MessagesByTopic` |

**用例数量基线**：

| 接口复杂度 | 建议最少用例数 |
|------------|---------------|
| 简单 GET（查询） | 4 |
| 写操作（POST） | 6-10 |
| 复杂业务链路（如多步校验） | 20+ |
| 管理操作（如 Admin POST 含审计日志） | 6-8 |

**生成后验证**：

```bash
cd test && go test -v ./cases/{scope}/... -run Test{Method}
```

测试必须全绿。**测试红了 = 未交付**。如果业务逻辑复杂短时间跑不通，**暂停交付**，先让测试绿再提。

每个 Skill 应在其"第 5 步"中**额外**列出 Skill 特定的测试位置（不同 Skill 对应不同的 test/cases/ 子目录）。

**例外**（不强制写测试的情况）：

- 内部纯数据结构定义（如 `service/{module}/types.go` 中只有 struct）
- 代码生成 / 非业务代码（framework 内部辅助）
- 设计文档中标注为 🚫 不实现的模块
- 维护类 Skill（doc-sync-check / update-index 不生成代码，无测试要求）

---

### §E 公共第 6 步：验证命令

每个 Skill 生成代码后，**必须**运行：

```bash
go build ./{path}/...
go vet ./{path}/...
```

最后运行测试：

```bash
cd test && go test ./cases/{relevant}/... 
```

通过 → 任务完成。失败 → 根据错误回到对应步骤修正（不要绕过测试或弱化断言）。

每个 Skill 应在其"第 6 步"中**填入** Skill 特定的 `{path}` 与 `{relevant}`。

---

## 第三部分：19 个 Skill 速查表

| Skill | 用途 | 触发示例 | 主 md |
|-------|------|---------|-------|
| new-model | 从 DDL 生成 GORM Model struct | `/new-model {table_name}` | schema/database_design.md |
| new-service | 从伪代码生成 Service 方法 | `/new-service {module}/{Method}` | service/{module}_service.md |
| new-controller | 从 API 文档生成 Controller handler | `/new-controller {audience}/{module}/{action}` | api/{audience}_interfaces.md |
| new-middleware | 生成 Gin 认证中间件 | `/new-middleware {name}` | architecture/middleware.md |
| new-router | 生成路由注册文件 | `/new-router {audience}` | architecture/routing.md |
| new-scheduled-task | 生成定时任务 | `/new-scheduled-task {task-name}` | service/scheduled_tasks_design.md |
| new-mq-consumer | 生成 MQ 消费者 | `/new-mq-consumer {topic}` | service/scheduled_tasks_design.md §3 |
| new-test | 补齐测试文件（4 类用例） | `/new-test {audience}/{module}/{action}` | testing/testing_design.md |
| doc-sync-check | 审计文档与代码一致性 | `/doc-sync-check [scope]` | INDEX + BUILD_STATUS + 范围判定表 |
| update-index | 自动维护 INDEX.md | `/update-index [dry-run]` | INDEX.md |
| rebuild-from-docs | 从 docs 重建任意模块或整工程 | `/rebuild-from-docs {module}` | architecture/mvp_rebuild_path.md |
| sync-feature-to-docs | 增量需求同步（B1 场景） | `/sync-feature-to-docs {feature}` | docs-spec/19_incremental_sync_spec.md |
| new-saga-step | 生成 Saga 步骤 + 补偿 + 幂等 Key | `/new-saga-step {module}/{step}` | service/transaction_design.md |
| concurrency-review | 审计代码 ↔ 并发安全约束一致性 | `/concurrency-review [scope]` | architecture/concurrency_safety.md |
| performance-review | 审计代码 ↔ 性能合约约束一致性 | `/performance-review [scope]` | architecture/performance_contract.md |
| io-review | 审计代码 ↔ IO 铁律一致性 | `/io-review [scope]` | architecture/io_contract.md |
| domain-invariant-check | 审计代码 ↔ 领域不变量一致性 | `/domain-invariant-check [scope]` | service/{module}_service.md §7 |
| failure-path-review | 审计失败路径文档 / 测试覆盖完整性 | `/failure-path-review [scope]` | service/transaction_design.md §6 |
| design-solution | 技术方案生成：过 design-spec 十三视角产出 TRD（不生成代码） | `/design-solution {需求}` | design-spec/00..13 |


---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
