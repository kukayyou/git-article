本文介绍使用gin+go-micro+consul搭建微服务

go-micro git地址：https://github.com/micro/go-micro
安装指令：go get -u github.com/micro/go-micro/v2
原来go-micro consul的支持已经迁移到了go-plugins里面

gin git地址：https://github.com/gin-gonic/gin
安装指令：go get -u github.com/gin-gonic/gin

ubuntu+docker+consul 使用说明

1. 可以使用虚拟机或者实体机安装ubuntu，版本选择14.04即可，安装指南，可参考链接：https://www.jianshu.com/p/0f0ed7d8e06e
2. ubuntu安装成功后安装docker，参考链接：https://www.runoob.com/docker/ubuntu-docker-install.html
3. 安装及使用consul（参考资料：https://yq.aliyun.com/articles/696142）
	a. docker pull consul（拉取最新版本的consul镜像）
	b. 启动一个名为consul_server_1的Docker容器
docker run -d -p 8500:8500 -v /data/consul:/consul/data -e CONSUL_BIND_INTERFACE='eth0' --name=consul_server_1  213e00e87c53 agent -server -bootstrap -ui -node=1 -client='0.0.0.0'
	注意：213e00e87c53为consul的镜像hash，每台机器都不同，可以通过docker ps -a 查看，如下：
	REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
	consul              latest              213e00e87c53        5 weeks ago         116MB
	hello-world         latest              bf756fb1ae65        3 months ago        13.3kB

	
使用docker container ls 可以查看容器启动情况
CONTAINER ID   IMAGE         COMMAND                  CREATED         STATUS         PORTS     																	NAMES
a75b434ca6df   213e00e87c53  "docker-entrypoint.s…"   53 seconds ago  Up 52 secon    8300-8302/tcp, 8301-8302/udp, 8600/tcp, 8600/udp, 0.0.0.0:8500->8500/tcp   consul_server_1

	c. 查看consul管理界面，http://192.168.109.131:8500/ui，192.168.109.131是ubuntu所在机器的ip

	d. Consul 命令简单介绍
		agent : 表示启动 Agent 进程。
		-server：表示启动 Consul Server 模式。
		-client：表示启动 Consul Cilent 模式。
		-bootstrap：表示这个节点是 Server-Leader ，每个数据中心只能运行一台服务器。技术角度上讲 Leader 是通过 Raft 算法选举的，但是集群第一次启动时需要一个引导 Leader，在引导群集后，建议不要使用此标志。
		-ui：表示启动 Web UI 管理器，默认开放端口 8500，所以上面使用 Docker 命令把 8500 端口对外开放。
		-node：节点的名称，集群中必须是唯一的。
		-client：表示 Consul 将绑定客户端接口的地址，0.0.0.0 表示所有地址都可以访问。
		-join：表示加入到某一个集群中去。 如：-join=192.168.109.131
		
	e. 查看consul集群状态
		docker exec -t consul_server_1 consul members
		
		Node  Address          Status  Type    Build  Protocol  DC   Segment
		1     172.17.0.2:8301  alive   server  1.7.2  2         dc1  <all>
		
		Status表示它们的状态，都是alive。Type表示它们的类型，DC表示数据中心，是dc1
		Address是引导consul的ip，创建consul集群时会用到这个地址
		
	f. 下面再添加两个节点，命名为 -node=2 、-node=3
	
		指令如下：
		docker run -d -e CONSUL_BIND_INTERFACE='eth0' --name=consul_server_2 213e00e87c53 agent -server -node=2 -join='172.17.0.2'
		docker run -d -e CONSUL_BIND_INTERFACE='eth0' --name=consul_server_3 213e00e87c53 agent -server -node=3 -join='172.17.0.2'
		
	g. 将Client加入集群
		Client 在 Consul 集群中起到了代理 Server 的作用，Client 模式不持久化数据。一般情况每台应用服务器都会安装一个 Client ，这样可以减轻跨服务器访问带来性能损耗。也可以减轻 Server的请求压力。
		
		指令如下：
		docker run -d -e CONSUL_BIND_INTERFACE='eth0' --name=consul_server_4 213e00e87c53 agent -client -node=clint -join='172.17.0.2' -client='0.0.0.0'
		docker run -d -e CONSUL_BIND_INTERFACE='eth0' --name=consul_server_5 213e00e87c53 agent -client -node=clint2 -join='172.17.0.2' -client='0.0.0.0'
		
		
