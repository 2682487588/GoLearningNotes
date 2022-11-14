## 1.解开Gin的神秘面纱

### 1.1.1 数据如何在gin流转

```go
package main

import "github.com/gin-gonic/gin"

func main() {
    r := gin.Default()
    r.GET("/ping", func(c *gin.Context) {
        c.JSON(200, gin.H{
            "message": "pong",
        })
    })
    r.Run() // listen and serve on 0.0.0.0:8080
}
```

1. r:= gin.Default 初始化相应的参数
2. /ping将路由及处理handler注册到路由树
3. 启动服务

r.Run()其实调用的是err = http.**ListenAndServe(address, engine)**

###  1.1.2. ServeHTTP的作用

DefaultServeMux实现了**ServeHTTP(ResponseWriter, *Request)**,

request执行到server.go的**serverHandler{c.server}**.ServeHTTP(w, w.req)这一行,DefaultServeMux**取到了相关路由的处理handler**

### 1.1.3. Engine

整个gin框架中最重要的一个**struct就是Engine**——包含**路由**, **中间件**, **相关配置信息**

```go
type Engine struct {
    RouterGroup // 路由
    pool             sync.Pool  // context pool  连接池
    trees            methodTrees // 路由树
    // html template及其他相关属性先暂时忽略
}
```

### 1.1.4. New(), Default()

#### NEW

```go
func New() *Engine {
    // ...
    engine := &Engine{
        RouterGroup: RouterGroup{
            Handlers: nil,
            basePath: "/",
            root:     true,
        },
        // ...
        trees: make(methodTrees, 0, 9),
    }
    engine.RouterGroup.engine = engine
    engine.pool.New = func() interface{} {
        return engine.allocateContext()
    }
    return engine
}
```

New()主要干的事情:

1. 初始化了Engine 
2. 将RouterGroup的**Handlers(数组)**设置成**nil**, **basePath设置成/** 
3. RouteGroup里面也有一个Engine指针,这里将刚刚初始化的**engine赋值**给了**RouterGroup的engine指针**
4. 为了**防止频繁的context GC**造成效率的降低, 在Engine里**使用了sync.Pool**

#### Default

```go
func Default() *Engine {
    debugPrintWARNINGDefault()
    engine := New()
    engine.Use(Logger(), Recovery())
    return engine
}
```

Default()跟New()几乎一模一样, 就是调用了**gin内置的Logger(), Recovery()**中间件.

### 1.1.5. Use()

```go
func (engine *Engine) Use(middleware ...HandlerFunc) IRoutes {
    engine.RouterGroup.Use(middleware...)
    engine.rebuild404Handlers()
    engine.rebuild405Handlers()
    return engine
}
```

Use()就是gin的引入中间件的入口了, 不难发现Use()其实是在给**RouteGroup引入中间件**的.

```go
engine.rebuild404Handlers()
engine.rebuild405Handlers()
```

这两句函数其实在这里没有任何用处. 我感觉这里是给gin的测试代码用的. 我们在使用gin的时候, 要想在404, 405添加处理过程, 可以通过NoRoute(), NoMethod()来处理.

### 1.1.6. addRoute()

```go
func (engine *Engine) addRoute(method, path string, handlers HandlersChain) {
    ...
    root := engine.trees.get(method)
    if root == nil {
        root = new(node)
        engine.trees = append(engine.trees, methodTree{method: method, root: root})
    }
    root.addRoute(path, handlers)
}
```

**将handler和path添加到engine的tree中**

### 1.1.8. ServeHTTP

这个函数的存在, **才能将请求转到gin中,** 使用gin的**相关函数处理request请求**

```go
t := engine.trees

for i, tl := 0, len(t); i < tl; i++ {
    if t[i].method != httpMethod {
        continue
    }
    root := t[i].root

    handlers, params, tsr := root.getValue(path, c.Params, unescape)
    if handlers != nil {
        c.handlers = handlers
        c.Params = params
        c.Next()
        c.writermem.WriteHeaderNow()
        return
    }
    ...
}
//利用request中的path, 从Engine的trees中获取已经注册的handler
```

```go
func (c *Context) Next() {
    c.index++
    for c.index < int8(len(c.handlers)) {
        c.handlers[c.index](c)
        c.index++
    }
}
//在Next()执行handler的操作. 其实也就是下面的函数
```

~~~go
func(c *gin.Context) {
    c.JSON(200, gin.H{
        "message": "pong",
    })
}
~~~

