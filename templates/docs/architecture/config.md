# 配置体系

> 规范来源：`aiweave/docs-spec/04_architecture_overview_spec.md` §2（总体结构 §1-§5）
> + `aiweave/docs-spec/26_config_center_spec.md`（配置上云 / 凭据加密 §6-§9）

## 1. 配置文件分类

- **mount/**：环境相关，配置中心动态修改
- **app/**：业务相关，随代码发布

## 2. 配置文件清单

| 文件 | 内容范围 | 何时修改 |
|------|---------|---------|
| `conf/mount/config.yaml` | 服务端口、日志、accessLog、pprof | 环境差异 |
| `conf/mount/api.yaml` | 外部 API 调用配置 | 上线新外部依赖 |
| `conf/mount/resource.yaml` | MySQL / Redis / Kafka / MQ | 资源迁移 |
| `conf/app/app.yaml` | 熔断器参数等业务参数 | 调优 |

## 3. 加载流程

```
helpers.PreInit() → conf.InitConf() → conf.RConf / conf.AConf 全局变量
```

## 4. 占位机制

敏感信息使用 `@@key` 格式占位，运行时由配置中心替换。

## 5. Go 结构体映射

| yaml 节点 | Go struct |
|----------|----------|
| resource.mysql | `TMysql` |
| resource.redis | `TRedis` |
| resource.kafkaPub | `map[string]TKafkaPub` |
| ... | ... |

---

> 以下 §6-§9 规范来源：`aiweave/docs-spec/26_config_center_spec.md`（配置上云 / 凭据加密）。
> 启用"配置上云"时强制填写；纯本地配置工程可标 🚫 不启用。

## 6. 配置分层与环境模型

### 6.1 权威源矩阵

| 配置类别 | 权威源 | 修改方式 | 入 git |
|---------|--------|---------|--------|
| 业务配置（随代码发布） | 本地仓库 | 改代码 + 发版 | ✅ |
| 环境配置（端口 / 资源地址 / 外部依赖） | 配置中心（云端） | 配置中心控制台 | ❌（启动落盘副本） |
| 连接凭据（配置中心 / 资源账号） | 加密密文 + 应用层密钥 | 重新生成密文 | 密文可入库；密钥 ❌ |

### 6.2 环境维度

- 环境标记来源：{env-source}（如环境变量 / 环境标记文件）
- 配置标识按环境拼接：{dataId} + {group}-{env}

## 7. 配置中心集成

### 7.1 Client 抽象（唯一进行配置中心网络调用的地方，可替换）

```go
type Client interface {
    Get(dataId, group string) (string, error) // 按 dataId + group 读取原始文本
}
```

### 7.2 连接信息

| 字段 | 含义 | 安全要求 |
|------|------|---------|
| {Addr} | 配置中心地址 | — |
| {Namespace} | 命名空间 / 租户隔离 | — |
| {Username} | 连接账号 | 密文，运行期解密 |
| {Password} | 连接密码 | 密文，运行期解密 |
| {TimeoutMs} | 请求超时（0=实现默认） | 必须有上限 |

### 7.3 拉取 → 落盘映射

| dataId | group | 本地落盘路径 |
|--------|-------|------------|
| {dataId-A} | {group}-{env} | conf/mount/{file-A}.yaml |

编排（Sync）：逐条拉取 → 自动建父目录覆盖写 → 空内容视为失败 → 任一失败 fail-fast。

## 8. 凭据加密

- 算法：{对称加密，如 AES-256-GCM}，随机 nonce 前置 + Base64，带完整性校验。
- 密钥来源：应用层持有 32 字节密钥，框架库不内置；{密钥注入方式，如环境变量 `{KEY-ENV}`}；不进 git。
- 密文一次性生成（Encrypt），运行期只解密（Decrypt）。
- 轮换：换密钥 → 重新生成所有密文 → 替换配置文件密文。轮换记录：{rotation-log}。

```go
func Encrypt(key []byte, plain string) (string, error) // → Base64(nonce ‖ ciphertext)
func Decrypt(key []byte, token string) (string, error)
```

**禁止**：明文凭据落盘/入库（`R-CONF-PLAINTEXT-CRED`）；密钥硬编码（`R-CONF-HARDCODE-SECRET`）；密钥/明文凭据文件入 git（`R-CONF-SECRET-COMMIT`）；拉取失败未 fail-fast（`R-CONF-NO-FAILFAST`）。

## 9. 启动期加载时序

```
读 env 标记 → 解密凭据(Decrypt) → 构造连接信息 → 连接配置中心
  → Sync 拉取落盘 → 本地解析为全局配置变量 → 初始化资源
任一步失败 → fail-fast（进程退出），禁止静默空/旧配置启动
```

## 10. 危险开关双门禁登记

> 规范来源：`aiweave/docs-spec/26_config_center_spec.md §7.5`；决策方法论见 `aiweave/design-spec/06_resilience_design.md §3.3`（危险开关双门禁）。
> 高风险开关（跳缓存 / 强制降级 / 危险调试开关等）必须 **配置档位 + 运行档位 `RUN_ENV` 双门禁叠加**，两道门同时满足才放行，绝不可能在线上误生效。仅配置档位单门禁即违规（`R-CONF-DANGER-SINGLE-GATE`）。
> 本工程每个高风险开关在此登记一行；无危险开关可标 🚫 不涉及。

| 开关名 | 门一·配置档位条件 | 门二·运行档位 `RUN_ENV` 条件 | 落点（代码路径） |
|--------|------------------|------------------------------|------------------|
| {危险开关-A} | {配置中心档位 / 请求标记，如仅 {test-audience} 允许} | {RUN_ENV 硬校验，如仅非生产 env 生效} | {path/to/gate} |


---

> 🧩 **AIWeave 骨架 · 作者 XuRuibo** <hustxurb@163.com> · Apache-2.0 · 模板文件，复制到工程后按业务语义填充
