# 03 - CLAUDE.md 写法规范

> 规定项目根目录 `CLAUDE.md` 的内容结构。CLAUDE.md 是 AI 进入工程的总入口，优先级高于一切其他规范文档。

---

## 1. 定位

`CLAUDE.md` 在每次对话启动时被 Claude Code 自动加载到上下文。它的职责：

1. **告诉 AI 项目是什么** —— 一段简短的项目概述
2. **告诉 AI 项目的当前状态** —— 指向 BUILD_STATUS.md
3. **声明硬纪律** —— 文档同步规则、暂不实现模块、范围判定
4. **快速命令清单** —— 常用 build/test/run 命令
5. **范围判定表** —— 代码路径 → 文档路径的双向映射

CLAUDE.md 是 **AI 的契约书**，不是项目说明文档。它的语气应当是"必须做 X / 不得做 Y"，不是"我们项目的特点是 X"。

---

## 2. 顶层结构（强制）

CLAUDE.md 必须按以下章节顺序组织：

- `## 项目概述`（1-2 段简介 + 当前实现状态指向 BUILD_STATUS；如有暂不实现模块加警示框）
- `## 常用命令`（构建与运行 / 测试 / 代码格式化）
- `## 项目架构`（目录结构 / 分层架构 / 配置体系 / 初始化流程 / 接口返回格式 / 错误码规范 / 路由注册 / 支持的基础设施 / 核心设计原则 / 外部 API 调用如有）
- `## 开发注意事项`（含日志规范、Hooks 机制（L0 强制）、新增接口/外部调用流程、定时任务类型等）
- `## 文档同步规则（强制）`（愿景 / 设计先行 / 双向同步 / 20 条具体规则 / 危险模式清单（含 R-IO-* / R-CONF-* / R-CACHE-* grep 锚）/ 范围判定表 / 输出格式 / 文档目录结构）

> **完整章节骨架见** [`templates/CLAUDE.md`](../templates/CLAUDE.md)——可直接复制到项目根。下面 §3-§7 给出每个章节的填充规则与必填要素。

---

## 3. §项目概述（必填模板）

```markdown
## 项目概述

{ProjectName}（{ProjectAlias}），基于 {Framework}（{Language} {Version}），是一个{独立微服务/库/...}，提供 {core capabilities}。架构上设计为 {N 种运行模式共一份代码库}，由 {cli framework} 子命令区分入口。

> **当前实现状态**：项目处于"{x}"阶段。完整实现状态一览见 [`docs/BUILD_STATUS.md`](docs/BUILD_STATUS.md)。当你读到文档里某个 service 方法、接口或定时任务时，**不要假设它已在代码中实现**——先查 BUILD_STATUS，再决定是"调用现有实现"还是"按 skill 生成新实现"。

> **⛔ 暂不实现模块（如有）**：（按 BUILD_STATUS §0 抽出关键内容并指向）
```

### 3.1 警示框语法

```markdown
> **⛔ 暂不实现模块**：**{module} 模块**（`{path1}`、`{path2}`）当前阶段**一律不进入代码实现**。设计文档完整保留，但 AI 在任何 skill 下都**不得**为这些模块生成代码。完整列表、检查规则、解除条件见 [`docs/BUILD_STATUS.md`](docs/BUILD_STATUS.md) §0。
```

### 3.2 不该写的内容

- 项目历史 / 团队组成（写在 README 不写在 CLAUDE.md）
- 运维**操作流程**（发布步骤 / 值班 / 应急预案，写在 runbook）。注意：部署**设计契约**（镜像必含项 / 探针语义 / 优雅启停时序 / 发布回滚策略）属系统蓝图，写在 `docs/architecture/deployment.md`，由 [`docs-spec/27`](27_deployment_runtime_spec.md) 治理——两者的判定标准是"能否服务于重建工程"
- 商业背景 / 市场分析（不该出现在工程文档）

