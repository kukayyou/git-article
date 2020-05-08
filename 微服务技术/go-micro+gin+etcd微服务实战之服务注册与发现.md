在构建微服务时，使用服务发现可以减少配置的复杂性，本文以go-micro为微服务框架，使用etcd作为服务发现服务，使用gin开发golang服务。

使用gin 的原因是gin能够很好的和go-micro进行集成。

本文主要介绍服务注册和发现的实现

关于如何搭建etcd服务可以移步：https://www.jianshu.com/p/ec0e4911236d

本文默认以搭建好了etcd服务，服务的地址是：192.168.109.131:12379
如果你搭建好了自己的etcd服务，可以按照上面文章的步骤做，会看到如下界面：
![图片.png](https://upload-images.jianshu.io/upload_images/13833591-a08df32097b1ed3d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这里我的etcd服务启用了 3个节点。

####开撸
####服务注册
我们预设两个server，userserver和orderserver
下面开始上代码：
userserver程序结果如下：
![图片.png](https://upload-images.jianshu.io/upload_images/13833591-1ff65a65f1a853ad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
有两个文件router.go和main.go
```main.go代码如下```
```main.go实现初始化路由，服务注册```
```
package main

import (
	"github.com/micro/go-micro/registry"//
	"github.com/micro/go-micro/web"//
	"github.com/micro/go-micro/registry/etcd"//
	"userserver/routers"
)

var etcdReg registry.Registry

func  init()  {
	//新建一个consul注册的地址，也就是我们consul服务启动的机器ip+端口
	etcdReg = etcd.NewRegistry(
		registry.Addrs("192.168.109.131:12379"),
	)
}

func main() {
	//初始化路由
	ginRouter := routers.InitRouters()

	//注册服务
	microService:= web.NewService(
		web.Name("api.tutor.com.userserver"),
		//web.RegisterTTL(time.Second*30),//设置注册服务的过期时间
		//web.RegisterInterval(time.Second*20),//设置间隔多久再次注册服务
		web.Address(":18001"),
		web.Handler(ginRouter),
		web.Registry(etcdReg ),
		)

	microService.Run()
}
```
```router.go代码如下```
```router.go主要用来定义程序的api接口，使用gin开发```
```
package routers

import "github.com/gin-gonic/gin"

func InitRouters() *gin.Engine {
	ginRouter := gin.Default()
	ginRouter.POST("/users/", func(context *gin.Context) {
		context.String(200, "get userinfos")
	})

	return ginRouter
}
```
注册的代码就写好了，启动userserver，我们在micro的服务界面，可以看到如下效果：
![图片.png](https://upload-images.jianshu.io/upload_images/13833591-c722dd2a9014154b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

说明我们注册成功了

####服务发现
服务发现，就是从etcd中获取到我们注册进去的服务，这样在调用别的服务时，就不用从配置文件获取，直接查询etcd即可。

orderserver我们除了实现服务注册外，也实现服务发现的功能
orderserver代码结构如下：
![图片.png](https://upload-images.jianshu.io/upload_images/13833591-4cfeb06bdec06b9b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上代码
```main.go代码如下```
```
package main

import (
	"bytes"
	"fmt"
	"github.com/micro/go-micro/client/selector"
	"github.com/micro/go-micro/registry"
	"github.com/micro/go-micro/web"
	"github.com/micro/go-micro/registry/etcd"
	"net/http"
	"orderserver/routers"
	"time"
)

var etcdReg registry.Registry

func init(){
	//新建一个consul注册的地址，也就是我们consul服务启动的机器ip+端口
	etcdReg = etcd.NewRegistry(
		registry.Addrs("192.168.109.131:12379"),
	)
}

func main() {
	//初始化路由
	ginRouter := routers.InitRouters()

	//注册服务
	microService:= web.NewService(
		web.Name("api.tutor.com.orderserver"),
		//web.RegisterTTL(time.Second*30),//设置注册服务的过期时间
		//web.RegisterInterval(time.Second*20),//设置间隔多久再次注册服务
		web.Address(":18002"),
		web.Handler(ginRouter),
		web.Registry(etcdReg ),
		)

    //获取服务地址
	hostAddress := GetServiceAddr("api.tutor.com.userserver")
	if len(hostAddress) <= 0{
		fmt.Println("hostAddress is null")
	}else{
		url := "http://"+ hostAddress + "/users"
		response, _ := http.Post(url, "application/json;charset=utf-8",bytes.NewBuffer([]byte("")))

		fmt.Println(response)
	}

	microService.Run()
}

func GetServiceAddr(serviceName string)(address string){
	var retryCount int
	for{
		servers,err :=etcdReg.GetService(serviceName)
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
```GetServiceAddr就是服务发现的代码```
```
首先，使用servers,err :=consulReg.GetService(serviceName)获取注册的服务
返回的servers是个slice

然后，使用next := selector.RoundRobin(services)获取其中一个服务的信息

这里注意：
在老版本中可以直接使用selector.RoundRobin(services)，但是在v2版本中需要做个转换处理：
var services []*registry.Service
for _,value := range servers{
	fmt.Println(value.Name, ":", value.Version)
	services = append(services, value)
}
因为使用的数据结构不同，感兴趣的可以细看下区别。
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
启动oerderserver 我们就能获取到userserver的地址，各位可以调试看下效果。

今天go-micro+gin+etcd微服务实战就介绍完了，是不是很简单
