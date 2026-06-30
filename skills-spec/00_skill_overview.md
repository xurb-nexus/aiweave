# 00 - Skill 体系总览

> 规定 `.claude/skills/` 下 Skill 文件的整体结构、命名约定、职责划分、与 docs/ 的协同。

---

## 1. Skill 是什么

Skill 是放在 `.claude/skills/{name}/SKILL.md` 的 markdown 文件，告诉 AI **"执行某个研发动作时，具体要做哪些步骤、生成哪些文件、遵循什么规范"**。

每个 Skill 包含：

- **触发条件**：什么时候用这个 Skill
- **拒绝规则**：触碰 🚫 暂不实现模块时立即停止
- **分步流程**：从第 0 步（前置检查）到最后一步（验证 + 测试同步），不可跳过
- **代码模板**：精确到包名、struct 命名、错误类型选择
- **文档同步**：生成代码后必须同步更新哪些 md
- **测试同步**：同步生成 / 扩展 test/cases/* 用例
- **验证命令**：`go build` / `go vet` 确保代码正确

---

## 2. Skill 体系全景（19 个 Skill / 5 类分组）

> 设计意图：`doc-sync-check` 审计的是"代码 ↔ 文档一致性"；其他审计类 Skill 审计的是"代码 ↔ 约束一致性"。两者维度不同，互补不替代——前者保证可重建（AIWeave 第一性目标），后者保证运行时正确。

```
┌─── 设计类（A0 设计阶段 / design-spec 驱动） 1 个 ───────────────────┐
│                                                                  │
│  design-solution      输入需求 → 过十三视角 → 产出技术方案(TRD)    │
│                       （docs↔代码生命周期最上游；不生成代码/docs）  │
│                                                                  │
├─── 创建类（A 建设模式 / B 增量场景共用） 9 个 ──────────────────────┤
│                                                                  │
│  new-model            从 DDL 生成 GORM Model struct               │
│  new-service          从伪代码生成 Service 方法                   │
│  new-controller       生成 HTTP Controller handler                │
│  new-middleware       生成 Gin 中间件                             │
│  new-router           生成路由注册                                │
│  new-scheduled-task   生成定时任务（cobra/cycle/crontab）         │
│  new-mq-consumer      生成 MQ 消费者                              │
│  new-test             生成测试文件（4 类用例 + 高级 3 类）        │
│  new-saga-step        生成 Saga 步骤 + 补偿 + 幂等 Key            │
│                                                                  │
├─── 维护类（B 模式日常） 2 个 ───────────────────────────────────┤
│                                                                  │
│  update-index         自动更新 INDEX.md                           │
│  sync-feature-to-docs 增量需求同步：代码改动反向更新到全链路文档    │
│                                                                  │
├─── 审计类 6 个 ────────────────────────────────────────────────┤
│                                                                  │
│  doc-sync-check          审计 代码 ↔ 文档 一致性                  │
│  concurrency-review      审计 代码 ↔ 并发安全约束                 │
│  performance-review      审计 代码 ↔ 性能合约约束                 │
│  io-review               审计 代码 ↔ IO 铁律（N+1/串行编排）      │
│  domain-invariant-check  审计 代码 ↔ 领域不变量约束               │
│  failure-path-review     审计 失败路径全景图覆盖完整性            │
│                                                                  │
├─── 终极类（A2 重建模式） 1 个 ──────────────────────────────────┤
│                                                                  │
│  rebuild-from-docs    从文档重建任意模块或整个工程                 │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**为什么分 5 类**：

- **设计** vs **创建** vs **维护** vs **审计** vs **终极**：按"在生命周期的位置"+"是否生成代码"+"是否常态使用"交叉
- **设计类**（A0 上游）不生成代码、不写 docs/，只把需求过 design-spec 十三视角产出技术方案(TRD)，是后续一切的前置决策——它是 `design-spec/` 支柱的执行器（对应关系见 [`01_skill_authoring_guide.md`](01_skill_authoring_guide.md) §6）
- **创建类**生成新代码（A 模式 + B 模式新增场景）
- **维护类**不生成代码，只反向同步文档与索引（B 模式日常）
- **审计类**不生成代码，只检查一致性（B 模式 / 发版前兜底）
- **终极类**专用于代码丢失或换技术栈时从 docs 全量重建（A2 模式）

**为什么审计类有 6 个**：

每个审计类 Skill 对应一个维度的"一致性"：

| Skill | 一致性维度 | 关联规范 |
|-------|----------|---------|
| `doc-sync-check` | 代码 ↔ 文档 | 全部 docs-spec |
| `concurrency-review` | 代码 ↔ 并发安全约束 | docs-spec/20 |
| `performance-review` | 代码 ↔ 性能合约约束 | docs-spec/22 |
| `io-review` | 代码 ↔ IO 铁律（N+1 / 串行编排 / 聚合并行） | docs-spec/25 |
| `domain-invariant-check` | 代码 ↔ 领域不变量 | docs-spec/09 §7 |
| `failure-path-review` | 失败路径文档 / 测试 覆盖 | docs-spec/21 §6 + 24 §4 |

6 个审计类 Skill 共同构成"L4 审计层"防御（见 OPERATIONS.md 附录 "6 层防御 × W 工作流对照表"）。`io-review` 与 `performance-review` 维度互补：22/perf 管单次延迟，25/io 管往返次数与并行化。

---

## 3. 与"建设模式 / 增量同步模式"的对应

| 模式 | 主要 Skill |
|------|-----------|
| **A0 design**（技术方案生成 / 上游） | **`/design-solution`**（过 design-spec 十三视角产出 TRD，再进 A1 / B1） |
| **A1 forward**（从零开发） | `/new-*` 系列按 Stage 顺序调用 + `/new-test` |
| **A2 rebuild**（从 docs 重建） | `/rebuild-from-docs {scope}` 按 Stage 顺序 |
| **B1 sync-feature**（需求迭代后） | **`/sync-feature-to-docs`** + `/doc-sync-check` |
| **B2 refactor**（演进重构） | `/rebuild-from-docs {module}` + `/sync-feature-to-docs` |
| 维护 / 发版前 | `/doc-sync-check all` + `/update-index` |

---

## 4. 命名约定

### 4.1 Skill 名

- 全小写 + 连字符（kebab-case）
- 创建类用 `new-*`（new-model / new-service / ...）
- 维护类用动词名（doc-sync-check / update-index / rebuild-from-docs / sync-feature-to-docs）

### 4.2 文件路径

```
.claude/skills/{skill-name}/SKILL.md
```

每个 Skill 一个独立目录（即使只有一个 SKILL.md），便于将来扩展辅助文件。

### 4.3 调用方式

用户在 Claude Code 中输入 `/{skill-name} [arguments]` 即可触发。

---

## 5. Skill 之间的调用顺序（典型工作流）

### 5.1 新增 API 接口

```
/new-service {module}/{Method}      ← 先生成 service 层
       ↓
/new-controller {audience}/{module}/{action}
       ↓
/new-router {audience}              ← 追加路由注册
       ↓
/new-test {audience}/{module}/{action}
       ↓
/doc-sync-check api                 ← 审计
       ↓
/update-index                       ← 同步索引
```

### 5.2 新增定时任务

```
/new-service {module}/{TaskMethod}
       ↓
/new-scheduled-task {task-name}
       ↓
/doc-sync-check
```

### 5.3 增量需求同步（B1）

```
/sync-feature-to-docs               ← 一站式
   （内部隐式调用 update-index、doc-sync-check 局部审计）
```

---

## 6. 与 BUILD_STATUS.md 的联动（强制）

每个 Skill 的第 0 步必须做 3 项检查：

```
0.1 检查 BUILD_STATUS.md §0：是否触碰 🚫 暂不实现模块？命中 → 立即拒绝
0.2 检查目标模块当前状态：
    - 🟢：已实现，停止并提示用户（避免重复创建）
    - 🟡：部分实现，仅补缺失方法
    - ⬜：可正常生成
    - ❌：未设计，停止先补 md
0.3 检查依赖模块状态：
    - 依赖未实现 → 先调对应 Skill 生成依赖
```

---

## 7. 与 docs/ 的协同（强制）

每个 Skill 的步骤中**必须**包含：

| 步骤 | 必做 |
|------|------|
| 第 1 步：读设计文档 | 列出本次任务需要读的所有 md |
| 中间：生成代码 | 严格按 md 中的伪码 / DDL / 接口规格 |
| 倒数第 2 步：文档同步 | 生成代码后**必须**确认对应 md 已更新；如发现需调整 md，**必须同步更新** |
| 倒数第 1 步：验证 | `go build` + `go vet` + 测试 |

详细规则见 `aiweave/skills-spec/01_skill_authoring_guide.md`。

---

## 8. Skill 与测试纪律（强制）

每个生成业务代码的 Skill（new-service / new-controller / new-middleware / new-router / new-scheduled-task / new-mq-consumer），都**必须**有"同步测试用例"步骤。

详细规则见：

- `aiweave/PRINCIPLES.md` §8（测试是验收的唯一标准）
- `aiweave/docs-spec/17_testing_design_spec.md` §5（测试纪律）

---

## 9. Skill 的演进维护

### 9.1 何时新增 Skill

满足以下任一条件时考虑新增：

- 工程出现新的代码层级（如 GraphQL resolver、gRPC handler）
- 工程出现新的设计视角（如 RPC 客户端、文件存储）
- 增量同步场景遇到现有 Skill 无法覆盖的特殊改动

### 9.2 何时修改 Skill

| 触发 | 动作 |
|------|------|
| 对应 docs-spec/ 规范升级 | 同步修改 SKILL.md 中的步骤与模板 |
| 工程实践发现 SKILL 步骤遗漏 | 增补步骤 + 在 BUILD_STATUS.md §9 中记录 |
| pkg 框架升级 | 同步 SKILL.md 中的代码模板 |

### 9.3 何时删除 Skill

- 对应代码层级被废弃 → 删除 Skill 目录 + 同步 INDEX
- 由于工程演进，多个 Skill 可合并 → 谨慎合并并显式记录

---

## 10. 参考资源

| 资源 | 用途 |
|------|------|
| `aiweave/skills-spec/01_skill_authoring_guide.md` | 编写 Skill 的标准结构与最佳实践 |
| `aiweave/skills-spec/02_settings_local_json_spec.md` | settings.local.json 的 permissions / hooks 配置 |
| `aiweave/templates/skills/{name}/SKILL.md` | 19 个 Skill 的可执行骨架（同时充当规范文档；公共章节真相源在 01_skill_authoring_guide.md §A-§E） |


---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
