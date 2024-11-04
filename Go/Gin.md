```toc
```
路由是指：将用户的请求url映射到特定的处理程序/功能上。我们需要根据url以及HTTP方法确定路由。
## 第一个Gin程序
`/`为默认的路径
```go
// 第一个Gin程序
func main() {
	r := gin.Default()                // 创建默认的路由实例
	r.GET("/", func(c *gin.Context) { // 定义路由
		c.String(200, "Hello Gin") // 设置响应体的内容
	})
	r.Run() // 默认监听8080端口
}
```

![QQ_1730442430236.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/QQ_1730442430236.png)

## 参数解析
通过c.Param()提取url中的参数，比如以下程序将提取name参数
```go
package main

import (
    "net/http"
    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default()
    r.GET("/user/:name", func(c *gin.Context) {
        name := c.Param("name")
        c.String(http.StatusOK, "Hello %s", name)
    })
    r.Run() // 启动服务
}
```

![QQ_1730442545647.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/QQ_1730442545647.png)
### Query解析
url中的?表示Query，通过gin.Context.Query()进行解析：
```go
// 匹配users?name=xxx&role=xxx，role可选  
r.GET("/users", func(c *gin.Context) {  
	name := c.Query("name")  
	role := c.DefaultQuery("role", "teacher")  
	c.String(http.StatusOK, "%s is a %s", name, role)  
})
```

![QQ_1730442932158.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/QQ_1730442932158.png)

DefaultQuery()多了一个形参表示默认值，如果Query无法查询到参数，则使用默认值。

有些数据将以表单的形式被提交，如用户登陆、注册、搜索等。如果以GET的方式进行请求，那么这些数据将出现在Query中。

POST请求中，表单数据通常出现在请求体中，需要使用其他进行解析。

### 请求体解析
```go
r.POST("/form", func(c *gin.Context) {  
	username := c.PostForm("username")  
	password := c.DefaultPostForm("password", "000000") // 可设置默认值  
  
	c.JSON(http.StatusOK, gin.H{  
		"username": username,  
		"password": password,  
	})  
})
```

![QQ_1730443277140.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/QQ_1730443277140.png)

### map数据解析
通过QueryMap解析Query中的map数据，通过PostFormMap解析请求体中的map数据。
```go
r.POST("/post", func(c *gin.Context) {  
	ids := c.QueryMap("ids")  
	names := c.PostFormMap("names")

	c.JSON(http.StatusOK, gin.H{
		"ids":   ids,  
		"names": names,  
	})  
})
```

![QQ_1730443982949.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/QQ_1730443982949.png)
## 重定向
两种方式：
1. HTTP重定向：使用`c.Redirect()`将请求重定向，以下代码表示：设置状态为301状态，并重定向到`/index`路由
2. 路由重定向：直接修改url，并立即处理该url。该方法不会修改地址栏的url，而是直接将请求逻辑进行转发
```go
r.GET("/redirect", func(c *gin.Context) {
	c.Redirect(http.StatusMovedPermanently, "/index")
})

r.GET("/goindex", func(c *gin.Context) {
	c.Request.URL.Path = "/"
	r.HandleContext(c)
})

r.GET("/index", func(c *gin.Context) {
	c.String(http.StatusOK, "这是/redirect的重定向")
})

r.GET("/", func(c *gin.Context) {
	c.String(http.StatusOK, "这是/goindex的重定向")
})
```

![QQ_1730444485057.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/QQ_1730444485057.png)

![QQ_1730444399184.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/QQ_1730444399184.png)
## 分组路由
分组路由可以简化权限管理，让代码更加模块化，清晰。

主要优势在于：可以对一类具有相同前缀或中间件的路由进行集中处理。
```go
r := gin.Default()
// 定义一个处理函数
defaultHandler := func(c *gin.Context) {
	c.JSON(http.StatusOK, gin.H{
		"path": c.FullPath(),
	})
}
// 通过路由引擎的Group()方法创建分组
v1 := r.Group("/v1")
{
	// 可以在某个组中，定义路由
	v1.GET("/posts", defaultHandler)
	v1.GET("/series", defaultHandler)
}
v2 := r.Group("/v2")
{
	v2.GET("/posts", defaultHandler)
	v2.GET("/series", defaultHandler)
}
```

此外，路由组还支持嵌套。比如v1路由组下还能定义新的路由组。

![QQ_1730445186068.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/QQ_1730445186068.png)

![QQ_1730445307544.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/QQ_1730445307544.png)
## 文件上传
### 单文件
主要就是通过FormFile获取请求中的文件，接着通过路由引擎的SaveUploadedFile()上传文件到指定目录。

```go
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
		dst := fmt.Sprintf("/tmp/%s", file.Filename)
		// 上传文件到指定的目录
		c.SaveUploadedFile(file, dst)
		c.JSON(http.StatusOK, gin.H{
			"message": fmt.Sprintf("'%s' uploaded!", file.Filename),
		})
	})
	r.Run()
}
```
### 多文件
主要就是通过MultipartForm获取整个表单信息form，接着通过form.File获取文件列表，并通过`[file]`指定需要获取多文件列表。

获取多个文件后，for range遍历这些文件并保存到本地。

```go
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
			dst := fmt.Sprintf("/tmp/%s_%d", file.Filename, index)
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

![QQ_1730448541910.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/QQ_1730448541910.png)
单文件与多文件都能成功上传。

## 中间件
定义路由时，在处理函数之前传入中间件。调用处理函数前，将先调用中间件函数。

![QQ_1730452135930.png](https://raw.githubusercontent.com/ren77281/pigco-image/main/img/QQ_1730452135930.png)

c.Next()将调用当时声明的处理函数，而c.Abort()将阻止后续的处理函数调用。

注意：在中间件或者处理函数中使用Context时，需要使用Copy()拷贝，只能使用拷贝后的副本。

## ShouldBind
使用ShouldBind可以提取表单，Query，JSON中的数据，并绑定到结构体中

## Cookie
通过Context.Cookie()获取指定名称的Cookie
```go
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
## 日志
`io.MultiWriter()`将接收多个`io.Writer`，并将输出写入所有的`io.Writer`。

接着Gin的默认日志写入器将被设置为这些`io.Writer`

```go
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

