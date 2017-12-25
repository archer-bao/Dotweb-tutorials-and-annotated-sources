# Dotweb教程与源码解读

------

DotWeb是一个快速开发 Go 应用的 HTTP 框架，是一个开源的项目，可在 github 上找到源码，地址如下：
### [DotWeb](https://github.com/devfeel/dotweb)

-----

*要求* ：
> * 安装配置 Go 开发环境：自行安装
> * 安装 Dotweb 库：go get github.com/devfeel/dotweb


*相关环境* ：
> * Ubuntu 16.04 x64
> * Go 1.9.2
> * DotWeb 1.3.7

DotWeb 版本查看：
> DotWeb源码目录下有个 version.md 文件，通过该文件可以查看到当前框架版本，通常最顶部的为当前版本。

-----

### **快速开始** ：

**第一步：**
创建文件 main.go
```
package main

import (
	"github.com/devfeel/dotweb"
)

func main() {
	//init DotApp
	app := dotweb.New()

	//set route
	app.HttpServer.GET("/index", func(ctx dotweb.Context) error {
		_, err := ctx.WriteString("welcome to my first web!")
		return err
	})

	//begin server
	app.StartServer(8888)
}
```

**第二步：** 
编译文件 main.go 并运行，在浏览器中访问结果如下图：
![first_dotweb](http://p1iazy1u3.bkt.clouddn.com/dotweb1-1.png)


------------

### **代码解读：**

上面的代码中，关键部分在于第9行、12-15行、18行；

**第9行：**
```
app := dotweb.New()
```
这一句代码用于获得一个DotWeb实例，一个DotWeb实例就是一个Web后台服务。
DotWeb 实例到底是什么样的一个存在呢？接口？自定义类型？我们可以通过阅读源码来看看，它的原型定义在 `github.com/devfeel/dotweb/dotweb.go` 文件中，如下：
```
type (
	DotWeb struct {
		HttpServer              *HttpServer
		cache                   cache.Cache
		OfflineServer           servers.Server
		Config                  *config.Config
		Middlewares             []Middleware
		ExceptionHandler        ExceptionHandle
		NotFoundHandler         StandardHandle
		MethodNotAllowedHandler StandardHandle 
		AppContext              *core.ItemContext
		middlewareMap           map[string]MiddlewareFunc
		middlewareMutex         *sync.RWMutex
	}

    ... // 省略
)
```
&emsp;&emsp;从代码中首先可以看到，它是一个自定义类型，内部组合了一些其他的结构和接口，不仅含有 `HttpServer` 模块，还有其他功能模块，比如 `cache.Cache（缓存）` 、 `config.Config（配置）` 、  `Middleware（中间件）` 、 `ExceptionHandle（异常处理）` 等等。

&emsp;&emsp;在 `github.com/devfeel/dotweb/dotweb.go` 文件中，我们同样能看到第9行用到的 `New()` 函数，该函数负责初始化一个 DotWeb 实例，并返回该实例的指针。另外，在该文件中同时可以看到 `dotweb` 结构提供了哪些方法，诸如：

* `Close() 关闭后台实例` 
* `initAppConfig() `
* `Use()`

以上三个只是简单节选列举，更多请查看源码。


&emsp;&emsp;上面的第9行代码得到了 `一个` 初始化的实例，那么我们可不可以同时获取 `两个` 并在稍后运行呢？答案是：`可以` 。代码如下：
```
package main

import (
	"github.com/devfeel/dotweb"
)

func main() {
	//init DotApp
	app := dotweb.New()
	app2 := dotweb.New()

	//set route
	app.HttpServer.GET("/index", func(ctx dotweb.Context) error {
		_, err := ctx.WriteString("welcome to my first web!")
		return err
	})
	app2.HttpServer.GET("/index2", func(ctx dotweb.Context) error {
		_, err := ctx.WriteString("welcome to my second web!")
		return err
	})

	//begin server
	go app.StartServer(8888)
	app2.StartServer(9999)
}

```

在浏览器中访问结果如下图：

![first_dotweb](http://p1iazy1u3.bkt.clouddn.com/dotweb1-2.png)

---

**12-15行**

```
app.HttpServer.GET("/index", func(ctx dotweb.Context) error {
    _, err := ctx.WriteString("welcome to my first web!")
    return err
})
```
&emsp;&emsp;这四行代码可以逻辑上视为一行，主要用来注册路由，通俗来说即是后端提供给前端的一个特定服务接口。其中 `app` 是第9行得到的后台服务实例，而 `HttpServer` 则是 `app` 实例中的 HTTP 子模块。该子模块是被内嵌于 `DotWeb` 结构中，它的原型定义在 `github.com/devfeel/dotweb/server.go` 文件中，如下：
```
type (
	//HttpServer定义
	HttpServer struct {
		stdServer      *http.Server
		router         Router
		Modules        []*HttpModule
		DotApp         *DotWeb
		sessionManager *session.SessionManager
		lock_session   *sync.RWMutex
		pool           *pool
		binder         Binder
		render         Renderer
		offline        bool
		Features       *feature.Feature
	}
    
    ... // 省略
)
```
&emsp;&emsp;和 `DotWeb` 结构类似，`HttpServer` 也是一个由多个子模块组合的自定义类型，每个子模块提供了相应的功能支持。在该文件中同样可以看到 `HttpServer` 结构提供了很多方法，目前我们 `主要关注` 其中的几个，它们分别对应了 `HTTP协议` 的几种 `method方法`，如下：

* `GET()` 
* `HEAD()`
* `PUT()`
* `POST()`
* `OPTIONS()`
* `DELETE()`
* `ANY()`

其中，最常用的两个便是 `GET()` 和 `POST()` 了。想要说明的是，这些方法设计非常简明直观，需要什么样方式的接口访问，直接使用对应的方法进行路由注册即可，如果不想要指定具体的 `method方法`，那么可以使用提供的 `ANY()` 方法。

&emsp;&emsp;继续前面的代码，我们通过 `GET` 向DotWeb实例 `app` 的 `HttpServer` 子模块注册了路由，路由的路径是 `/index`，对应的处理函数是一个匿名函数：
```
func(ctx dotweb.Context) error {
    _, err := ctx.WriteString("welcome to my first web!")
    return err
})
```
也就是说，每次当我们访问 `/index` 路由时，都会调用这个匿名函数进行对应处理，因此从最前面的截图中，我们可以看到浏览器输出了
```
welcome to my first web!
```

----

**第18行**
```
app.StartServer(8888)
```
&emsp;&emsp;这一行代码指定了DotWeb后台监听的端口 `8888` 并启动后台服务，但没有指定本地IP，默认是接受所有网络的连接，比如 `loopback（环回）` 。如果我们想只接受本地 `以太网网络` 或者 `loopback（环回）`，那么可以使用另外一个方法 `ListenAndServe()`，该函数原型如下：
```
func (app *DotWeb) ListenAndServe(addr string) error
```
我们可以这样调用并启动后台服务：
```
app.ListenAndServe("127.0.0.1:8888")
```
这样启动后台后，后台会绑定 `loopback（环回）`，只能接受来自 `127.0.0.1/24` 的数据，而其他的类似以太网络的数据都会被拒绝。
