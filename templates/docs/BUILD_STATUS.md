# 实现进度一览（BUILD_STATUS）

> 版本：1.0 | 日期：{YYYY-MM-DD}
>
> 本文档是 docs/ 中的**一等公民**。AI 在生成任何"已有文档描述"的代码前，**必须先读本文档**，区分「已实现—可直接调用」与「待实现—需按 skill 生成」。

---

## 0. ⛔ 暂不实现的模块（如有）

> 按 `aiweave/docs-spec/02_build_status_md_spec.md` §3 模板填写。本节存在时，AI 在所有 Skill 中遇到对应模块都必须立即拒绝。

以下模块在 `docs/` 中已有完整设计，但**当前阶段明确不进入代码实现**。

| 模块 | 涉及代码路径（全部标为 🚫 暂停） | 涉及文档（保留完整设计，不动） | 在调用链中的处理方式 |
|------|------|------|------|
| **{module}** | `service/{...}/`、`controllers/.../{...}/` | `docs/service/{...}.md` | 调用链中的替代实现 |

### 0.1 附带关系
- 哪些 Redis Key / 表 / 配置项保留 md 完整但运行时不写
- 哪些定时任务设计保留但代码不实现
- 哪些接口定义在 md 中但路由不注册

### 0.2 检查规则
- `/new-service {module}/*` → 立即拒绝
- `/new-controller .../{module}/*` → 立即拒绝
- `/new-scheduled-task {task}` → 立即拒绝
- `/rebuild-from-docs {module}` → 立即拒绝
- `/rebuild-from-docs all` → 跳过本模块

### 0.3 解除条件
启用时按以下顺序执行：
1. 删除本文档 §0
2. 删除相关 md 顶部的警示框
3. 恢复调用链中的完整实现
4. 执行 /new-service / /new-controller / /new-scheduled-task 生成代码
5. 搜索全 md 中 "🚫" / "暂不实现" 关键字逐个清理

---

## 1. 状态图例

| 图例 | 含义 | AI 处理方式 |
|------|------|-----------|
| 🟢 | 已实现且在运行时装配 | 按既有签名调用，勿重复创建 |
| 🟡 | 代码已落盘但未装配 | 启用前先读取代码，确认签名 |
| ⬜ | 设计已定稿，代码尚未创建 | 使用对应 `/new-*` skill |
| 🚫 | 设计已定稿，但当前阶段**明确不实现**（见 §0） | **拒绝生成** |
| ❌ | 未设计且未实现 | **停止**生成，先补 md |

---

## 2. 整体进度快照

| 层 | 状态 | 已实现 / 规划 | 下一步 |
|----|------|--------------|-------|
| `go.mod` / 模块根 | ⬜ | 0 / 1 | 待初始化 |
| `main.go` + helpers 资源初始化 | ⬜ | 0 / 1 | 待生成 |
| `conf/` 配置 | ⬜ | 0 / 3 | 待生成 |
| `components/error.go` | ⬜ | 0 / 1 | 待生成 |
| `components/constants.go` | ⬜ | 0 / 1 | 待生成 |
| `models/` | ⬜ | 0 / N | 按 /new-model 生成 |
| `helpers/` 基础设施 | ⬜ | 0 / N | 按需生成 |
| `middleware/` | ⬜ | 0 / N | 按 /new-middleware 生成 |
| `service/` | ⬜ | 0 / N | 按 /new-service 生成 |
| `controllers/http/` | ⬜ | 0 / N | 按 /new-controller 生成 |
| `controllers/command/` | ⬜ | 0 / N | 按 /new-scheduled-task 生成 |
| `controllers/mq/` | ⬜ | 0 / N | 按 /new-mq-consumer 生成 |
| `router/http.go` | ⬜ | 0 / 1 | 按 /new-router 生成 |
| `router/command.go` | ⬜ | 0 / 1 | 按 /new-scheduled-task 生成 |
| `router/mq.go` | ⬜ | 0 / 1 | 按 /new-mq-consumer 生成 |

---

## 3. 模型层（models/）明细

