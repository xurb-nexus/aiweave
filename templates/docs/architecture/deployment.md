# 部署与运行时生命周期

> 项目启动时复制本骨架到 `docs/architecture/deployment.md`，按 [`aiweave/docs-spec/27_deployment_runtime_spec.md`](../../../docs-spec/27_deployment_runtime_spec.md) 规范填充。
>
> 版本：1.0 | 更新日期：{YYYY-MM-DD}
>
> 本文是"部署**设计契约**"（产物含什么 / 时序是什么）；部署**操作手册**（发布步骤 / 值班 / 应急）放 `runbook/`，不写在此。

---

## 1. 部署产物蓝图

### 1.1 产物清单

| 产物 | 落地路径 | 必含项 |
|------|---------|--------|
| 容器镜像定义 | `{Dockerfile-path}` | 多阶段构建 / 非 root / ENTRYPOINT = `{binary}` `{run-mode}` |
| 编排清单 | `{deploy-manifest-path}` | 探针 3 件套 / requests+limits / 副本数 / 终止宽限期 |
| CI-CD 流水线 | `{pipeline-path}` | 构建 → 测试（-race / -bench 门禁）→ 镜像推送 → 部署 → 回滚钩子 |

### 1.2 重建约束

- 重建工程时，§1.1 每个产物都必须生成（缺一视为重建未完成）。
- 镜像引用按内容寻址（digest / 不可变 tag），禁止 `latest` 类可变 tag。

---

## 2. 容器镜像约束

| 约束 | 本工程取值 | grep 锚 |
|------|-----------|---------|
| 多阶段构建 | {是/否；构建层 `{build-base}` → 运行层 `{run-base}`} | — |
| 非 root 运行 | {运行用户 `{nonroot-user}`} | `R-DEPLOY-ROOT` |
| 最小基础镜像 | {`{run-base}`} | — |
| 镜像内健康指令 | {探针端点 `{live-path}` / `{ready-path}`} | `R-PROBE-MISSING` |
| 终止宽限期 | {`{grace-period}` s，必须 > §5 排空超时} | — |

---

## 3. 健康探针语义

| 探针 | 端点 | 检查内容 | 超时 / 失败阈值 | 失败动作 |
|------|------|---------|----------------|---------|
| liveness | `{live-path}` | 仅进程级自检（**禁止**查外部依赖） | {T} / {N} | 重启实例 |
| readiness | `{ready-path}` | 配置就绪 + 关键依赖可达（按 cross_service_contract.md 依赖图）+ 未关闭中 | {T} / {N} | 摘流量（不重启） |
| startup | `{startup-path}` | 启动时序（§4）走完 | {T} / {N} | 延后 live/ready 判定 |

> 分层铁律：liveness 与 readiness 职责不可互换。依赖检查只放 readiness。

依赖检查清单（readiness 检查哪些依赖，取自 `cross_service_contract.md` 下游依赖图）：

- {dependency-1}（{检查方式：Ping / 轻量探测}）
- {dependency-2}

---

## 4. 启动就绪时序

```
进程启动
  → 解析运行模式（{cli} 子命令）
  → 配置就绪（详见 config.md §9：解密 → 拉取落盘 → 解析）  ── 失败即 fail-fast
  → 资源初始化（{DB} / {缓存} / {MQ} / 熔断器，按 infrastructure.md）  ── 失败即 fail-fast
  → 注册信号处理（SIGTERM/SIGINT → §5 优雅关闭）
  → startup 通过 → 启动常驻 goroutine（按 concurrency_safety.md §1.1 清单）
  → 预热（可选 / 尾敏感服务强烈推荐）：连接池预拨 MinIdle 条连接 + 缓存预热  ── 避开首请求握手与缓存 miss 长尾（决策见 design-spec/07 §3.5）
  → readiness 置 true（此刻开始 accept 流量；置 true 晚于预热）
```

