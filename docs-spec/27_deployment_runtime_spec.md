# 27 - 部署与运行时生命周期规范

> 规定 `docs/architecture/deployment.md` 中"**部署产物蓝图 + 容器镜像约束 + 健康探针语义 + 启动就绪时序 + 优雅关闭时序 + 发布回滚策略**"的内容结构与维护规则。
>
> 本规范是 AIWeave 的 P0 规范。它补齐第一性目标的最后一块短板——**"仅凭 docs/ + .claude/skills/ 重建整个工程"必须包含"重建出的工程能被部署、能优雅启停"**，否则重建产物虽行为等价却无法上线。

---

## 1. 定位

`deployment.md` 是 **AI 在生成任何"进程入口 / 信号处理 / 健康探针 / 资源关闭 / 部署产物"代码之前必须读的文档**。它回答五个问题：

> 1. 重建工程需要哪些部署产物（容器镜像定义 / 编排清单 / 流水线），各自必须包含什么？
> 2. 对外服务暴露哪些健康探针，liveness / readiness / startup 各自检查什么、返回什么？
> 3. 进程启动到"可接入流量"经过哪些步骤、什么顺序（与配置拉取、资源初始化的衔接）？
> 4. 收到终止信号后如何优雅关闭——排空在途请求、停掉常驻 goroutine、安全提交位点、释放连接？
> 5. 新版本如何发布（滚动 / 蓝绿 / 金丝雀）、出问题如何回滚？

没有这篇文档：

- AI 生成的进程**不处理终止信号**，发布滚动时正在处理的请求被直接掐断（数据半写、客户端 5xx）
- AI 把 **liveness 探针做成深度依赖检查**（探活里查 DB / 缓存），下游一抖动就被编排系统误杀重启，放大故障
- AI 生成的常驻 goroutine / 消费者**在关闭时泄漏**（位点不提交、在途消息丢失）
- 重建出的工程**没有镜像定义 / 编排清单**，无法部署——第一性目标在"上线"这一步断裂

---

## 2. 与相邻规范的边界划分（强制 / 双源真相规避）

| 内容 | 真相源 | 本规范（27） |
| --- | --- | --- |
| 配置拉取 / 凭据解密 / 启动 fail-fast | [`26`](26_config_center_spec.md) config.md §6-§9 | **引用**：把 26 的"配置就绪"作为启动时序的前置节点（§4） |
| 常驻 goroutine 生命周期登记 / cancel 路径 | [`20`](20_concurrency_safety_spec.md) concurrency_safety.md §1.1 / §5.2 | **引用**：优雅关闭必须 drain §1.1 登记的**每一个**常驻 goroutine（§5） |
| 代码"行为变更"灰度（shadow / 双写 / 1%→100%） | [`18`](18_mvp_rebuild_path_spec.md) mvp_rebuild_path.md §11.5.4 | **互补**：18 切"流量行为比例"，27 切"实例版本"（滚动 / 金丝雀），两者正交（§6） |
| 探针对应的 health 指标 | [`23`](23_observability_spec.md) observability.md | **引用**：27 定义探针**语义**，23 定义其 Metric；探针状态变化产生的指标归 23 |
| 下游超时 / 熔断 / 故障传播 | [`24`](24_cross_service_contract_spec.md) cross_service_contract.md | **引用**：readiness 的依赖检查清单取自 24 的下游依赖图 |
| 部署**操作手册**（值班 / 发布步骤清单 / 应急预案） | `runbook/` 或 `ops/`（docs/ 之外，见 [`00 §5`](00_directory_layout.md) / [`03 §3.2`](03_claude_md_spec.md)） | **不写**：27 是"部署**设计契约**"（产物含什么、时序是什么），不是"怎么操作发布" |

> **核心边界**：**部署设计契约入 docs/（本规范治理），部署操作手册入 runbook/。** 二者的判定标准——"能否服务于重建工程"：镜像必须含哪些层、探针必须返回什么、关闭必须排空什么 → 属蓝图，入 docs；"周五几点发布、谁审批、回滚找谁" → 属运维流程，入 runbook。本规范修正了 03 §3.2 / 00 §5"部署一律外置"的旧表述（见 §11 与 03/00 的衔接）。

---

## 3. 核心原则（强制）

