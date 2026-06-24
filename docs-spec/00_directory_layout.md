# 00 - 目录布局规范

> 规定 `docs/` 目录的标准结构、子目录命名与新增目录的判定规则。

---

## 1. 标准布局

```
<project-root>/
├── CLAUDE.md                # AI 总入口（强制，详见 03_claude_md_spec.md）
├── docs/
│   ├── INDEX.md             # 目录索引（强制，详见 01_index_md_spec.md）
│   ├── BUILD_STATUS.md      # 实现进度（强制，详见 02_build_status_md_spec.md）
│   │
│   ├── architecture/        # 架构与跨模块设计
│   │   ├── overview.md              # 架构总纲（强制）
│   │   ├── config.md                # 配置体系
│   │   ├── infrastructure.md        # 基础设施初始化
│   │   ├── routing.md               # 路由与中间件链
│   │   ├── middleware.md            # 中间件设计
│   │   ├── status_codes.md          # 错误码体系
│   │   ├── {核心校验链}_flow.md      # 业务校验流程（如有）
│   │   ├── audit_log.md             # 审计日志（如有）
│   │   ├── render_functions.md      # 响应渲染函数（如有多种响应格式）
│   │   ├── helpers_api.md           # 工具函数与全局变量 API（强制）
│   │   ├── pkg_api.md               # 内部框架 API 契约（强制）
│   │   ├── go_module.md             # Go 模块与依赖
│   │   ├── constants.md             # 业务常量唯一真相源
│   │   ├── logging.md               # 日志规范
│   │   ├── ai_dev_guide.md          # AI 研发手册（推荐）
│   │   └── mvp_rebuild_path.md      # MVP 分阶段重建路径（强制）
│   │
│   ├── api/                 # 接口规格（每类调用方一份）
│   │   ├── {audience1}_interfaces.md        # 如 audience_a_interfaces.md
│   │   ├── {audience2}_interfaces.md        # 如 audience_b_interfaces.md
│   │   └── {audience3}_interfaces.md        # 如 audience_c_interfaces.md
│   │
│   ├── service/             # Service 层设计
│   │   ├── service_design.md                # Service 总览
│   │   ├── {module}_service.md              # 每个 service 模块一份
│   │   └── scheduled_tasks_design.md        # 定时任务 + MQ 消费者
│   │
│   ├── schema/              # 数据库设计
│   │   └── database_design.md               # DDL + 分库 + Redis Key 注册
│   │
│   ├── cache/               # 缓存设计
│   │   ├── cache_design.md                  # 缓存策略 + Key 模式 + Hash 字段映射
│   │   └── cache_helpers.md                 # 缓存辅助函数（如有 L1 本地缓存层）
│   │
│   ├── circuit_breaker/     # 熔断 / 限流 / 降级
│   │   └── circuit_breaker_design.md
│   │
│   └── testing/             # 测试设计
│       └── testing_design.md
│
└── .claude/
    ├── settings.local.json  # 项目级设置
    └── skills/              # 19 个 Skill（详见 skills-spec/）
        ├── new-model/SKILL.md
        ├── new-service/SKILL.md
        └── ...
```

---

## 2. 强制 vs 可选

| 文件 / 目录 | 状态 | 缺失会导致 |
|------------|------|-----------|
| `CLAUDE.md` | 强制 | AI 不知道项目规范，每次都要重新探索 |
| `docs/INDEX.md` | 强制 | AI 无法定位文档之间的引用关系 |
| `docs/BUILD_STATUS.md` | 强制 | AI 无法判断哪些代码已实现、哪些是 🚫 暂不实现 |
| `docs/architecture/overview.md` | 强制 | 无法建立系统全局心智模型 |
| `docs/architecture/helpers_api.md` | 强制 | 工具函数签名无真相源，业务代码会跨包乱调 |
| `docs/architecture/pkg_api.md` | 强制 | 框架类型签名无真相源，AI 频繁猜错导致编译失败 |
| `docs/architecture/mvp_rebuild_path.md` | 强制 | `/rebuild-from-docs all` 没有分阶段保护，出错难以回滚 |
| `docs/api/*.md` | 强制（如有 HTTP 接口） | Controller 无法生成 |
| `docs/service/service_design.md` | 强制（如有业务逻辑） | Service 无法生成 |
| `docs/schema/database_design.md` | 强制（如有数据库） | Model 无法生成、缓存 Key 无对照表 |
| `docs/cache/cache_design.md` | 强制（如有 Redis） | 缓存 Key 无注册表 |
| `docs/circuit_breaker/` | 推荐 | 故障保护不可重建，但短期可不写 |
| `docs/testing/testing_design.md` | 推荐 | 没有测试纪律和框架契约 |
| `docs/architecture/audit_log.md` | 按需 | 仅当有审计需求时 |
| `docs/architecture/{核心校验链}_flow.md` | 按需 | 仅当有复杂业务校验时 |
| `docs/architecture/render_functions.md` | 按需 | 仅当多种响应格式时 |

