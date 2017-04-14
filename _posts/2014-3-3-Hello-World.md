---
layout: post
title: go test 外部服务的“麻烦”解决办法
---


咱们在写golang的unit test时，经常会碰到依赖一些外部服务；比如依赖第三方HTTP接口、redis server、TCP server

单元测试有一种叫测试替身（Mock）的解决办法，但是通常我们依赖服务是在被测函数里边的，难道为了跑一次test，还得改被测函数么（完美情况我们可以重构代码让其不需要改动也能跑test）


***

下边说的办法也比较麻烦，但这辈子可能只需要麻烦一次

先来说最常用的第三方HTTP接口，golang官方其实内置了一个pkg，叫 **net/http/httptest**，通过这个pkg可以在test.go中去自己实现接口逻辑，从而返回我们预期的数据，最终实现unit test的数据检查

举个简单的例子（代码意思下，未必能跑通）

```golang
import (
    "net/http"
    "net/http/httptest"
    "testing"
)

type TestHandler struct {
    token string
    expiredAt int64
}

func (h *TestHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    if strings.HasPrefix(r.URL.String(), `/oauth/token`) {
        h.token = "gagaga"
        h.expiredAt = time.Now().Unix() + 10
        fmt.Fprintf(w, `{"status": "ok", "access_token": "gagaga", "expires_in": 10}`)
    } else {
        fmt.Fprintf(w, `hello!`)
    }
}

func TestGetOAuthToken(t *testing.T) {
    // 启动外部服务
    server := httptest.NewServer(&TestHandler{})
    defer server.Close()

    // 调用接口
    resp, err := http.Get(server.URL + `/oauth/token`)
    
    // TODO 检查resp的数据是否和预期一致
    ...
}


```

从代码来看，其实就是在test中自己实现http server并启动，保证了该接口正常提供服务


***

如果咱们的test依赖redis server，同样可以自己实现（本质实现一个TCP server）

还是上例子

```
type TestRedisHandler struct {
    addr string
    server *redcon.Server
}

func (h *TestRedisHandler) start() {
    h.addr   = "127.0.0.1:5987"
    h.server = redcon.NewServer(h.addr,
		func(conn redcon.Conn, cmd redcon.Command) {
			switch strings.ToLower(string(cmd.Args[0])) {
			default:
				conn.WriteError("ERR unknown command '" + string(cmd.Args[0]) + "'")
			case "quit":
				conn.WriteString("OK")
				conn.Close()
			case "get":
				conn.WriteBulk([]byte(`hello!`))
			}
		},
		func(conn redcon.Conn) bool {
			return true
		},
		func(conn redcon.Conn, err error) {
		},
    )
    go h.server.ListenAndServe()
}

func (h *TestRedisHandler) stop() {
    h.server.Close()
}

func TestGetDataFromRedis(t *testing.T) {
    rserver := &TestRedisHandler{}
    rserver.start()
    defer rserver.stop()
    
    pool := connectRedis(rserver.addr)
    conn := pool.Get()
    conn.Do("GET", "redis_key")
    ...
}

```

如果依赖Postgresql、beanstalk、elasticsearch服务呢？按照这些服务的协议实现server就行了

BTW，咱们在跑go test时，可以使用 **go test -coverprofile=cover.out && go tool cover -html=cover.out** 