| 原则 | 含义 |
| --- | --- |
| **可部署即蓝图一部分** | 重建工程必须重建出部署产物（镜像定义 + 编排清单 + 流水线骨架）；缺部署产物视为重建未完成 |
| **优雅关闭** | 收到终止信号后**先转不就绪、停止接入新请求**，再**排空在途**，最后释放资源；**禁止**收到信号即 `os.Exit` / 硬退出 |
| **探针分层** | liveness（活着吗）只做**进程级自检**，不查外部依赖；readiness（能接流量吗）做**依赖就绪检查**；startup（启动完成了吗）保护慢启动。三者职责不可混 |
| **启动有序就绪** | 进程必须在"配置就绪（26）→ 资源初始化 → 自检通过"后才置 readiness=true；**禁止**未就绪即对外暴露 |
| **关闭有序排空** | 关闭顺序 = 转不就绪 → 停止 accept → 排空在途（带超时）→ 停常驻 goroutine / 消费者（提交位点）→ 关连接池 → 退出；超时强杀兜底 |
| **发布可回滚** | 任何发布策略都必须保留**即时回滚**路径；版本不可变（镜像按内容寻址，不可原地覆盖 tag） |

---

## 4. 顶层结构（deployment.md 章节）

`deployment.md` 强制包含以下章节（启用本规范时）：

- `## 1.` 部署产物蓝图（镜像定义 / 编排清单 / CI-CD 流水线，各自必含项清单）
- `## 2.` 容器镜像约束（多阶段构建 / 非 root / 最小基础镜像 / 镜像内健康检查指令）
- `## 3.` 健康探针语义（liveness / readiness / startup 的检查内容、返回、超时、失败阈值）
- `## 4.` 启动就绪时序（衔接 26 配置就绪 → 资源初始化 → readiness 置位）
- `## 5.` 优雅关闭时序（信号 → 排空 → 停 goroutine → 提交位点 → 关连接，强制 ASCII 时序图）
- `## 6.` 发布与回滚策略（滚动 / 蓝绿 / 金丝雀 / 即时回滚，与 18 §11.5.4 边界）
- `## 7.` 资源配额与弹性（requests/limits / 副本数 / 水平弹性触发指标）

> **完整章节骨架见** [`templates/docs/architecture/deployment.md`](../templates/docs/architecture/deployment.md)。

---

## 5. §1 部署产物蓝图（强制）

```markdown
### 1.1 产物清单
| 产物 | 落地路径（占位） | 必含项 |
|------|-----------------|--------|
| 容器镜像定义 | `{Dockerfile-path}` | 多阶段构建 / 非 root / ENTRYPOINT = `{binary}` `{run-mode}` |
| 编排清单 | `{deploy-manifest-path}` | 探针 3 件套 / requests+limits / 副本数 / 终止宽限期 |
| CI-CD 流水线 | `{pipeline-path}` | 构建 → 测试（含 -race / -bench 门禁）→ 镜像推送 → 部署 → 回滚钩子 |

### 1.2 重建约束
- 重建工程时，§1.1 每个产物都必须生成（缺一视为重建未完成）。
- 产物中所有镜像引用**按内容寻址**（digest / 不可变 tag），禁止 `latest` 类可变 tag。
```

---

## 6. §2 容器镜像约束（强制）

```markdown
| 约束 | 要求 | grep 锚 |
|------|------|---------|
| 多阶段构建 | 构建层与运行层分离，运行层不含编译器 / 源码 | — |
| 非 root 运行 | 运行层声明非 root 用户；禁止以 root 启动进程 | `R-DEPLOY-ROOT` |
| 最小基础镜像 | 运行层用最小镜像（distroless / 静态 + scratch 等） | — |
| 镜像内健康指令 | 暴露探针端点供编排系统调用 | `R-PROBE-MISSING` |
| 终止宽限期 | 编排清单声明 > §5 排空超时 的终止宽限，避免排空未完即被强杀 | — |
```

---

## 7. §3 健康探针语义（强制 / 分层）

