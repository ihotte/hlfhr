# HTTPS Listener For HTTP Redirect

这个 mod 通过劫持 `net.Conn` 实现了一个功能:   
当用户使用 http 协议访问 https 端口时，服务器返回302重定向。  
改编自 `net/http` 。  

This mod implements a feature by hijacking `net.Conn` :  
If a user accesses an https port using http, the server returns 302 redirection.  
Adapted from `net/http`  and `crypto/tls` .  

Related Issue:  
[net/http: configurable error message for Client sent an HTTP request to an HTTPS server. #49310](https://github.com/golang/go/issues/49310)  


***
## Get
```
go get github.com/bddjr/hlfhr
```


***
## Example
[See example/main.go](example/main.go)  

Example:  
```go
var srv *hlfhr.Server

func main() {
	// Use hlfhr.New
	srv = hlfhr.New(&http.Server{
		Addr: ":5678",
		Handler: http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			// Write something...
		}),
		ReadHeaderTimeout: 10 * time.Second,
		IdleTimeout:       10 * time.Second,
	})
	// Then just use it like http.Server .

	err := srv.ListenAndServeTLS("localhost.crt", "localhost.key")
	fmt.Println(err)
}
```
```go
var l net.Listener
var srv *http.Server

func main() {
	srv = &http.Server{
		Addr: ":5678",
		Handler: http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			// Write something...
		}),
		ReadHeaderTimeout: 10 * time.Second,
		IdleTimeout:       10 * time.Second,
	}

	var err error
	l, err = net.Listen("tcp", srv.Addr)
	if err != nil {
		fmt.Println(err)
		return
	}
	defer l.Close()

	// Use hlfhr.NewListener
	l = hlfhr.NewListener(l, srv, nil)

	err = srv.ServeTLS(l, "localhost.crt", "localhost.key")
	fmt.Println(err)
}
```

Run:  
```
git clone https://github.com/bddjr/hlfhr
cd hlfhr
cd example

go build
./example
```

```

  test:
  cmd /C curl -v -k -L http://localhost:5678/

2024/06/20 11:50:09 http: TLS handshake error from [::1]:60470: hlfhr: Client sent an HTTP request to an HTTPS server
```

[See request](README_curl.md)  


***
## Logic

[See request](README_curl.md)  

HTTPS Server Start -> Hijacking net.Listener.Accept  

### Client HTTPS 
Accept hijacking net.Conn.Read -> First byte not looks like HTTP -> ✅Continue...  

### Client HTTP/1.1
Accept hijacking net.Conn.Read -> First byte looks like HTTP -> HttpOnHttpsPortErrorHandler

If handler nil -> Read Host header and path -> 🔄302 Redirect.  

### Client HTTP/???
Accept hijacking net.Conn.Read -> First byte looks like HTTP -> HttpOnHttpsPortErrorHandler

If handler nil -> Missing Host header -> ❌400 ScriptRedirect.  

### Client ???
Accept hijacking net.Conn.Read -> First byte does not looks like TLS handshake or HTTP request -> Close.

***
## Option Example

#### HttpOnHttpsPortErrorHandler
```go
// Default
srv.HttpOnHttpsPortErrorHandler = http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
	http.Redirect(w, r, "https://"+r.Host+r.URL.RequestURI(), http.StatusFound)
})
```
```go
// Check Host header
srv.HttpOnHttpsPortErrorHandler = http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
	r.URL.Host = r.Host
	switch r.URL.Hostname() {
	case "localhost":
		http.Redirect(w, r, "https://"+r.Host+r.URL.RequestURI(), http.StatusFound)
	case "www.localhost", "127.0.0.1":
		s := "https://localhost"
		if port := r.URL.Port(); port != "" {
			s += ":" + port
		}
		s += r.URL.RequestURI()
		http.Redirect(w, r, s, http.StatusFound)
	default:
		w.WriteHeader(421)
	}
})
```

***
## Feature Example

#### New  
```go
srv := hlfhr.New(&http.Server{})
```

#### NewServer  
```go
srv := hlfhr.NewServer(&http.Server{})
```

#### Server.IsShuttingDown
```go
var srv *hlfhr.Server
isShuttingDown := srv.IsShuttingDown()
```

#### Server.NewListener
```go
var l net.Listener
var srv *http.Server
l = hlfhr.New(srv).NewListener(l)
```

#### NewListener
```go
var l net.Listener
var srv *http.Server
var h http.Handler
l = hlfhr.NewListener(c, srv, h)
```

#### IsMyListener
```go
var l net.Listener
isHlfhrListener := hlfhr.IsMyListener(l)
```

#### NewConn
```go
var c net.Conn
var srv *http.Server
var h http.Handler
c = hlfhr.NewConn(c, srv, h)
```

#### IsMyConn
```go
var c net.Conn
isHlfhrConn := hlfhr.IsMyConn(c)
```

#### NewResponse
```go
resp := hlfhr.NewResponse()
```

#### NewResponseWriter
```go
var c net.Conn
w := hlfhr.NewResponseWriter(c, nil)
hw := http.ResponseWriter(w)
```

#### ResponseWriter.WriteLock
```go
var w *hlfhr.ResponseWriter
w.WriteLock()
```


***
## License
[BSD-3-clause license](LICENSE.txt)  
