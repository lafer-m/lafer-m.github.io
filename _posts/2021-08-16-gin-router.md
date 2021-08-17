---
layout: post
title: gin web框架详解
tags: [golang]
---

gin框架的使用及图解分析

### gin简单使用
gin.Default() 返回的gin.Engine实现了http.Handler接口，即是实现了ServeHTTP(ResponseWriter, *Request)方法， 所以我们可以将gin router当作一个简单的http handler来使用， gin同样可以作为handler无缝插入到其他的web框架中。  
https://github.com/gin-gonic/gin 建议可以详读一下gin的源码。
  
```
package main

import "gopkg.in/gin-gonic/gin.v1"

func main() {
    router := gin.Default()        
    http.ListenAndServe(":8080", router)    
}

```

### gin路由
http request => gin.Context => gin.methodTrees => walk(gin.node)获取handlersChain => gin.Context.Next()进入中间件处理和业务处理。  

gin路由最关键的数据结构就是node了，node存储了对应一种method的所有路由及路由处理链函数。  
图解gin路由树:      
/  
/test/*files          *通配符，表示匹配所有，限制在url最后一个/后面，如/ttt*files或者/*files/test这样的格式都是不支持的。  
/abc/:id/test       :匹配符，两个/之间最多支持一个匹配。  
/abd/:id/test  

wildChild节点总是在子节点数组的最后一个，且一个节点只能有一个  
节点保存了该路由所有的处理函数链，包含了已经注册了的各种中间件等。  

![gin-router.png](http://www.mrzzjiy.cn/assets/gin-router.png)


### gin路由及中间件使用示例
支持路由组，中间件支持到单接口级别，及获取params/uri参数等，详细参考如下的示例代码
```
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

// 针对api级别的中间处理
func FackerApiMiddle() gin.HandlerFunc {
	return func(c *gin.Context) {
		c.Set("test", "test")
		c.Next()
	}
}

// 虚拟的handler
func FakerJsonHandler(ctx *gin.Context) {
	name := ctx.Param("name")
	// 获取query参数  xxx?test=xxx&a=c
	_ = ctx.Query("test")
	_ = ctx.DefaultQuery("a", "b")
	ctx.JSON(200, gin.H{"status": 200, "msg": fmt.Sprintf("hello, world! %s", name)})
}

// 针对api组级别的中间处理
func FackerHeaderMiddle() gin.HandlerFunc {
	return func(c *gin.Context) {
		token := c.GetHeader("Authorizion")
		if token == "" {
			c.AbortWithStatusJSON(http.StatusForbidden, gin.H{"status": 200, "msg": "not found token"})
		}
		c.Set("header_token", token)
		c.Next()
        
        // 可以继续做一些操作
	}
}

// 针对api组级别的中间处理
func FackerCookieMiddle() gin.HandlerFunc {
	return func(c *gin.Context) {
		cookie, err := c.Cookie("token")
		if err != nil || cookie == "" {
			c.AbortWithStatusJSON(http.StatusForbidden, gin.H{"status": 403, "msg": "not found cookie token"})
		}
		c.Set("token", cookie)
		c.Next()
	}
}

func main() {
	// include Logger() Recover()中间件
	// router := gin.Default()

	// 不包含任何中间件
	router := gin.New()
    // 注册全局中间件
    router.Use(gin.Recovery())

    // 组级别中间件
    groupV1 := router.Group("/v1", gin.Logger(), FackerCookieMiddle())
	groupV2 := router.Group("/v2", FackerHeaderMiddle())

    // curl会返回 {"msg":"hello, world! test","status":200}
	router.GET("/v3/:name", FakerJsonHandler)

    // api 方法
	groupV1.GET("/test", FakerJsonHandler)
    // api 级别中间件及方法
	groupV2.GET("/test1", FackerApiMiddle(), FakerJsonHandler)

	http.ListenAndServe(":8081", router)
}
```


### 日志使用示例
默认的Logger()是通过中间件的形式插入到gin router中的，当然也就可以自定义中间件来使用任何日志框架， 都可以以中间件的形式进行记录。

```

// 日志记录到 MongoDB
func LoggerToMongo() gin.HandlerFunc {
    // init mongoDB conn
	return func(c *gin.Context) {
		
	}
}

// 日志记录到 ES
func LoggerToES() gin.HandlerFunc {
    // init ES conn
	return func(c *gin.Context) {

	}
    }
```
使用默认日志中间件写日志到文件
```
func main() {
    // 写文件的时候不需要颜色
    gin.DisableConsoleColor()

    // Logging to a file.
    f, _ := os.Create("gin.log")
    gin.DefaultWriter = io.MultiWriter(f)

    // 输出到标准输出和文件
    // gin.DefaultWriter = io.MultiWriter(f, os.Stdout)

    router := gin.Default()
    router.GET("/ping", func(c *gin.Context) {
        c.String(200, "pong")
    })

    router.Run(":8080")
}
```

###  上传文件
curl -X POST http://localhost:8080/upload \
  -F "file=@/Users/appleboy/test.zip" \
  -H "Content-Type: multipart/form-data"

```
func main() {
	router := gin.Default()
	// Set a lower memory limit for multipart forms (default is 32 MiB)
	router.MaxMultipartMemory = 8 << 20  // 8 MiB
	router.POST("/upload", func(c *gin.Context) {
		// single file
		file, _ := c.FormFile("file")
		log.Println(file.Filename)

		// Upload the file to specific dst.
		c.SaveUploadedFile(file, dst)

		c.String(http.StatusOK, fmt.Sprintf("'%s' uploaded!", file.Filename))
	})
	router.Run(":8080")
}
```


### 数据绑定及验证
数据binding分为两类：  
一类是mustBind, 方法如Bind() BindJSON() 等等，出现错误会返回400及text/plain的http返回。  
第二类是ShouldBind开头的，方法如ShouldBind, ShouldBindJSON, ShouldBindXML等等，出现错误，会返回给调用者。


```
// json绑定
type Login struct {
	User     string `form:"user" json:"user" xml:"user"  binding:"required"`
	Password string `form:"password" json:"password" xml:"password" binding:"required"`
}

func main() {
	router := gin.Default()

	// json ({"user": "manu", "password": "123"})
	router.POST("/loginJSON", func(c *gin.Context) {
		var json Login
		if err := c.ShouldBindJSON(&json); err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
			return
		}

		if json.User != "manu" || json.Password != "123" {
			c.JSON(http.StatusUnauthorized, gin.H{"status": "unauthorized"})
			return
		}

		c.JSON(http.StatusOK, gin.H{"status": "you are logged in"})
	})

	// Example for binding XML (
	//	<?xml version="1.0" encoding="UTF-8"?>
	//	<root>
	//		<user>manu</user>
	//		<password>123</password>
	//	</root>)
    router.POST("/loginXML", func(c *gin.Context) {
		var xml Login
		if err := c.ShouldBindXML(&xml); err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
			return
		}

		if xml.User != "manu" || xml.Password != "123" {
			c.JSON(http.StatusUnauthorized, gin.H{"status": "unauthorized"})
			return
		}

		c.JSON(http.StatusOK, gin.H{"status": "you are logged in"})
	})

	// Listen and serve on 0.0.0.0:8080
	router.Run(":8080")
}
```
示例请求

```
$ curl -v -X POST \
  http://localhost:8080/loginJSON \
  -H 'content-type: application/json' \
  -d '{ "user": "manu" }'
> POST /loginJSON HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.51.0
> Accept: */*
> content-type: application/json
> Content-Length: 18
>
* upload completely sent off: 18 out of 18 bytes
< HTTP/1.1 400 Bad Request
< Content-Type: application/json; charset=utf-8
< Date: Fri, 04 Aug 2017 03:51:31 GMT
< Content-Length: 100
<
{"error":"Key: 'Login.Password' Error:Field validation for 'Password' failed on the 'required' tag"}
```
如上的请求报错了，是因为定义password字段的时候设置了binding:"required" tag，是必须要传的，如果设置为binding:"-" ， 则不会出现上述的错误。



自定义字段验证， gin使用了https://github.com/go-playground/validator这个开源库来做验证，详细可以参考下这个库的文档。


```
package main