```markdown
| 探针 | 问题 | 检查内容 | 失败动作 | 禁止 |
|------|------|---------|---------|------|
| liveness | 进程死锁/卡死了吗 | **仅进程级自检**（事件循环可响应、无死锁）；**不查外部依赖** | 重启实例 | 禁止在 liveness 内查 DB/缓存/下游（依赖抖动→误杀重启→放大故障）`R-PROBE-LIVENESS-DEEP` |
| readiness | 现在能接流量吗 | 配置已就绪（26）+ 关键依赖可达（按 24 依赖图）+ 未处于关闭中 | 摘除流量（不重启） | 关闭中必须立即转 false |
| startup | 慢启动完成了吗 | 启动时序（§4）是否走完 | 延后 liveness/readiness 判定 | 慢启动场景必须配置，避免 liveness 在启动期误杀 |
```

> **分层铁律**：liveness 与 readiness **职责不可互换**。把依赖检查放进 liveness 是最常见的放大故障反模式——下游短暂不可达本应只摘流量（readiness），却触发了实例重启（liveness），把局部抖动升级为滚动重启风暴。

---

## 8. §4 启动就绪时序（强制 ASCII）

```
进程启动
  │  解析运行模式（{cli} 子命令 → HTTP / 消费者 / 定时任务入口）
  ▼
配置就绪（详见 26 §9）：解密凭据 → 拉取落盘 → 解析 → 全局配置
  │  任一步失败 ──► ✗ fail-fast（进程退出，见 26）
  ▼
资源初始化（DB / 缓存 / MQ / 熔断器，按 infrastructure.md 时序）
  │  初始化失败 ──► ✗ fail-fast
  ▼
注册信号处理（SIGTERM / SIGINT → 触发 §5 优雅关闭）
  ▼
startup 探针通过 → 启动常驻 goroutine（按 concurrency_safety.md §1.1 登记的清单）
  ▼
预热（可选 / 尾敏感服务强烈推荐）：连接池预拨 MinIdle 条连接 + 缓存预热（05 §3.8）
  │  避开首请求的 TCP/TLS 握手与缓存 miss 长尾（决策见 design-spec/07 §3.5）
  ▼
readiness 置 true（此刻才开始 accept 外部流量；置 true 晚于预热）
```

**就绪铁律**：readiness 置 true **必须**晚于"配置就绪 + 资源初始化 + 信号处理注册（+ 预热，尾敏感服务）"。任何"未注册关闭信号就 accept 流量"或"资源未就绪就置 readiness"都违反本节。

---

## 9. §5 优雅关闭时序（强制 ASCII）

```
收到 SIGTERM / SIGINT
  │  readiness 立即转 false（编排系统据此摘除新流量）
  ▼
停止 accept 新请求（HTTP server 进入 Shutdown；消费者停止拉取新消息）
  │
  ▼
排空在途（drain）：等待在途请求 / 在途消息处理完
  │  带超时 {drain-timeout}；超时则进入强制阶段
  ▼
停止常驻 goroutine / 消费者：
  │  对 concurrency_safety.md §1.1 登记的【每一个】常驻 goroutine 触发 cancel（§5.2 路径）
  │  消费者：处理完当前批 → 安全提交位点（offset / ack）→ 退出
  ▼
释放资源：关闭连接池（DB / 缓存 / MQ）、flush 缓冲、关闭文件/日志
  ▼
进程退出（exit 0）
  ─────────────────────────────────────────────
  ⏱ 兜底：超过 {shutdown-deadline} 仍未退出 → 强制退出（记录未排空项）
```

**关闭铁律**：
1. **先转不就绪再排空**——顺序颠倒会导致摘流量前就停止处理，新请求落到正在关闭的实例。
2. **排空必须有超时**，超时进入强制阶段；**强制退出前必须记录未完成项**（在途数、未提交位点），不可静默丢弃。
3. **每个常驻 goroutine 都要被关闭覆盖**——与 20 §1.1 的登记清单**逐一对账**，未接入关闭信号的常驻 goroutine 即泄漏（`R-SHUTDOWN-LEAK-GOROUTINE`，与 `R-CONC-GOROUTINE-LEAK` 互补：20 管"是否登记 + 有无 cancel 路径"，27 管"进程关闭是否真的触发了它"）。
4. **消费者关闭必须先提交位点再退出**，否则重启后重复消费 / 丢消息（依赖 21 的幂等设计兜底重复，但不可主动制造丢失）。

---

## 10. §6 发布与回滚策略（强制）

