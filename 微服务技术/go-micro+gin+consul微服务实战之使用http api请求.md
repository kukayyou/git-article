本文是基于上一篇【go-micro+gin+consul微服务实战之服务注册与发现】的，没看过的同学，请移步：https://www.jianshu.com/p/757dc1bb3930

我们在使用微服务构建系统时，必然会用到http api，下面介绍下，在如何使用go-micro自带的http库构建http api 请求

我们还使用上一篇【go-micro+gin+consul微服务实战之服务注册与发现】中的orderserver和userserver作为示例。

看过上一篇的，就会注意到我们有一段代码使用了http请求，如下：
 ```
//获取服务地址
    hostAddress := GetServiceAddr("userserver")
    if len(hostAddress) <= 0{
        fmt.Println("hostAddress is null")
    }else{
        url := "http://"+ hostAddress + "/users"
        response, _ := http.Post(url, "application/json;charset=utf-8",bytes.NewBuffer([]byte("")))

        fmt.Println(response)
    }
```
这里我们使用的是 golang的【net/http】包，来实现的，过程相对复杂，因为需要使用```GetServiceAddr("userserver")```现获取到userserver的地址。
在GetServiceAddr中我们使用for的方式来获取，代码复杂，如下：
```
func GetServiceAddr(serviceName string)(address string){
	var retryCount int
	for{
		servers,err :=consulReg.GetService(serviceName)
		if err !=nil {
			fmt.Println(err.Error())
		}
		var services []*registry.Service
		for _,value := range servers{
			fmt.Println(value.Name, ":", value.Version)
			services = append(services, value)
		}
		next := selector.RoundRobin(services)
		if node , err := next();err == nil{
			address = node.Address
		}
		if len(address) > 0{
			return
		}
		//重试次数++
		retryCount++
		time.Sleep(time.Second * 1)
		//重试5次为获取返回空
		if retryCount >= 5{
			return
		}
	}
}
```

下面我们使用go-micro中自带的http包来实现http api请求
设计的包有
 ```
"github.com/micro/go-micro/client"
"github.com/micro/go-micro/client/selector"
"github.com/micro/go-plugins/client/http"
```
实现代码如下：
 ```
func main() {
	//初始化路由
	ginRouter := routers.InitRouters()
	//新建一个consul注册的地址，也就是我们consul服务启动的机器ip+端口
	consulReg = consul.NewRegistry(
		registry.Addrs("192.168.109.131:8500"),
	)

	//注册服务
	microService:= web.NewService(
		web.Name("orderserver"),
		//web.RegisterTTL(time.Second*30),//设置注册服务的过期时间
		//web.RegisterInterval(time.Second*20),//设置间隔多久再次注册服务
		web.Address(":18002"),
		web.Handler(ginRouter),
		web.Registry(consulReg),
		)

	microselector := selector.NewSelector(
		selector.Registry(consulReg),//传入consul注册
		selector.SetStrategy(selector.RoundRobin),//指定查询机制
	)
	microClient := microhttp.NewClient(
		client.Selector(microselector),
		client.ContentType("application/json"))
	req:=microClient.NewRequest("userserver", "/users",map[string]string{})
	var resp map[string]interface{}

	err := microClient.Call(context.Background(), req, &resp)
	if err == nil{
		fmt.Println(resp)
	}

	microService.Run()
}
```

核心代码是这段
```
microselector := selector.NewSelector(
		selector.Registry(consulReg),//传入consul注册
		selector.SetStrategy(selector.RoundRobin),//指定查询机制
	)
	microClient := microhttp.NewClient(
		client.Selector(microselector),
		client.ContentType("application/json"))
	req:=microClient.NewRequest("userserver", "/users",map[string]string{})
	var resp map[string]interface{}

	err := microClient.Call(context.Background(), req, &resp)
	if err == nil{
		fmt.Println(resp)
	}
```

这里需要注意，这种方式是依赖consul的，```consulReg = consul.NewRegistry(
       registry.Addrs("192.168.109.131:8500"),
   )```

这种实现共分为3步
```
第一步，构建selectort，通过指定selector.Registry(consulReg)就会去consul服务请求服务的地址；selector.SetStrategy(selector.RoundRobin)是指定获取服务的策略纬轮询的方式
第二步，构建request请求，这里指定selector和ContentType，请求的服务名和接口名，map[string]string{}为请求参数，这里我们传空
第三步，请求接口，调用Call函数，传入请求，第一个参数默认传空，第三个参数是响应的返回数据，这个是根据我们返回的数据格式定义的，我们在userserver的/users接口中返回的数据如下：
"status": "1",
"data":   "get userinfos",
所以这里我们把resp定义为map[string]interface{}，你也可以根据自己的返回定义
```
到这里就完成请求http api 的实现，运行一下就可以看到效果
```
map[data:get userinfos status:1]
```

orderserver的完整代码如下，userserver的可以在【go-micro+gin+consul微服务实战之服务注册与发现】中查看
orderserver代码结构如下：

![image](https://upload-images.jianshu.io/upload_images/13833591-4cfeb06bdec06b9b.png?imageMogr2/auto-orient/strip|imageView2/2/w/227)
```main.go代码如下```
```
package main

import (
	"context"
	"fmt"
	"github.com/micro/go-micro/client"
	"github.com/micro/go-micro/client/selector"
	"github.com/micro/go-micro/registry"
	"github.com/micro/go-micro/web"
	microhttp "github.com/micro/go-plugins/client/http"
	"github.com/micro/go-plugins/registry/consul"
	"orderserver/routers"
	"time"
)

var consulReg registry.Registry

func init(){

}

func main() {
	//初始化路由
	ginRouter := routers.InitRouters()
	//新建一个consul注册的地址，也就是我们consul服务启动的机器ip+端口
	consulReg = consul.NewRegistry(
		registry.Addrs("192.168.109.131:8500"),
	)

	//注册服务
	microService:= web.NewService(
		web.Name("orderserver"),
		//web.RegisterTTL(time.Second*30),//设置注册服务的过期时间
		//web.RegisterInterval(time.Second*20),//设置间隔多久再次注册服务
		web.Address(":18002"),
		web.Handler(ginRouter),
		web.Registry(consulReg),
		)

	microselector := selector.NewSelector(
		selector.Registry(consulReg),//传入consul注册
		selector.SetStrategy(selector.RoundRobin),//指定查询机制
	)
	microClient := microhttp.NewClient(
		client.Selector(microselector),
		client.ContentType("application/json"))
	req:=microClient.NewRequest("userserver", "/users",map[string]string{})
	var resp map[string]interface{}

	err := microClient.Call(context.Background(), req, &resp)
	if err == nil{
		fmt.Println(resp)
	}

	microService.Run()
}
```

```router.go代码如下```
```
package routers

import "github.com/gin-gonic/gin"

func InitRouters() *gin.Engine {
    ginRouter := gin.Default()
    ginRouter.POST("/orders/", func(context *gin.Context) {
        context.String(200, "get orderinfos")
    })

    return ginRouter
}
```