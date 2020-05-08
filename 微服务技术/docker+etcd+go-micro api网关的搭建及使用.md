在我们使用go-micro框架时，会用到其api网关功能。
本文以etcd作为服务注册和发现工具，实现通过api网关和etcd实现服务间的调用
本文以下内容为基础，未看过的请移步：
【ubuntu+docker搭建etcd集群】：https://www.jianshu.com/p/ec0e4911236d
【go-micro+gin+etcd微服务实战之服务注册与发现】：https://www.jianshu.com/p/1e14a5b0a9db

现默认已经将etcd集群启动，且已了解使用go-micro+gin+etcd实现服务注册与发现
```etcd集群地址为：192.168.109.131:12379```
```micro网关地址为：192.168.109.131:8080```
（这是以我的虚拟机地址进行设置的，实际使用时请按照自己的配置来）

本文仍以orderserver和userserver为基础。

在orderserver中实现以下功能：
```
1. 注册到etcd
2. 通过api网关请求userserver接口/userserver/user/infos
```
在orderserver中实现以下功能：
```
1. 注册到etcd
```

####架构图如下：
![图片.png](https://upload-images.jianshu.io/upload_images/13833591-3b0b6444184d19d1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

外部请求通过API GW进入，API GW查询etcd对应orderserver的地址并请求orderserver，orderserver请求查询etcd对应userserver的地址并请求userserver。

####上代码
下面代码中有关日志的部分可以忽略，不影响我们此次的功能。
orderserver代码结构：
![图片.png](https://upload-images.jianshu.io/upload_images/13833591-42b5e4f362d659ae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
```
conf是配置文件
config是配置变量
controllers是控制层代码
routers是路由层
```
```main.go代码如下```
```
package main

import (
	"github.com/kukayyou/commonlib/myconfig"
	"github.com/kukayyou/commonlib/myhttp"
	"github.com/kukayyou/commonlib/mylog"
	"github.com/micro/go-micro/registry"
	"github.com/micro/go-micro/registry/etcd"
	"github.com/micro/go-micro/web"
	//"github.com/micro/go-plugins/registry/consul"
	"go.uber.org/zap"
	"orderserver/config"
	"orderserver/routers"
)

var sugarLogger *zap.SugaredLogger

func main() {
	//初始化路由
	ginRouter := routers.InitRouters()
	//新建一个consul注册的地址，也就是我们consul服务启动的机器ip+端口
	/*consulReg := consul.NewRegistry(
		registry.Addrs(config.ConsulAddress),
	)*/
	etcdReg := etcd.NewRegistry(
		registry.Addrs("192.168.109.131:12379"))
	myhttp.EtcdAddr = "192.168.109.131:12379"
	//注册服务
	microService:= web.NewService(
		web.Name("api.tutor.com.orderserver"),
		//web.RegisterTTL(time.Second*30),//设置注册服务的过期时间
		//web.RegisterInterval(time.Second*20),//设置间隔多久再次注册服务
		web.Address(":18002"),
		web.Handler(ginRouter),
		web.Registry(etcdReg),
		)

	microService.Run()
}

func initConfig() {
	myconfig.LoadConfig("./conf/config.conf")
	config.ConsulAddress = myconfig.Config.GetString("consul_address")
	config.LogPath =  myconfig.Config.GetString("log_path")
	config.LogLevel =  int8(myconfig.Config.GetInt64("log_level"))
	config.LogMaxAge =  int(myconfig.Config.GetInt64("log_max_age"))
	config.LogMaxSize =  int(myconfig.Config.GetInt64("log_max_size"))
	config.LogMaxBackups =  int(myconfig.Config.GetInt64("log_max_backups"))
}

func init() {
	initConfig()
	mylog.InitLog(config.LogPath,"orderserver", config.LogMaxAge, config.LogMaxSize, config.LogMaxBackups, config.LogLevel)
}
```
```initConfig和init是初始化内容，不用关注主要关注如下内容：```

####服务注册
下面的代码实现了将orderserver注册到etcd，这里注意```web.Name("api.tutor.com.orderserver")```这句是指定服务名，但为什么这么写，是为了后面通过api网关访问特意这么写的，后面我们的api网关的namespace会指定为```api.tutor.com```所以这里就必须把名称写成这样。
```
    etcdReg := etcd.NewRegistry(
        registry.Addrs("192.168.109.131:12379"))
    myhttp.EtcdAddr = "192.168.109.131:12379"
    //注册服务
    microService:= web.NewService(
        web.Name("api.tutor.com.orderserver"),
        //web.RegisterTTL(time.Second*30),//设置注册服务的过期时间
        //web.RegisterInterval(time.Second*20),//设置间隔多久再次注册服务
        web.Address(":18002"),
        web.Handler(ginRouter),
        web.Registry(etcdReg),
        )

    microService.Run()
```
```router.go代码如下```
这里注意，我的代码示例使用rest接口的方式来实现的，这里每个接口的跟节点必须是```/orderserver```，否则后面访问接口时会无法访问到，因为我们的请求地址是```http://192.168.109.131:8080/orderserver/order/infos```,所以接口结构要设计为```/orderserver/order/infos```
```
package routers

import (
	"github.com/gin-gonic/gin"
	"orderserver/controllers"
)

func InitRouters() *gin.Engine {
	ginRouter := gin.Default()
	root := ginRouter.Group("/orderserver")
	order := root.Group("/order")
	order.POST("/infos", controllers.GetOrderController{}.GetOrderInfosApi)

	return ginRouter
}
```
```base_controller.go代码如下：```
主要实现请求参数解析和请求requestid设置
```
package controllers

import (
	"github.com/gin-gonic/gin"
	"github.com/kukayyou/commonlib/mylog"
	"io/ioutil"
)

type BaseController struct {
	mylog.LogInfo
	ReqParams []byte
}

func (bc *BaseController) Prepare(c *gin.Context) {
	bc.SetRequestId()

	bc.ReqParams, _ = ioutil.ReadAll(c.Request.Body)

	mylog.Info("requestId:%s, params : %s", bc.GetRequestId(), string(bc.ReqParams))
}
```
```order_infos.go代码如下：```
这里主要实现请求userserver

```
package controllers

import (
	"github.com/gin-gonic/gin"
	"github.com/kukayyou/commonlib/myhttp"
	"github.com/kukayyou/commonlib/mylog"
	"encoding/json"
	"time"
)

type GetOrderController struct {
	BaseController
}

type RequestData struct {
	Data int `json:"data"`
}

type OrderInfo struct {
	OrderID   string                 `json:"orderId"`
	UserInfos map[string]interface{} `json:"userInfos"`
}

type UserInfo struct {
	UserID   uint64 `json:"userId"`
	UserName string `json:"userName"`
	Mobile   string `json:"mobile"`
	Email    string `json:"email"`
	Sex      string `json:"sex"` //male or female
	Age      uint64 `json:"age"`
}

func (this GetOrderController)GetOrderInfosApi(c *gin.Context) {
	this.Prepare(c)
	var params RequestData
	json.Unmarshal(this.ReqParams, &params)

	var orderInfo OrderInfo
	mylog.Info("requestID:%s:, GetOrderInfosApi start ... ", this.GetRequestId())
	data := orderInfo.GetOrderInfos(this.GetRequestId())
	c.JSON(200,
		gin.H{
			"status": "1",
			"data":   data,
		})

	go func() {
		time.Sleep(time.Second*2)
		mylog.Info("requestID:%s:, 延时日志：%s", this.GetRequestId(), time.Now().Format("2006-01-02 15:04:05"))
	}()

	return
}

func (oi *OrderInfo) GetOrderInfos(requestID string) []OrderInfo {
	var orderInfo OrderInfo
	if userInfos := GetUserInfoByIDs(requestID);userInfos!=nil{
		orderInfo.UserInfos = userInfos
		o,_:= json.Marshal(orderInfo)
		mylog.Info("requestID:%s orderInfo is :%v", requestID, string(o))
		return []OrderInfo{orderInfo}
	}
	return nil
}

func GetUserInfoByIDs(requestID string) map[string]interface{} {
	resp := myhttp.RequestWithHytrix("api.tutor.com.userserver", "/userserver/user/infos", map[string]string{})
	if resp != nil{
		r,_:=json.Marshal(resp)
		mylog.Info("requestID:%s, resp is :%s", requestID, string(r))
		return resp
	}
	return nil
}

```
请求这块用到了go-micro的方法，我自己封装了一下，具体代码如下：
```myhttp```是封装的包，里面的```RequestWithHytrix```方法，支持consul和etcd两种服务发现工具，只需通过```RegistryType```指定就可以了；
看过【go-micro+gin+consul微服务实战之使用http api请求】：https://www.jianshu.com/p/1e14a5b0a9db，这篇文章的就不陌生如何通过consul请求，其实etcd跟consul类似，在代码上并没有太大区别，唯一的区别就是初始化```registry.Registry```是所使用的的库不同。这里不再细说，不了解的可以看看上面链接的文章
```
package myhttp

import (
	"context"
	"encoding/json"
	hystrixGo "github.com/afex/hystrix-go/hystrix"
	"github.com/kukayyou/commonlib/mylog"
	"github.com/micro/go-micro/client"
	"github.com/micro/go-micro/client/selector"
	"github.com/micro/go-micro/registry"
	"github.com/micro/go-micro/registry/etcd"
	microhttp "github.com/micro/go-plugins/client/http"
	"github.com/micro/go-plugins/registry/consul"
	"github.com/micro/go-plugins/wrapper/breaker/hystrix"
)

var (
	ConsulAddr string//consul地址：ip+port
	EtcdAddr string//consul地址：ip+port
	DefaultSleepWindow int = 5000//重试时间窗口
	DefaultTimeOut int = 5000//默认超时时间
	DefaultVolumeThreshold int = 2//默认最大失败次数
	RegistryType int = 0//0:etcd ,1:consul
)

func RequestWithHytrix(serverName, url string, req interface{})map[string]interface{}{
	var reg registry.Registry
	switch RegistryType {
	case 0:
		reg = etcd.NewRegistry(
			registry.Addrs(EtcdAddr),
		)
	case 1:
		reg = consul.NewRegistry(
			registry.Addrs(ConsulAddr),
		)
	default:
	}

	microSelector := selector.NewSelector(
		selector.Registry(reg),              //传入consul注册
		selector.SetStrategy(selector.RoundRobin), //指定查询机制
	)
	microClient := microhttp.NewClient(
		client.Selector(microSelector),
		client.ContentType("application/json"),
		client.Wrap(hystrix.NewClientWrapper()), //熔断操作
	)
	hystrixGo.DefaultSleepWindow = DefaultSleepWindow//重试时间窗口
	hystrixGo.DefaultTimeout = DefaultTimeOut//默认超时时间
	hystrixGo.DefaultVolumeThreshold = DefaultVolumeThreshold//默认最大失败次数

	reqInfo := microClient.NewRequest(serverName, url, req)
	r, _ := json.Marshal(req)
	mylog.Info("RegistryType:%d, serverName:%s, url:%s, req:%s", RegistryType, serverName, url, string(r))
	var resp map[string]interface{}

	if err := microClient.Call(context.Background(), reqInfo, &resp); err != nil {
		mylog.Error("request error:%s", err.Error())
		return nil
	}

	re, _ := json.Marshal(resp)
	mylog.Info("response is:%s", string(re))
	return  resp
}
```

至此，orderserver的核心功能就已经完全实现。

下面看userserver

userserver代码结构如下，跟orderserver类似：
![图片.png](https://upload-images.jianshu.io/upload_images/13833591-abddc0140a2b8f75.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 ```main.go代码如下：```
```
package main

import (
	"fmt"
	"github.com/micro/go-micro/client/selector"
	"github.com/micro/go-micro/registry"
	"github.com/micro/go-micro/registry/etcd"
	"github.com/micro/go-micro/web"
	"github.com/micro/go-plugins/registry/consul"
	"time"
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
	etcdReg := etcd.NewRegistry(
		registry.Addrs("192.168.109.131:12379"))
	//注册服务
	microService:= web.NewService(
		web.Name("api.tutor.com.userserver"),
		//web.RegisterTTL(time.Second*30),//设置注册服务的过期时间
		//web.RegisterInterval(time.Second*20),//设置间隔多久再次注册服务
		web.Address(":18001"),
		web.Handler(ginRouter),
		web.Registry(etcdReg),
		)

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
核心代码是，如下内容，实现注册到etcd，名称的设置原理跟orderserver一样，设置为```api.tutor.com.userserver```
```
etcdReg := etcd.NewRegistry(
        registry.Addrs("192.168.109.131:12379"))
    //注册服务
    microService:= web.NewService(
        web.Name("api.tutor.com.userserver"),
        //web.RegisterTTL(time.Second*30),//设置注册服务的过期时间
        //web.RegisterInterval(time.Second*20),//设置间隔多久再次注册服务
        web.Address(":18001"),
        web.Handler(ginRouter),
        web.Registry(etcdReg),
        )

    microService.Run()
```
 ```router.go代码如下：```
实现接口路由设置
```
package routers

import (
	"github.com/gin-gonic/gin"
	"userserver/controllers"
)

func InitRouters() *gin.Engine {
	ginRouter := gin.Default()
	root := ginRouter.Group("/userserver")
	userGroup := root.Group("/user")
	userGroup.POST("/infos", controllers.GetUserInfosApi)
	return ginRouter
}

```
```user_infos.go代码如下```
实现返回数据userinfo，这里没做实现，直接返回空结构体
```
package controllers

import (
	"github.com/gin-gonic/gin"
)

type UserInfo struct {
	UserID   uint64 `json:"userId"`
	UserName string `json:"userName"`
	Mobile   string `json:"mobile"`
	Email    string `json:"email"`
	Sex      string `json:"sex"` //male or female
	Age      uint64 `json:"age"`
}

func GetUserInfosApi(c *gin.Context) {
	var userInfo UserInfo
	data := userInfo.GetUserInfo()

	c.JSON(200,
		gin.H{
			"status": "1",
			"data":   data,
		})
	return
}

func (uc *UserInfo) GetUserInfo() []UserInfo {
	var userInfo UserInfo
	return []UserInfo{userInfo}
}

```

至此，代码就已经实现完毕。
####下面来设置go-micro的api网关
这里使用ubuntu+docker+microhq/micro镜像的方式
####首先
下载镜像
```
    docker pull microhq/micro
```
####其次，启动镜像并设置参数
```
docker run -d -p 8080:8080 --name=micro_api_gw ba526346c047 --registry=etcd --registry_address=192.168.109.131:12379 --api_namespace=api.tutor.com --api_handler=http api
```
参数说明：
```
-d 为后台运行模式
-p指定镜像对外端口第一个8080是镜像对外的端口，第二个8080是micro 网关默认端口
--name=micro_api_gw 指定镜像名称为 micro_api_gw
ba526346c047 为镜像ID，可通过指令 docker images 查看
--registry=etcd 指定服务注册的类型是etcd
--registry_address=192.168.109.131:12379  指定服务注册的地址是192.168.109.131:12379，这个要根据自己的etcd集群来调整，我的设置的是这个
--api_namespace=api.tutor.com 指定网关的命名空间为api.tutor.com，这个就是我们刚刚在设置server名称时用到的，可根据自己的情况调整
--api_handler=http 指定以http的方式请求server，micro还支持rpc，api等方式，可以自己研究下
```
镜像启动后，我们的api网关就设置好了

####下面就是使用postman演示效果
请求接口为```http://192.168.109.131:8080/orderserver/order/infos```，返回结果如下，
![图片.png](https://upload-images.jianshu.io/upload_images/13833591-626a89a4926882f7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
