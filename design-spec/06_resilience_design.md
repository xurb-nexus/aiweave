# 06 - 稳定性设计决策规范

> 回答"面对一个需求，怎么选熔断 / 降级 / 限流策略与连接池 / 配置 / 分层 / 可观测性基线"。这套底座支撑 IO 设计在生产稳定跑。
>
> **边界先声明**：熔断算法选型 / 差异化参数 / 接入方式（[`docs-spec/12`](../docs-spec/12_circuit_breaker_spec.md)）、全局 SLA / 热路径 / 内存预算 / 背压（[`docs-spec/22`](../docs-spec/22_performance_contract_spec.md)）、可观测性 Metrics（[`docs-spec/23`](../docs-spec/23_observability_spec.md)）、配置中心 / 凭据加密（[`docs-spec/26`](../docs-spec/26_config_center_spec.md)）、结构化日志（[`docs-spec/13`](../docs-spec/13_logging_spec.md)）等**规则真相源在对应 docs-spec**，本篇只引用不复制。本篇新增决策资产：熔断形态选型、连接池基线、分层与数据层禁日志、危险开关双门禁。

---

## 1. 设计目标

- **不被拖垮**：依赖故障时熔断 / 降级，不让单点拖垮整体；恢复时平滑回升不雪崩。
- **过载可控**：入口限流，热路径有 SLA 与背压。
- **数据层可无副作用复用**：分层单向依赖，数据层不打日志（横切关注点归请求边界）。
- **高风险开关有运行时硬门禁**：不可能在线上误生效。

---

## 2. 输入信号识别

| 迹象 | 典型描述 |
| --- | --- |
| 外部依赖 | "调用下游 / 外部 API，可能慢 / 失败" |
| 热路径 | "关键路径接口，QPS 高，有 SLO" |
| 过载风险 | "可能被突发流量打满" |
| 多资源接入 | "Redis / MySQL / 外部 API 都要保护" |
| 凭据 / 危险开关 | "敏感密钥""跳缓存等高风险开关" |

---

## 3. 决策树（强制）

### 3.1 熔断 / 降级 / 限流选型

```
稳定性风险是哪种形态？
│
├─ 外部依赖可能故障 ──► 自适应熔断（概率丢弃，非二态开关）+ 超时 + 差异化降级
│                        └─ 接入按基础设施类型选：注册表(Redis) / GORM 插件(MySQL) / 显式包裹(外部 API)
├─ 入口过载风险 ──────► 限流（令牌桶 / 漏桶 / 滑窗）
├─ 热路径性能 ────────► 标热路径 + 定 SLA + 往返预算（衔接 01 §1）
└─ 资源耗尽风险 ──────► 背压 / 内存预算 / 有界队列（22 §6.1）
```

### 3.2 为什么概率丢弃 > 二态开关

```
二态 Open/Closed：恢复瞬间全部放行 → 再次打满 → 雪崩
SRE 概率丢弃：     K = 1/percent；丢弃概率 p = max(0, (total - K×accepts)/(total+1))
                  后端恢复时 accepts↑ → K×accepts 放大快于 total → p 自然归零 → 流量平滑回升
按组件特性分级调参：越快 / QPS 越高 / 容错越严 → 越激进（短窗口 + 高 percent）
```

> 这是 **Google SRE 客户端自适应限流（client-side throttling）**——调用方在本地按 `accepts/requests` 自我丢弃、丢在请求发出前。K 值权衡（节流启动阈值 = 失败率超 `1−1/K`）与"作用于依赖侧 vs 入口侧"的边界见 §9.1。

### 3.3 分层、配置、可观测性

```
├─ 分层依赖 ──► 严格单向 controllers→service/api→models/data/helpers→pkg→conf（禁反向）
│               └─ service / data 层禁打日志 → return error 上传，由 controller 定 WARN/ERROR
│                  （日志归请求边界：那里才有完整上下文；同一 Repo 被 HTTP/cron/job 复用，自打会重复+误判）
├─ 配置 / 凭据 ──► 框架层薄（纯 HTTP 取配置）；凭据加密入库但解密密钥留代码（钥匙锁分离）
│                  高风险开关双重硬门禁（配置档位 + 运行档位 RUN_ENV 叠加）
└─ 可观测性 ──► 关键路径直方图盯 SLO（桶按 SLO 设，label 带 status/mode）+ 业务计数器
```

