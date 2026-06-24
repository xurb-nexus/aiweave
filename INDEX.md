# AIWeave 索引

## 0. AIWeave 采纳进度

> 落地工程把本表复制到 `docs/INDEX.md §0`，按工程实际启用情况标记每个规范的状态。
>
> AIWeave 骨架本身的所有 28 篇 docs-spec + 13 篇 design-spec + 19 个 Skill 都已就位；具体哪些规范在某个工程内强制生效，由该工程自行决定。本表是工程级开关。

### 0.1 规范启用状态

| 规范 | 启用状态 | 启用日期 | 备注 |
| --- | --- | --- | --- |
| `docs-spec/00-19` 基础规范 | 🟢 启用 | {YYYY-MM-DD} | — |
| `docs-spec/20_concurrency_safety` 并发安全 | 🟢 / 🟡 / ⬜ / 🚫 | — | 推荐启用 |
| `docs-spec/21_distributed_transaction` 分布式事务 | 🟢 / 🟡 / ⬜ / 🚫 | — | 仅有跨数据源时启用 |
| `docs-spec/22_performance_contract` 性能合约 | 🟢 / 🟡 / ⬜ / 🚫 | — | 推荐启用 |
| `docs-spec/23_observability` 可观测性（Metrics） | 🟢 / 🟡 / ⬜ / 🚫 | — | 推荐启用 |
| `docs-spec/24_cross_service_contract` 跨服务合约 | 🟢 / 🟡 / ⬜ / 🚫 | — | 单体服务可标 🚫 |
| `docs-spec/25_io_aggregation` IO 极致与聚合并行 | 🟢 / 🟡 / ⬜ / 🚫 | — | 强烈推荐（三条 IO 铁律） |
| `docs-spec/26_config_center` 配置中心与凭据加密 | 🟢 / 🟡 / ⬜ / 🚫 | — | 配置上云时启用 |
| `docs-spec/27_deployment_runtime` 部署与运行时生命周期 | 🟢 / 🟡 / ⬜ / 🚫 | — | 强烈推荐（容器化部署 / 优雅启停 / 探针） |
| `design-spec/*` 高性能架构方法论（十二视角） | 🟢 / 🟡 / ⬜ | — | 推荐启用（技术方案生成阶段；IO 视角强烈推荐） |
| Hooks 机制（L0 自动化防御，skills-spec/02 §4） | 🟢 / 🟡 / ⬜ | — | 强烈推荐（团队共享 settings.json 强制项） |
| `templates/docs/architecture/runtime_profile.md` 运行时基线 | 🟢 / ⬜ | — | 涉及人工运行时数据 |
| `OPERATIONS.md` §11 演进效果度量季度复盘 | 🟢 / ⬜ | — | 需工程实际运行 2 季度建立基线后启用 |

### 0.2 采纳建议（三档）

| 工程复杂度 | 推荐启用集 |
| --- | --- |
| **简单 CRUD / 内部工具服务** | 基础规范 + 20 §2 共享状态注册表 + 09 §7 领域不变量 + CLAUDE.md 危险模式清单 |
| **生产服务常态（推荐起点）** | 基础规范 + 20 + 21 + 22 §2 热路径 + 23 + 25 IO 铁律 + 27 部署与运行时生命周期 + Hooks（L0）+ 18 §11 安全重构 + 审计 Skill `concurrency-review` / `io-review` / `domain-invariant-check` |
| **高并发 / 分布式核心** | 全部启用（含 24 + 25 + 26 + 27 + runtime_profile + Hooks + 全部审计 Skill + §11 演进效果度量） |

### 0.3 状态语义

- 🟢 启用：规范本体已就位，CLAUDE.md / OPERATIONS / 范围判定表已联动，AI 写代码时强制遵守
- 🟡 部分：规范本体就位但部分子节未联动（如 grep 锚已定义但审计 Skill 未实现）
- ⬜ 未启用：规范本体未引入，AI 不应触发对应约束
- 🚫 不启用：本工程明确不需要

---

## 顶层入口

