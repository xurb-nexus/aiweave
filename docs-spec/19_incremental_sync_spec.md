# 19 - 增量需求同步规范（核心场景）

> 规定 **"开发完一个需求后，把需求设计 + 代码改动 + 本规范一起交给 AI，让 AI 补全/完善工程的全链路文档"** 这一场景的标准流程、输入信物、判定逻辑、输出契约。
>
> 这是规范的**第二种使用模式**——与"从零建设" / "从 docs/ 重建代码"并列的核心场景。

---

## 1. 为什么这个场景重要

工程一旦上线，进入持续演进期。每个新需求都会带来：

- 新增 / 修改的 API 接口
- 新增 / 修改的 Service 方法
- 新增的数据表 / 表字段 / 索引
- 新增的 Redis Key / Hash 字段
- 新增的错误码 / 中间件 / 定时任务 / MQ topic
- 新增的常量 / 工具函数 / 全局变量

如果每次需求迭代后**没有把改动同步回 docs/**，几次迭代之内：
- BUILD_STATUS 跟实际不符
- INDEX 漏登记
- 接口规格文档过时
- 缓存设计的 Hash 字段映射缺失新字段
- AI 在后续工作中拿到的是过时的 md，开始生成错代码

**结论**：每次需求交付，**必须**触发一次"增量同步"，把改动反映到全链路文档。

---

## 2. 与"从零建设"模式的区别

| 维度 | 建设模式（forward / rebuild） | 增量同步模式 |
|------|----------------------------|------------|
| 起点 | docs/ 完整 / 代码空白 | docs/ + 代码都已存在 |
| 输入 | docs/ + skills/ | docs/ + skills/ + **新需求技术文档** + **代码改动 diff/PR** |
| AI 任务 | 按 docs/ 生成代码 | 按代码 + 需求文档反向更新 docs/ |
| 触发频率 | 工程启动时 / 大重构时 | 每次需求迭代结束时 |
| 关键 Skill | `/new-*` 系列 + `/rebuild-from-docs` | `/sync-feature-to-docs` + `/doc-sync-check` |

---

## 3. 输入信物（用户给 AI 必须提供）

AI 要做高质量增量同步，需要 **3 类输入** 全部到位：

### 3.1 输入 A：本规范（aiweave/）

- 整套规范的 docs-spec/ + skills-spec/
- 让 AI 知道每类改动应该反映在哪些 md 中

### 3.2 输入 B：新需求的技术设计文档

包括但不限于：

- **需求描述**：业务背景、目标、用户故事
- **接口变更清单**：新增 / 修改的接口（含字段定义、校验规则）
- **数据模型变更**：新增表 / 修改表 / 索引调整
- **缓存变更**：新增 Redis Key / Hash 字段 / TTL 调整
- **业务规则变更**：状态转换、业务规则、限流策略
- **依赖变更**：新外部 API、新基础设施

### 3.3 输入 C：代码改动的具体形式

至少之一：

- **Git diff 或 PR**：`git diff main..feature`
- **改动文件清单**：列出所有新增 / 修改 / 删除的文件
- **关键代码片段**：复杂逻辑的实际实现

---

## 4. 输出契约（AI 必须交付）

AI 完成增量同步后，必须产出：

### 4.1 全链路文档变更清单

按 **范围判定表**（CLAUDE.md / 规范的核心机制）扫描，列出：

```markdown
## 文档变更清单

### 新增的 md
- docs/api/{audience}_interfaces.md → 新增 §N.X 接口规格
- docs/service/{module}_service.md → 新增 §N.X 方法详情
- docs/cache/cache_design.md → 新增 §2.X.Y Hash 字段映射

### 修改的 md
- docs/schema/database_design.md → §3.N 表结构（新增字段）
- docs/architecture/status_codes.md → §3.3 新增错误码
- docs/architecture/constants.md → §X 新增常量
- docs/INDEX.md → 同步登记
- docs/BUILD_STATUS.md → 状态更新

### 无变更（已审查）
- docs/architecture/middleware.md → 无影响
- docs/circuit_breaker/circuit_breaker_design.md → 无影响
```

### 4.2 每条变更的 diff 摘要

```markdown
### docs/api/audience_b_interfaces.md
\`\`\`diff
@@ §6.3 新增 ...
+ ### 6.3.5 批量导出 Key
+
+ \`\`\`
+ POST /{prefix}/operator/{module}/{action}
+ \`\`\`
+ ...
\`\`\`
```

### 4.3 自检结果

```markdown
## 自检清单

- [x] 新增代码文件 → 都已找到对应 md（如 service/foo.go → service/foo_service.md）
- [x] 修改代码文件 → 都已更新关联 md
- [x] 新增 md → 已登记到 INDEX
- [x] 删除代码 → 同步清理 md
- [x] 新增的代码已同步测试用例（test/cases/...）
- [x] 改动是否触碰了 🚫 暂不实现模块？否
- [x] 已更新 BUILD_STATUS 对应行
```

### 4.4 命中"签名冲突裁决"的报告

如果发现新代码的签名与既有 md 描述冲突：

```markdown
## 签名冲突报告（裁决：以代码为准）

- `helpers.NewFooClient` 的签名变了（新增了 timeout 参数）
- 已修正 docs/architecture/helpers_api.md §3 的函数签名
- 影响范围：3 个 service 调用点（service/foo / service/bar / service/baz）已升级
```

### 4.5 文档同步声明（CLAUDE.md 输出格式）

```markdown
文档同步：更新了 docs/api/audience_b_interfaces.md、docs/service/{module-2}_service.md、docs/schema/database_design.md、docs/INDEX.md、docs/BUILD_STATUS.md
```

---

## 5. AI 的判定流程（标准 SOP）

### 5.1 第 1 步：读输入

```
1.1 读取 aiweave/PRINCIPLES.md + aiweave/docs-spec/00..19（本规范全部）
1.2 读取本工程的 docs/INDEX.md + docs/BUILD_STATUS.md（理解当前状态）
1.3 读取本工程的 CLAUDE.md（拿到范围判定表）
1.4 读取用户提供的"需求技术文档"
1.5 读取 git diff / 改动文件清单
```

### 5.2 第 2 步：拒绝规则

```
2.1 检查改动是否触碰 BUILD_STATUS §0 中的 🚫 模块
    → 命中：立即停止，向用户提示"触碰了暂不实现模块，请确认"
2.2 检查改动是否包含 🚫 模块的代码（应当不存在）
    → 命中：报告异常，由用户裁决
```

### 5.3 第 3 步：差异分类

按 **CLAUDE.md 范围判定表** 对改动文件分类：

```
3.1 模型层改动（models/）→ docs/schema/database_design.md
3.2 Service 改动 → docs/service/{module}_service.md + service_design.md
3.3 Controller 改动 → docs/api/{audience}_interfaces.md
3.4 Middleware 改动 → docs/architecture/middleware.md
3.5 Router 改动 → docs/architecture/routing.md
3.6 定时任务改动 → docs/service/scheduled_tasks_design.md
3.7 MQ 消费者改动 → docs/service/scheduled_tasks_design.md §3
3.8 Helpers 全局变量 / 工具函数 → docs/architecture/helpers_api.md
3.9 缓存操作 / 新 Redis Key → docs/cache/cache_design.md
3.10 错误码 → docs/architecture/status_codes.md
3.11 常量 → docs/architecture/constants.md
3.12 配置 → docs/architecture/config.md
3.13 中间件链 / 路由组 → docs/architecture/routing.md
3.14 性能 / 安全约束 → docs/architecture/overview.md
3.15 测试用例 → 校验 test/cases/{audience}/ 是否同步
```

**约束类代码迹象 → 约束类文档**

```
3.16 sync.Mutex / sync.RWMutex / sync.Map / atomic.* 新增
     → docs/architecture/concurrency_safety.md §2 共享状态注册表
3.17 go func() / errgroup.Go / make(chan T) / make(chan T, N)
     → docs/architecture/concurrency_safety.md §1.1 / §4.1
3.18 db.Transaction() / BeginTx / tx.Commit / tx.Rollback / Saga 步骤标记
     → docs/service/transaction_design.md §2 / §3 / §6 失败路径全景图
     → 同步 docs/service/{module}_service.md §4.N.8 事务与一致性
3.19 SETNX / 唯一索引 / 幂等校验类代码
     → docs/service/transaction_design.md §4 幂等性设计
3.20 涉及业务规则校验的 if 分支（如 if {amount-field} < 0）
     → docs/service/{module}_service.md §7 领域不变量（业务规则 / 隐式约束）
     → 伪码 §4.N.3 添加 [INVARIANT-CHECK: I-N] 标记
```

**新增（已激活）：**

```
3.21 涉及 metrics 注册（helpers/metrics.go）
     → docs/architecture/observability.md §2 Metrics 采集范围 + §3 Cardinality 评估
3.22 涉及热路径文件（performance_contract.md §2 清单）
     → docs/architecture/performance_contract.md §2 / §3 / §6
     → 方法伪码加 [HOT-PATH] 标记
3.23 新增告警规则（YAML / Grafana / Alertmanager）
     → docs/architecture/observability.md §5.2 基础告警规则
3.24 新增 sync.Pool / *_bench_test.go 文件
     → docs/architecture/performance_contract.md §3.2 / §7.1
3.25 GORM struct 字段加 column:"-" / DROP COLUMN / 字段类型变更
     → docs/schema/database_design.md §8 字段演进历史
3.26 提交说明含 "refactor:" 前缀且改动 ≥ 100 行
     → docs/architecture/mvp_rebuild_path.md §11 安全重构方法论
     → 如涉及行为变更 → §11.5.4 灰度切流量
```

**跨服务合约 / 不变量 / 失败路径相关：**

```
3.27 api/ 目录新增 client / pkg/ 引入新 RPC 包
     → docs/architecture/cross_service_contract.md §3 下游合约
3.28 新增 server 端路由被外部调用方接入 / 修改对外接口字段
     → docs/architecture/cross_service_contract.md §2 上游合约 + §5 接口版本管理
3.29 修改下游调用的超时 / 重试参数
     → cross_service_contract.md §2/§3 + 评估对 §4 故障传播矩阵的影响
3.30 引入新的 5xx / 业务错误分支 / Service 方法新增 return Error 路径
     → cross_service_contract.md §4 故障传播矩阵 + transaction_design.md §6 失败路径
     → 同步测试用例（按 testing_design.md §5.5 高级用例 3 类）

3.31 新增 / 修改业务规则校验代码（如 if {amount-field} < 0）
     → {module}_service.md §7.1 业务规则 + 伪码 [INVARIANT-CHECK: I-N] 标记
     → BUILD_STATUS.md §11.3 领域不变量计数

3.32 修改状态字段流转条件（UPDATE ... SET status=?）
     → {module}_service.md §7.3 业务状态机转换条件表
```

约束类文档同步动作的详细 B1 反向同步规则见各规范的"维护流程"章节。

### 5.4 第 4 步：每篇 md 的更新动作

按各 md 对应的 docs-spec/ 规范执行：

| 改动类型 | 必须更新的字段 |
|---------|-------------|
| 新增 API | docs-spec/07 §5 单接口完整规格 / docs-spec/09 §10 单 service 方法 / docs-spec/04 routing 表 |
| 新增表 | docs-spec/05 §3 完整 DDL / §4 Redis Key / §6 GORM 规范 |
| 新增 Hash 字段 | docs-spec/06 §4.2 Hash 字段映射 + database_design §4.6 同步 |
| 新增错误码 | docs-spec/14 §2.3 / §3.3 清单 + 业务状态枚举 |
| 新增常量 | docs-spec/16 §X + §10 components/constants.go 同步 |
| 新增中间件 | docs-spec/04 §5 中间件子节 |
| 新增 Skill 调用方 | docs-spec/02 BUILD_STATUS / 范围判定表 |

### 5.5 第 5 步：交叉一致性检查

```
5.1 INDEX.md 是否登记新增 md？
5.2 BUILD_STATUS 状态是否更新（⬜ → 🟢 / 🟡）？
5.3 新增的 Redis Key 在 cache_design.md §4 与 database_design.md §4 是否双向出现？
5.4 新增的常量是否在 constants.md §X 表 与 §10 Go 文件都出现？
5.5 新增的接口是否在 api/*.md 与 routing.md §5 路由表都登记？
5.6 新增的 service 方法是否在 service_design.md §6 接口映射表登记？
5.7 新增的 helpers 函数是否在 helpers_api.md §1 / §2 登记？
5.8 新增 / 修改的代码是否同步了测试用例？

5.9  新增共享状态（sync.* / chan）是否在 concurrency_safety.md §2 / §4 双向登记？
     是否同步 BUILD_STATUS §11 约束清单状态轨道？
5.10 新增事务边界（BeginTx / Saga 步骤）是否在 transaction_design.md §2 + 
     service docs §4.N.8 双向登记？
5.11 新增业务规则校验代码是否在 {module}_service.md §7.1 业务规则表登记？
     伪码 §4.N.3 是否添加 [INVARIANT-CHECK: I-N] 标记？
5.12 新增 / 修改的事务相关方法伪码是否使用统一标记语法
     （[TXN-*] / [SAGA-STEP-N] / [IDEMPOTENT-CHECK] / [INVARIANT-CHECK] / [LOCK-*]）？

【约束类扩展】
5.13 新增 Metric 是否在 observability.md §2 登记？label 是否命中 §3.2 禁止清单？
     Cardinality 是否超 §3.1 默认上限？超出是否在 ai_dev_guide.md §9 登记理由？
5.14 新增热路径方法是否在 performance_contract.md §2 清单登记？
     伪码是否加 [HOT-PATH] 标记？是否触犯 §2.2 禁止操作清单？
5.15 channel 容量是否在 performance_contract.md §6.1 与 concurrency_safety.md §4.1
     双向对齐？
5.16 修改字段类型 / 删除字段是否在 database_design.md §8 字段演进历史登记？
     物理删除前是否经过至少一个发版周期的软兼容期？
5.17 涉及 refactor 类提交是否符合 mvp_rebuild_path.md §11 安全重构约束
     （≤ 200 行 / ≤ 5 commit / 灰度切流量）？
5.18 涉及方法伪码使用 [HOT-PATH] / [METRIC-EMIT] 标记的，对应的代码实现是否真发出指标？

5.19 新增 api/ client / pkg/ RPC 包 是否在 cross_service_contract.md §3 下游合约登记？
     是否显式设了超时（< 上游超时）？
5.20 新增对外路由是否在 cross_service_contract.md §2 上游合约登记？
     接口签名变更是否按 §5 评估是否升主版本号？
5.21 Service 方法新增 return Error 路径是否在 transaction_design.md §6 失败路径全景图
     或 cross_service_contract.md §4 故障传播矩阵覆盖？是否有对应测试用例？
5.22 修改业务规则校验代码是否同步 {module}_service.md §7.1 + 伪码 [INVARIANT-CHECK] 标记？
     [INVARIANT-CHECK] 标记与实际校验代码匹配率 ≥ 95%？
```

### 5.6 第 6 步：输出与确认

按 §4 输出契约组织最终回复。**不要静默修改 md** — 必须列出所有变更清单让用户审阅。

---

## 6. 不允许的简化

```markdown
❌ 仅看代码 diff，不读需求技术文档
   → 易遗漏"业务背景"、"为什么这样设计"等必要描述

❌ 只更新接口/Service/Schema 文档，不查 INDEX/BUILD_STATUS
   → 索引漂移、状态不准

❌ 看到代码用了某个 Redis Key 就只更新 cache_design.md
   → 必须同步 database_design §4 + constants §7（如果是新模板）

❌ 把"自检"留给用户做
   → §4.3 自检清单是 AI 必须输出的一部分

❌ 在签名冲突时不报告，直接静默改 md
   → 必须按 §4.4 显式报告，由用户裁决
```

---

## 7. 与"建设模式"的衔接

增量同步与建设模式可以共存于同一工程：

```
建设模式（forward） ──→ 工程启动 + Stage 1-N 完成 ──→ 投产
                                                       │
                                                       ▼
                                              进入持续迭代
                                                       │
                       ┌─────────────────────┬─────────┘
                       ▼                     ▼
                需求 1：增量同步         需求 N：增量同步
```

每个生产需求迭代结束时，触发一次增量同步——这是工程演进的常态。

---

## 8. 与 doc-sync-check 的关系

| Skill | 用途 |
|-------|------|
| `/sync-feature-to-docs` | **主动同步**：开发完需求后调用，根据需求文档 + 代码改动更新全链路文档 |
| `/doc-sync-check` | **被动审计**：扫描已有代码 + 文档，发现不一致项 |

最佳实践：

- 每次需求合并前 → `/sync-feature-to-docs`（保证 PR 与文档同步合并）
- 每次发版前 → `/doc-sync-check all`（兜底审计）

---

## 9. 输入质量与输出质量的关系

### 9.1 高质量输入

```
✅ 完整的需求技术文档（含背景、设计、字段定义、缓存策略）
✅ 干净的 git diff（含描述性的 commit message）
✅ 关键决策的设计依据
✅ 测试用例的覆盖说明
```

→ 高质量输出：每篇文档变更精准、自检清单全过、零遗漏

### 9.2 低质量输入

```
❌ 只给"实现了批量导出功能"一句话
❌ 只给一个 PR 链接，不附技术设计
❌ 改动横跨多个需求且未拆分
```

→ AI 必须**先提问**而不是硬猜：
- "缺少需求技术文档，本次同步可能遗漏 X 处" → 让用户补充
- "PR 改动涉及多个需求（看起来涉及 A 和 B）" → 让用户裁决拆分

---

## 10. 维护

| 触发 | 动作 |
|------|------|
| 范围判定表变更 | 同步 §5.3 |
| 输出契约调整 | §4 + 同步 sync-feature-to-docs Skill 模板 |
| 新增 docs-spec/ 视角 | §5.4 表格 |

---

## 11. 与 sync-feature-to-docs Skill 的关系

本规范是 `aiweave/templates/skills/sync-feature-to-docs/SKILL.md` 的**蓝本**——Skill 是这套流程的**可机械执行版本**，规范是 Skill 的**理论依据**。修改规范时同步检查 Skill。


---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