```markdown
### 6.1 发布策略选型
| 策略 | 适用 | 要点 |
|------|------|------|
| 滚动 | 默认 / 无状态服务 | 分批替换；每批之间 readiness 通过才继续；最大不可用数受限 |
| 蓝绿 | 需整体切换 / 快速回滚 | 新版本整组就绪后切流量；旧组保留待回滚 |
| 金丝雀 | 高风险变更 | 先小比例实例承接，观测指标无异常再全量 |

### 6.2 与 18 §11.5.4 的边界
- **27（本节）切"实例版本"**：滚动 / 金丝雀替换的是**运行实例**，粒度是 Pod / 进程。
- **18 §11.5.4 切"流量行为"**：shadow read / 双写 / 1%→10%→100% 切换的是**代码路径行为**，同一版本内灰度。
- 两者正交：一次高风险上线常**先金丝雀实例（27）+ 再行为灰度（18）**。

### 6.3 回滚契约（强制）
- 任何策略都必须保留**即时回滚**到上一已知良好版本的路径。
- 镜像**版本不可变**（内容寻址）；回滚 = 指向旧 digest，不依赖重新构建。
- 含**不兼容 Schema 变更**的发布，回滚前必须确认 DB 已按 [`05 §8`](05_schema_design_spec.md) 走 expand-contract 软兼容（旧版本能读新 schema），否则回滚会破坏数据兼容。
```

---

## 11. §7 资源配额与弹性（强制）

```markdown
| 项 | 要求 | grep 锚 |
|----|------|---------|
| requests / limits | 编排清单必须声明 CPU / 内存 requests 与 limits | `R-DEPLOY-NO-LIMITS` |
| 副本数 | 声明最小副本（避免单点）；有状态/选主任务说明唯一性保证 | — |
| 水平弹性 | 弹性触发指标取自 23 observability 的服务级指标（如 CPU / QPS / 队列深度）；禁止用业务量纲指标做弹性 | — |
| 内存 limit 与预算对齐 | limit 与 [`22 §3`](22_performance_contract_spec.md) 内存预算一致，避免 OOMKilled | — |
| CPU limit ↔ `GOMAXPROCS` | `GOMAXPROCS` 必须对齐容器 CPU limit（automaxprocs 读 cgroup 配额）；默认=宿主核数 → 容器被限流时 P 数超配额 → 调度争用 + 节流抖动抬 P99 | `R-DEPLOY-GOMAXPROCS` |
| mem limit ↔ `GOMEMLIMIT` | `GOMEMLIMIT` 设为 mem limit 的 ~90%（软上限），GC 提前回收防 OOMKill；与 [`22 §3.4`](22_performance_contract_spec.md) 运行时旋钮一致 | `R-RUNTIME-MEMLIMIT` |
```

---

## 12. AI 行为约束（强制 / 对应 CLAUDE.md 规则 19）

```markdown
### 部署与运行时生命周期约束（强制）

- 新增 / 修改进程入口（main / 子命令）→ 必须注册 SIGTERM/SIGINT 信号并接入 §5 优雅关闭；缺失 → `R-SHUTDOWN-NO-SIGNAL`
- 优雅关闭必须：先转 readiness=false → 停 accept → 排空（带超时）→ 停 goroutine/消费者（提交位点）→ 关连接池；直接 `os.Exit` / 无排空 → `R-SHUTDOWN-NO-DRAIN`
- 新增常驻 goroutine / 消费者 → 必须同时在 concurrency_safety.md §1.1 登记**且**接入 §5 关闭排空；只登记不排空 → `R-SHUTDOWN-LEAK-GOROUTINE`
- liveness 探针**禁止**查外部依赖（DB/缓存/下游）；依赖检查只放 readiness → `R-PROBE-LIVENESS-DEEP`
- 对外服务**必须**暴露 liveness + readiness（慢启动加 startup）探针端点；缺失 → `R-PROBE-MISSING`
- readiness 置 true 必须晚于"配置就绪（26）+ 资源初始化 + 信号注册"；未就绪即 accept → 拒绝
- 容器镜像运行层**禁止** root 用户 → `R-DEPLOY-ROOT`；编排清单**禁止**缺 requests/limits → `R-DEPLOY-NO-LIMITS`
- 镜像引用**禁止**可变 tag（`latest`）；必须内容寻址（digest / 不可变 tag）
- 部署进容器 → `GOMAXPROCS` 必须对齐 CPU limit（automaxprocs）、`GOMEMLIMIT` 对齐 mem limit（~90%）；缺失 → `R-DEPLOY-GOMAXPROCS` / `R-RUNTIME-MEMLIMIT`
- 尾敏感 / 高扇出服务 → 启动时序应含连接池预热（MinIdle 预拨）+ 缓存预热，readiness 晚于预热（缺失则头部 P99 飙高，决策见 design-spec/07）
- 含不兼容 Schema 变更的发布 → 回滚前必须确认 05 §8 软兼容已就位
- 拿不准关闭是否覆盖了全部常驻 goroutine → 主动与 20 §1.1 清单对账，不擅自略过
```

