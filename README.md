# Service Discovery

Service Discovery 套件能夠讓你透過服務探索自動註冊服務，並且以負載平衡的方式來平均連線請求。

# 索引

* [安裝方式](#安裝方式)
* [使用方式](#使用方式)
    * [Consul](#Consul)
        * [服務註冊](#服務註冊)
        * [服務移除](#服務移除)
* [負載平衡](#負載平衡)
    * [進階自訂](#進階自訂)
        * [Consul](#Consul)

# 安裝方式

打開終端機並且透過 `go get` 安裝此套件即可。

```bash
$ go get github.com/go-mego/sd
```

# 使用方式

此套件有許多使用方式與支援不同的服務探索中心，下列是目前支援的列表。

* 簡易本地列表
* Consul
* Etcd（_尚未支援_）
* Eureka（_尚未支援_）
* ZooKeeper（_尚未支援_）

## Consul

Consul 是一個服務探索中心，伺服器可以在上線的時候自動向服務探索中心註冊自己的 IP 位置，接著負載平衡就能得知該伺服器並均攤請求負載。

### 服務註冊

透過 `Register` 函式並傳入伺服器資訊，就能將伺服器註冊到遠端的服務探索中心列表令其他服務發現自己並提供使用。

```go
package main

import (
	"github.com/go-mego/sd/consul"
	"github.com/go-mego/sd/version"
	"github.com/hashicorp/consul/api"
)

func main() {
	// 以預設設定檔建立 Consul 客戶端。
	c, _ := consul.NewClient(api.DefaultConfig())
	// 註冊服務到服務探索中心。
	c.Register(&api.AgentServiceRegistration{
		// 透過 `.ID()` 替這個服務產生一個獨立的 UUID 識別編號。
		ID: c.ID(),
        // 將服務的版本號帶入標籤中，所以我們就能在負載平衡透過標籤篩選服務的版本。
        // 請參閱 Version 套件，這會產生一個 `1.0.0+stable` 的字串。
		Tags: []string{version.Define(1, 0, 0, "stable").String()},
		Name: "Database",
		Port: 1234,
	})
}
```

### 服務移除

若服務即將下線，則可以透過 `Deregister` 將自己從遠端的服務探索中心移除來避免其他服務繼續使用自己。

```go
package main

import (
	"github.com/go-mego/sd/consul"
	"github.com/hashicorp/consul/api"
)

func main() {
	// 以預設設定檔建立 Consul 客戶端。
    c, _ := consul.NewClient(api.DefaultConfig())
	// 將指定編號的服務從服務探索中心裡移除。
	c.Deregister(&api.AgentServiceRegistration{
        // 如果先前有透過 `.Register()` 與 `.ID()` 註冊，
        // 那麼這次就可以透過 `.ID()` 取得上次產生的編號進而從列表中移除自己。
		ID: c.ID(),
	})
}
```

# 負載平衡

負載平衡可以讓你平均地配發請求到不同伺服器，你只需要提供伺服器的 IP 位置列表即可。但這種方式僅能使用固定的列表且不能移除伺服器。若需要有更彈性的使用方式，請參考下列與服務探索中心搭配的用法。

透過 `NewBasic` 可以初始化一個最簡單並以輪詢當作查詢模式的負載平衡器。

```go
package main

import (
	"github.com/go-mego/sd/lb"
)

func main() {
	// 初始化一個最基本的負載平衡，並且設置可用的伺服器位置。
	l := lb.NewBasic([]string{
		"localhost:1234",
		"localhost:4567",
		"localhost:8910",
	})
	// 透過 `.Get()` 來透過負載平衡取得其中一個可用的伺服器。
	// 基本負載平衡會以輪詢當作查詢模式，每次會取得不同的伺服器，不會與上一個重複。
    l.Get().String() // localhost:1234
    l.Get().String() // localhost:4567
    l.Get().String() // localhost:8910
    l.Get().String() // localhost:1234
}
```

## 進階自訂

若要與其他服務探索中心搭配使用，這裡有幾種不同的查詢用法。雖然查詢方法不同，但是設置都是一樣的，詳細的設置請參考下方範例。

```go
// 輪詢模式，依照順序呼叫不同伺服器。
lb.NewRoundRobin(lb.Option{ ... })
// 隨機模式，依照亂數呼叫不同伺服器。（*）
lb.NewRandom(lb.Option{ ... })
// （尚未支援）最近模式，永遠呼叫最低 Ping 值的伺服器。（*）
lb.NewClosest(lb.Option{ ... })
// （尚未支援）寬鬆模式，永遠呼叫最少連線數的伺服器。（*）
lb.NewFastest(lb.Option{ ... })
```

`（*）`：表示有可能與上一次的伺服器重複。

### Consul

負載平衡也能夠使用 Consul 作為基礎，這能夠讓你有更大的彈性來取得伺服器位置，而不是僅能使用固定寫死的列表。

```go
package main

import (
	"github.com/go-mego/sd/consul"
	"github.com/go-mego/sd/lb"
	"github.com/go-mego/sd/version"
	"github.com/hashicorp/consul/api"
)

func main() {
	// 以預設設定檔建立 Consul 客戶端。
	c, _ := consul.NewClient(api.DefaultConfig())
	// 以 Consul 為基礎，初始化一個輪詢負載平衡器。
	l := lb.NewRoundRobin(lb.Option{
		// 請參閱 Version 套件，這會產生一個 `1.0.0+stable` 的字串。
		Tag:    version.Define(1, 0, 0, "stable").String(),
		Name:   "Database",
		Client: c,
	})
	// 透過 `Get` 取得 Consul 服務探索中心裡其中一個
	// 帶有 `1.0.0+stable` 標籤的 `Database` 伺服器 IP 位置。
	l.Get().String()
}
```