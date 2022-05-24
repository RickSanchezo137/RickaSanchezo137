# [返回](/)

# 介绍

- 基于 Radix 树的路由，小内存占用。没有反射。可预测的 API 性能
- 传入的 HTTP 请求可以由一系列中间件和最终操作来处理。 例如：Logger，Authorization，GZIP，最终操作 DB
- Gin 可以 catch 一个发生在 HTTP 请求中的 panic 并 recover 它。这样，你的服务器将始终可用。例如，你可以向 Sentry 报告这个 panic
- Gin 可以解析并验证请求的 JSON，例如检查所需值的存在
- 更好地组织路由。是否需要授权，不同的 API 版本…… 此外，这些组可以无限制地嵌套而不会降低性能
- Gin 提供了一种方便的方法来收集 HTTP 请求期间发生的所有错误。最终，中间件可以将它们写入日志文件，数据库并通过网络发送
- Gin 为 JSON，XML 和 HTML 渲染提供了易于使用的 API

# 使用

**入门例子：**

```go
func main() {
	r := gin.Default()
	r.GET("/hello", func(context *gin.Context) {
		context.JSON(http.StatusOK, gin.H{
			"message": "start",
		})
	})
	r.Run()
}
```

Engine 是 Gin 框架最重要的数据结构，它是框架的入口。通过 Engine 对象来定义服务路由信息、组装插件、运行服务。正如 Engine 的中文意思「引擎」一样，它就是框架的核心发动机，整个 Web 服务的都是由它来驱动的

> 发动机属于精密设备，构造非常复杂，不过 Engine 对象很简单，因为引擎最重要的部分 —— 底层的 HTTP 服务器使用的是 Go 语言内置的 http server，Engine 的本质只是对内置的 HTTP 服务器的包装，让它使用起来更加便捷

gin.Default() 函数会生成一个默认的 Engine 对象，里面包含了 2 个默认的常用插件，分别是 Logger 和 Recovery，Logger 用于输出请求日志，Recovery 确保单个请求发生 panic 时记录异常堆栈日志，输出统一的错误响应

Context是web运行期间的一个上下文容器，封装了request、response以及web通信期间一些共享变量等等

## 自定义HTTP服务器

```go
func main() {
  router := gin.Default()

  s := &http.Server{
    Addr:           ":8080",
    Handler:        router,
    ReadTimeout:    10 * time.Second,
    WriteTimeout:   10 * time.Second,
    MaxHeaderBytes: 1 << 20,
  }
  s.ListenAndServe()
}
```

## HTML渲染

```go
func main() {
   r := gin.Default()
   r.LoadHTMLGlob("./templates/*")
   r.GET("demo", func(c *gin.Context) {
      c.HTML(http.StatusOK, "index.html", gin.H{
         "name": "admin",
         "pwd": "123456",
      })
   })
   r.Run()
}
```

## 获取参数

- url参数

在Gin框架中，可以通过Query来获取URL中？后面所携带的参数。例如`/name=admin&pwd=123456`。获取方法如下

```go
package main

import (
	"net/http"
	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()
	r.GET("/", func(c *gin.Context) {
		name := c.Query("name")
		pwd := c.Query("pwd")
		// fmt.Printf("name:%s ; pwd:%s",name,pwd)
		c.JSON(http.StatusOK, gin.H{
			"name": name,
			"pwd":  pwd,
		})
	})
	r.Run()
}
```

- Form参数

当前端请求的数据通过form表单提交时，例如向`/user/reset`发送了一个POST请求，获取请求数据方法如下

```go
package main

import (
	"net/http"
	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()
	r.LoadHTMLFiles("./login.html", "./index.html") //加载页面
	r.GET("/", func(c *gin.Context) {
		c.HTML(http.StatusOK, "login.html", nil)

	})
	r.POST("/", func(c *gin.Context) {
		username := c.PostForm("username") //对应h5表单中的name字段
		password := c.PostForm("password")
		c.HTML(http.StatusOK, "index.html", gin.H{
			"username": username,
			"password": password,
		})
	})
	r.Run()
}
```

- Path参数

请求的参数通过URL路径传递，例如`/user/admin`，获取请求URL路径中的参数方法如下