import (
	"net/http"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/gin-gonic/gin/binding"
	"github.com/go-playground/validator/v10"
)

// Booking contains binded and validated data.
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

func main() {
	route := gin.Default()

    // 注册验证函数
	if v, ok := binding.Validator.Engine().(*validator.Validate); ok {
		v.RegisterValidation("bookabledate", bookableDate)
	}
    route.GET("/bookable", getBookable)
	route.Run(":8085")
}

func getBookable(c *gin.Context) {
	var b Booking
	if err := c.ShouldBindWith(&b, binding.Query); err == nil {
		c.JSON(http.StatusOK, gin.H{"message": "Booking dates are valid!"})
	} else {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
	}
}
```

返回

```
$ curl "localhost:8085/bookable?check_in=2030-04-16&check_out=2030-04-17"
{"message":"Booking dates are valid!"}

$ curl "localhost:8085/bookable?check_in=2030-03-10&check_out=2030-03-09"
{"error":"Key: 'Booking.CheckOut' Error:Field validation for 'CheckOut' failed on the 'gtfield' tag"}

$ curl "localhost:8085/bookable?check_in=2000-03-09&check_out=2000-03-10"
{"error":"Key: 'Booking.CheckIn' Error:Field validation for 'CheckIn' failed on the 'bookabledate' tag"}%
```

### gin opentracing 链路追踪的方案
https://github.com/jaegertracing/jaeger-client-go
OpenTracing的数据模型: Trace / Span

Trace: 调用链, 其中包含了多个Span.
Span: 跨度, 计量的最小单位, 每个跨度都有开始时间与截止时间. Span和Span之间可以存在References(关系): ChildOf 与 FollowsFrom

```
单个 Trace 中，span 间的因果关系


        [Span A]  ←←←(the root span)
            |
     +------+------+
     |             |
 [Span B]      [Span C] ←←←(Span C 是 Span A 的孩子节点, ChildOf)
     |             |
 [Span D]      +---+-------+
               |           |
           [Span E]    [Span F] >>> [Span G] >>> [Span H]
                                       ↑
                                       ↑
                                       ↑
                         (Span G 在 Span F 后被调用, FollowsFrom)
```

* 如果是跟踪grpc调用，则可以通过grpc中间件及metadata来传递trace span信息
* 当然也是可以在代码中进行函数跟踪，gin中可以通过自定义中间件来进行跟踪

[参考资料](https://www.jianshu.com/p/b5cd7b07a24e)