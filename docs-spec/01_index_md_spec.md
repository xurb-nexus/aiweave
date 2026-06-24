# 01 - INDEX.md 写法规范

> 规定 `docs/INDEX.md` 的内容结构、登记格式、维护规则。

---

## 1. 定位

`docs/INDEX.md` 是 **AI 在阅读任何 md 之前的第一站**。它的职责是：

1. **目录式入口**：列出 docs/ 下全部 md 文件
2. **路标**：每篇 md 一行描述，点出**本篇能回答什么问题、不能回答什么问题**
3. **关系图**：让 AI 知道"如果我要做 X，应该先读哪一篇"

INDEX.md 不是导航条目的简单堆砌，而是 AI 理解系统的入口图。

---

## 2. 顶层结构（强制）

INDEX.md 必须按以下章节顺序组织：**顶层入口 → 架构设计 → 接口设计 → 数据模型 → 缓存设计 → Service 层设计 → 熔断器设计 → 变更日志**。每个章节都用 markdown 表格，两列：`| 文档 | 说明 |`。

> **完整章节骨架见** [`templates/docs/INDEX.md`](../templates/docs/INDEX.md)——可直接复制到工程作起点。

---

## 3. 单条登记格式

### 3.1 表格列定义

```markdown
| 文档 | 说明 |
|------|------|
| [path/to/file.md](path/to/file.md) | 一句话说明本篇能回答什么问题，最长不超过 200 字 |
```

### 3.2 描述精度

每条描述必须做到：

- **回答"我想知道 X，是不是该读这一篇"**
- **列出本篇覆盖的关键内容**（具体到节标题或核心信息）
- **必要时点出"对面"**（即不该来这里读什么）

### 3.3 反例

```markdown
| [overview.md](architecture/overview.md) | 架构总纲 |   ← ❌ 太空，无法判断该不该读

| [overview.md](architecture/overview.md) | 系统总纲：定位、整体架构图、N 步业务校验链、缓存架构、Redis-first 写入与增量 Flush、统一熔断器、性能/稳定性/成本设计亮点、AI 原生工程   ← ✅ 让人一目了然
```

### 3.4 描述需点明的元素

| 元素 | 例子 |
|------|------|
| 覆盖的核心子主题 | "N 步业务校验链、Pipeline 优化、缓存预热" |
| 量化指标（如有） | "P99 < {N}ms、QPS > {N}" |
| 数量（如可统计） | "N 张表 DDL"、"N 个接口" |
| 状态标注（如适用） | "🚫 当前阶段不实现"、"⬜ 设计已定稿待实现" |
| 关键依赖 | "AI 生成代码前**必读**" |

---

## 4. 引用关系标注

如果某篇 md 与其他 md 有强引用关系，**在描述中显式标注**：

```markdown
| [pkg_api.md](architecture/pkg_api.md) | **pkg/ 内部框架 API 契约**：业务代码依赖的全部公开类型和函数签名。AI 生成任何业务代码前**必读**。 |

| [BUILD_STATUS.md](BUILD_STATUS.md) | **实现进度一览**（🟢/🟡/⬜ 每层状态、AI 生成代码前必查） |
```

加粗 / 斜体 / **必读** 等字样允许使用，目的是吸引 AI 注意力，避免遗漏前置依赖。

---

## 5. 维护规则

### 5.1 新增/重命名/删除 md 时

| 动作 | 必做 |
|------|------|
| 新增 md | 在对应章节表格中追加一行；如需新章节，在 README 风格的表格中新增章节 |
| 重命名 | 更新表格中的链接路径和文件名引用；如描述需调整一并改 |
| 删除 md | 移除对应行；检查其他 md 是否还引用了被删 md，连带清理 |

### 5.2 描述漂移检测

每次修改 md **正文**（非 typo / 排版），重读 INDEX.md 中对应行的描述，确认它仍准确。

### 5.3 计数同步

如果描述中包含数量（"N 张表"、"N 个接口"），实际新增/删除时同步更新计数。可由 `/update-index` Skill 自动维护。

---

## 6. 行长与可读性

### 6.1 行长上限

每条登记建议不超过 **300 字符**。超过时考虑拆分章节或精简描述。

不强行限制，但要明白：INDEX.md 会被 AI 全文加载到上下文，过长会浪费 tokens。

### 6.2 表格格式

- 使用标准 markdown 表格（不用 HTML 表）
- 不混用对齐方式
- 链接使用 `[显示文本](路径)` 格式

---

## 7. 顶层入口章节的特殊性

`## 顶层入口` 章节列出 **AI 必须最先读** 的文档：

- `BUILD_STATUS.md` —— 任何代码生成前的拒绝规则
- `testing_design.md`（如有） —— 测试纪律
- `mvp_rebuild_path.md` —— 分阶段构建蓝图

这一节的描述要更详尽，因为它是 AI 进入工程的第一道门。

---

## 8. 章节合理性

INDEX.md 的章节划分应当与 `docs/` 子目录划分一致：

| INDEX 章节 | 对应目录 |
|-----------|----------|
| 架构设计 | `docs/architecture/` |
| 接口设计 | `docs/api/` |
| 数据模型 | `docs/schema/` |
| 缓存设计 | `docs/cache/` |
| Service 层设计 | `docs/service/` |
| 熔断器设计 | `docs/circuit_breaker/` |
| 测试设计 | `docs/testing/` |

新增子目录时同步新增章节（详见 `00_directory_layout.md` §4.3）。

---

## 9. 自动化维护

`/update-index` Skill 自动扫描 `docs/` 并对比 `INDEX.md`：

- 新增的 md → 提示登记
- 删除的 md → 提示移除
- 计数差异 → 提示更新

详见 `aiweave/templates/skills/update-index/SKILL.md`。

---

## 10. 与其他 md 的关系

| 文件 | 与 INDEX.md 的关系 |
|------|------------------|
| `BUILD_STATUS.md` §8 | 文档状态总览，列举每个目录的 md 篇数（与 INDEX 计数同源） |
| `CLAUDE.md` 范围判定表 | 代码路径 → md 路径映射，与 INDEX.md 互补（CLAUDE.md 是反向：从代码改动找文档，INDEX.md 是正向：从文档主题找文件） |

---

## 11. 完整模板

参见 `aiweave/templates/docs/INDEX.md`。


---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