---

## 3. 命名规则

### 3.1 文件名

- **统一小写 + 下划线**：`scheduled_tasks_design.md`、`status_codes.md`
- **接口规格按调用方命名**：`{audience}_interfaces.md`，如 `audience_a_interfaces.md`
- **service 模块按名称**：`{module}_service.md`，如 `{module-1}_service.md`
- **MD 后缀小写**：`.md` 不是 `.MD` 也不是 `.markdown`

### 3.2 目录名

- 按业务/技术维度划分，单数或不可数名词：`architecture/`、`service/`、`schema/`、`cache/`、`testing/`
- 不使用复数前缀：避免 `architectures/`
- 不使用动词目录：避免 `building/`、`deploying/`

### 3.3 锚点（章节标题）

- 一级标题用项目名：`# {ProjectName} — {文档主题}`
- 二级及以下用主题名，不重复项目名

---

## 4. 新增目录的判定

### 4.1 何时新增子目录

满足以下任一条件时，可在 `docs/` 下新增子目录：

1. **覆盖一个独立的设计视角**（如 `cache/`、`circuit_breaker/`），且预计将有 ≥ 2 篇 md
2. **对应一个独立的代码层/包**（如 `service/`、`api/`）
3. **服务于一个明确的工程能力**（如 `testing/`、`migration/`）

### 4.2 何时不新增

- 单篇 md 能容纳的内容（直接放 `architecture/`）
- 临时工作笔记（不应进入 docs/）
- 用户故事 / 需求描述（这是产品文档，不是工程蓝图）

### 4.3 新增后必须做

1. 在 `docs/INDEX.md` 中**新增对应的二级标题章节**和表格
2. 更新 `CLAUDE.md` 的「范围判定表」（添加该目录的代码路径 → 文档路径映射）
3. 在 `docs/architecture/ai_dev_guide.md` 中提及（如有）
4. 与团队/AI 沟通该目录的语义

---

## 5. 不应该出现在 docs/ 的内容

| 内容 | 应放在哪里 |
|------|----------|
| 个人笔记 / 调研草稿 | 项目外（如 Notion、Obsidian） |
| 临时讨论记录 | 项目外 |
| 产品需求文档 (PRD) | 产品库（不混入技术蓝图） |
| 用户故事 | 产品库 |
| 部署运维**操作手册**（发布步骤 / 值班 / 应急） | `runbook/` 或 `ops/`（独立目录，不和 docs/ 混）。但部署**设计契约**（镜像/探针/优雅启停/发布回滚）入 `docs/architecture/deployment.md`，见 docs-spec/27 |
| 历史变更日志 | `CHANGELOG.md`（项目根） |
| 实现细节注释 | 代码内的 godoc |

`docs/` 是**系统蓝图**，不是工作日志。任何不能服务于"重建工程"的内容都应排除。

---

## 6. 与 .claude/ 的关系

`.claude/` 与 `docs/` 是平级关系：

- `docs/` 描述"系统是什么"
- `.claude/skills/` 描述"如何根据 docs/ 生成代码"

两者通过 `CLAUDE.md` 串联——CLAUDE.md 既指向 `docs/INDEX.md`，也指向 `.claude/skills/`。

---

## 7. 自检

新项目按本规范初始化 `docs/` 后，运行以下命令应当无错：

```bash
# 1. 所有强制目录都已创建
ls docs/architecture docs/api docs/service docs/schema docs/cache

# 2. 所有强制 md 都已存在（即使内容为空白模板）
test -f docs/INDEX.md && \
test -f docs/BUILD_STATUS.md && \
test -f docs/architecture/overview.md && \
test -f docs/architecture/helpers_api.md && \
test -f docs/architecture/pkg_api.md && \
test -f docs/architecture/mvp_rebuild_path.md

# 3. .claude/ 骨架就位
ls .claude/skills/
test -f CLAUDE.md
```

不过则补齐缺失项再开工。


---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
