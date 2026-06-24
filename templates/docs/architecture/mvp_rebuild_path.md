# 分阶段构建路径

> 规范来源：`aiweave/docs-spec/18_mvp_rebuild_path_spec.md`

## 1. 文档定位

### 1.1 三种使用场景

| 场景 | 触发 | 输入 | 输出 |
|------|------|------|------|
| **forward** | 新工程启动 | docs/ + skills/ | 按 Stage 逐个生成 |
| **rebuild** | 代码丢失/换栈 | docs/ + skills/ | 按 Stage 逐个重建 |
| **refactor** | 大重构 | 旧代码 + 新 docs | 按 Stage 切片替换 |

### 1.2 与 BUILD_STATUS 的关系
BUILD_STATUS 反映"当前实际状态"，本文档反映"目标路径"。

### 1.3 何时不使用本文档
- 单一 bug 修复
- 单 API 接口的迭代（走 OPERATIONS.md W2 工作流）
- 配置参数调整

## 2. 阶段拆分方法论

### 2.1 拆分原则
1. 依赖最少 → 最多
2. 每个 Stage 可独立运行
3. 业务价值递增
4. 单 Stage 代码量 < 2000 行

### 2.2 单 Stage 的判定标准
- [ ] 有明确的"完成验证"
- [ ] 有限的依赖范围
- [ ] 有限的代码量（< 2000 行）
- [ ] 有清晰的回退机制

### 2.3 Stage 间的依赖图谱
```
[L0] go.mod / pkg/ / 配置 / components / models / helpers
        ↓
[L1] 公共工具函数 + 业务常量
        ↓
[L2] 最小关键链路
        ↓
[L3] 业务模块 A / B / C 并行
```

## 3. 前置约束

- [ ] go.mod / module / Go 版本已就绪
- [ ] pkg/ 内部框架已存在
- [ ] conf/ 配置骨架已存在
- [ ] components/error.go 已存在
- [ ] models/ 已生成或可由 /new-model all 生成
- [ ] helpers/ 资源初始化已就绪
- [ ] middleware/ 通用中间件已就绪
- [ ] router/http.go 主入口已就绪
- [ ] main.go 已就绪

## 4. 当前工程的具体阶段

### Stage 1 — 公共工具 & 常量（约 X 行）
- **目标**：
- **涉及文档**：
- **生成的文件**：
- **临时简化**：
- **验证**：
- **测试交付**：
- **MVP 里程碑**：
- **BUILD_STATUS 更新**：

### Stage 2 — 最小关键链路（约 X 行）

### Stage 3 — 业务自助（约 X 行）

### Stage 4 — 业务核心闭环（约 X 行）

### Stage 5 — 后台 / 长尾（约 X 行）

### Stage 6（可选）— 启用 🚫 模块（如有）

## 5. 阶段间依赖关系

```
Stage 1 → Stage 2 → Stage 3 → Stage 4 → Stage 5 → Stage 6（可选）
```

## 6. AI 执行协议

每个 Stage 开工时：
1. 读本文档对应章节
2. 读 BUILD_STATUS 标记 🟡（进行中）
3. 按 skill 生成代码
4. 每层 go build 验证
5. 全量 go build + vet
6. 同步测试用例
7. 更新 BUILD_STATUS → 🟢
8. 跑 /doc-sync-check
9. 汇报用户确认后进入下一阶段

## 7. 失败回退策略
- 不硬写 → 修 md → 用户评审 → 再继续
- Git revert 回到 Stage N-1

## 8. 与其他 skill 的协作

| Skill | 在哪些 Stage 触发 |
|-------|-----------------|
| ... | ... |

## 9. 三种使用场景的具体路径

### 9.1 forward
### 9.2 rebuild
### 9.3 refactor

## 10. 当前阶段决策记录（如有 🚫 模块）


---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
