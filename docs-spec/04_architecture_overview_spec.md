# 04 - 架构总纲文档规范

> 规定 `docs/architecture/` 下系列 md 的写法。架构层是除接口/Schema 外另一个 AI 必读的视角。

本规范覆盖以下文档：

- `overview.md` —— 架构总纲（强制）
- `config.md` —— 配置体系
- `infrastructure.md` —— 基础设施初始化
- `routing.md` —— 路由与中间件链
- `middleware.md` —— 中间件设计
- `{核心校验链}_flow.md` —— 业务校验流程（如有）
- `audit_log.md` —— 审计日志（如有）
- `render_functions.md` —— 响应渲染函数（如有多种响应格式）
- `go_module.md` —— Go 模块与依赖
- `ai_dev_guide.md` —— AI 研发手册（推荐）

`status_codes.md`、`logging.md`、`constants.md`、`helpers_api.md`、`pkg_api.md`、`mvp_rebuild_path.md` 各有专门规范，分别见 §14、§13、§16、§15、§15、§18。

---

## 1. overview.md（架构总纲）

### 1.1 章节结构（强制）

overview.md 必须按以下章节顺序组织：

- `## 1.` 系统定位（做什么/不做什么 / 上下游关系图 / 核心能力清单）
- `## 2.` 整体架构（整体架构图 ASCII / 关键设计 5-10 项）
- `## 3.` 核心业务链路（如有，如 N 步业务校验链）
- `## 4.` 性能 / 稳定性 / 成本设计亮点
- `## 5.` AI 原生工程（如适用，引用 ai_dev_guide.md）

> **完整章节骨架见** [`templates/docs/architecture/overview.md`](../templates/docs/architecture/overview.md)。

### 1.2 必填要素

- **系统定位** —— 一句话总结，让人不用翻代码就能知道"这个服务在系统里干什么"
- **上下游关系** —— 谁调它、它调谁
- **整体架构图** —— 一张 ASCII 图，从外部调用方到 Gin → 中间件 → 路由组 → service → 数据访问 → 熔断器
- **关键设计** —— 至少列出 5 条本工程的非通用设计决策（如 "Redis-first 写入"、"{核心校验流程}与{数值变更流程}分离"、"Dirty Set + SPOP 增量 Flush"）
- **性能目标** —— 如有 SLA 必须显式写出（QPS、P99 延迟、可用性）

### 1.3 ASCII 图绘制规范

- 使用 ` ┌─┐ │ │ └─┘` 等盒形字符
- 调用方向用 `→` 或 `▼`
- 反向回调用 `◄────►`
- 组件内列出 2-5 个关键名词，不展开实现细节
- 单图不超过 25 行

### 1.4 反例

```markdown
本服务负责 API 的业务校验和状态变更。   ← ❌ 太空，没有上下游、没有量化、没有关键设计
```

正确：

```markdown
{ProjectName}（目标服务）是一个独立微服务，处于上游网关的关键路径上——每一次 API 请求都必须经过业务校验判定。系统以"极致性能、极高稳定性、极低成本"为核心设计目标，提供以下核心能力：身份认证、权限管控、用量统计、限流防护、{受限模块}拦截、合规校验。

性能目标：P99 < {N}ms，QPS > {N}，可用性 {X}%。
```

---

## 2. config.md（配置体系）

### 2.1 章节结构（强制）

config.md 必须按以下章节顺序组织（§1-§5 由本规范治理；§6-§9 配置上云 / 凭据加密由 [`docs-spec/26`](26_config_center_spec.md) 治理）：

- `## 1.` 配置文件分类（mount/ 环境相关 + app/ 业务相关）
- `## 2.` 配置文件清单（每个 yaml/json：文件名 / 内容范围 / 何时修改）
- `## 3.` 加载流程（PreInit → InitConf → 资源初始化 的代码路径与时序）
- `## 4.` 占位机制（如 `@@key` 格式，由配置中心运行时替换）
- `## 5.` Go 结构体映射（每个 yaml 节点 → Go struct 字段）
- `## 6.~9.` 配置上云 / 配置中心集成 / 凭据加密 / 启动加载时序 —— **详见 [`docs-spec/26_config_center_spec.md`](26_config_center_spec.md)**（启用配置上云时强制）

