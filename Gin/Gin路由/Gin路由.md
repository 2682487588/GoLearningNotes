### **HttpRouter**

[HttpRouter 是用于Go](https://golang.org/)的轻量级高性能 HTTP 请求路由器（也称为*多路复用器*或简称*mux*）

### 特征

#### **仅显式匹配：**

对于其他路由器，例如[`http.ServeMux`](https://golang.org/pkg/net/http/#ServeMux)，请求的 URL 路径可以匹配多个模式。因此他们有一些尴尬的模式优先规则，比如*最长匹配*或*先注册，先匹配*。通过这个路由器的设计，一个请求只能匹配一个或不匹配一个路由。结果，也没有意外的匹配，这使得它非常适合 SEO 并改善了用户体验。

#### **停止关心尾部斜线：**

选择您喜欢的 URL 样式，如果缺少尾部斜线或有多余的斜线，路由器会自动重定向客户端。当然，只有在新路径有处理程序的情况下才会这样做。如果你不喜欢它，你可以[关闭这个行为](https://godoc.org/github.com/julienschmidt/httprouter#Router.RedirectTrailingSlash)。

#### **路径自动更正：**

除了检测丢失的或额外的斜杠，路由器还可以修复错误的情况并删除多余的路径元素（如`../`或`//`）。[CAPTAIN CAPS LOCK](http://www.urbandictionary.com/define.php?term=Captain+Caps+Lock)是您的用户之一吗？HttpRouter 可以通过进行不区分大小写的查找并将他重定向到正确的 URL 来帮助他。

#### **路由模式中的参数：**

停止解析请求的 URL 路径，只需为路径段命名，路由器将动态值传递给您。由于路由器的设计，路径参数非常便宜。

**零垃圾：**匹配和调度过程产生零字节的垃圾。唯一进行的堆分配是为路径参数构建键值对的切片，以及构建新的上下文和请求对象（后者仅在标准`Handler`/ `HandlerFunc`API 中）。在 3 参数 API 中，如果请求路径不包含参数，则不需要单个堆分配。

#### **最佳表现：**

 [基准不言自明](https://github.com/julienschmidt/go-http-routing-benchmark)。有关实施的技术细节，请参见下文。

#### **不再有服务器崩溃：**

您可以设置[Panic 处理程序](https://godoc.org/github.com/julienschmidt/httprouter#Router.PanicHandler)来处理在处理 HTTP 请求期间发生的恐慌。然后路由器恢复并让`PanicHandler`日志记录发生的事情并提供一个很好的错误页面。

#### **非常适合 API：**

路由器设计鼓励构建合理的、分层的 RESTful API。此外，它还内置了对[OPTIONS 请求](http://zacstewart.com/2012/04/14/http-options-method.html)和`405 Method Not Allowed`回复的原生支持。

当然，您也可以设置**自定义[`NotFound`](https://godoc.org/github.com/julienschmidt/httprouter#Router.NotFound)和 [`MethodNotAllowed`](https://godoc.org/github.com/julienschmidt/httprouter#Router.MethodNotAllowed)处理程序**并[**提供静态文件**](https://godoc.org/github.com/julienschmidt/httprouter#Router.ServeFiles)。

### 1. Restful风格的API

- gin支持Restful风格的API
- 即Representational State Transfer的缩写。直接翻译的意思是"表现层状态转化"，是一种互联网应用程序的API设计理念：URL定位资源，用HTTP描述操作

1.获取文章 /blog/getXxx Get blog/Xxx

2.添加 /blog/addXxx POST blog/Xxx

3.修改 /blog/updateXxx PUT blog/Xxx

4.删除 /blog/delXxxx DELETE blog/Xxx

### API参数

- 可以通**过Context的Param方法来获取API参数**
- localhost:8000/xxx/zhangsan

```go
package main

import (
    "net/http"
    "strings"
    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default()
    r.GET("/user/:name/*action", func(c *gin.Context) {
        name := c.Param("name")
        action := c.Param("action")
        //截取/
        action = strings.Trim(action, "/")
        c.String(http.StatusOK, name+" is "+action)
    })
    //默认为监听8080端口
    r.Run(":8000")
}
```

![image-20220120152736276](C:%5CUsers%5C%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20220120152736276.png)

###  URL参数

- URL参数可以通过**DefaultQuery()**或**Query()**方法获取
- DefaultQuery()若参数不对则，**返回默认值**，Query()若不存在，**返回空串**

~~~go
package main

import (
    "fmt"
    "net/http"

    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default()
    r.GET("/user", func(c *gin.Context) {
        //指定默认值
        //http://localhost:8080/user 才会打印出来默认的值
        name := c.DefaultQuery("name", "枯藤")
        c.String(http.StatusOK, fmt.Sprintf("hello %s", name))
    })
    r.Run()
}
~~~

#### 不传递参数输出的结果：

![不传递参数输出的结果](https://www.topgoer.com/static/gin/1.1/3.png)

#### 传递参数输出的结果：

![image-20220120153828246](C:%5CUsers%5C%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20220120153828246.png)

### 表单参数

- 表单传输为**post请求**，http常见传输格式为四种:
  - application/json
  - application/x-www-form-urlencoded
  - application/xml
  - multipart/form-data
- 表单参数可以通过**PostForm()**方法获取,该方法默认解析的是**x-www-form-urlencoded**或**from-data格**式的参数

~~~go
package main

//
import (
	"fmt"
	"net/http"

	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()
	r.POST("/form", func(c *gin.Context) {
		types := c.DefaultPostForm("type", "post")
		username := c.PostForm("username")
		password := c.PostForm("userpassword")
		// c.String(http.StatusOK, fmt.Sprintf("username:%s,password:%s,type:%s", username, password, types))
		c.String(http.StatusOK, fmt.Sprintf("username:%s,password:%s,type:%s", username, password, types))
	})
	r.Run()
}
~~~



~~~html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
</head>
<body>
<form action="http://localhost:8080/form" method="post" action="application/x-www-form-urlencoded">
    用户名：<input type="text" name="username" placeholder="请输入你的用户名">  <br>
    密&nbsp;&nbsp;&nbsp;码：<input type="password" name="userpassword" placeholder="请输入你的密码">  <br>
    <input type="submit" value="提交">
</form>
</body>
</html>
~~~



![image-20220120163550662](C:%5CUsers%5C%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20220120163550662.png)

### 上传单个文件

- multipart/form-data用于文件上传
- gin文件上传与原生net/http方法类似，不同在于gin把原生request封装到c.request

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
</head>
<body>
    <form action="http://localhost:8080/upload" method="post" enctype="multipart/form-data">
          上传文件:<input type="file" name="file" >
          <input type="submit" value="提交">
    </form>
</body>
</html>
```

```go
package main

import (
    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default()
    //限制上传最大尺寸
    r.MaxMultipartMemory = 8 << 20
    r.POST("/upload", func(c *gin.Context) {
        file, err := c.FormFile("file")
        if err != nil {
            c.String(500, "上传图片出错")
        }
        // c.JSON(200, gin.H{"message": file.Header.Context})
        c.SaveUploadedFile(file, file.Filename)
        c.String(http.StatusOK, file.Filename)
    })
    r.Run()
}
```

#### 上传特定文件

```go
package main

import (
    "fmt"
    "log"
    "net/http"

    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default()
    r.POST("/upload", func(c *gin.Context) {
        _, headers, err := c.Request.FormFile("file")
        if err != nil {
            log.Printf("Error when try to get file: %v", err)
        }
        //headers.Size 获取文件大小
        if headers.Size > 1024*1024*2 {
            fmt.Println("文件太大了")
            return
        }
        //headers.Header.Get("Content-Type")获取上传文件的类型
        if headers.Header.Get("Content-Type") != "image/png" {
            fmt.Println("只允许上传png图片")
            return
        }
        c.SaveUploadedFile(headers, "./video/"+headers.Filename)
        c.String(http.StatusOK, headers.Filename)
    })
    r.Run()
}
```

### 上传多个文件

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
</head>
<body>
    <form action="http://localhost:8000/upload" method="post" enctype="multipart/form-data">
          上传文件:<input type="file" name="files" multiple>
          <input type="submit" value="提交">
    </form>
</body>
</html>
```

```go
package main

import (
   "github.com/gin-gonic/gin"
   "net/http"
   "fmt"
)

// gin的helloWorld

func main() {
   // 1.创建路由
   // 默认使用了2个中间件Logger(), Recovery()
   r := gin.Default()
   // 限制表单上传大小 8MB，默认为32MB
   r.MaxMultipartMemory = 8 << 20
   r.POST("/upload", func(c *gin.Context) {
      form, err := c.MultipartForm()
      if err != nil {
         c.String(http.StatusBadRequest, fmt.Sprintf("get err %s", err.Error()))
      }
      // 获取所有图片
      files := form.File["files"]
      // 遍历所有图片
      for _, file := range files {
         // 逐个存
         if err := c.SaveUploadedFile(file, file.Filename); err != nil {
            c.String(http.StatusBadRequest, fmt.Sprintf("upload err %s", err.Error()))
            return
         }
      }
      c.String(200, fmt.Sprintf("upload ok %d files", len(files)))
   })
   //默认端口号是8080
   r.Run(":8000")
}
```

### routes group

- 为了管理相同的URL

```go
package main

import (
   "github.com/gin-gonic/gin"
   "fmt"
)

// gin的helloWorld

func main() {
   // 1.创建路由
   // 默认使用了2个中间件Logger(), Recovery()
   r := gin.Default()
   // 路由组1 ，处理GET请求
   v1 := r.Group("/v1")
   // {} 是书写规范
   {
      v1.GET("/login", login)
      v1.GET("submit", submit)
   }
   v2 := r.Group("/v2")
   {
      v2.POST("/login", login)
      v2.POST("/submit", submit)
   }
   r.Run(":8000")
}

func login(c *gin.Context) {
   name := c.DefaultQuery("name", "jack")
   c.String(200, fmt.Sprintf("hello %s\n", name))
}

func submit(c *gin.Context) {
   name := c.DefaultQuery("name", "lily")
   c.String(200, fmt.Sprintf("hello %s\n", name))
}
```

![image-20220120191734143](C:%5CUsers%5C%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20220120191734143.png)

### 路由拆分与注册

#### 基本的路由注册

```go
package main

import (
    "net/http"

    "github.com/gin-gonic/gin"
)

func helloHandler(c *gin.Context) {
    c.JSON(http.StatusOK, gin.H{
        "message": "Hello www.topgoer.com!",
    })
}

func main() {
    r := gin.Default()
    r.GET("/topgoer", helloHandler)
    if err := r.Run(); err != nil {
        fmt.Println("startup service failed, err:%v\n", err)
    }
}
```

#### 路由拆分成文件荷包

当项目的规模增大后就不太适**合继续在项目的main.go文件**中去实现路由注册相关逻辑了，我们会倾向于**把路由部分的代码都拆分出来**，形成一个单独的文件或包

**routers.go**

~~~go
package main

import (
    "net/http"

    "github.com/gin-gonic/gin"
)

func helloHandler(c *gin.Context) {
  c.JSON(http.StatusOK,gin.H{
    "message":"Hello www.com!!"
  })
}

func setupRouter() *gin.Engine {
  r:=gin.Default()
  r.GET("/topgoer",helloHandler)
  return r
}
~~~

此时**main.go**中调用上面定义好的**setupRouter函数**：

```go
func main() {
    r := setupRouter()
    if err := r.Run(); err != nil {
        fmt.Println("startup service failed, err:%v\n", err)
    }
}
```

**此时的目录结构**

```go
gin_demo
├── go.mod
├── go.sum
├── main.go
└── routers.go
```

#### 路由拆分成多个文件

当**我们的业务规模继续膨胀**，单独的**一个routers文件或包**已经**满足不了我们的需求**了，

routers/shop.go中添加一个**LoadShop的函数**，将**shop相关的路由注册**到指定的路由器：

```
func LoadShop(e *gin.Engine)  {
    e.GET("/hello", helloHandler)
  e.GET("/goods", goodsHandler)
  e.GET("/checkout", checkoutHandler)
  ...
}
```

routers/blog.go中添加一个**LoadBlog的函**数，将**blog相关的路由**注册到指定的路由器：

```go
func LoadBlog(e *gin.Engine) {
    e.GET("/post", postHandler)
  e.GET("/comment", commentHandler)
  ...
}
```

在main函数中实现最终的注册逻辑如下：

```go
func main() {
    r := gin.Default()
    routers.LoadBlog(r)
    routers.LoadShop(r)
    if err := r.Run(); err != nil {
        fmt.Println("startup service failed, err:%v\n", err)
    }
}
```

#### 路由拆分到不同的APP

```
gin_demo
├── app
│   ├── blog
│   │   ├── handler.go
│   │   └── router.go
│   └── shop
│       ├── handler.go
│       └── router.go
├── go.mod
├── go.sum
├── main.go
└── routers
    └── routers.go
```

其中app/blog/router.go用来定义post相关路由信息，具体内容如下：

```go
func Routers(e *gin.Engine) {
    e.GET("/post", postHandler)
    e.GET("/comment", commentHandler)
}
```

app/shop/router.go用来定义shop相关路由信息，具体内容如下：

```go
func Routers(e *gin.Engine) {
    e.GET("/goods", goodsHandler)
    e.GET("/checkout", checkoutHandler)
}
```

routers/routers.go中根据需要定义Include函数用来注册子app中定义的路由，Init函数用来进行路由的初始化操作：

```go
type Option func(*gin.Engine)

var options = []Option{}

// 注册app的路由配置
func Include(opts ...Option) {
    options = append(options, opts...)
}

// 初始化
func Init() *gin.Engine {
    r := gin.New()
    for _, opt := range options {
        opt(r)
    }
    return r
}
```

main.go中按如下方式先注册子app中的路由，然后再进行路由的初始化：

~~~go
func main() {
    // 加载多个APP的路由配置
    routers.Include(shop.Routers, blog.Routers)
    // 初始化路由
    r := routers.Init()
    if err := r.Run(); err != nil {
        fmt.Println("startup service failed, err:%v\n", err)
    }
}
~~~