```go
package main

import (
	"net/http"
	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()
	r.GET("/user/:username", func(c *gin.Context) {
		username := c.Param("username")
		c.JSON(http.StatusOK, gin.H{
			"username": username,
		})
	})
	r.Run()
}
```

# 路由

基于radix树

## 普通路由

```go
r.GET("/get",func(c *gin.Context) {})
r.GET("/login",func(c *gin.Context) {})
r.POST("/login",func(c *gin.Context) {})
```

此外，还有一个可以匹配所有请求方法的Any方法如下

```go
r.Any("/test",func(c *gin.Context) {})
```

为没有配置处理函数的路由添加处理程序，默认情况下它返回404代码，以下为没有匹配到路由的请求返回的是`templates/404.html`页面

```go
r.NoRoute(func(c *gin.Context) {
		c.HTML(http.StatusNotFound,"templates/404.html",nil)
})
```

## 路由组

我们可以将拥有共同前缀URL的路由划分为一个路由组

```go
package main

import (
	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()
	user := r.Group("/user")
	user.GET("/index", func(c *gin.Context) {})
	user.POST("/login", func(c *gin.Context) {})
	r.Run()
}
```

路由组也是支持嵌套的

```
func main() {
	r := gin.Default()
	user := r.Group("/user")
	user.GET("/index", func(c *gin.Context) {})
	user.POST("/login", func(c *gin.Context) {})
	pwd:=user.Group("/pwd")
	pwd.GET("/pwd",func(c *gin.Context) {})
	r.Run()
}
```

# Gin 中间件

Gin框架允许开发者在处理请求的过程中，加入钩子函数，这个钩子函数就叫中间件。中间件适合处理一些公共的业务逻辑，比如登陆认证，权限校验，记录日志等。具体使用方法如下

```go
package main

import (
	"fmt"
	"net/http"
	"time"
	"github.com/gin-gonic/gin"
)

//定义一个中间键m1统计请求处理函数耗时
func m1(c *gin.Context) {
	fmt.Println("m1 in...")
	start := time.Now()
	// c.Next() //调用后续的处理函数
	c.Abort()//阻止调用后续的处理函数
	cost := time.Since(start)
	fmt.Printf("cost:%v\n", cost)
}

func index(c *gin.Context) {
	c.JSON(http.StatusOK, gin.H{
		"msg": "ok",
	})
}

func main() {
	r := gin.Default()
	r.GET("/", m1, index)
	r.Run()
}
```

```go
package main
    
import (
    "github.com/gin-gonic/gin"
    "log"
    "time"
)
    
func Logger() gin.HandlerFunc {
    return func(c *gin.Context) {
        t := time.Now()
    
        // 设置 example 变量
        c.Set("example", "12345")
    
        // 请求之前...
    
        c.Next()
    
        // 请求之后...
    
        latency := time.Since(t)
        // 打印请求处理时间
        log.Print(latency)
    
        // 访问即将发送的响应状态码
        status := c.Writer.Status()
        log.Println(status)
    }
}
    
func main() {
    r := gin.New()
    // 使用自定义的 Logger 中间件
    r.Use(Logger())
    
    // 定义路由
    r.GET("/test", func(c *gin.Context) {
        example := c.MustGet("example").(string)
        // 打印 example 值 12345
        log.Println(example)
    })
    
    // 监听 0.0.0.0:8080
    r.Run(":8080")
}
```

## BasicAuth中间件

```go
// 模拟一些私有数据
var secrets = gin.H{
    "foo":    gin.H{"email": "foo@bar.com", "phone": "123433"},
    "austin": gin.H{"email": "austin@example.com", "phone": "666"},
    "lena":   gin.H{"email": "lena@guapa.com", "phone": "523443"},
}
    
func main() {
    r := gin.Default()
    
    // 在 /admin 分组中使用 gin.BasicAuth() 中间件
    // 通过 gin.Accounts 来初始化一些测试用户信息（用户名/密码键值对）
    authorized := r.Group("/admin", gin.BasicAuth(gin.Accounts{
        "foo":    "bar",
        "austin": "1234",
        "lena":   "hello2",
        "manu":   "4321",
    }))
    
    // /admin/secrets 
    // 匹配 "localhost:8088/admin/secrets
    authorized.GET("/secrets", func(c *gin.Context) {
        // 获取用户，通过 BasicAuth 中间件设置
        user := c.MustGet(gin.AuthUserKey).(string)
        if secret, ok := secrets[user]; ok {
            c.JSON(http.StatusOK, gin.H{"user": user, "secret": secret})
        } else {
            c.JSON(http.StatusOK, gin.H{"user": user, "secret": "NO SECRET :("})
        }
    })
    
    // 监听 0.0.0.0:8080，等待客户端请求
    r.Run(":8088")
}
```