就绪铁律：readiness 置 true 必须晚于"配置就绪 + 资源初始化 + 信号注册（+ 预热，尾敏感服务）"。

---

## 5. 优雅关闭时序

```
收到 SIGTERM / SIGINT
  → readiness 立即转 false（编排摘新流量）
  → 停止 accept（HTTP Shutdown / 消费者停止拉取）
  → 排空在途（drain，超时 {drain-timeout}）
  → 停常驻 goroutine：对 concurrency_safety.md §1.1 登记的【每一个】触发 cancel
  → 消费者：处理完当前批 → 提交位点 → 退出
  → 释放资源：关连接池 / flush 缓冲 / 关日志
  → exit 0
  ⏱ 兜底：超过 {shutdown-deadline} 强制退出，记录未排空项
```

### 5.1 关闭排空清单（与 concurrency_safety.md §1.1 逐一对账）

| 常驻 goroutine / 消费者 | cancel 路径 | 关闭动作 | 位点提交 |
|------------------------|------------|---------|---------|
| {resident-goroutine-1} | {cancel-path} | {drain 动作} | {N/A 或 commit offset} |
| {consumer-1} | {cancel-path} | 处理完当前批 | commit offset |

> 关闭铁律：先转不就绪再排空；排空必须有超时；每个常驻 goroutine 都要被覆盖；消费者先提交位点再退出。

---

## 6. 发布与回滚策略

### 6.1 发布策略

| 服务 / 变更类型 | 策略 | 要点 |
|----------------|------|------|
| {无状态服务默认} | 滚动 | 分批；每批 readiness 通过才继续；最大不可用 {N} |
| {需快速回滚} | 蓝绿 | 新组就绪后切流量，旧组保留 |
| {高风险变更} | 金丝雀 | 小比例承接，观测无异常再全量 |

### 6.2 与行为灰度（mvp_rebuild_path.md §11.5.4）的边界

- 本文切"实例版本"（滚动 / 金丝雀，粒度 = 进程 / Pod）。
- §11.5.4 切"流量行为"（shadow / 双写 / 1%→100%，同版本内灰度）。
- 两者正交，高风险上线常先金丝雀实例 + 再行为灰度。

### 6.3 回滚契约

- 保留即时回滚到上一已知良好版本的路径。
- 镜像版本不可变；回滚 = 指向旧 digest，不重新构建。
- 含不兼容 Schema 变更的发布，回滚前确认 `database_design.md §8` 已 expand-contract 软兼容。

---

## 7. 资源配额与弹性

| 项 | 本工程取值 | grep 锚 |
|----|-----------|---------|
| requests / limits | CPU `{cpu-req}`/`{cpu-lim}`，内存 `{mem-req}`/`{mem-lim}` | `R-DEPLOY-NO-LIMITS` |
| 副本数 | 最小 `{min-replicas}`；{有状态/选主唯一性说明} | — |
| 水平弹性 | 触发指标 `{scale-metric}`（取自 observability.md 服务级指标） | — |
| 内存 limit 对齐 | 与 `performance_contract.md §3` 内存预算一致 | — |
| CPU limit ↔ `GOMAXPROCS` | `GOMAXPROCS` 对齐容器 CPU limit（automaxprocs 读 cgroup 配额）；默认=宿主核数 → 被限流时 P 数超配额 → 调度争用 + 节流抖动抬 P99 | `R-DEPLOY-GOMAXPROCS` |
| mem limit ↔ `GOMEMLIMIT` | `GOMEMLIMIT` 设为 mem limit 的 ~90%（软上限），GC 提前回收防 OOMKill；与 `performance_contract.md §3.4` 运行时旋钮一致 | `R-RUNTIME-MEMLIMIT` |

---

## 变更日志

| 日期 | 变更 | 触发事件 |
|------|------|---------|
| {YYYY-MM-DD} | 初版 | 项目启动 |


---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
