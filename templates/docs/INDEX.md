# {ProjectName} 文档索引

> 项目启动时复制本骨架到 `docs/INDEX.md`，按 `aiweave/docs-spec/01_index_md_spec.md` 规范填充。

---

## 顶层入口

| 文档 | 说明 |
|------|------|
| [BUILD_STATUS.md](BUILD_STATUS.md) | **实现进度一览**（🟢/🟡/⬜/🚫 每层状态、AI 生成代码前必查） |
| [testing/testing_design.md](testing/testing_design.md) | **测试框架设计**：in-process 启动、业务校验注入、seed 数据、断言 API、测试纪律 |

## 架构设计

| 文档 | 说明 |
|------|------|
| [architecture/overview.md](architecture/overview.md) | 系统总纲 |
| [architecture/config.md](architecture/config.md) | 配置体系 |
| [architecture/infrastructure.md](architecture/infrastructure.md) | 基础设施初始化 |
| [architecture/routing.md](architecture/routing.md) | 路由与中间件链 |
| [architecture/middleware.md](architecture/middleware.md) | 中间件设计 |
| [architecture/status_codes.md](architecture/status_codes.md) | 错误码体系 |
| [architecture/{核心校验链}_flow.md](architecture/auth_flow.md) | 业务校验流程（如有，template 中以 auth_flow.md 为骨架示例） |
| [architecture/audit_log.md](architecture/audit_log.md) | 审计日志（如有） |
| [architecture/render_functions.md](architecture/render_functions.md) | 响应渲染函数（如有多种格式） |
| [architecture/helpers_api.md](architecture/helpers_api.md) | **辅助函数与全局变量 API 参考**（业务代码必读） |
| [architecture/pkg_api.md](architecture/pkg_api.md) | **pkg/ 内部框架 API 契约**（AI 生成业务代码前必读） |
| [architecture/go_module.md](architecture/go_module.md) | Go 模块与依赖规范 |
| [architecture/constants.md](architecture/constants.md) | 业务常量唯一真相源 |
| [architecture/logging.md](architecture/logging.md) | 日志规范 |
| [architecture/ai_dev_guide.md](architecture/ai_dev_guide.md) | AI 原生研发手册 |
| [architecture/mvp_rebuild_path.md](architecture/mvp_rebuild_path.md) | 分阶段构建路径 |
| [architecture/concurrency_safety.md](architecture/concurrency_safety.md) | 并发安全：共享状态注册表 / 锁策略 / Channel / 危险操作清单（约束类，推荐启用） |
| [architecture/performance_contract.md](architecture/performance_contract.md) | 性能合约：SLA / 热路径 / 内存预算 / 背压（约束类，推荐启用） |
| [architecture/observability.md](architecture/observability.md) | 可观测性：服务级 Metrics / Cardinality 控制 / 告警规则（约束类，推荐启用） |
| [architecture/io_contract.md](architecture/io_contract.md) | IO 合约：三条 IO 铁律 / 聚合器 / 并行编排原语 / 往返预算（约束类，强烈推荐） |
| [architecture/cross_service_contract.md](architecture/cross_service_contract.md) | 跨服务合约：上下游依赖图 / 故障传播矩阵 / 接口版本（约束类，多服务时启用） |
| [architecture/runtime_profile.md](architecture/runtime_profile.md) | 运行时基线：人工运行时数据快照（如有） |
| [architecture/deployment.md](architecture/deployment.md) | 部署与运行时生命周期：部署产物 / 镜像约束 / 探针分层 / 优雅启停 / 发布回滚（约束类，强烈推荐） |

## 接口设计

| 文档 | 说明 |
|------|------|
| [api/{audience1}_interfaces.md](api/{audience1}_interfaces.md) | 调用方 1 接口规格（N 个接口） |
| [api/{audience2}_interfaces.md](api/{audience2}_interfaces.md) | 调用方 2 接口规格（N 个接口） |
| [api/{audience3}_interfaces.md](api/{audience3}_interfaces.md) | 调用方 3 接口规格（N 个接口） |

## 数据模型

| 文档 | 说明 |
|------|------|
| [schema/database_design.md](schema/database_design.md) | 数据库设计：N 张表 DDL、分库设计、Redis Key 注册表、GORM 规范 |

## 缓存设计

| 文档 | 说明 |
|------|------|
| [cache/cache_design.md](cache/cache_design.md) | 缓存设计：Redis-first / Key 模式 / Hash 字段映射 / Flush / 限流 |
| [cache/cache_helpers.md](cache/cache_helpers.md) | 缓存辅助函数（如有 L1 本地缓存层） |

## Service 层设计

| 文档 | 说明 |
|------|------|
| [service/service_design.md](service/service_design.md) | Service 层总览：错误处理 / 模块依赖 / 调用矩阵 |
| [service/{module}_service.md](service/{module}_service.md) | 单模块详细设计 |
| [service/scheduled_tasks_design.md](service/scheduled_tasks_design.md) | 定时任务 + MQ 消费者设计 |
| [service/transaction_design.md](service/transaction_design.md) | 分布式事务：事务边界清单 / 补偿 / 幂等 / 失败路径全景图（约束类，跨数据源时启用） |

## 熔断器设计

| 文档 | 说明 |
|------|------|
| [circuit_breaker/circuit_breaker_design.md](circuit_breaker/circuit_breaker_design.md) | 熔断器配置：算法 / 差异化参数 / 接入方式 |

---

## 变更日志

记录文档结构演进。

| 日期 | 变更 | 触发事件 |
|------|------|---------|
| {YYYY-MM-DD} | 初版 | 项目启动 |


---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