## 在中间件中使用Goroutine

当在中间件或 handler 中启动新的 Goroutine 时，不能使用原始的上下文，必须使用只读副本

```go
package main

import (
	"github.com/gin-gonic/gin"
	"log"
	"time"
)

func main() {
	r := gin.Default()

	r.GET("/long_async", func(c *gin.Context) {
		// 创建在 goroutine 中使用的副本
		tmp := c.Copy()
		go func() {
			// 用 time.Sleep() 模拟一个长任务。
			time.Sleep(5 * time.Second)

			// 请注意您使用的是复制的上下文 "tmp"，这一点很重要
			log.Println("Done! in path " + tmp.Request.URL.Path)
		}()
	})

	r.GET("/long_sync", func(c *gin.Context) {
		// 用 time.Sleep() 模拟一个长任务。
		time.Sleep(5 * time.Second)

		// 因为没有使用 goroutine，不需要拷贝上下文
		log.Println("Done! in path " + c.Request.URL.Path)
	})
	r.Run()
}
```

# Gin 参数绑定

为了能够更方便的获取请求相关参数，提高开发效率，我们可以使用ShouldBind，它能够基于请求自动提取JSON，Form表单，Query等类型的值，并把值绑定到指定的结构体对象，具体使用方法如下

```go
package main

import (
	"fmt"
	"net/http"
	"github.com/gin-gonic/gin"
)

type Userinfo struct {
	Username string `form:"username"`
	Password string `form:"password"`
}

func main() {
	r := gin.Default()
	r.GET("/user", func(c *gin.Context) {
		var u Userinfo
		err := c.ShouldBind(&u)
		if err != nil {
			c.JSON(http.StatusBadGateway, gin.H{
				"error": err.Error(),
			})
		} else {
			c.JSON(http.StatusOK, gin.H{
				"status": "ok",
			})
		}
		fmt.Printf("%#v\n", u)
	})
	r.Run()
}
```

ShouldBind会按照以下顺序解析请求中的数据并完成绑定：

- 如果是GET请求，只使用Form绑定引擎（Query）
- 如果是POST请求，首先检查content-type是否为JSON或XML，然后再使用Form（form-data）

Gin 框架提供了两种绑定方案：

- Must Bind
  - 方法：Bind, BindJSON, BindXML, BindQuery, BindYAML
  - 行为：这些方法会调用底层的 `MustBindWith` 方法，如果出现绑定错误，会通过 `c.AbortWithError(400, err).SetType(ErrorTypeBind)` 退出请求，如果你想要对该行为有更多的控制，请使用下面 Should Bind 这套方案
- Should Bind
  - 方法：ShouldBind, ShouldBindJSON, ShouldBindXML, ShouldBindQuery, ShouldBindYAML
  - 行为：这些方法会调用底层的 `ShouldBindWith` 方法，如果出现绑定错误，需要开发者自己来处理

当使用上述绑定方法时，Gin 框架会根据请求头 `Content-Type` 推断绑定方案，如果你对要绑定的类型非常确定，可以直接使用 `MustBindWith` 或 `ShouldBindWith` 方法

你还可以通过 `binding:"required"` 标签来指定哪些字段是必需的，如果必需字段为空会报错，示例代码如下（`src/gin-demo/examples/model_binding.go`）：