| DB | 表 | 代码文件 | 状态 |
|----|----|---------|------|
| {db_1} | {table_1} | `models/{db_1}/{table_1}.go` | ⬜ |
| ... | ... | ... | ⬜ |

---

## 4. Service 层（service/）明细

| 模块 | 文档 | 方法数 | 状态 |
|------|------|--------|------|
| `service/{module_1}/` | `{module_1}_service.md` | N | ⬜ |
| ... | ... | ... | ⬜ |

**生成顺序建议**（依赖最少 → 最多）：
1. {independent_modules}
2. {modules_depending_on_above}
3. ...

---

## 5. Controller 层（controllers/http/）明细

| 路由组 | 文档 | 接口数 | 已实现 | 状态 |
|--------|------|-------|-------|------|
| {audience_1} | `{audience_1}_interfaces.md` | N | 0 | ⬜ |
| ... | ... | ... | ... | ⬜ |

---

## 6. 定时任务与 MQ 消费者

### 6.1 定时任务

| 任务名 | 类型 | 文件 | 状态 |
|-------|------|------|------|
| {task_1} | cobra/cycle/crontab | `{path}` | ⬜ |
| ... | ... | ... | ⬜ |

### 6.2 Kafka 消费者

| 消费者 | topic | 文件 | 状态 |
|--------|-------|------|------|
| {consumer_1} | `{topic}` | `controllers/mq/{topic}_consumer.go` | ⬜ |

---

## 7. helpers 工具函数

| 文件 | 导出函数 | 状态 | 依赖者 |
|------|---------|------|-------|
| `helpers/ip.go` | `IsInternalIP` | ⬜ | middleware |
| `helpers/crypto.go` | `Md5Hex / Generate* / HashPassword / CheckPassword` | ⬜ | middleware / service |
| `helpers/math.go` | `AbsInt64` | ⬜ | middleware |
| ... | ... | ... | ... |

---

## 8. 文档状态（docs/）

| 目录 | 篇数 | 状态 | 说明 |
|------|------|------|------|
| `docs/architecture/` | N | ⬜ / 🟢 | — |
| `docs/api/` | N | ⬜ / 🟢 | — |
| `docs/service/` | N | ⬜ / 🟢 | — |
| `docs/schema/` | 1 | ⬜ / 🟢 | — |
| `docs/cache/` | N | ⬜ / 🟢 | — |
| `docs/circuit_breaker/` | 1 | ⬜ / 🟢 | — |
| `docs/testing/` | 1 | ⬜ / 🟢 | — |
| `docs/INDEX.md` | 1 | 🟢 | — |
| `docs/BUILD_STATUS.md` | 1 | 🟢 | 本文档 |

---

## 9. Skill 状态（.claude/skills/）

| Skill | 用途 | 状态 |
|-------|------|------|
| `new-model` | DDL → GORM struct | 🟢 |
| `new-service` | 伪码 → service 方法 | 🟢 |
| `new-controller` | API md → handler | 🟢 |
| `new-middleware` | middleware.md → 中间件 | 🟢 |
| `new-router` | routing.md → 路由注册 | 🟢 |
| `new-scheduled-task` | scheduled_tasks → cobra/cycle | 🟢 |
| `new-mq-consumer` | MQ consumer 生成 | 🟢 |
| `new-test` | 测试用例 | 🟢 |
| `doc-sync-check` | 多维度一致性 | 🟢 |
| `update-index` | 自动维护 INDEX | 🟢 |
| `rebuild-from-docs` | 任意模块/整工程重建 | 🟢 |
| `sync-feature-to-docs` | 增量需求同步 | 🟢 |

---

## 10. 维护流程

- 每次 `/new-*` skill 生成新代码后，**必须**更新本文档对应行的状态列
- 每次 `/doc-sync-check all` 完成后，**必须**与本文档对比，发现差异立即补齐
- 新增文档或 skill 时，**必须**在 §8 / §9 中登记一行
- 删除代码时，**必须**把对应行从 🟢 改回 ⬜（如果仍计划实现）或移除

**AI 自检**：生成代码前，如果无法快速判定目标模块当前状态，**停下**先读本文档。

