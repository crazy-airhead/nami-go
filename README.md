# nami-go

English | [中文](README_zh.md)

A lightweight HTTP RPC client framework for Go, ported from Java [Solon](https://solon.noear.org/) Nami.

It provides:

- **Channel** abstraction for transport (HTTP built-in via `net/http`).
- **Encoder / Decoder** for serialization (JSON built-in via `encoding/json`).
- **Filter** chain for request interception.
- **Upstream / Discovery** for service discovery.
- A fluent `Nami` client, a `Builder`, and a `ClientFactory` for per-path clients.
- A `util` package for the simplest possible URL-based requests.

## Install

```bash
go get github.com/crazy-airhead/nami-go
```

## Quick start with `util`

No setup required — import and call. `util` resolves relative paths against a base URL.

```go
import "github.com/crazy-airhead/nami-go/util"

util.SetBaseURL("http://api.example.com")
util.SetTimeout(10) // per-request timeout in seconds (0 = use channel default)

// Relative paths resolve against the base URL
body, _ := util.Get("/users")
users, _ := util.GetJSON[[]User]("/users")
result, _ := util.PostJSON[Order]("/orders", newOrder)

// Full URLs are used as-is and bypass the base URL
ip, _ := util.GetJSON[IPInfo]("https://api.ipify.org?format=json")
```

## Comparison with `net/http`

`nami-go` is a thin layer on top of `net/http` — the HTTP channel ultimately uses
Go's standard client. The difference is ergonomics and abstraction, not a
replacement transport.

**GET, parse JSON, treat non-2xx as error:**

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

| Aspect | `net/http` | `nami-go` |
|---|---|---|
| Body close + read + error glue | manual, every call | built in |
| Non-2xx treated as error | manual status check | automatic (`Get` / `Post` / `GetJSON` …) |
| JSON decode to typed value | manual `json.Unmarshal` | `GetJSON[T]` / `PostJSON[T]` generics |
| Base URL + relative paths | manual joining per call | `util.SetBaseURL` once |
| Query params / headers | `url.Values`, `req.Header.Set` | `map[string]string` |
| Request interception | `http.RoundTripper` (transport) | `Filter` chain + `Channel` / `Encoder` / `Decoder` |
| Service discovery | build it yourself | `Upstream` / `Discovery` |
| Streaming, custom `Transport` | first-class | limited (pass a custom `*http.Client` to the channel) |
| Timeout / cancellation | `context.Context` + `Client.Timeout` | `util.SetTimeout` / `Config.SetTimeout` (seconds); `context.Context` not wired per request |
| Dependencies | none (stdlib) | this module |

**When to use which**

- Use `util` / the `nami` client for everyday JSON request/response APIs where
  less boilerplate and built-in status checks win.
- Drop down to raw `net/http` when you need streaming, a custom `http.Transport`,
  per-request `context.Context` cancellation, or to inspect the raw request/response.
- The HTTP channel accepts a custom client via `http.NewWithClient(client)`, so you
  can tune the underlying transport (connection pooling, TLS, timeouts) and still
  keep the nami ergonomics.

## Core `nami` client

```go
import (
	_ "github.com/crazy-airhead/nami-go/channel/http" // registers the HTTP channel
	_ "github.com/crazy-airhead/nami-go/coder/json"   // registers the JSON codec
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

### Builder

```go
import (
	_ "github.com/crazy-airhead/nami-go/channel/http" // registers the HTTP channel
	_ "github.com/crazy-airhead/nami-go/coder/json"   // registers the JSON codec
	"github.com/crazy-airhead/nami-go/nami"
)

n := nami.NewBuilder().
	Timeout(5).
	Upstream(nami.NewUpstreamFixed([]string{"http://localhost:8080"})).
	Name("user-service").
	Path("/api/users").
	Build()

// GET http://localhost:8080/api/users — URL is resolved from Upstream + Path
var users []User
if err := n.Action(nami.MethodGet).CallAndBind(nil, nil, nil, &users); err != nil {
	return err
}
```

### ClientFactory — one service, many paths

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

Each client returned by `For` is independent — mutating one does not affect others.

## Packages

| Path | Purpose |
|---|---|
| `nami` | Core framework: Channel, Encoder/Decoder, Filter, Upstream, Config, Nami client |
| `channel/http` | HTTP transport channel |
| `coder/json` | JSON encoder/decoder |
| `util` | Simplest URL-based request helpers |

## License

[Apache-2.0](LICENSE)