---

## 4. 默认选型与升级路径

| 场景 | 默认选型 | 升级触发 → 升级到 |
| --- | --- | --- |
| 外部依赖 | 超时 + 自适应熔断 + 降级 | 多依赖 → 按组件特性差异化调参 |
| 限流 | 入口令牌桶 | 多租户 → 分租户配额 |
| 热路径 | SLA + 往返预算 | 超预算 → 异步化 / 预计算 |
| 连接池 | 经验基线（§9.2） | 实测压测后按 QPS / 时延调 |
| 凭据 | 加密入库 + 密钥留代码 | 合规要求 → 外置 KMS |

---

## 5. 权衡矩阵

| 手段 | 保护力 | 恢复平滑 | 复杂度 | 适用 |
| --- | --- | --- | --- | --- |
| 二态熔断 | 中 | 差（恢复雪崩） | 低 | 不推荐 |
| SRE 概率丢弃 | 高 | 好 | 中 | 默认 |
| 限流 | 高 | — | 中 | 入口过载 |
| 差异化降级 | 高 | — | 中 | 多依赖（只读依赖返空 / 关键依赖拒绝 / best-effort 依赖静默） |

---

## 6. 反模式速查

| 反模式 | 正确方向（详见 docs-spec/12 / 22 / 23 / 13 / 26） |
| --- | --- |
| 二态开关（恢复瞬间雪崩） | SRE 概率丢弃自适应 |
| 调外部依赖无超时 / 无熔断 | 超时 + 熔断 + 差异化降级 |
| 熔断全局一刀切 | 按集群 / 实例独立命名隔离 + 按组件分级 |
| service / data 层打日志 | 数据层禁日志，error 上传由 controller 记 |
| 热路径无 SLA / 无直方图 | 标热路径 + 直方图盯 SLO |
| 危险开关只靠配置 | 配置档位 + 运行时 RUN_ENV 双门禁 |
| 解密密钥与密文一起入配置 | 钥匙锁分离（密钥留代码） |
| 无界队列 / 无背压 | 有界队列 + 背压（22 §6.1） |

---

## 7. 产出与落地

结论落到 **`docs/circuit_breaker/circuit_breaker_design.md`**（熔断 / 降级 / 限流，docs-spec/12）与 **`docs/architecture/performance_contract.md`**（SLA / 热路径 / 预算 / 背压，docs-spec/22）。连接池基线落 `infrastructure.md`（docs-spec/04 §3）；配置 / 凭据落 `config.md §6-§9`（docs-spec/26）；可观测性落 `observability.md`（docs-spec/23）；分层 / 数据层禁日志落 `CLAUDE.md`（docs-spec/03）+ `logging.md`（docs-spec/13）。往返预算与 [`01`](01_io_design.md) 共用。

---

## 8. 与既有规范的边界（双源真相规避）

| 内容 | 06（本篇） | docs-spec/12 | docs-spec/22 | docs-spec/26 / 23 / 13 | design-spec/01 |
| --- | --- | --- | --- | --- | --- |
| 稳定性风险判定 / 概率丢弃选型 / 分层与禁日志 / 双门禁取舍 | ✅ 真相源 | 不写 | 不写 | 不写 | 不写 |
| 熔断算法 / 差异化参数 / 接入方式 | 引用 | ✅ 真相源 | 不写 | 不写 | 不写 |
| 全局 SLA / 热路径 / 内存预算 / 背压 | 引用 | 不写 | ✅ 真相源 | 不写 | 引用 |
| 配置中心 / 凭据加密 / Metrics / 日志规范 | 引用 | 不写 | 不写 | ✅ 真相源 | 不写 |
| IO 往返预算 / 聚合并行 | 引用 | 不写 | 引用 | 不写 | ✅ 真相源 |