```go
package main

import (
  "github.com/gin-gonic/gin"
  "net/http"
)

// Binding from JSON
type Login struct {
  User     string `form:"user" json:"user" xml:"user"  binding:"required"`
  Password string `form:"password" json:"password" xml:"password" binding:"required"`
}

func main() {
  router := gin.Default()

  // Example for binding JSON ({"user": "manu", "password": "123"})
  router.POST("/loginJSON", func(c *gin.Context) {
    var json Login
    if err := c.ShouldBindJSON(&json); err != nil {
      c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
      return
    }

    if json.User != "xueyuanjun" || json.Password != "123456" {
      c.JSON(http.StatusUnauthorized, gin.H{"status": "unauthorized"})
      return
    } 

    c.JSON(http.StatusOK, gin.H{"status": "you are logged in"})
  })

  // Example for binding XML (
  //	<?xml version="1.0" encoding="UTF-8"?>
  //	<root>
  //		<user>user</user>
  //		<password>123</password>
  //	</root>)
  router.POST("/loginXML", func(c *gin.Context) {
    var xml Login
    if err := c.ShouldBindXML(&xml); err != nil {
      c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
      return
    }

    if xml.User != "xueyuanjun" || xml.Password != "123456" {
      c.JSON(http.StatusUnauthorized, gin.H{"status": "unauthorized"})
      return
    } 

    c.JSON(http.StatusOK, gin.H{"status": "you are logged in"})
  })

  // Example for binding a HTML form (user=manu&password=123)
  router.POST("/loginForm", func(c *gin.Context) {
    var form Login
    // This will infer what binder to use depending on the content-type header.
    if err := c.ShouldBind(&form); err != nil {
      c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
      return
    }

    if form.User != "xueyuanjun" || form.Password != "123456" {
      c.JSON(http.StatusUnauthorized, gin.H{"status": "unauthorized"})
      return
    } 

    c.JSON(http.StatusOK, gin.H{"status": "you are logged in"})
  })

  // Listen and serve on 0.0.0.0:8080
  router.Run(":8080")
}
```

# Gin 表单验证器

```go
package main

import (
  "net/http"
  "time"

  "github.com/gin-gonic/gin"
  "github.com/gin-gonic/gin/binding"
  "gopkg.in/go-playground/validator.v9"
)

// Booking 中包含了绑定的表单请求字段和验证规则
type Booking struct {
  CheckIn  time.Time `form:"check_in" binding:"required,bookabledate" time_format:"2006-01-02"`
  CheckOut time.Time `form:"check_out" binding:"required,gtfield=CheckIn" time_format:"2006-01-02"`
}

var bookableDate validator.Func = func(fl validator.FieldLevel) bool {
  date, ok := fl.Field().Interface().(time.Time)
  if ok {
    today := time.Now()
    if today.After(date) {
      return false
    }
  }
  return true
}

func getBookable(c *gin.Context) {
  var b Booking
  if err := c.ShouldBindWith(&b, binding.Query); err == nil {
    c.JSON(http.StatusOK, gin.H{"message": "预定日期有效!"})
  } else {
    c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
  }
}

func main() {
  route := gin.Default()

  // 注册新的自定义验证规则
  if v, ok := binding.Validator.Engine().(*validator.Validate); ok {
    v.RegisterValidation("bookabledate", bookableDate)
  }

  route.GET("/bookable", getBookable)
  route.Run(":8085")
}
```

启动服务器，测试结果如下：

![-w974](https://laravel.gstatics.cn/storage/uploads/images/gallery/2020-08/image-15784598817169.jpg)

- 单个文件上传

前端页面代码

```go
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <title>上传文件示例</title>
</head>
<body>
<form action="/upload" method="post" enctype="multipart/form-data">
    <input type="file" name="f1">
    <input type="submit" value="上传">
</form>
</body>
</html>
```

后端Gin框架部分代码

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
	// 处理multipart forms提交文件时默认的内存限制是32 MiB
	// 可以通过下面的方式修改
	// r.MaxMultipartMemory = 8 << 20  // 8 MiB
	r.POST("/upload", func(c *gin.Context) {
		// 单个文件
		file, err := c.FormFile("f1")
		if err != nil {
			c.JSON(http.StatusInternalServerError, gin.H{
				"message": err.Error(),
			})
			return
		}

		log.Println(file.Filename)
		dst := fmt.Sprintf("C:/tmp/%s", file.Filename)
		// 上传文件到指定的目录
		c.SaveUploadedFile(file, dst)
		c.JSON(http.StatusOK, gin.H{
			"message": fmt.Sprintf("'%s' uploaded!", file.Filename),
		})
	})
	r.Run()
}
```

- 多个文件上传

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
	// 处理multipart forms提交文件时默认的内存限制是32 MiB
	// 可以通过下面的方式修改
	// r.MaxMultipartMemory = 8 << 20  // 8 MiB
	r.POST("/upload", func(c *gin.Context) {
		// Multipart form
		form, _ := c.MultipartForm()
		files := form.File["file"]

		for index, file := range files {
			log.Println(file.Filename)
			dst := fmt.Sprintf("C:/tmp/%s_%d", file.Filename, index)
			// 上传文件到指定的目录
			c.SaveUploadedFile(file, dst)
		}
		c.JSON(http.StatusOK, gin.H{
			"message": fmt.Sprintf("%d files uploaded!", len(files)),
		})
	})
	r.Run()
}
```

