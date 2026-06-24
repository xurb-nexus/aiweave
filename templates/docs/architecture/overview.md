# {ProjectName} — 架构总纲

> 版本：1.0 | 日期：{YYYY-MM-DD}
> 规范来源：`aiweave/docs-spec/04_architecture_overview_spec.md`

---

## 1. 系统定位

{一段话说明本服务做什么、不做什么}。性能目标：QPS > {X}，P99 < {Y}ms，可用性 {99.X}%。

```
{ASCII 上下游关系图}
```

## 2. 整体架构

```
{ASCII 整体架构图：调用方 → Gin 中间件 → 路由组 → service → 数据访问 → 熔断器}
```

### 关键设计

**①** {设计 1，例：Redis-first 写入策略}

**②** {设计 2}

**③** ...

## 3. 核心业务链路（如有）

```
{N 步链路图}
```

性能目标：{量化指标}

## 4. 性能 / 稳定性 / 成本设计亮点

### 4.1 性能
- {QPS / P99 / 缓存命中率}

### 4.2 稳定性
- {熔断 / 降级 / 容错}

### 4.3 成本
- {资源消耗 / 横向扩展}

## 5. AI 原生工程

详见 [`docs/architecture/ai_dev_guide.md`](ai_dev_guide.md)。


---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
