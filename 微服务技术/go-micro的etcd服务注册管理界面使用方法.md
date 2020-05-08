我们在使用consul时，consul提供了管理界面，可很直观的看到我们注册到consul的服务及健康状况。
etcd并未提供此功能，但是我们可以使用go-micro提供的一个简易界面查看我们注册到etcd中的服务
本文是基于【docker+etcd+go-micro api网关的搭建及使用】：https://www.jianshu.com/p/13d1df6e6731，这篇文章的环境基础来实现的，没有搭建docker+etcd+go-micro api网关的，可以按照上面的链接搭建一遍。

启动这个管理界面也是使用go-micor的镜像来操作，只是指令上有些变化，在启动api网关时，我们使用的是api指令，如下：
```
docker run -d -p 8080:8080 --name=micro_api_gw ba526346c047 --registry=etcd --registry_address=192.168.109.131:12379 --api_namespace=api.tutor.com --api_handler=http api
```
这里我们使用web指令,如下：
```
docker run -d -p 8082:8082 --name=micro_etcd_monitor ba526346c047 --registry=etcd --registry_address=192.168.109.131:12379 --api_namespace=api.tutor.com web
```
####注意
```
这里端口有变化
我们指定的端口是8082，而不是8080
并且少了--api_handler=http
```
启动完成后，在浏览器输入```http://192.168.109.131:8082/registry```，就可以看到如下界面（192.168.109.131是我的虚拟机ip，可以根据自己的机器调整）：
![图片.png](https://upload-images.jianshu.io/upload_images/13833591-8ede8456aaa41e74.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
红框里的就是我注册进去的服务
其中，go.micor.api是我启动的api网关；go.micro.web就是我们刚刚启动的
在这个界面就可以看到我们在etcd注册的服务
```
今天就介绍到这里