---

## 13. 禁止清单（强制 / grep 锚）

| 禁止 | 风险 | grep 锚 |
| --- | --- | --- |
| 进程入口未注册终止信号 / 收到信号即硬退出 | 发布即掐断在途请求 | `R-SHUTDOWN-NO-SIGNAL` |
| 优雅关闭缺在途排空（直接 `os.Exit`） | 在途请求 / 消息丢失 | `R-SHUTDOWN-NO-DRAIN` |
| 常驻 goroutine / 消费者未接入关闭排空 | goroutine 泄漏 / 位点不提交 | `R-SHUTDOWN-LEAK-GOROUTINE` |
| liveness 探针内做依赖检查（DB/缓存/下游） | 依赖抖动→误杀重启→放大故障 | `R-PROBE-LIVENESS-DEEP` |
| 对外服务缺 readiness / liveness 端点 | 编排无法判定就绪/存活，流量打到未就绪实例 | `R-PROBE-MISSING` |
| 容器以 root 运行 | 提权风险 | `R-DEPLOY-ROOT` |
| 编排清单缺 requests/limits | 资源争抢 / OOMKilled / 调度不可控 | `R-DEPLOY-NO-LIMITS` |
| `GOMAXPROCS` 未对齐容器 CPU limit（用默认宿主核数） | 调度争用 + 节流抖动抬 P99 | `R-DEPLOY-GOMAXPROCS` |
| `GOMEMLIMIT` 未设 / 未对齐 mem limit | GC 太晚 → OOMKilled | `R-RUNTIME-MEMLIMIT` |

> grep 锚定位为"信号级"非"判定级"。命中标 🟡 待复核，最终判定权在人工 reviewer。误报通过行末 `// aiweave:allow=<rule-id>` 注解抑制（每文件 ≤ 3 处）。完整锚定义见 `ai_dev_guide.md §10`；由 `/doc-sync-check`（折叠 `R-SHUTDOWN-*` / `R-PROBE-*` / `R-DEPLOY-*` / `R-RUNTIME-*`，参照 26 的 `R-CONF-*` 模式）+ L0 Hooks 双层兜底。

---

## 14. §维护流程（含 B1 反向同步规则）

### 14.1 B1 反向同步规则（强制）

| 代码迹象（git diff） | 反向同步动作 |
| --- | --- |
| 修改进程入口 / 新增 `signal.Notify` / `Shutdown` | §4 启动 + §5 关闭时序图同步 |
| 新增常驻 goroutine / 消费者 | §5 排空清单 +1，并与 concurrency_safety.md §1.1 对账 |
| 新增 / 修改探针 handler（`/live` `/ready` 等） | §3 探针语义表同步；liveness 误加依赖检查 → 拒绝（`R-PROBE-LIVENESS-DEEP`） |
| 修改 `{Dockerfile-path}` / 编排清单 | §1 产物清单 + §2 镜像约束 + §7 配额同步 |
| 修改发布策略 / 流水线 | §6 发布回滚策略同步 |
| 编排清单改 CPU / mem limit | §7 校对 `GOMAXPROCS` / `GOMEMLIMIT` 对齐；与 22 §3.4 对账 |
| 新增连接池预热 / MinIdle 预拨 | §4 启动时序预热节点同步 |
| 出现 root 运行 / 缺 limits / 可变 tag | 命中禁止清单 → 拒绝合并 |

### 14.2 维护触发