| 文档 | 说明 |
|------|------|
| [README.md](README.md) | 规范的定位、适用范围、目录全景、启动新项目的 4 阶段路径 |
| [PRINCIPLES.md](PRINCIPLES.md) | 核心原则：双向同步、文档优先、签名冲突裁决、BUILD_STATUS 先查、暂不实现模块、测试纪律 |
| [OPERATIONS.md](OPERATIONS.md) | 操作手册：8 类工作流（设计先行 / 新增 API / 新增表 / 新增任务 / 新增 MQ / 重构 / 增量同步 / 定期审计）+ 3 类自检清单（交付前 / 代码变更 / 文档变更）+ 增量同步专属清单 |

## docs-spec/ —— `docs/` 目录的规范

编号 00-27，覆盖一个 Go 后端工程所需的全部设计视角，并支持"建设模式 + 增量同步模式"两种使用方式。

| 序号 | 规范 | 对应 docs/ 路径 | 说明 |
|------|------|---------------|------|
| 00 | [docs-spec/00_directory_layout.md](docs-spec/00_directory_layout.md) | docs/* | 标准目录结构（哪些子目录、命名规则、新增目录的判定） |
| 01 | [docs-spec/01_index_md_spec.md](docs-spec/01_index_md_spec.md) | docs/INDEX.md | 索引文件写法（每篇 md 的登记格式、描述精度、维护规则） |
| 02 | [docs-spec/02_build_status_md_spec.md](docs-spec/02_build_status_md_spec.md) | docs/BUILD_STATUS.md | 实现进度文件 + 暂不实现模块（§0）模板 + 状态图例 + 维护流程 |
| 03 | [docs-spec/03_claude_md_spec.md](docs-spec/03_claude_md_spec.md) | CLAUDE.md（项目根） | AI 总入口：项目概述 + 分层架构 + 开发规范 + 文档同步纪律 + 范围判定表 |
| 04 | [docs-spec/04_architecture_overview_spec.md](docs-spec/04_architecture_overview_spec.md) | docs/architecture/overview.md 等 | 架构总纲及衍生文档（config / infrastructure / routing / middleware / auth_flow / audit_log / render_functions / go_module / ai_dev_guide） |
| 05 | [docs-spec/05_schema_design_spec.md](docs-spec/05_schema_design_spec.md) | docs/schema/database_design.md | 数据库 DDL + 分库设计 + Redis Key 注册表 + GORM Model 规范 |
| 06 | [docs-spec/06_cache_design_spec.md](docs-spec/06_cache_design_spec.md) | docs/cache/cache_design.md + cache_helpers.md | KV 缓存设计：Redis-first / 三级缓存 / Key 模式 / Hash 字段映射 / Flush 机制 / 限流 |
| 07 | [docs-spec/07_api_interfaces_spec.md](docs-spec/07_api_interfaces_spec.md) | docs/api/*.md | API 接口详细设计与实现：请求/响应 JSON / Go struct / 校验规则 / Controller 伪代码 |
| 08 | [docs-spec/08_api_flow_spec.md](docs-spec/08_api_flow_spec.md) | docs/architecture/{auth,billing,...}_flow.md | API 逻辑描述与流程串联：跨接口/跨模块的业务链路 |
| 09 | [docs-spec/09_service_design_spec.md](docs-spec/09_service_design_spec.md) | docs/service/*.md | Service 层方法签名 + 步骤级伪代码 + 数据访问 + 错误类型选择 |
| 10 | [docs-spec/10_scheduled_tasks_spec.md](docs-spec/10_scheduled_tasks_spec.md) | docs/service/scheduled_tasks_design.md | 定时任务（cobra 外部调度 / cycle 内部协程 / crontab 定点）+ 并行策略 + 异常容错 |
| 11 | [docs-spec/11_mq_consumer_spec.md](docs-spec/11_mq_consumer_spec.md) | docs/service/scheduled_tasks_design.md §MQ | MQ 消费者设计：攒批 / 落库 / 死信队列 / offset 管理 |
| 12 | [docs-spec/12_circuit_breaker_spec.md](docs-spec/12_circuit_breaker_spec.md) | docs/circuit_breaker/circuit_breaker_design.md | 熔断 / 限流 / 降级：算法选型 / 差异化参数 / GORM Plugin / 接入方式 |
| 13 | [docs-spec/13_logging_spec.md](docs-spec/13_logging_spec.md) | docs/architecture/logging.md | 结构化日志：API 风格 / 级别判定 / 字段约定 / 敏感数据 |
| 14 | [docs-spec/14_status_codes_spec.md](docs-spec/14_status_codes_spec.md) | docs/architecture/status_codes.md | 错误码 / 状态码体系：双错误类型 / 范围划分 / 业务状态枚举 |
| 15 | [docs-spec/15_helpers_pkg_api_spec.md](docs-spec/15_helpers_pkg_api_spec.md) | docs/architecture/helpers_api.md + pkg_api.md | 工具函数 + 框架 API 契约（一切签名的真相源） |
| 16 | [docs-spec/16_constants_spec.md](docs-spec/16_constants_spec.md) | docs/architecture/constants.md | 业务常量 / 魔法数字唯一真相源 |
| 17 | [docs-spec/17_testing_design_spec.md](docs-spec/17_testing_design_spec.md) | docs/testing/testing_design.md | 测试框架 + 测试纪律 + 4 类用例 + 与 Stage 协同 |
| 18 | [docs-spec/18_mvp_rebuild_path_spec.md](docs-spec/18_mvp_rebuild_path_spec.md) | docs/architecture/mvp_rebuild_path.md | 分阶段构建路径（通用方法论）：覆盖 forward / rebuild / refactor 三种场景 |
| 19 | [docs-spec/19_incremental_sync_spec.md](docs-spec/19_incremental_sync_spec.md) | （流程文档，不直接对应一篇工程 md，但塑造工程演进的 SOP） | **增量需求同步规范**：开发完需求后如何把代码改动反向同步到全链路 docs/，B1 场景的核心规则。 |
| 20 | [docs-spec/20_concurrency_safety_spec.md](docs-spec/20_concurrency_safety_spec.md) | docs/architecture/concurrency_safety.md | **并发安全与运行时约束规范**：共享状态注册表 / 锁策略 / Channel / 资源生命周期 / 危险操作清单 / 并发测试策略 + B1 反向同步规则 |
| 21 | [docs-spec/21_distributed_transaction_spec.md](docs-spec/21_distributed_transaction_spec.md) | docs/service/transaction_design.md | **分布式事务与补偿设计规范**：事务模型选型 / 事务边界清单 / 补偿逻辑 / 幂等性 / 一致性窗口 / 失败路径全景图 / 伪码标记语法 + B1 反向同步规则 |
| 22 | [docs-spec/22_performance_contract_spec.md](docs-spec/22_performance_contract_spec.md) | docs/architecture/performance_contract.md | **性能合约与热路径规范**：全局 SLA / 热路径标注 / 内存预算 / 数据访问约束 / 背压策略 / 性能回归测试 / 与 12 边界划分 + B1 反向同步规则 |
| 23 | [docs-spec/23_observability_spec.md](docs-spec/23_observability_spec.md) | docs/architecture/observability.md | **可观测性规范（Metrics 限服务级）**：仅服务级基础指标 / Cardinality 控制 / Label 禁止清单 / 告警规则 / 与 13 边界划分 + B1 反向同步规则 |
| 24 | [docs-spec/24_cross_service_contract_spec.md](docs-spec/24_cross_service_contract_spec.md) | docs/architecture/cross_service_contract.md | **跨服务合约规范**：上下游依赖图 / 上游合约 / 下游合约 / 故障传播矩阵 / 接口版本管理 + B1 反向同步规则 |
| 25 | [docs-spec/25_io_aggregation_spec.md](docs-spec/25_io_aggregation_spec.md) | docs/architecture/io_contract.md | **IO 极致与聚合并行规范**：三条 IO 铁律（批量优先 / 禁止 N+1 / 禁止独立串行编排）/ 聚合器模式（收集→批量读→并行回源→单飞→异步写回）/ 并行编排原语 / 数据访问聚合约束 / IO 往返预算 / IO 回归测试 + B1 反向同步规则 |
| 26 | [docs-spec/26_config_center_spec.md](docs-spec/26_config_center_spec.md) | docs/architecture/config.md §6-§9 | **配置中心与凭据加密规范**：配置上云权威源模型 / 配置中心 Client 抽象与拉取落盘编排 / 凭据对称加密（密钥应用层持有不入库）/ 启动期 fail-fast 加载时序（治理 config.md §6-§9，与 04 §2 边界划分） |
| 27 | [docs-spec/27_deployment_runtime_spec.md](docs-spec/27_deployment_runtime_spec.md) | docs/architecture/deployment.md | **部署与运行时生命周期规范**：部署产物蓝图（镜像/编排/流水线）/ 容器镜像约束 / 健康探针分层语义（liveness/readiness/startup）/ 启动就绪时序（衔接 26）/ 优雅关闭时序（drain + 提交位点，对账 20 §1.1）/ 发布回滚策略（与 18 §11.5.4 边界）+ B1 反向同步规则。补齐"重建产物能上线"的第一性闭环 |

## design-spec/ —— 高性能架构方法论（技术方案生成参考）

编号 00-12，AIWeave 的**第三支柱**。与 docs-spec（怎么写下来）、skills-spec（怎么执行）正交：design-spec 回答"面对需求怎么**做出**架构决策"。每个视角是一个决策透镜（识别形态 → 决策树 → 默认选型 → 权衡 → 反模式 → 落到 docs/）。决策真相源在此，硬规则与表示真相源仍在对应 docs-spec（引用不复制，见各篇 §8 边界）。**01-07 是旗舰范本写透的同步请求路径核心，08-12 把方法论扩到异步面（08）/ 契约面（09）/ 多副本协调（10）/ 在线演进（11）/ 容量伸缩（12）四条边。**

| 序号 | 规范 | 决策落到 docs/ | 表示真相源 | 说明 |
|------|------|---------------|-----------|------|
| 00 | [design-spec/00_design_overview.md](design-spec/00_design_overview.md) | — | — | 设计方法论总纲：第三支柱定位 / 十二视角清单（四组：同步路径 01-07 / 异步 08 / 契约 09 / 多副本+时间轴 10-12）/ 统一 lens 九节骨架 / 双源真相边界 / 技术方案(TRD)产物结构 / 占位符纪律继承 |
| 01 | [design-spec/01_io_design.md](design-spec/01_io_design.md) ⭐ | architecture/io_contract.md | [docs-spec/25](docs-spec/25_io_aggregation_spec.md) | **旗舰范本（写透）**：IO 三维度（次数/串并行/往返）+ 读写路径分治 + 读/写两棵决策树 + 三句话默认（批量/并行/单次往返优先）+ 借纪律不借实现（按数据形态重选原语）+ 编排权衡矩阵 + 截止预算逐跳传播 + 扇出失败语义（all-or-nothing vs best-effort）+ 跨请求微批 + 反模式速查（R-IO-*）+ 机械闸门。硬规则（三条铁律 / 聚合器）引用 25 不复制 |
| 02 | [design-spec/02_data_model_design.md](design-spec/02_data_model_design.md) | schema/database_design.md | [docs-spec/05](docs-spec/05_schema_design_spec.md) | 数据建模决策（写透）：数据访问层范式（私有纯查询+公有缓存编排+成对批量）/ 分库分表决策树（业务域+量级+读写模式 / 逻辑物理表名解耦）/ shard 分组并行 / 批量三段式 / 组装层 map 入参纪律 / 分页（keyset 游标 / 大结果集流式） |
| 03 | [design-spec/03_concurrency_design.md](design-spec/03_concurrency_design.md) | architecture/concurrency_safety.md | [docs-spec/20](docs-spec/20_concurrency_safety_spec.md) | 并发模型决策（写透）：并发原语选型 / **协程池容量定量（Little 定律，远大于核数）** / 阻塞不变量 / 内联兜底提交器 |
| 04 | [design-spec/04_transaction_design.md](design-spec/04_transaction_design.md) | service/transaction_design.md | [docs-spec/21](docs-spec/21_distributed_transaction_spec.md) | 事务一致性决策（写透）：一致性强度判定 / 本地 vs 分布式 / **弱一致+边界兜底+监控取舍** / 读写分离消除幂等复杂度 |
| 05 | [design-spec/05_caching_design.md](design-spec/05_caching_design.md) | cache/cache_design.md | [docs-spec/06](docs-spec/06_cache_design_spec.md) | 缓存策略决策（写透）：统一访问层 / **读写非对称降级（加速层 vs 数据源）** / 多级缓存 / 值编码 / Hash 分桶 / 本地 SQLite 镜像 / 确定性加密 ID / 穿透防护（空值+布隆）/ 序列化格式选型 / 世代式失效（杜绝 KEYS）/ 键生命周期+冷启预热 |
| 06 | [design-spec/06_resilience_design.md](design-spec/06_resilience_design.md) | circuit_breaker/ + architecture/performance_contract.md | [docs-spec/12](docs-spec/12_circuit_breaker_spec.md) / [22](docs-spec/22_performance_contract_spec.md) | 稳定性决策（写透）：**自适应熔断（SRE 概率丢弃 > 二态）** / 零侵入接入 / 连接池基线 / 分层与数据层禁日志 / 危险开关双门禁 |
| 07 | [design-spec/07_tail_latency_design.md](design-spec/07_tail_latency_design.md) | architecture/performance_contract.md + cross_service_contract.md | [docs-spec/22](docs-spec/22_performance_contract_spec.md) / [24](docs-spec/24_cross_service_contract_spec.md) / [12](docs-spec/12_circuit_breaker_spec.md) | 尾延迟与自适应过载决策（写透）：尾延迟三源识别 / **对冲·备份请求** / 静态池→**自适应并发限制** / **优先级·截止感知 load shedding** / 重试预算 / 冷启预热。压 P99·P999 长尾与过载有序降级（与 06 扛故障互补，见 07 §8 边界） |
| 08 | [design-spec/08_messaging_design.md](design-spec/08_messaging_design.md) 🆕 | service/scheduled_tasks_design.md §MQ | [docs-spec/11](docs-spec/11_mq_consumer_spec.md) | **异步消息 / 事件驱动（写透）**：同步/异步解耦判定 / **投递语义决策树（at-most/at-least/exactly-once）** / 顺序与分区键 / **双写一致（Outbox / CDC）** / 消费幂等 / 消费者扩展与背压 / 扇出（工作队列 vs 发布订阅）/ 消息 schema 演进。读写分治的第三面（把工作解耦出请求生命周期） |
| 09 | [design-spec/09_api_contract_design.md](design-spec/09_api_contract_design.md) 🆕 | api/*.md + architecture/*_flow.md | [docs-spec/07](docs-spec/07_api_interfaces_spec.md) / [08](docs-spec/08_api_flow_spec.md) | **API 契约 / 接口粒度（写透）**：**接口粒度（chatty vs chunky，直接决定调用方 N+1）** / 批量·聚合端点 / 分页契约（offset vs cursor）/ **版本兼容 expand-contract** / 字段投影 / 幂等键 / 协议选型。比 IO 还上游的 N+1 闸门 |
| 10 | [design-spec/10_scheduling_coordination_design.md](design-spec/10_scheduling_coordination_design.md) 🆕 | service/scheduled_tasks_design.md | [docs-spec/10](docs-spec/10_scheduled_tasks_spec.md) | **定时任务 / 分布式协调（写透）**：**单例 vs 每副本判定** / 协调原语（原子认领 / 分布式锁 / leader 选举 / 外部调度）/ 幂等可重入调度 / **fencing 防脑裂**。补"单后台协程多副本陷阱"的正确性缺口 |
| 11 | [design-spec/11_evolution_migration_design.md](design-spec/11_evolution_migration_design.md) 🆕 | 分布落地（schema/database_design.md §8 + mvp_rebuild_path.md §11） | [docs-spec/05](docs-spec/05_schema_design_spec.md) / [19](docs-spec/19_incremental_sync_spec.md) | **演进与在线迁移（写透）**：兼容 vs 破坏判定 / **扩展-收缩（expand-contract）** / 在线 DDL / **限流回填（backfill）** / 双写双读 + 影子比对灰度 / 回滚闸门。零停机时间轴安全变更（无独立 docs-spec，同 07 分布落地） |
| 12 | [design-spec/12_capacity_scaling_design.md](design-spec/12_capacity_scaling_design.md) 🆕 | 分布落地（performance_contract.md + deployment.md） | [docs-spec/22](docs-spec/22_performance_contract_spec.md) / [27](docs-spec/27_deployment_runtime_spec.md) | **容量规划与弹性伸缩（写透）**：**从 SLA 反推容量** / 余量（排队律 ρ≈0.8 膝盖）/ 资源配比一致性（副本×连接≤下游）/ 伸缩信号选型与防抖 / 有状态 vs 无状态伸缩。把 07 排队律正向用作容量设计输入（无独立 docs-spec，同 07 分布落地） |

## skills-spec/ —— `.claude/skills/` 的规范

| 文档 | 说明 |
|------|------|
| [skills-spec/00_skill_overview.md](skills-spec/00_skill_overview.md) | Skill 体系总览：5 类分组（创建 9 / 维护 2 / 审计 6 / 终极 1 / 设计 1）、命名约定、调用顺序、与 Stage 的协同 |
| [skills-spec/01_skill_authoring_guide.md](skills-spec/01_skill_authoring_guide.md) | Skill 编写指南：frontmatter / 步骤化结构 / 文档同步 / 测试同步 / 验证 |
| [skills-spec/02_settings_local_json_spec.md](skills-spec/02_settings_local_json_spec.md) | settings.local.json 规范：permissions / hooks / 推荐配置 |

`templates/skills/{name}/SKILL.md` 是 **19 个** Skill 的可执行骨架（每个 SKILL.md 同时充当规范与可复制的执行手册；公共章节真相源在 [skills-spec/01_skill_authoring_guide.md](skills-spec/01_skill_authoring_guide.md) §A-§E）：

| Skill | 骨架文件 |
|-------|----------|
| new-model | [templates/skills/new-model/SKILL.md](templates/skills/new-model/SKILL.md) |
| new-service | [templates/skills/new-service/SKILL.md](templates/skills/new-service/SKILL.md) |
| new-controller | [templates/skills/new-controller/SKILL.md](templates/skills/new-controller/SKILL.md) |
| new-middleware | [templates/skills/new-middleware/SKILL.md](templates/skills/new-middleware/SKILL.md) |
| new-router | [templates/skills/new-router/SKILL.md](templates/skills/new-router/SKILL.md) |
| new-scheduled-task | [templates/skills/new-scheduled-task/SKILL.md](templates/skills/new-scheduled-task/SKILL.md) |
| new-mq-consumer | [templates/skills/new-mq-consumer/SKILL.md](templates/skills/new-mq-consumer/SKILL.md) |
| new-test | [templates/skills/new-test/SKILL.md](templates/skills/new-test/SKILL.md) |
| doc-sync-check | [templates/skills/doc-sync-check/SKILL.md](templates/skills/doc-sync-check/SKILL.md) |
| update-index | [templates/skills/update-index/SKILL.md](templates/skills/update-index/SKILL.md) |
| rebuild-from-docs | [templates/skills/rebuild-from-docs/SKILL.md](templates/skills/rebuild-from-docs/SKILL.md) |
| sync-feature-to-docs | [templates/skills/sync-feature-to-docs/SKILL.md](templates/skills/sync-feature-to-docs/SKILL.md) — **B1 增量同步核心 Skill** |
| concurrency-review | [templates/skills/concurrency-review/SKILL.md](templates/skills/concurrency-review/SKILL.md) — 审计代码 ↔ 并发安全约束一致性 |
| performance-review | [templates/skills/performance-review/SKILL.md](templates/skills/performance-review/SKILL.md) — 审计代码 ↔ 性能合约约束一致性 |
| io-review | [templates/skills/io-review/SKILL.md](templates/skills/io-review/SKILL.md) — 审计代码 ↔ IO 铁律（N+1 / 串行编排 / 聚合并行）一致性 |
| new-saga-step | [templates/skills/new-saga-step/SKILL.md](templates/skills/new-saga-step/SKILL.md) — 生成 Saga 步骤代码 + 补偿 + 幂等 Key |
| domain-invariant-check | [templates/skills/domain-invariant-check/SKILL.md](templates/skills/domain-invariant-check/SKILL.md) — 审计代码 ↔ 领域不变量约束一致性 |
| failure-path-review | [templates/skills/failure-path-review/SKILL.md](templates/skills/failure-path-review/SKILL.md) — 审计失败路径文档 / 测试覆盖完整性 |
| design-solution | [templates/skills/design-solution/SKILL.md](templates/skills/design-solution/SKILL.md) — **设计类（A0 上游）**：输入需求 → 过 design-spec 十二视角 → 产出技术方案(TRD)，衔接 A1 forward / B1 sync |

## templates/ —— 可直接复制的骨架

| 路径 | 说明 |
|------|------|
| [templates/CLAUDE.md](templates/CLAUDE.md) | 项目根 CLAUDE.md 骨架，复制到新项目根目录后填空即可 |
| [templates/docs/](templates/docs/) | 完整 docs/ 骨架（每篇 md 含目录结构 + 待填字段） |
| [templates/skills/](templates/skills/) | 19 个 .claude/skills/{name}/SKILL.md 骨架（5 类分组：创建 9 / 维护 2 / 审计 6 / 终极 1 / 设计 1） |

---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
