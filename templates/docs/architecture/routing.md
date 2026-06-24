# 路由与控制器设计

> 规范来源：`aiweave/docs-spec/04_architecture_overview_spec.md` §4

## 1. 中间件链

```
请求进入
  ├── recovery        （golib 内置）
  ├── accessLog       （golib 内置）
  ├── pprof           （golib 内置）
  ├── Cors            （middleware/cors.go，全局）
  ├── Metrics         （middleware/metrics.go，全局）
  ├── [路由组中间件]
  └── Handler 执行
```

| 中间件 | 文件 | 注册位置 | 触发条件 | 对 ctx 的影响 |
|--------|------|---------|---------|--------------|
| ... | ... | ... | ... | ... |

## 2. 三类路由组定义

### 2.1 {Audience1} 路由组

| 维度 | 说明 |
|------|------|
| 路径前缀 | `/{prefix}/...` |
| 调用方 | ... |
| 业务校验中间件 | ... |
| 接口数量 | N |
| 接口文档 | [`docs/api/{audience1}_interfaces.md`](../api/...) |

### 2.2 {Audience2}

### 2.3 {Audience3}

## 3. 路由注册代码

### 3.1 router/http.go

（按 `aiweave/templates/skills/new-router/SKILL.md` 模板）

### 3.2 router/http_{audience}.go

（同上）

## 4. 包导入别名约定

| import 路径 | 别名 |
|------------|------|
| `{project_module}/controllers/http/operator/{module}` | `{audience}_{module}` |
| ... | ... |

## 5. 完整路由表

| 方法 | 路径 | 控制器 | 业务校验 | 文档 |
|------|------|--------|------|------|
| GET | `/{prefix}/ready` | `probe.Ready` | 无 | — |
| ... | ... | ... | ... | ... |

## 6. 控制器分包策略

```
controllers/http/{audience}/{module}/{action}.go
```

## 7. MQ 消费路由（如有）

## 8. 命令行任务路由


---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