| 触发 | 动作 |
|------|------|
| 新增运行模式（子命令） | §4 启动时序补该模式入口 + §5 该模式关闭路径 |
| 调整排空 / 关闭超时 | §5 时序图数值同步 + 编排清单终止宽限期对齐（必须 > 排空超时） |
| 接入新下游依赖 | §3 readiness 依赖检查清单同步（取自 24 依赖图） |

---

## 15. 与其他文档的关系

| 内容 | deployment.md（27） | config.md（26） | concurrency_safety.md（20） | mvp_rebuild_path.md（18） | observability.md（23） |
|------|---------------------|------------------|-----------------------------|--------------------------|------------------------|
| 部署产物 / 镜像 / 探针语义 | ✅ 真相源 | 不写 | 不写 | 不写 | 不写 |
| 启动配置就绪 | 引用（§4 节点） | ✅ 真相源（§9） | 不写 | 不写 | 不写 |
| 常驻 goroutine 登记 / cancel | 引用（§5 排空对账） | 不写 | ✅ 真相源（§1.1/§5.2） | 不写 | 不写 |
| 流量行为灰度 | 互补（§6.2 边界） | 不写 | 不写 | ✅ 真相源（§11.5.4） | 不写 |
| 探针状态指标 | 引用（§3） | 不写 | 不写 | 不写 | ✅ 真相源 |

---

## 16. 与 Skill 的联动

- **`/doc-sync-check`**：扫描进程入口 / 探针 handler / 镜像与编排清单（`R-SHUTDOWN-*` / `R-PROBE-*` / `R-DEPLOY-*`），核对 §5 排空清单与 concurrency_safety.md §1.1 一致（折叠审计，参照 26 的 `R-CONF-*`，不另设审计 Skill）
- **`/concurrency-review`**：交叉复核——§5 优雅关闭排空的 goroutine 清单与 20 §1.1 登记清单对账
- **`/rebuild-from-docs`**：重建工程时，§1 部署产物（镜像定义 / 编排清单 / 流水线）必须一并重建，否则重建未完成
- **`/new-scheduled-task`** / **`/new-mq-consumer`**：生成常驻 goroutine / 消费者时，提示同步 §5 关闭排空路径

---

## 17. 参考示例（仅示意，落地按业务替换）

> ⚠️ 以下为示意，规范本体（§1-§16）已用占位符表达；落地工程时按业务语义替换占位符 / 技术栈，不得直接照搬本节具体命令、端口、镜像名作为规范。
>
> 示例服从 [`PRINCIPLES.md §12 占位符规则`](../PRINCIPLES.md#12-占位符规则强制--双轨结构) 的"参考示例段豁免"。

### 17.1 §5 优雅关闭（示例：Go HTTP + Kafka 消费者）

```
收到 SIGTERM
  → readiness handler 置 false（k8s 摘流量）
  → srv.Shutdown(ctx)  // 停 accept，等在途 HTTP 完成（ctx 超时 25s）
  → cancel()           // 触发所有常驻 goroutine 的 <-ctx.Done()
  → consumer 处理完当前批 → commit offset → 退出
  → db.Close() / redis.Close() / producer.Close()
  → exit 0
  ⏱ 超过 30s 仍未退出 → os.Exit(1) 并日志记录未排空数
```

terminationGracePeriodSeconds 设 35s（> 关闭 deadline 30s），避免排空未完即被 SIGKILL。

### 17.2 §3 探针（示例）

```
GET /live   → 仅检查事件循环可响应，恒返回 200（除非进程死锁）
GET /ready  → 检查 config 已加载 + DB Ping 可达 + 未处于 shutting-down；任一不满足返回 503
GET /startup→ 启动时序走完返回 200，期间 503（保护慢启动期不被 liveness 误杀）
```

### 17.3 §2 镜像（示例：多阶段 + 非 root + distroless）

```
# build stage：golang 镜像编译出静态 binary
# run stage：gcr.io/distroless/static，USER nonroot，ENTRYPOINT ["/app","http"]
# 镜像按 digest 引用：registry/app@sha256:...（禁止 :latest）
```


---

> 📝 **作者** XuRuibo <hustxurb@163.com> · `SPDX-FileCopyrightText: 2026 XuRuibo` · `SPDX-License-Identifier: Apache-2.0`
