# nami-go

[English](README.md) | 中文

一个轻量级的 Go HTTP RPC 客户端框架，移植自 Java [Solon](https://solon.noear.org/) Nami。

提供以下能力：

- **Channel**（通道）传输抽象（内置基于 `net/http` 的 HTTP 实现）。
- **Encoder / Decoder**（编解码器）序列化抽象（内置基于 `encoding/json` 的 JSON 实现）。
- **Filter**（过滤器）请求拦截链。
- **Upstream / Discovery**（上游 / 服务发现）集成。
- 流式 API 的 `Nami` 客户端、`Builder` 构建器，以及按路径生成客户端的 `ClientFactory`。
- 一个 `util` 包，提供最简单的基于 URL 的请求封装。

## 安装

```bash
go get github.com/crazy-airhead/nami-go
```

## 用 `util` 快速上手

无需配置，导入即用。`util` 会把相对路径解析到已设置的 base URL 上。

```go
import "github.com/crazy-airhead/nami-go/util"

util.SetBaseURL("http://api.example.com")
util.SetTimeout(10) // 每个请求的超时（秒，0 = 用通道默认值）

// 相对路径解析到 base URL 上
body, _ := util.Get("/users")
users, _ := util.GetJSON[[]User]("/users")
result, _ := util.PostJSON[Order]("/orders", newOrder)

// 完整 URL 原样使用，不走 base URL
ip, _ := util.GetJSON[IPInfo]("https://api.ipify.org?format=json")
```

## 与 `net/http` 的对比

`nami-go` 是 `net/http` 之上的一层薄封装——HTTP 通道最终使用的就是 Go 标准库客户端。
区别在于**易用性和抽象层次**，而非另起炉灶的传输实现。

**同样的「GET + 解析 JSON + 非 2xx 视为错误」：**

```go
// net/http
resp, err := http.Get(url)
if err != nil {
	return err
}
defer resp.Body.Close()
if resp.StatusCode >= 400 {
	return fmt.Errorf("status %d", resp.StatusCode)
}
var users []User
if err := json.NewDecoder(resp.Body).Decode(&users); err != nil {
	return err
}

// nami-go util
users, err := util.GetJSON[[]User](url)
```

| 维度 | `net/http` | `nami-go` |
|---|---|---|
| 关闭 body、读取、错误胶水代码 | 每次手动处理 | 内置 |
| 非 2xx 视为错误 | 手动检查状态码 | 自动（`Get` / `Post` / `GetJSON` …） |
| JSON 解码为强类型值 | 手动 `json.Unmarshal` | `GetJSON[T]` / `PostJSON[T]` 泛型 |
| Base URL + 相对路径 | 每次手动拼接 | `util.SetBaseURL` 设置一次 |
| Query 参数 / 请求头 | `url.Values`、`req.Header.Set` | `map[string]string` |
| 请求拦截 | `http.RoundTripper`（传输层） | `Filter` 链 + `Channel` / `Encoder` / `Decoder` |
| 服务发现 | 自行实现 | `Upstream` / `Discovery` |
| 流式、自定义 `Transport` | 一等公民 | 受限（可向通道传入自定义 `*http.Client`） |
| 超时 / 取消 | `context.Context` + `Client.Timeout` | `util.SetTimeout` / `Config.SetTimeout`（秒）；不支持按请求传入 `context.Context` |
| 依赖 | 无（标准库） | 本模块 |

**何时选哪个**

- 日常的 JSON 请求/响应 API，追求更少样板代码和内置状态检查时，用 `util` / `nami` 客户端。
- 当你需要流式传输、自定义 `http.Transport`、按请求 `context.Context` 取消、或检查原始请求/响应时，回退到原生 `net/http`。
- HTTP 通道可通过 `http.NewWithClient(client)` 传入自定义客户端，这样既能调优底层传输（连接池、TLS、超时），又能保留 nami 的易用性。

## 核心 `nami` 客户端

```go
import (
	_ "github.com/crazy-airhead/nami-go/channel/http" // 注册 HTTP 通道
	_ "github.com/crazy-airhead/nami-go/coder/json"   // 注册 JSON 编解码器
	"github.com/crazy-airhead/nami-go/nami"
)

n := nami.New().
	URL("http://example.com/api/users").
	Action(nami.MethodPost)

n.Call(nil, nil, body)

var item Order
if err := n.GetObject(&item); err != nil {
	return err
}
```

### Builder 构建器

```go
import (
	_ "github.com/crazy-airhead/nami-go/channel/http" // 注册 HTTP 通道
	_ "github.com/crazy-airhead/nami-go/coder/json"   // 注册 JSON 编解码器
	"github.com/crazy-airhead/nami-go/nami"
)

n := nami.NewBuilder().
	Timeout(5).
	Upstream(nami.NewUpstreamFixed([]string{"http://localhost:8080"})).
	Name("user-service").
	Path("/api/users").
	Build()

// GET http://localhost:8080/api/users —— URL 由 Upstream + Path 解析得到
var users []User
if err := n.Action(nami.MethodGet).CallAndBind(nil, nil, nil, &users); err != nil {
	return err
}
```

### ClientFactory —— 一个服务，多条路径

```go
import (
	_ "github.com/crazy-airhead/nami-go/channel/http"
	_ "github.com/crazy-airhead/nami-go/coder/json"
	"github.com/crazy-airhead/nami-go/nami"
)

factory := nami.NewClientFactory().
	ServiceName("user-service").
	Upstream(nami.NewUpstreamFixed([]string{"http://localhost:8080"})).
	Timeout(10)

users  := factory.For("/api/v1/users")
orders := factory.For("/api/v1/orders")

// GET http://localhost:8080/api/v1/users
var list []User
if err := users.Action(nami.MethodGet).CallAndBind(nil, nil, nil, &list); err != nil {
	return err
}

// POST http://localhost:8080/api/v1/orders
var created Order
if err := orders.Action(nami.MethodPost).CallAndBind(nil, nil, newOrder, &created); err != nil {
	return err
}
```

`For` 返回的每个客户端彼此独立——修改其中一个不会影响其他客户端。

## 包一览

| 路径 | 用途 |
|---|---|
| `nami` | 核心框架：Channel、Encoder/Decoder、Filter、Upstream、Config、Nami 客户端 |
| `channel/http` | HTTP 传输通道 |
| `coder/json` | JSON 编解码器 |
| `util` | 最简单的基于 URL 的请求封装 |

## 许可证

[Apache-2.0](LICENSE)
