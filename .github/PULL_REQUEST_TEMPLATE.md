## 改动概述

<!-- 一句话说明本 PR 的目的 -->

## 改动类型

- [ ] 🐛 修复笔误 / 链接 / 格式
- [ ] 📝 新增规则 / docs-spec / Skill / 模板
- [ ] 🔧 更新现有规则
- [ ] 🎨 重构（不改变规则语义，仅调整结构）
- [ ] 📖 文档增强（README / INDEX / OPERATIONS / PRINCIPLES）

## 关联 Issue

Closes #

## AIWeave 双向同步自检清单（强制）

### §12 占位符规则
- [ ] 主体段无业务名泄漏（按 PRINCIPLES §12.4 关键词清单自检）
- [ ] 涉及示例的内容已迁入末尾"参考示例"段并带 `> ⚠️` 警告行
- [ ] 占位符使用了规范的命名（`{Module}` / `{Method}` / `{ns}` / `{example_table_*}` 等）

### 双向同步纪律
- [ ] 涉及 docs-spec 新增 → INDEX.md `docs-spec/` 表已新增一行
- [ ] 涉及 Skill 新增 → INDEX.md Skill 表 + skills-spec/00 体系全景已更新
- [ ] 涉及新规范 → CLAUDE.md 范围判定表已联动 + OPERATIONS 工作流已联动
- [ ] 涉及新规范 → 已配套 B1 反向同步规则（镜像建设规则）

### 变更日志
- [ ] INDEX.md §变更日志已新增一行（日期 + 触发事件 + 改动摘要）

## 测试

<!-- 如改动涉及 Skill 模板、grep 锚等可机械验证项，说明如何验证 -->

## 其他说明

<!-- 任何 reviewer 需要特别注意的点 -->


---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
