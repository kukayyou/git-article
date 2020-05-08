#####1.安装ubuntu 
可以使用虚拟机或者实体机安装ubuntu，版本选择14.04即可，安装指南，可参考链接：https://www.jianshu.com/p/0f0ed7d8e06e
######2. 安装docker
ubuntu安装成功后安装docker，参考链接：https://www.runoob.com/docker/ubuntu-docker-install.html
######3. 安装及使用consul
（参考资料：https://yq.aliyun.com/articles/696142）

***拉取最新版本的consul镜像***
  ```
  docker pull consul
  ```
***启动一个名为consul_server_1的Docker容器***

```
docker run -d -p 8500:8500 -v /data/consul:/consul/data -e CONSUL_BIND_INTERFACE='eth0' --name=consul_server_1  213e00e87c53 agent -server -bootstrap -ui -node=1 -client='0.0.0.0'
```
注意：
1. eth0是网卡名，根据自己的网卡名做修改，可使用```ifconfig```查看
2. /consul/data 是 Consul 持久化地方，如果需要持久化那 Dooker 启动时候需要给它指定一个数据卷 -v /data/consul:/consul/data，若不存在可以创建
3. 213e00e87c53为consul的镜像hash，每台机器都不同，可以通过```docker images``` 查看，如下图：
![图片.png](https://upload-images.jianshu.io/upload_images/13833591-bcf5ca149d26c8e3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
使用```docker container ls ```可以查看容器启动情况
![图片.png](https://upload-images.jianshu.io/upload_images/13833591-fef379ad6a12d4da.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
***查看consul管理界面***
```http://192.168.109.131:8500/ui```，192.168.109.131是ubuntu所在机器的ip
如下图：
![图片.png](https://upload-images.jianshu.io/upload_images/13833591-05f478e5e55bb494.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
1、2、3是节点名称

***Consul 命令简单介绍***
```
agent : 表示启动 Agent 进程。
-server：表示启动 Consul Server 模式。
-client：表示启动 Consul Cilent 模式。
-bootstrap：表示这个节点是 Server-Leader ，每个数据中心只能运行一台服务器。技术角度上讲 Leader 是通过 Raft 算法选举的，但是集群第一次启动时需要一个引导 Leader，在引导群集后，建议不要使用此标志。
-ui：表示启动 Web UI 管理器，默认开放端口 8500，所以上面使用 Docker 命令把 8500 端口对外开放。
-node：节点的名称，集群中必须是唯一的。
-client：表示 Consul 将绑定客户端接口的地址，0.0.0.0 表示所有地址都可以访问。
-join：表示加入到某一个集群中去。 如：-join=192.168.109.131
```
***查看consul集群状态***
```docker exec -t consul_server_1 consul members```
![图片.png](https://upload-images.jianshu.io/upload_images/13833591-c046c52d7278127f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
Status表示它们的状态，都是alive。Type表示它们的类型，DC表示数据中心，是dc1
Address是引导consul的ip，创建consul集群时会用到这个地址

***下面再添加两个节点，命名为 -node=2 、-node=3***
指令如下：
```
docker run -d -e CONSUL_BIND_INTERFACE='eth0' --name=consul_server_2 213e00e87c53 agent -server -node=2 -join='172.17.0.2'
docker run -d -e CONSUL_BIND_INTERFACE='eth0' --name=consul_server_3 213e00e87c53 agent -server -node=3 -join='172.17.0.2'
```
***将Client加入集群***
Client 在 Consul 集群中起到了代理 Server 的作用，Client 模式不持久化数据。一般情况每台应用服务器都会安装一个 Client ，这样可以减轻跨服务器访问带来性能损耗。也可以减轻 Server的请求压力。
指令如下：
```
docker run -d -e CONSUL_BIND_INTERFACE='eth0' --name=consul_server_4 213e00e87c53 agent -client -node=clint -join='172.17.0.2' -client='0.0.0.0'
docker run -d -e CONSUL_BIND_INTERFACE='eth0' --name=consul_server_5 213e00e87c53 agent -client -node=clint2 -join='172.17.0.2' -client='0.0.0.0'
```
至此，consul集群就搭建完毕了，可以写代码，实现服务注册了

	
		
		
		
		