> **完整章节骨架见** [`templates/docs/architecture/config.md`](../templates/docs/architecture/config.md)。

### 2.2 必填要素

- 列出**所有**配置文件，无遗漏
- 列出每个 yaml 节点对应的 Go struct（如 `TApi`、`TResource`）
- 敏感信息占位机制
- **配置上云 / 配置中心集成 / 凭据加密**（如启用）—— 本规范只承载 §1-§5；这部分的真相源是 [`docs-spec/26`](26_config_center_spec.md)，§2 此处仅引用不展开（双源真相规避）

---

## 3. infrastructure.md（基础设施初始化）

### 3.1 章节结构（强制）

infrastructure.md 必须按以下章节顺序组织：

- `## 1.` 资源清单（一张表，列出所有运行时资源：MySQL/Redis/Kafka/本地缓存数）
- `## 2.` 初始化时序（PreInit → InitResource 完整调用链；时序依赖）
- `## 3.` 各资源初始化函数（Init* 函数位置与职责）
- `## 4.` 全局客户端（导出的全局变量 → 业务代码如何引用）
- `## 5.` 与外部服务对比（如适用，本工程的极简主义裁剪）
- `## 6.` 连接池配置基线登记（每资源一行登记 `{maxIdle}` / `{maxOpen}` / `{conn-timeout-ms}` / `{read-timeout-ms}` / `{write-timeout-ms}` / `{conn-max-lifetime}` / 重试；约束提示"写超时 ≥ 读超时、外部短超时 + 重试"。基线方法论真相源在 [`design-spec/06 §9.2`](../design-spec/06_resilience_design.md)，本节只登记"选了什么"不复制）
- `## 7.` 嵌入式库资源登记（如启用本地镜像：读池 `{read-maxOpen}` 按 `min(max(2×NumCPU,16),64)` 长存保温 / 写池 `MaxOpenConns=1` 串行。方法论真相源在 [`design-spec/03 §9.4`](../design-spec/03_concurrency_design.md)，本节只登记不复制）

> **完整章节骨架见** [`templates/docs/architecture/infrastructure.md`](../templates/docs/architecture/infrastructure.md)。
>
> §6 / §7 是**表示记录槽位**（工程填空"选了什么"），非方法论；连接池基线 / 嵌入式库读写双池的取舍方法只在 design-spec/06 §9.2 与 design-spec/03 §9.4，本规范引用不复制（双源真相规避）。

### 3.2 关键纪律

- **资源极简主义**：每个资源都要回答"为什么需要它"
- **时序明确**：哪些资源初始化必须先于其他资源
- **全局变量集中说明**：业务代码不应"猜"全局变量名

---

## 4. routing.md（路由与中间件链）

### 4.1 章节结构（强制）

routing.md 必须按以下章节顺序组织：

- `## 1.` 中间件链（执行顺序 ASCII 图 + 中间件详情表）
- `## 2.` 三类路由组定义（按调用方）
- `## 3.` 路由注册代码（每个路由组的注册函数）
- `## 4.` 包导入别名约定
- `## 5.` 完整路由表（**强制**：每条路由一行）
- `## 6.` 控制器分包策略
- `## 7.` MQ 消费路由（如有）
- `## 8.` 命令行任务路由（如有）

> **完整章节骨架见** [`templates/docs/architecture/routing.md`](../templates/docs/architecture/routing.md)。

### 4.2 必填精度

- **完整路由表**：必须列出所有路由（不要"等等"省略）
- 每条路由的业务校验中间件明确标出
- 路径命名规范（小写 / 连字符 / 不下划线）显式说明

---

## 5. middleware.md（中间件设计）

### 5.1 单个中间件的完整规格

每个中间件按以下结构描述（`## N.` 一个中间件，子节 `### N.1` 到 `### N.7`）：

- `### N.1` 职责（一句话）
- `### N.2` 注册位置（哪个路由组、middleware 文件路径）
- `### N.3` 输入（提取的参数 / 依赖的资源）
- `### N.4` 处理步骤（编号伪码）
- `### N.5` 输出（ctx.Set 注入字段 / 错误响应格式）
- `### N.6` 性能 / 缓存（如有热路径，显式说明本地缓存策略）
- `### N.7` 边界情况（缺参数 / 资源未找到 / 时间窗口超时 → 错误码）

