### 1.Cookie介绍

- HTTP是**无状态协议**，服务器**不能记录浏览器的访问状态**，也就是说服务器**不能区分两次请求是否由同一个客户端发出**
- Cookie是**解决HTTP协议无状态的方案**之一
- Cookie实际上就是**服务器保存在浏览器上的一段信息**。浏览器有了Cookie之后，每次向服务器**发送请求时**都会同时将该**信息发送给服务器，**服务器收到请求后，就可以**根据该信息处理请求**
- Cookie由**服务器创建**，并发送给浏览器，最终由**浏览器保存**

#### Cookie用途

Cookie由服务器创建，并发送给浏览器，最终由浏览器保存

### 2.Cookie的使用

- 测试服务端发送cookie给客户端，客户端请求时携带cookie

```go
package main

import (
   "github.com/gin-gonic/gin"
   "fmt"
)

func main() {
   // 1.创建路由
   // 默认使用了2个中间件Logger(), Recovery()
   r := gin.Default()
   // 服务端要给客户端cookie
   r.GET("cookie", func(c *gin.Context) {
      // 获取客户端是否携带cookie
      cookie, err := c.Cookie("key_cookie")
      if err != nil {
         cookie = "NotSet"
         // 给客户端设置cookie
         //  maxAge int, 单位为秒
         // path,cookie所在目录
         // domain string,域名
         //   secure 是否智能通过https访问
         // httpOnly bool  是否允许别人通过js获取自己的cookie
         c.SetCookie("key_cookie", "value_cookie", 60, "/",
            "localhost", false, true)
      }
      fmt.Printf("cookie的值是： %s\n", cookie)
   })
   r.Run(":8000")
}
```

### 3.Cookie   练习

- 模拟实现权限验证中间件

  - 有2个路由，login和home
  - login用于**设置cookie**
  - home是**访问查看信息的请求**
  - 在请求**home之前**，先跑中间件代码，**检验是否存在cook**ie

- 访问home，会显示错误，因为权限校验未通过

- ![img](https://www.topgoer.com/static/gin/5.1/1.jpg)

  - 然后访问登录的请求，登录并设置cookie

  ![img](https://www.topgoer.com/static/gin/5.1/2.jpg)

  - 再次访问home，访问成功

  ![img](https://www.topgoer.com/static/gin/5.1/3.jpg)

~~~go
package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

func Auth()  gin.HandlerFunc {
	return func(c *gin.Context) {
	 //value:=	c.Cookie(key)
		//err ==nil
		if cookie,err:= c.Cookie("abc");err==nil {
			if cookie == "123" {
				//如果等于自己要匹配的值 那么就返回
				c.Next()
				return
			}
		}
		//返回错误
		c.JSON(http.StatusUnauthorized,gin.H{"error":"err"})
		//验证不通过就不用调用后续函数
        c.Abort()
		return
	}
}

//c.SetCookie("key_cookie", "value_cookie", 60, "/",
//"localhost", false, true)

func main() {
    r:= gin.Default()
    r.GET("/login", func(c *gin.Context) {
		c.SetCookie("abc","123",6000,"/","localhost",false,true)
		c.String(200,"Login Success!")

	})
    r.GET("/home",Auth(), func(c *gin.Context) {
		c.JSON(200,gin.H{"data":"home"})
	})
    r.Run(":8000")
}
~~~

### 4.Cookie的缺点

- **不安全**，明文
- 增加带宽消耗
- 可以**被禁用**
- cookie**有上限**

### 5.Sessions

gorilla/sessions为**自定义session**后端**提供cookie**和**文件系统session**以及基础结构。

- 简单的API：将其用作设**置签名（以及可选的加密）cookie**的简便方法。
- 内置的后端可将session**存储在cookie或文件系统**中。
- Flash消息：一直**持续读取的session**值。
- 切换session持久性（又称“记住我”）和**设置其他属性**的便捷方法。
- 旋转**身份验证**和**加密密钥**的机制。
- 每个**请求有多个session**，即使使用不同的后端也是如此。
- 自定义session后端的**接口和基础结构**：可以使用通用API检索并**批量保存来自不同商店的session**。

~~~go
package main

import (
    "fmt"
    "net/http"

    "github.com/gorilla/sessions"
)

// 初始化一个cookie存储对象
// something-very-secret应该是一个你自己的密匙，只要不被别人知道就行
var store = sessions.NewCookieStore([]byte("something-very-secret"))

func main() {
    http.HandleFunc("/save", SaveSession)
    http.HandleFunc("/get", GetSession)
    err := http.ListenAndServe(":8080", nil)
    if err != nil {
        fmt.Println("HTTP server failed,err:", err)
        return
    }
}

func SaveSession(w http.ResponseWriter, r *http.Request) {
    // Get a session. We're ignoring the error resulted from decoding an
    // existing session: Get() always returns a session, even if empty.

    //　获取一个session对象，session-name是session的名字
    session, err := store.Get(r, "session-name")
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    // 在session中存储值
    session.Values["foo"] = "bar"
    session.Values[42] = 43
    // 保存更改
    session.Save(r, w)
}
func GetSession(w http.ResponseWriter, r *http.Request) {
    session, err := store.Get(r, "session-name")
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    foo := session.Values["foo"]
    fmt.Println(foo)
}
~~~

```go
    // 删除
    // 将session的最大存储时间设置为小于零的数即为删除
    session.Options.MaxAge = -1
    session.Save(r, w)
```