---

## 9. 参考示例（仅示意，落地按业务替换）

> ⚠️ 以下为示意：规范本体（§1-§8）已用占位符表达；本节数值 / 代码为某真实工程实现，**只借纪律不借实现**。本节服从 [`PRINCIPLES.md §12`](../PRINCIPLES.md#12-占位符规则强制--双轨结构) 参考示例段豁免。

### 9.1 自适应熔断按组件分级（系统 A/B）

```
K = 1/percent；丢弃概率 p = max(0, (total - K×accepts)/(total+1))   // 恢复时 p 自然归零
```

| 组件 | request | percent | window | 定位 |
| --- | --- | --- | --- | --- |
| Redis（缓存） | 200 | 0.96 | 5s | 最激进（正常 >99.9%，掉 96% 即雪崩前兆） |
| MySQL | 50 | 0.92 | 10s | 平衡（容忍偶发慢查询） |
| ES / 慢查询 | 30 | 0.9 | 15s | 最宽容（慢是常态，长窗口防震荡） |
| 外部 API | 100 | 0.92 | 10s | 外部依赖保护 |

三种零侵入接入：Redis 以"连接指针身份"挂注册表懒加载（数据层零改动）；MySQL 实现 `gorm.Plugin` 给六类操作注册 before/after（`before` 调 `Allow()` 拒绝则 `AddError`，`ErrRecordNotFound` 视为成功）；外部 API 显式 `cb.Do(func() error{...})` 包裹 + 差异化降级。

**客户端自适应限流（client-side throttling）+ K 值权衡**：本算法是 Google SRE 的客户端节流——调用方在**本地**按 `requests`（自己发起总数）与 `accepts`（被后端接受数）之比自行丢弃，**丢在请求发出之前**。两层收益：① 后端 Open 时仍放一小撮流量探活（非二态全断），后端 `accepts` 一回升，`K×accepts` 放大快于 `requests` → `p` 自然归零、平滑恢复；② 注定失败的调用不再空耗本端连接 / 超时预算。**K 决定激进度**：节流仅在实际失败率超过 `1−1/K` 时启动（即 `accepts/requests < 1/K`）——

| K | 节流启动阈值（失败率 >） | 取向 |
| --- | --- | --- |
| 1.1 | ~9% | 最激进：偶发失败即开始丢（内部强依赖如 Redis） |
| 1.5 | ~33% | 平衡 |
| 2.0 | 50% | 最宽容：过半失败才启动（慢是常态的外部依赖） |

> 此处概率丢弃**作用于依赖侧**（调用方对某下游自我节流）；与 [`07 §3.4`](07_tail_latency_design.md) 作用于**入口侧**（系统自身过载时按优先级丢入站请求）的 load shedding 公式同构、位置不同——前者治"依赖故障别拖垮自己"，后者治"自身过载有序丢"。二者叠加而非替代。

### 9.2 连接池配置基线

| 资源 | 关键参数 |
| --- | --- |
| MySQL（每库） | maxIdle/maxOpen 各 100、connMaxLifetime 3600s、connTimeout 1500ms、读写超时各 3s、`interpolateParams=true`（省 prepared statement 往返） |
| Redis | maxIdle/maxActive 各 100、connTimeout 300ms、readTimeout 300ms、writeTimeout 600ms（压缩大 value 稍慢）、maxConnLifetime 10m |
| 外部 API | timeout 300ms、maxIdleConns 100、idleConnTimeout 300s、retry 1 |

写超时 > 读超时（压缩 / 大包）；外部调用短超时 + 重试。经验基线，非金科玉律。

### 9.3 数据层禁日志的理由

同一 Repo 方法被 HTTP / cron / job 多入口复用——自打日志会重复打点 + 级别误判（"未找到"在 DB 层是 `ErrRecordNotFound`，在 controller 层才是正常业务 201）。禁日志让数据层成为**可无副作用复用的纯取数单元**，支撑 IO 编排在上层统一并行 / 批量。


---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