> **完整章节骨架见** [`templates/docs/architecture/middleware.md`](../templates/docs/architecture/middleware.md)。

### 5.2 错误响应格式

中间件的错误响应**必须显式写出 JSON 结构**——不同路由组可能用不同响应格式。

---

## 6. {核心校验链}_flow.md（业务校验流程）

如有复杂核心业务链路（多步校验），单独成篇。

### 6.1 章节结构（强制）

按以下章节顺序组织：

- `## 1.` 签名算法（完整公式 + 时间窗口 + 大小写规则）
- `## 2.` 业务校验链概览（N 步链路总览图）
- `## 3.` 每步详细（编号 + 检查内容 + 错误码）
- `## 4.` 性能优化（Pipeline / 本地缓存 / short-circuit）
- `## 5.` 失败处理（每个失败点对应的状态码 / 是否计入状态）

> **完整章节骨架见** [`templates/docs/architecture/{核心校验链}_flow.md`](../templates/docs/architecture/auth_flow.md)（template 文件实际名 `auth_flow.md` 为示例骨架，落地按业务重命名）。

---

## 7. audit_log.md（审计日志）

如有审计需求，单独成篇。

### 7.1 章节结构（强制）

按以下章节顺序组织：

- `## 1.` insertOperationLog 函数（完整签名 + 字段映射表）
- `## 2.` 触发表（每个端点的 action / targetType / targetId / detail 取值规则）
- `## 3.` 数据流（端点 → service → insertOperationLog → MySQL log 库）
- `## 4.` DDL 引用（指向 schema/database_design.md 对应表）
- `## 5.` 查询接口（如有）

> **完整章节骨架见** [`templates/docs/architecture/audit_log.md`](../templates/docs/architecture/audit_log.md)。

---

## 8. render_functions.md（响应渲染函数）

如项目有多种响应格式（如 Internal `{status,message,data}` vs UserSelf `{errNo,errStr,data}`），单独成篇。

### 8.1 章节结构（强制）

按以下章节顺序组织：

- `## 1.` 响应模式对比表（每种模式 | JSON 结构 | 错误处理方式）
- `## 2.` 每个 render 函数的完整定义（签名 + 行为规则 + ctx 状态码）
- `## 3.` 自定义 Render 注册机制（如有）
- `## 4.` base.RenderJson 系列方法（错误参数支持的不同类型）
- `## 5.` 设计决策（为什么有多种格式）

> **完整章节骨架见** [`templates/docs/architecture/render_functions.md`](../templates/docs/architecture/render_functions.md)。

---

## 9. go_module.md（Go 模块与依赖）

### 9.1 章节结构（强制）

按以下章节顺序组织：

- `## 1.` 模块路径（`module {name}` + Go 版本）
- `## 2.` 直接依赖清单（每个依赖：名称 / 版本 / 用途）
- `## 3.` 不可替换依赖（哪些依赖不能升级及原因）
- `## 4.` go.mod 生成顺序（重建工程时）

> **完整章节骨架见** [`templates/docs/architecture/go_module.md`](../templates/docs/architecture/go_module.md)。

---

## 10. ai_dev_guide.md（AI 研发手册）

推荐写。是 `CLAUDE.md` 的扩展版，详细说明：

- 什么是 AI 原生工程
- 核心资产一览表
- 12 大 Skill 体系详解
- 设计先行研发流程
- 文档同步纪律
- 典型工作流示例
- 对研发人员的建议

> **完整章节骨架见** [`templates/docs/architecture/ai_dev_guide.md`](../templates/docs/architecture/ai_dev_guide.md)。

---

## 11. 通用要素

所有架构 md 都应：

- 顶部标注版本号 + 日期
- 章节编号统一 `## N.` `### N.M.`
- ASCII 图为优（mermaid 也可，但要确保可被纯文本工具读取）
- 引用其他 md 时使用 `[显示文本](相对路径)` 格式
- 表格使用标准 markdown


---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
