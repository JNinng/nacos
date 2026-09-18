# nacos

> **已封存**（2026-09-19）：组件并入 go_template 模板内置组件库
> `internal/components/nacos`（随模板分发，删留自便）。本仓库保留 v0.1.0 历史与离线测试，不再演进。

[go_template](https://github.com/JNinng/go_template) 体系的原生组件资产：nacos 双角色客户端（单仓库单包）。模板侧的接入示例与组件约定见其 [docs/DESIGN.md](https://github.com/JNinng/go_template/blob/master/docs/DESIGN.md)。

- **CfgClient**——配置中心客户端，Source 兼容签名（结构化类型满足，零 import 模板即可直传 `config.Tree.Attach`）
- **Reg**——服务注册客户端，标准生命周期签名（`New` / `Start` / `Stop`）

两者分立配置、独立客户端、互不共享连接；实例标识（service name / port）由装配点显式传入（service name 传模板元数据 `meta.Name`）。

## 安装

```
go get github.com/jninng/nacos
```

Go 版本要求：1.23+。被包装库：`github.com/nacos-group/nacos-sdk-go/v2`（gRPC 协议）。依赖 observ（日志契约）。

## 配置节（可粘贴）

```yaml
nacos:
  config:
    enabled: true
    unreachable: fail
    addr: "127.0.0.1:8848"
    namespace: ""
    group: DEFAULT_GROUP
    data_id: app.yaml
    username: ""
    password: ""
  registrar:
    enabled: false
    unreachable: fail
    addr: "127.0.0.1:8848"
    namespace: ""
    group: DEFAULT_GROUP
    service_ip: ""
    username: ""
    password: ""
```

## 字段表

### nacos.config（配置中心）

| 名称 | 类型 | 缺省 | 热更 | 说明 |
|---|---|---|---|---|
| enabled | bool | true | 否 | 角色开关；false 时推送空快照、热更停摆 |
| unreachable | string | fail | 否 | 不可达策略：fail = 启动报错；disable = 告警后纯本地配置继续 |
| addr | string | 127.0.0.1:8848 | 否 | 服务地址 host:port（gRPC 端口由 SDK 按偏移推导） |
| namespace | string | "" | 否 | 命名空间 ID |
| group | string | DEFAULT_GROUP | 否 | 配置分组 |
| data_id | string | app.yaml | 否 | 订阅的配置集（全量快照来源，yaml 格式） |
| username / password | string | "" | 否 | 凭据（与 registrar 分立） |

### nacos.registrar（服务注册）

| 名称 | 类型 | 缺省 | 热更 | 说明 |
|---|---|---|---|---|
| enabled | bool | false | 否 | 角色开关；启用时实例标识必填（装配点传参） |
| unreachable | string | fail | 否 | fail = 启动报错；disable = 告警后旁路运行（暂不被发现） |
| addr | string | 127.0.0.1:8848 | 否 | 服务地址 |
| namespace / group | string | "" / DEFAULT_GROUP | 否 | 注册分组 |
| service_ip | string | 自动探测 | 否 | 缺省探测本机出口 IP（UDP 拨号不实际发包） |
| username / password | string | "" | 否 | 凭据 |

**本资产全部字段均不热更**（nacos 节为启动期配置：连接参数无法热切换，不实现 `ApplyConfig`，变更仅下次启动生效）。nacos 节只能来自本地配置文件——读它时远程尚未连通。

## 生命周期 API

| API | 说明 |
|---|---|
| `Default() Config` | 默认值基座（与模板 `config.Decode` 成对使用） |
| `NewCfgClient(cfg Config, opts ...Option) (*CfgClient, error)` | 构造即校验（策略/地址），不连接；失败无资源需清理 |
| `(*CfgClient).Name() string` | 源名称 "nacos" |
| `(*CfgClient).Start(ctx, push func(map[string]any)) error` | Source 签名：连接 + 首份全量快照 + 订阅变更；阻塞至 ctx 取消。ctx 取消即停止 |
| `NewReg(cfg Config, serviceName string, port int, opts ...Option) (*Reg, error)` | 构造即校验；enabled=false 返回旁路实例（Start 空操作） |
| `(*Reg).Start(ctx) error` | 注册实例（Ephemeral，心跳由 SDK 维持） |
| `(*Reg).Stop(ctx) error` | 注销 + 释放连接；幂等可重入 |
| `WithLogger(observ.Logger) Option` | 显式注入日志面（保留给测试捕获；缺省动态读，见下） |

## 装配示例（模板侧）

```go
// internal/app/wire.go
func wireSource(t *config.Tree) error {
    cfg, err := config.Decode(t, "nacos", nacos.Default())
    if err != nil { return err }
    cc, err := nacos.NewCfgClient(cfg)
    if err != nil { return err }
    return t.Attach(cc) // CfgClient 以 Source 兼容签名直传，零适配胶水
}

func wire(t *config.Tree, r *runner, meta app.Meta) error {
    cfg, err := config.Decode(t, "nacos", nacos.Default())
    if err != nil { return err }
    reg, err := nacos.NewReg(cfg, meta.Name, 8080) // service name 传 meta.Name
    if err != nil { return err }
    r.Add("nacos-reg", reg.Start, reg.Stop)
    return nil
}
```

## 并发与回调纪律

- 回调（`push`）在 SDK 回调 goroutine 内同步执行且快速返回；不阻塞不重入
- 全量快照语义：每次变更推送整份 dataId 内容解析结果；空内容 = 空快照（删除语义）
- 变更解析失败：告警丢弃本次、维持上一有效快照（最终一致）
- **日志缺省动态读 `observ.DefaultLogger()`（非构造期快照）**：本资产可能在 `wireSource` 构造（早于日志装配），快照会把告警永久固定在 Noop
- **不可达策略降级告警走双通道**：observ 动态读 + 直写 stderr——降级发生在引导窗口时 observ 可能仍是 Noop，stderr 保证高信号运维事实永不丢失
- 日志消息/字段 snake_case；进程内无全局状态重赋值