---

## 4. §常用命令（必填）

```markdown
## 常用命令

### 构建与运行

\`\`\`bash
# 启动 HTTP 服务（默认）
go run main.go

# 启动指定任务
go run main.go {task-name}

# 构建二进制文件
go build -o {binary-name}
\`\`\`

### 测试

\`\`\`bash
go test ./...
go test ./{package}/...
go test -cover ./...
\`\`\`

### 代码格式化

\`\`\`bash
go fmt ./...
\`\`\`
```

---

## 5. §项目架构（必填）

包含以下子节：

### 5.1 目录结构

用 ASCII 树画出顶层目录，每行一句注释：

```
project/
├── main.go              # 入口文件
├── conf/                # 配置目录
│   ├── app/             # 业务配置
│   └── mount/           # 环境配置
├── controllers/         # 请求入口
│   ├── command/         # 定时任务
│   ├── http/            # HTTP 入口
│   └── mq/              # MQ 消费
...
```

### 5.2 分层架构

用 ASCII 图描述层级关系：

```
controllers/ → service/ 或 api/ → models/ 或 data/ 或 helpers/
     ↓               ↓                       ↓
  参数校验        业务逻辑              数据访问
```

并明确每层职责：

- **controllers/** —— ...
- **service/** —— ...
- **api/** —— ...
- **models/** —— ...
- **data/** —— ...
- **helpers/** —— ...

### 5.3 配置体系

详见 `aiweave/templates/CLAUDE.md`，描述：

- 配置文件分类（mount / app）
- 加载机制
- 敏感信息占位规则

### 5.4 初始化流程

按编号列出 main.go → helpers.PreInit → InitResource 的实际顺序，每行一句简介。

### 5.5 接口返回格式

如果项目有多种返回格式（如 Internal vs UserSelf），列举每种格式的 JSON 结构。

### 5.6 错误码规范

简略列出错误码范围划分（详细见 `docs/architecture/status_codes.md`）。

### 5.7 核心设计原则

3-5 条最核心的设计原则，用一句话表达。例如：

- Redis-first：高流量操作先写 Redis，定时任务 flush 到 MySQL
- MQ 异步落库：业务事件流水走 Kafka 异步持久化
- 统一熔断器：Redis、MySQL、外部 API 全部由同一套接口保护

---

## 6. §开发注意事项（必填）

包含但不限于：

- 控制器只负责参数校验，不写业务逻辑
- URI 命名规范
- 外部 API 调用必须使用熔断器
- 新增错误码的位置
- 测试文件的位置和命名
- **日志规范（强制）** —— 引用 `docs/architecture/logging.md`
- 新增接口的标准流程
- 新增外部 API 调用的标准流程
- 定时任务类型说明

---

## 7. §文档同步规则（强制，最重要）

这一章是 CLAUDE.md 的核心。**必须照抄以下结构**，仅替换工程特定内容：

### 7.1 愿景

```markdown
> **本规则是本项目的最高优先级纪律，优先级高于一切其他开发规范。任何不满足文档同步的交付都视为未完成，必须补齐后才能结束。**

`docs/` 中的 md 是整个系统的完整蓝图。目标是：**仅凭 docs/ 中的 md 就能用 AI 从零重建整个工程**。
```

### 7.2 研发流程：设计先行

按 `aiweave/PRINCIPLES.md` §10 的流程图照抄。

### 7.3 双向同步（核心原则）

变更方向 → 强制要求的对照表，按 `aiweave/PRINCIPLES.md` §2.1 照抄。

### 7.4 具体执行规则（9 条）

按以下编号顺序列出：

```markdown
#### 规则 1：修改代码前 — 查找关联文档
#### 规则 2：修改代码后 — 同步更新文档
#### 规则 3：新增代码 — 必须有文档（零容忍）
#### 规则 4：修改 md 后 — 检查代码一致性
#### 规则 5：更新 INDEX.md
#### 规则 6：自检清单（每次交付前必须过一遍）
#### 规则 7：文档优先级的例外——签名冲突裁决
#### 规则 8：BUILD_STATUS 先查
#### 规则 9：代码必须同步测试用例（硬约束）
```

每条的具体内容见 `aiweave/PRINCIPLES.md`。

### 7.5 范围判定表

```markdown
### 范围判定表

| 变更的代码路径 | 检查的文档 |
|--------------|----------|
| `main.go`、`helpers/init.go` | `docs/architecture/overview.md` |
| `go.mod`、`go.sum` | `docs/architecture/go_module.md` |
| `helpers/`（资源初始化） | `docs/architecture/infrastructure.md` |
| `conf/` | `docs/architecture/config.md` |
| `router/` | `docs/architecture/routing.md` |
| `middleware/` | `docs/architecture/middleware.md` |
| `components/error.go` | `docs/architecture/status_codes.md` |
| `components/`（业务常量） | `docs/architecture/constants.md` |
| `api/` | `docs/architecture/overview.md`（外部 API 调用部分） |
| `models/` | `docs/schema/database_design.md` |
| `service/{module}/` | `docs/service/{module}_service.md`、`docs/service/service_design.md` |
| `service/{audit-module}/` | `docs/architecture/audit_log.md` |
| `controllers/http/{audience}/` | `docs/api/{audience}_interfaces.md` |
| `controllers/command/`、`router/command.go` | `docs/service/scheduled_tasks_design.md` |
| `controllers/mq/`、`router/mq.go` | `docs/service/scheduled_tasks_design.md` §MQ |
| `helpers/rediscache/` | `docs/cache/cache_design.md`、`docs/architecture/helpers_api.md` |
| `helpers/circuit_breaker.go` | `docs/circuit_breaker/circuit_breaker_design.md` |
| `data/` | `docs/cache/cache_design.md`（缓存数据访问层） |
| `conf/app/app.yaml` | `docs/circuit_breaker/circuit_breaker_design.md` |
| `helpers/`（工具函数、全局变量） | `docs/architecture/helpers_api.md` |
| `pkg/**` | `docs/architecture/pkg_api.md` |
| 日志相关 | `docs/architecture/logging.md` |
| 实现进度 / 代码落盘状态 | `docs/BUILD_STATUS.md` |
| `test/framework/` | `docs/testing/testing_design.md` |
| `test/cases/` | `docs/testing/testing_design.md`、对应 `docs/api/*.md` |
```

**判定原则**：每行格式 `代码路径 → 文档路径`。代码层级与文档层级 1:1 对应。新增任何代码目录时，必须新增对应行。

### 7.6 输出格式

```markdown
### 输出格式

每次回复的末尾，必须包含文档同步状态声明，二选一：

- **有变更**：「文档同步：更新了 `docs/xxx.md`、新建了 `docs/yyy.md`」
- **无变更**：「文档同步：本次修改不涉及文档变更」
```

### 7.7 文档目录结构

简短列出 docs/ 子目录及作用，与 INDEX.md 章节呼应。

---

## 8. 长度上限

CLAUDE.md 总长度建议不超过 **500 行**。每次对话都加载，过长拖累所有交互。

如果某节内容过多，应**外移到 docs/architecture/** 下的专题 md，CLAUDE.md 只保留指针。

---

## 9. 维护

| 触发 | 动作 |
|------|------|
| 新增代码目录 | 在范围判定表新增一行 |
| 新增 docs/ 子目录 | 在文档目录结构补充 |
| 修改文档同步规则 | 同时更新 `aiweave/PRINCIPLES.md`（保持一致） |
| 启用 🚫 模块 | 删除 §项目概述 中的警示框 + 同步删 BUILD_STATUS §0 |

---

## 10. 模板

完整模板见 `aiweave/templates/CLAUDE.md`。


---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
