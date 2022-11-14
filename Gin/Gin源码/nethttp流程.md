### 1net/http的大概流程

**明白几个问题**

- request数据是如何流转的
- gin框架到底**扮演了什么角色**
- 请求从**gin流入net/http**, 最后又是**如何回到gin中**
- gin的**context为何能承担起来复杂的需求**
- gin的**路由算法**
- gin的**中间件**是什么
- gin的**Engine**具体是个什么东西
- net/http的requeset, **response都提供了哪些有用的东西**

#### 1.1.1 gin框架预览 

![img](https://www.topgoer.com/static/gin/yuanma/1.png)

#### 1.1.2 request数据是如何流转的

r.Run()的源码:

```go
func (engine *Engine) Run(addr ...string) (err error) {
    defer func() { debugPrintError(err) }()

    address := resolveAddress(addr)
    debugPrint("Listening and serving HTTP on %s\n", address)
    err = http.ListenAndServe(address, engine)
    return
}
```

```go
func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte("Hello World"))
    })

    if err := http.ListenAndServe(":8000", nil); err != nil {
        fmt.Println("start http server fail:", err)
    }
}
```

输入localhost:8000, 会看到Hello World.

#### 1.1.3  HTTP如何建立起来

**HTTP三次握手**

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210503115020503.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ0NDQzOTg2,size_16,color_FFFFFF,t_70)

第一次

- **客户端发送**SYN，**服务端接收**，服务端得出**客户端的发送能力**和**服务端的接收能力都正常**

第二次

- **服务端发送**SYN+ACK，客户端接收，**客户端**得出**客户端发送接收能力正常**，**服务端发送接收能力也都正常**，但是服务端**不能确认客户端的接收能力是否正常**

第三次

- 客户端发送ACK，服务器接收，服务端才能**得出客户端发送接收能力正常，服务端自己发送接收能力也都正常**

#### 四次挥手

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210503115031239.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ0NDQzOTg2,size_16,color_FFFFFF,t_70)

#### 为什么握手是三次，而挥手时需要四次呢？

其实在TCP握手的时候，接收端将SYN包和ACK确认包合并到一个包中发送的，所以减少了一次包的发送

对于四次挥手，由于TCP是全双工通信，主动关闭方**发送FIN请求不代表完全断开连接**，只能表示主**动关闭方不再发送数据了**。而接收方可能还要发送数据，就不能立即关闭服务器端到客户端的数据通道，所以就**不能将服务端的FIN包和对客户端的ACK包合并发送**，**只能先确认ACK**，等服务器**无需发送数据时在发送FIN包**，所以四次挥手时需要四次数据包的交互


##### 1.1.3正文

在TCP/IP五层模型下, **HTTP位于应用层**, 需要有**传输层**来承载HTTP协议. 传输层比较常见的协议是TCP,UDP, SCTP等. 由于**UDP不可靠**, SCTP有自己特殊的运用场景, 所以一般情况下**HTTP是由TCP协议承载的(**可以使用wireshark抓包然后查看各层协议)

所以说, 要想能够对客户端http请求进行回应的话, 就**首先需要建立起来TCP连接,** 也就是**socket**. 下面要看下net/http是如何建立起来socket?

#### 1.1.4 net/http是如何建立socket

```go
func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte("Hello World"))
    })

    if err := http.ListenAndServe(":8000", nil); err != nil {
        fmt.Println("start http server fail:", err)
    }
}
```

#### 1.1.5. 注册路由

```go
http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte("Hello World"))
    })
```

这段代码是在注册一个路由及这个路由的**handler**到**DefaultServeMux中**

```go
func (mux *ServeMux) Handle(pattern string, handler Handler) {
    mux.mu.Lock()
    defer mux.mu.Unlock()

    if pattern == "" {
        panic("http: invalid pattern")
    }
    if handler == nil {
        panic("http: nil handler")
    }
    if _, exist := mux.m[pattern]; exist {
        panic("http: multiple registrations for " + pattern)
    }

    if mux.m == nil {
        mux.m = make(map[string]muxEntry)
    }
    mux.m[pattern] = muxEntry{h: handler, pattern: pattern}

    if pattern[0] != '/' {
        mux.hosts = true
    }
}
```