# Gin 重定向

## HTTP重定向

```go
package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

func main() {
	r := gin.Default()
	r.GET("/test", func(c *gin.Context) {
		c.Redirect(http.StatusMovedPermanently, "https://www.w3cschool.cn/")
	})
	r.Run()
}
```

以上代码执行结果如下

![动画](https://atts.w3cschool.cn/attachments/image/20220302/1646185788687296.gif)

## 路由重定向

```go
package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

func main() {
	r := gin.Default()
	r.GET("/test1", func(c *gin.Context) {
		// 指定重定向的URL
		c.Request.URL.Path = "/test2"
		r.HandleContext(c)
	})
	r.GET("/test2", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{"hello": "world"})
	})
	r.Run()
}
```

以上代码执行结果如下

![img](data:image/png;base64,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)

**网页url不发生变化，不作跳转**

# Gin 设置和获取Cookie

在Gin框架中设置和获取Cookie的方法如下

```go
package main

import (
	"fmt"
	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()
	r.GET("/cookie", func(c *gin.Context) {
		cookie, err := c.Cookie("gin_cookie")
		if err != nil {
			cookie = "NotSet"
			c.SetCookie("gin_cookie", "test", 3600, "/", "localhost", false, true)
		}
		fmt.Printf("Cookie value: %s \n", cookie)
	})
	r.Run()
}
```

执行结果如下

![img](data:image/png;base64,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)

# Gin 如何记录日志

在Gin框架中记录日志方法如下

```go
package main

import (
	"io"
	"os"
	"github.com/gin-gonic/gin"
)

func main() {
	// 禁用控制台颜色，将日志写入文件时不需要控制台颜色。
	gin.DisableConsoleColor()

	// 记录到文件。
	f, _ := os.Create("gin.log")
	gin.DefaultWriter = io.MultiWriter(f)

	// 如果需要同时将日志写入文件和控制台，请使用以下代码。
	// gin.DefaultWriter = io.MultiWriter(f, os.Stdout)

	r := gin.Default()
	r.GET("/ping", func(c *gin.Context) {
		c.String(200, "pong")
	})

	r.Run()
}
```

以上代码执行结果如下![动画](https://atts.w3cschool.cn/attachments/image/20220302/1646186777447261.gif)

## 定义路由日志格式

想要记录指定格式（如 JSON、键值）的信息，可以通过 `gin.DebugPrintRouteFunc` 来定义这个格式，在下面这个例子中，我们将通过标准日志包记录所有路由信息，你也可以根据需要自定义日志格式：

```go
func main() {
  r := gin.Default()

  // 默认路由输出格式
  gin.DebugPrintRouteFunc = func(httpMethod, absolutePath, handlerName string, nuHandlers int) {
    log.Printf("endpoint %v %v %v %v\n", httpMethod, absolutePath, handlerName, nuHandlers)
  }

  r.POST("/foo", func(c *gin.Context) {
    c.JSON(http.StatusOK, "foo")
  })

  r.GET("/bar", func(c *gin.Context) {
    c.JSON(http.StatusOK, "bar")
  })

  r.GET("/status", func(c *gin.Context) {
    c.JSON(http.StatusOK, "ok")
  })

  // Listen and Server in http://0.0.0.0:8080
  r.Run()
}
```

启动该服务器，输出结果如下：

![-w867](https://laravel.gstatics.cn/storage/uploads/images/gallery/2020-08/image-15784607395887.jpg)

# Gin 绑定HTML复选框

前端代码

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
    <form action="/" method="POST">
        <p>Check some colors</p>
        <label for="red">Red</label>
        <input type="checkbox" name="colors[]" value="red" id="red" />
        <label for="green">Green</label>
        <input type="checkbox" name="colors[]" value="green" id="green" />
        <label for="blue">Blue</label>
        <input type="checkbox" name="colors[]" value="blue" id="blue" />
        <input type="submit" />
    </form>
</body>
</html>
```

Gin框架代码

```go
package main

import (
    "github.com/gin-gonic/gin"
)

type myForm struct {
    Colors []string `form:"colors[]"`
}

func main() {
    r := gin.Default()

    r.LoadHTMLGlob("views/*")
    r.GET("/", indexHandler)
    r.POST("/", formHandler)

    r.Run(":8080")
}

func indexHandler(c *gin.Context) {
    c.HTML(200, "form.html", nil)
}

func formHandler(c *gin.Context) {
    var fakeForm myForm
    c.Bind(&fakeForm)
    c.JSON(200, gin.H{"color": fakeForm.Colors})
}
```

# Gin AsciiJSON

使用 `AsciiJSON`生成具有转义的非 `ASCII`字符的 `ASCII-only JSON`

生成只包含 ASCII 字符的 JSON 格式数据，对于非 ASCII 字符会进行转义

```go
func main() {
	r := gin.Default()

	r.GET("/someJSON", func(c *gin.Context) {
		data := map[string]interface{}{
			"lang": "GO语言",
			"tag":  "<br>",
		}

		// 输出 : {"lang":"GO\u8bed\u8a00","tag":"\u003cbr\u003e"}
		c.AsciiJSON(http.StatusOK, data)
	})

	// 监听并在 0.0.0.0:8080 上启动服务
	r.Run(":8080")
}
```

# Gin HTTP2 server 推送

服务端主动推送

```go
http.Pusher` 仅支持 `go1.8+
package main

import (
	"html/template"
	"log"

	"github.com/gin-gonic/gin"
)

var html = template.Must(template.New("https").Parse(`
<html>
<head>
  <title>Https Test</title>
  <script src="/assets/app.js"></script>
</head>
<body>
  <h1 style="color:red;">Welcome, Ginner!</h1>
</body>
</html>
`))

func main() {
	r := gin.Default()
	r.Static("/assets", "./assets")
	r.SetHTMLTemplate(html)

	r.GET("/", func(c *gin.Context) {
		if pusher := c.Writer.Pusher(); pusher != nil {
			// 使用 pusher.Push() 做服务器推送
			if err := pusher.Push("/assets/app.js", nil); err != nil {
				log.Printf("Failed to push: %v", err)
			}
		}
		c.HTML(200, "https", gin.H{
			"status": "success",
		})
	})

	// 监听并在 https://127.0.0.1:8080 上启动服务
	r.RunTLS(":8080", "./testdata/server.pem", "./testdata/server.key")
}
```

# Gin JSONP

使用 `JSONP`向不同域的服务器请求数据。如果查询参数存在回调，则将回调添加到响应体中

```go
func main() {
	r := gin.Default()

	r.GET("/JSONP", func(c *gin.Context) {
		data := map[string]interface{}{
			"foo": "bar",
		}
		
		// /JSONP?callback=x
		// 将输出：x({\"foo\":\"bar\"})
		c.JSONP(http.StatusOK, data)
	})

	// 监听并在 0.0.0.0:8080 上启动服务
	r.Run(":8080")
}
```

# Gin PureJSON

通常，`JSON`使用 `unicode`替换特殊 `HTML`字符，例如 `<` 变为 `\ u003c`。如果要按字面对这些字符进行编码，则可以使用 `PureJSON`。`Go 1.6` 及更低版本无法使用此功能

```go
func main() {
	r := gin.Default()
	
	// 提供 unicode 实体
	r.GET("/json", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"html": "<b>Hello, world!</b>",
		})
	})
	
	// 提供字面字符
	r.GET("/purejson", func(c *gin.Context) {
		c.PureJSON(200, gin.H{
			"html": "<b>Hello, world!</b>",
		})
	})
	
	// 监听并在 0.0.0.0:8080 上启动服务
	r.Run(":8080")
}
```

# Gin SecureJSON

使用 `SecureJSON`防止 `json`劫持。如果给定的结构是数组值，则默认预置 `"while(1),"` 到响应体

```go
func main() {
	r := gin.Default()

	// 你也可以使用自己的 SecureJSON 前缀
	// r.SecureJsonPrefix(")]}',\n")

	r.GET("/someJSON", func(c *gin.Context) {
		names := []string{"lena", "austin", "foo"}

		// 将输出：while(1);["lena","austin","foo"]
		c.SecureJSON(http.StatusOK, names)
	})

	// 监听并在 0.0.0.0:8080 上启动服务
	r.Run(":8080")
}
```

# Gin XML/JSON/YAML/ProtoBuf 渲染

```go
func main() {
	r := gin.Default()

	// gin.H 是 map[string]interface{} 的一种快捷方式
	r.GET("/someJSON", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{"message": "hey", "status": http.StatusOK})
	})

	r.GET("/moreJSON", func(c *gin.Context) {
		// 你也可以使用一个结构体
		var msg struct {
			Name    string `json:"user"`
			Message string
			Number  int
		}
		msg.Name = "Lena"
		msg.Message = "hey"
		msg.Number = 123
		// 注意 msg.Name 在 JSON 中变成了 "user"
		// 将输出：{"user": "Lena", "Message": "hey", "Number": 123}
		c.JSON(http.StatusOK, msg)
	})

	r.GET("/someXML", func(c *gin.Context) {
		c.XML(http.StatusOK, gin.H{"message": "hey", "status": http.StatusOK})
	})

	r.GET("/someYAML", func(c *gin.Context) {
		c.YAML(http.StatusOK, gin.H{"message": "hey", "status": http.StatusOK})
	})

	r.GET("/someProtoBuf", func(c *gin.Context) {
		reps := []int64{int64(1), int64(2)}
		label := "test"
		// protobuf 的具体定义写在 testdata/protoexample 文件中。
		data := &protoexample.Test{
			Label: &label,
			Reps:  reps,
		}
		// 请注意，数据在响应中变为二进制数据
		// 将输出被 protoexample.Test protobuf 序列化了的数据
		c.ProtoBuf(http.StatusOK, data)
	})

	// 监听并在 0.0.0.0:8080 上启动服务
	r.Run(":8080")
}
```

# Gin 优雅地重启或停止

你想优雅地重启或停止 web 服务器吗？有一些方法可以做到这一点

我们可以使用 `fvbock/endless` 来替换默认的 `ListenAndServe`

```
router := gin.Default()
router.GET("/", handler)
// [...]
endless.ListenAndServe(":4242", router)
```

替代方案：

- `manners`：可以优雅关机的 Go Http 服务器
- `graceful`：`Graceful`是一个 Go 扩展包，可以优雅地关闭 http.Handler 服务器
- `grace`：Go 服务器平滑重启和零停机时间部署

如果你使用的是 Go 1.8，可以不需要这些库！考虑使用 `http.Server` 内置的 `Shutdown()` 方法优雅地关机. 请参阅 gin 完整的 [graceful-shutdown](https://github.com/gin-gonic/examples/tree/master/graceful-shutdown) 示例

```go
// +build go1.8

package main

import (
	"context"
	"log"
	"net/http"
	"os"
	"os/signal"
	"time"

	"github.com/gin-gonic/gin"
)

func main() {
	router := gin.Default()
	router.GET("/", func(c *gin.Context) {
		time.Sleep(5 * time.Second)
		c.String(http.StatusOK, "Welcome Gin Server")
	})

	srv := &http.Server{
		Addr:    ":8080",
		Handler: router,
	}

	go func() {
		// 服务连接
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatalf("listen: %s\n", err)
		}
	}()

	// 等待中断信号以优雅地关闭服务器（设置 5 秒的超时时间）
	quit := make(chan os.Signal)
	signal.Notify(quit, os.Interrupt)
	<-quit
	log.Println("Shutdown Server ...")

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	if err := srv.Shutdown(ctx); err != nil {
		log.Fatal("Server Shutdown:", err)
	}
	log.Println("Server exiting")
}
```