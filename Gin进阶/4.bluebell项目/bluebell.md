### 4.3 雪花算法

![image-20220303093259706](C:%5CUsers%5C%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20220303093259706.png)

~~~go
package snow

import (
	"time"

	sf "github.com/bwmarrin/snowflake"
)

var node *sf.Node

func Init(startTime string, machineID int64) (err error) {
	var st time.Time
	st, err = time.Parse("2006-03-04", startTime)
	if err != nil {
		return
	}
	sf.Epoch = st.UnixNano() / 1000000
	node, err = sf.NewNode(machineID)
	return
}
func GenID() int64 {
	return node.Generate().Int64()
}
~~~

### 4.6调用Validator库进行校验

```go
package main

import (
	"fmt"
	"net/http"

	"github.com/gin-gonic/gin"
	"github.com/gin-gonic/gin/binding"
	"github.com/go-playground/locales/en"
	"github.com/go-playground/locales/zh"
	ut "github.com/go-playground/universal-translator"
	"github.com/go-playground/validator/v10"
	enTranslations "github.com/go-playground/validator/v10/translations/en"
	zhTranslations "github.com/go-playground/validator/v10/translations/zh"
)

// 定义一个全局翻译器T
var trans ut.Translator

// InitTrans 初始化翻译器
func InitTrans(locale string) (err error) {
	// 修改gin框架中的Validator引擎属性，实现自定制
	if v, ok := binding.Validator.Engine().(*validator.Validate); ok {

		zhT := zh.New() // 中文翻译器
		enT := en.New() // 英文翻译器

		// 第一个参数是备用（fallback）的语言环境
		// 后面的参数是应该支持的语言环境（支持多个）
		// uni := ut.New(zhT, zhT) 也是可以的
		uni := ut.New(enT, zhT, enT)

		// locale 通常取决于 http 请求头的 'Accept-Language'
		var ok bool
		// 也可以使用 uni.FindTranslator(...) 传入多个locale进行查找
		trans, ok = uni.GetTranslator(locale)
		if !ok {
			return fmt.Errorf("uni.GetTranslator(%s) failed", locale)
		}

		// 注册翻译器
		switch locale {
		case "en":
			err = enTranslations.RegisterDefaultTranslations(v, trans)
		case "zh":
			err = zhTranslations.RegisterDefaultTranslations(v, trans)
		default:
			err = enTranslations.RegisterDefaultTranslations(v, trans)
		}
		return
	}
	return
}

type SignUpParam struct {
	Age        uint8  `json:"age" binding:"gte=1,lte=130"`
	Name       string `json:"name" binding:"required"`
	Email      string `json:"email" binding:"required,email"`
	Password   string `json:"password" binding:"required"`
	RePassword string `json:"re_password" binding:"required,eqfield=Password"`
}

func main() {
	if err := InitTrans("zh"); err != nil {
		fmt.Printf("init trans failed, err:%v\n", err)
		return
	}

	r := gin.Default()

	r.POST("/signup", func(c *gin.Context) {
		var u SignUpParam
		if err := c.ShouldBind(&u); err != nil {
			// 获取validator.ValidationErrors类型的errors
			errs, ok := err.(validator.ValidationErrors)
			if !ok {
				// 非validator.ValidationErrors类型错误直接返回
				c.JSON(http.StatusOK, gin.H{
					"msg": err.Error(),
				})
				return
			}
			// validator.ValidationErrors类型错误则进行翻译
			c.JSON(http.StatusOK, gin.H{
				"msg":errs.Translate(trans),
			})
			return
		}
		// 保存入库等具体业务逻辑代码...

		c.JSON(http.StatusOK, "success")
	})
	_ = r.Run(":8999")
}
```

### 4.11基于Cookie-Session和基于Token认证

![image-20220305094039869](C:%5CUsers%5C%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20220305094039869.png)

![image-20220305094230519](C:%5CUsers%5C%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20220305094230519.png)

#### JWT

![image-20220305094539505](C:%5CUsers%5C%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20220305094539505.png)

![image-20220305094803973](C:%5CUsers%5C%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20220305094803973.png)



#### 使用

![image-20220305095700187](C:%5CUsers%5C%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20220305095700187.png)

1.声明结构体

2.声明过期时间

3.声明加密的secret

#### 生成Token

![image-20220305095937210](C:%5CUsers%5C%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20220305095937210.png)

#### 解析JWT

```go
// ParseToken 解析JWT
func ParseToken(tokenString string) (*MyClaims, error) {
   var mcl = new(MyClaims)
   // 解析token
   token, err := jwt.ParseWithClaims(tokenString, mcl, func(token *jwt.Token) (i interface{}, err error) {
      return mySecret, nil
   })
   if err != nil {
      return nil, err
   }
   if token.Valid { // 校验token
      return mcl, nil
   }
   return nil, errors.New("invalid token")
}
```

#### 中间件

```go
/ JWTAuthMiddleware 基于JWT的认证中间件
func JWTAuthMiddleware() func(c *gin.Context) {
   return func(c *gin.Context) {
      // 客户端携带Token有三种方式 1.放在请求头 2.放在请求体 3.放在URI
      // 这里假设Token放在Header的Authorization中，并使用Bearer开头
      // 这里的具体实现方式要依据你的实际业务情况决定
      authHeader := c.Request.Header.Get("Authorization")
      if authHeader == "" {
         controller.ResponseError(c, controller.CodeNeedLogin)
         c.Abort()
         return
      }
      // 按空格分割
      parts := strings.SplitN(authHeader, " ", 2)
      if !(len(parts) == 2 && parts[0] == "Bearer") {
         controller.ResponseError(c, controller.CodeInvalidToken)
         c.Abort()
         return
      }
      // parts[1]是获取到的tokenString，我们使用之前定义好的解析JWT的函数来解析它
      mc, err := jwt.ParseToken(parts[1])
      if err != nil {
         controller.ResponseError(c, controller.CodeInvalidToken)
         c.Abort()
         return
      }
      // 将当前请求的username信息保存到请求的上下文c上
      c.Set(controller.UserIDKey, mc.UserID)
      c.Next() // 后续的处理函数可以用过c.Get("username")来获取当前请求的用户信息
   }
}
```