#### 1.1.6. 服务监听及响应

上面路由已经**注册到net/http**了, 下面就该如何**建立socket**了, 以及最后又如何取到已经注册到的路由, 将**正确的响应信息从handler中取出来返回给客户端**

```go
if err := http.ListenAndServe(":8000", nil); err != nil {
    fmt.Println("start http server fail:", err)
}
// net/http/server.go:L3002-3005
func ListenAndServe(addr string, handler Handler) error {
    server := &Server{Addr: addr, Handler: handler}
    return server.ListenAndServe()
}
// net/http/server.go:L2752-2765
func (srv *Server) ListenAndServe() error {
    // ... 省略代码
    ln, err := net.Listen("tcp", addr) // <-----看这里listen
    if err != nil {
        return err
    }
    return srv.Serve(tcpKeepAliveListener{ln.(*net.TCPListener)})
}
// net/http/server.go:L2805-2853
func (srv *Server) Serve(l net.Listener) error {
    // ... 省略代码
    for {
        rw, e := l.Accept() // <----- 看这里accept
        if e != nil {
            select {
            case <-srv.getDoneChan():
                return ErrServerClosed
            default:
            }
            if ne, ok := e.(net.Error); ok && ne.Temporary() {
                if tempDelay == 0 {
                    tempDelay = 5 * time.Millisecond
                } else {
                    tempDelay *= 2
                }
                if max := 1 * time.Second; tempDelay > max {
                    tempDelay = max
                }
                srv.logf("http: Accept error: %v; retrying in %v", e, tempDelay)
                time.Sleep(tempDelay)
                continue
            }
            return e
        }
        tempDelay = 0
        c := srv.newConn(rw)
        c.setState(c.rwc, StateNew) // before Serve can return
        go c.serve(ctx) // <--- 看这里
    }
}
// net/http/server.go:L1739-1878
func (c *conn) serve(ctx context.Context) {
    // ... 省略代码
    serverHandler{c.server}.ServeHTTP(w, w.req)
    w.cancelCtx()
    if c.hijacked() {
        return
    }
    w.finishRequest()
    // ... 省略代码
}
// net/http/server.go:L2733-2742
func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
    handler := sh.srv.Handler
    if handler == nil {
        handler = DefaultServeMux
    }
    if req.RequestURI == "*" && req.Method == "OPTIONS" {
        handler = globalOptionsHandler{}
    }
    handler.ServeHTTP(rw, req)
}
// net/http/server.go:L2352-2362
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
    if r.RequestURI == "*" {
        if r.ProtoAtLeast(1, 1) {
            w.Header().Set("Connection", "close")
        }
        w.WriteHeader(StatusBadRequest)
        return
    }
    h, _ := mux.Handler(r) // <--- 看这里
    h.ServeHTTP(w, r)
}

// net/http/server.go:L1963-1965
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
    f(w, r)
}
```

这基本是**整个过程的代码了**

- **ln, err := net.Listen("tcp", addr)**做了初始化了**socket, bind, listen**的操作.
- **rw, e := l.Accept()**进行accept, 等待**客户端进行连接**
- go c.serve(ctx) 启动**新的goroutine**来处理本次请求. 同时主goroutine**继续等待客户端连接**, 进行高并发操作
- h, _ := mux.Handler(r) **获取注册的路由**, 然后拿到这个路由的handler, 然后将处理结果返回给客户端

```go
// net/http/server.go:L2218-2238
func (mux *ServeMux) match(path string) (h Handler, pattern string) {
    // Check for exact match first.
    v, ok := mux.m[path]
    if ok {
        return v.h, v.pattern
    }

    // Check for longest valid match.
    var n = 0
    for k, v := range mux.m {
        if !pathMatch(k, path) {
            continue
        }
        if h == nil || len(k) > n {
            n = len(k)
            h = v.h
            pattern = v.pattern
        }
    }
    return
}
```

**gin就是一个httprouter也不过分**, 当然gin也提供了其他比较主要的功能, 后面会一一介绍