---

## 11. 约束清单状态轨道

> 与 §3-§9 代码模块状态轨道并列的第二条轨道。代码模块状态回答"哪些代码已实现"；本节回答"哪些约束类清单已设计 / 已启用"。
>
> ⬜ 在本轨道允许长期保持，与代码模块状态轨道的语义不同。详见 [`aiweave/docs-spec/02 §11`](../../docs-spec/02_build_status_md_spec.md)。

### 11.1 共享状态约束（对应 docs/architecture/concurrency_safety.md）

| 约束类 | 已设计条目数 | 已启用条目数 | 状态 | 说明 |
|--------|-------------|-------------|------|------|
| §2 共享状态注册表 | 0 | 0 | ⬜ | 工程实施时填充 |
| §3 锁粒度决策表 | 0 | 0 | ⬜ | — |
| §4.1 Channel 清单 | 0 | 0 | ⬜ | — |
| §4.2 Goroutine 池配置 | 0 | 0 | ⬜ | — |
| §5.1 连接池配置 | 0 | 0 | ⬜ | — |
| §6 危险操作清单 | — | — | 🟢 | 7 条 rule-id 已在 ai_dev_guide.md §10 登记 |

### 11.2 事务约束（对应 docs/service/transaction_design.md）

| 约束类 | 已设计条目数 | 已启用条目数 | 状态 | 说明 |
|--------|-------------|-------------|------|------|
| §2 事务边界清单 | 0 | 0 | ⬜ | — |
| §3 补偿逻辑表 | 0 | 0 | ⬜ | — |
| §4 幂等接口清单 | 0 | 0 | ⬜ | — |
| §5 一致性窗口承诺 | 0 | 0 | ⬜ | — |

### 11.3 领域不变量（对应 docs/service/{module}_service.md §7）

| 模块 | 业务规则 (I-N) 数 | 隐式约束数 | 状态 | 说明 |
|------|-----------------|----------|------|------|
| `{Module-A}` | 0 | 0 | ⬜ | 工程实施时填充 |
| `{Module-B}` | 0 | 0 | ⬜ | — |

### 11.4 grep 锚 rule-id 索引（对应 docs/architecture/ai_dev_guide.md §10）

| Rule-id | 含义 | 状态 |
|---------|------|------|
| `R-CONC-LOCK-IO` | 锁内做网络 IO | 🟢 |
| `R-CONC-LOCK-LOG` | 锁内打日志 | 🟢 |
| `R-CONC-MAP-RACE` | map 并发写 | 🟢 |
| `R-CONC-DOUBLE-CLOSE` | 关闭已关闭的 channel | 🟢 |
| `R-CONC-SEND-CLOSED` | 向已关闭的 channel 发送 | 🟢 |
| `R-RESOURCE-DEFER-LOOP` | defer 在 for 循环内 | 🟢 |
| `R-CONC-GOROUTINE-LEAK` | 忽略 ctx.Done() | 🟢 |
| `R-TXN-NO-IDEMPOTENT` | 写入类接口无幂等 Key | 🟢 |
| `R-TXN-CROSS-SOURCE` | 跨数据源写入未登记事务边界 | 🟢 |
| `R-PERF-LOOP-DB-QUERY` | for 循环内 DB 查询（N+1） | 🟢 |
| `R-PERF-LOOP-ALLOC` | for 循环内大对象分配 | 🟢 |
| `R-PERF-HOT-REFLECT` | 热路径用反射 | 🟢 |
| `R-PERF-HOT-FMT` | 热路径用 fmt.Sprintf | 🟢 |
| `R-PERF-FULL-COUNT` | 全表 COUNT(*) | 🟢 |
| `R-RESOURCE-SLEEP-SYNC` | time.Sleep 做同步 | 🟢 |
| `R-CACHE-LARGE-KEY` | Redis 大 Key（> 1MB） | 🟢 |
| `R-CACHE-BIGKEY-DEL` | 大 key 用 `DEL` 阻塞单线程 | 🟢 |
| `R-CACHE-HOTKEY-SINGLE-SHARD` | 热 key 流量全砸单分片 | 🟡 |
| `R-CACHE-L1-HIGH-CARD` | 高基数 / 高频变更 key 塞进 L1 | 🟡 |
| `R-CACHE-MOD-N-REHASH` | 分片用裸 `mod N` 重哈希 | 🟢 |
| `R-CONF-DANGER-SINGLE-GATE` | 高风险开关缺 `RUN_ENV` 运行档位双门禁 | 🟢 |