![image-20220305112743061](C:%5CUsers%5C%E4%B8%BF%E5%89%91%E6%9D%A5%C2%B7%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20220305112743061.png)

### 4.19  MakeFile

`Makefile`由多条规则组成，每条规则主要由两个部分组成，分别是依赖的关系和执行的命令。

其结构如下所示：

```makefile
[target] ... : [prerequisites] ...
<tab>[command]
    ...
    ...
```

- targets：规则的目标
- prerequisites：可选的要生成 targets 需要的文件或者是目标。
- command：make 需要执行的命令（任意的 shell 命令）。可以有多条命令，每一条命令占一行。

#### 示例

```makefile
.PHONY: all build run gotool clean help

BINARY="bluebell"

all: gotool build

build:
	CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o ${BINARY}

run:
	@go run ./

gotool:
	go fmt ./
	go vet ./

clean:
	@if [ -f ${BINARY} ] ; then rm ${BINARY} ; fi

help:
	@echo "make - 格式化 Go 代码, 并编译生成二进制文件"
	@echo "make build - 编译 Go 代码, 生成二进制文件"
	@echo "make run - 直接运行 Go 代码"
	@echo "make clean - 移除二进制文件和 vim swap files"
	@echo "make gotool - 运行 Go 工具 'fmt' and 'vet'"
```

- `BINARY="bluebell"`是定义变量。
- `.PHONY`用来定义伪目标。不创建目标文件，而是去执行这个目标下面的命令。

### 4.41 swagger

#### gin-swagger实战

想要使用`gin-swagger`为你的代码自动生成接口文档，一般需要下面三个步骤：

1. 按照swagger要求给接口代码添加声明式注释，具体参照[声明式注释格式](https://swaggo.github.io/swaggo.io/declarative_comments_format/)。
2. 使用swag工具扫描代码自动生成API接口文档数据
3. 使用gin-swagger渲染在线接口文档页面

~~~go
go get -u github.com/swaggo/swag/cmd/swag
~~~

#### 第一步 添加注释

~~~go
package main

// @title 这里写标题
// @version 1.0
// @description 这里写描述信息
// @termsOfService http://swagger.io/terms/

// @contact.name 这里写联系人信息
// @contact.url http://www.swagger.io/support
// @contact.email support@swagger.io

// @license.name Apache 2.0
// @license.url http://www.apache.org/licenses/LICENSE-2.0.html

// @host 这里写接口服务的host
// @BasePath 这里写base path
func main() {
	r := gin.New()
	// liwenzhou.com ...
	r.Run()
}
~~~

代码中**处理请求的接口函数（通常位于controller层**）按如下方式写上注释

```go
// GetPostListHandler2 升级版帖子列表接口
// @Summary 升级版帖子列表接口
// @Description 可按社区按时间或分数排序查询帖子列表接口
// @Tags 帖子相关接口
// @Accept application/json
// @Produce application/json
// @Param Authorization header string false "Bearer 用户令牌"
// @Param object query models.ParamPostList false "查询参数"
// @Security ApiKeyAuth
// @Success 200 {object} _ResponsePostList
// @Router /posts2 [get]
func GetPostListHandler2(c *gin.Context) {
	// GET请求参数(query string)：/api/v1/posts2?page=1&size=10&order=time
	// 初始化结构体时指定初始参数
	p := &models.ParamPostList{
		Page:  1,
		Size:  10,
		Order: models.OrderTime,
	}

	if err := c.ShouldBindQuery(p); err != nil {
		zap.L().Error("GetPostListHandler2 with invalid params", zap.Error(err))
		ResponseError(c, CodeInvalidParam)
		return
	}
	data, err := logic.GetPostListNew(p)
	// 获取数据
	if err != nil {
		zap.L().Error("logic.GetPostList() failed", zap.Error(err))
		ResponseError(c, CodeServerBusy)
		return
	}
	ResponseSuccess(c, data)
	// 返回响应
}
```

上面注释中参数类型使用了`object`，`models.ParamPostList`具体定义如下

```go
// bluebell/models/params.go

// ParamPostList 获取帖子列表query string参数
type ParamPostList struct {
	CommunityID int64  `json:"community_id" form:"community_id"`   // 可以为空
	Page        int64  `json:"page" form:"page" example:"1"`       // 页码
	Size        int64  `json:"size" form:"size" example:"10"`      // 每页数据量
	Order       string `json:"order" form:"order" example:"score"` // 排序依据
}
```

#### 第二步  生成接口文档数据

```bash
swag init
```

#### 第三步：引入gin-swagger渲染文档数据

```go
import (
	_ "bluebell/docs"  // 千万不要忘了导入把你上一步生成的docs
	gs "github.com/swaggo/gin-swagger"
	"github.com/swaggo/gin-swagger/swaggerFiles"

	"github.com/gin-gonic/gin"
)
//注册swagger api相关路由
//routes.go
//r.GET("/swagger/*any", gs.WrapHandler(swaggerFiles.Handler))
```