至此，consul集群就搭建完毕了，可以写代码，实现服务注册了
		
4. 使用go-micro和consul实现服务注册和管理
	
	在构建微服务时，使用服务发现可以减少配置的复杂性，本文以go-micro为微服务框架，使用consul作为服务发现服务，使用gin开发golang服务。

使用gin 的原因是gin能够很好的和go-micro进行集成。

本文主要介绍服务注册和发现的实现

关于如何搭建consul服务可以移步：https://www.jianshu.com/p/271d490929a5

本文默认以搭建好了consul服务，服务的地址是：192.168.109.131:8500
如果你搭建好了自己的consul服务，可以在浏览器内输入192.168.109.131:8500（地址根据自己的consul服务做调整），会看到如下界面：
![图片.png](https://upload-images.jianshu.io/upload_images/13833591-2e56d358c91362b8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里我的consul服务启用了 3个节点。

####填坑
在开始写代码前，先给大家避一避坑，目前go-micro已经更新到v2版本，此版本去除了对consul 的支持，但支持etcd、mdns作为服务发现，但是老版本的go-micro仍支持consul，但是有些地方做了调整。
```
首先，需要go 1.13的支持，所以小伙伴们需要升级下golang

然后，在获取go-micro库时，不能使用这个指令了 go get -u github.com/micro/go-micro
      改为:go get -u github.com/micro/go-micro/v2
原来go-micro consul的支持已经迁移到了go-plugins里面
我们的代码里在导入consul库时，也变为了：
"github.com/micro/go-plugins/registry/consul"
这个在下面的代码里可以看到

然后，没有安装gin的同学，需要使用如下指令获取下：
go get -u github.com/gin-gonic/gin
```
这些小编折腾了很久才搞明白，这里先给大家提醒下，避免走我的老路
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
	"github.com/micro/go-micro/registry"//注意这些地址变了
	"github.com/micro/go-micro/web"//注意这些地址变了
	"github.com/micro/go-plugins/registry/consul"//注意这些地址变了
	"userserver/routers"
)

var consulReg registry.Registry

func  init()  {
	//新建一个consul注册的地址，也就是我们consul服务启动的机器ip+端口
	consulReg = consul.NewRegistry(
		registry.Addrs("192.168.109.131:8500"),
	)
}

func main() {
	//初始化路由
	ginRouter := routers.InitRouters()

	//注册服务
	microService:= web.NewService(
		web.Name("userserver"),
		//web.RegisterTTL(time.Second*30),//设置注册服务的过期时间
		//web.RegisterInterval(time.Second*20),//设置间隔多久再次注册服务
		web.Address(":18001"),
		web.Handler(ginRouter),
		web.Registry(consulReg),
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
注册的代码就写好了，启动userserver，我们在consul服务界面，可以看到如下效果：
![图片.png](https://upload-images.jianshu.io/upload_images/13833591-dcdf0078d9e4e93a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
说明我们注册成功了

####服务发现
服务发现，就是从consul中获取到我们注册进去的服务，这样在调用别的服务时，就不用从配置文件获取，直接查询consul即可。

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
	"github.com/micro/go-plugins/registry/consul"
	"net/http"
	"orderserver/routers"
	"time"
)

var consulReg registry.Registry

func init(){
	//新建一个consul注册的地址，也就是我们consul服务启动的机器ip+端口
	consulReg = consul.NewRegistry(
		registry.Addrs("192.168.109.131:8500"),
	)
}

func main() {
	//初始化路由
	ginRouter := routers.InitRouters()

	//注册服务
	microService:= web.NewService(
		web.Name("orderserver"),
		//web.RegisterTTL(time.Second*30),//设置注册服务的过期时间
		//web.RegisterInterval(time.Second*20),//设置间隔多久再次注册服务
		web.Address(":18002"),
		web.Handler(ginRouter),
		web.Registry(consulReg),
		)

    //获取服务地址
	hostAddress := GetServiceAddr("userserver")
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

今天go-micro+gin+consul微服务实战就介绍完了，是不是很简单

	
	
	