### 11.5 性能合约约束（新增，对应 docs/architecture/performance_contract.md）

| 约束类 | 已设计条目数 | 已启用条目数 | 状态 | 说明 |
|--------|-------------|-------------|------|------|
| §1 全局 SLA | 0 | 0 | ⬜ | 工程实施时填充 |
| §2 热路径清单 | 0 | 0 | ⬜ | — |
| §3 内存分配预算 + 对象池 | 0 | 0 | ⬜ | — |
| §6.1 队列 / channel 容量上限 | 0 | 0 | ⬜ | 与 §11.1 Channel 清单互引用 |
| §6.2 链路超时协调 | 0 | 0 | ⬜ | — |
| §7.1 基准测试清单 | 0 | 0 | ⬜ | — |

### 11.6 可观测性约束（新增，对应 docs/architecture/observability.md）

| 约束类 | 已设计条目数 | 已启用条目数 | 状态 | 说明 |
|--------|-------------|-------------|------|------|
| §2 Metrics 清单（HTTP / 健康 / Runtime） | 0 | 0 | ⬜ | 工程实施时填充 |
| §3.1 Cardinality 上限 | — | — | 🟢 | 默认上限已启用 |
| §3.2 Label 禁止清单 | — | — | 🟢 | 强制清单已启用 |
| §5.2 告警规则 | 0 | 0 | ⬜ | — |

### 11.7 Schema 演进约束（对应 docs/schema/database_design.md §8）

| 约束类 | 已设计条目数 | 已启用条目数 | 状态 | 说明 |
|--------|-------------|-------------|------|------|
| §8.1 废弃字段注册表 | 0 | 0 | ⬜ | — |
| §8.2 字段语义变更记录 | 0 | 0 | ⬜ | — |
| §8.3 Schema 兼容性规则 | — | — | 🟢 | 规则已启用（强制） |

### 11.8 跨服务合约约束（对应 docs/architecture/cross_service_contract.md）

| 约束类 | 已设计条目数 | 已启用条目数 | 状态 | 说明 |
|--------|-------------|-------------|------|------|
| §2 上游合约 | 0 | 0 | ⬜ | 工程实施时填充 |
| §3 下游合约 | 0 | 0 | ⬜ | — |
| §4 故障传播矩阵 | 0 | 0 | ⬜ | — |
| §5 接口版本管理 | — | — | 🟢 | 规则已启用 |

---

## 12. 运行时基线区域（如启用 runtime_profile.md）

> 与 §3-§9 代码模块状态轨道、§11 约束清单状态轨道并列的第三条轨道。详见 [`aiweave/docs-spec/02 §12`](../../docs-spec/02_build_status_md_spec.md)。

| 项 | 上次校对日期 | 数据来源 | 与上次基线偏离 | 备注 |
|----|-------------|---------|--------------|------|
| §1 流量模式 | — | 待录入 | — | runtime_profile.md 未启用 |
| §2 数据规模 | — | 待录入 | — | — |
| §3 资源消耗基线 | — | 待录入 | — | — |
| §4 关键超时链 | — | 待录入 | — | 与 performance_contract.md §6.2 双向对齐 |

**维护节奏**：

- 每季度技术评审后必须更新
- 偏离 > 30% → 触发 runtime_profile.md 重写
- 数据来源变更（人工估算 → APM 实测）→ 更新"数据来源"列

### 11.5 维护

- 新增 §2 / §3.1 / §4.1 等表内一行 → 对应类目"已设计条目数" +1
- 校验代码 / 审计 Skill 联动后 → "已启用条目数" +1，状态可转 🟢
- 启用某类约束的总开关→ 该类目状态从 ⬜ 转 🟢


---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
