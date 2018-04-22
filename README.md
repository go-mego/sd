# Service Discovery [![GoDoc](https://godoc.org/github.com/go-mego/sd?status.svg)](https://godoc.org/github.com/go-mego/sd)

Service Discovery 套件能夠讓你透過服務探索自動註冊服務，並且以負載平衡的方式來平均連線請求。

# 索引

* [安裝方式](#安裝方式)
* [使用方式](#使用方式)
    * [Consul](#Consul)
        * [服務註冊](#服務註冊)
        * [服務移除](#服務移除)

# 安裝方式

打開終端機並且透過 `go get` 安裝此套件即可。

```bash
$ go get github.com/go-mego/sd
```

# 使用方式

此套件有許多使用方式與支援不同的服務探索中心，下列是目前支援的列表。

* Consul
* Etcd（_尚未支援_）
* Eureka（_尚未支援_）
* ZooKeeper（_尚未支援_）

## Consul

Consul 是一個服務探索中心，伺服器可以在上線的時候自動向服務探索中心註冊自己的 IP 位置，接著負載平衡就能得知該伺服器並均攤請求負載。

### 建立客戶端

透過 `NewClient` 與傳入設定檔來建立一個客戶端連線。

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
}
```

### 服務註冊

透過 `Register` 函式並傳入伺服器資訊，就能將伺服器註冊到遠端的服務探索中心列表令其他服務發現自己並提供使用。

```go
func main() {
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
func main() {
    c, _ := consul.NewClient(api.DefaultConfig())
	// 將指定編號的服務從服務探索中心裡移除。
	c.Deregister(&api.AgentServiceRegistration{
        // 如果先前有透過 `.Register()` 與 `.ID()` 註冊，
        // 那麼這次就可以透過 `.ID()` 取得上次產生的編號進而從列表中移除自己。
		ID: c.ID(),
	})
}
```