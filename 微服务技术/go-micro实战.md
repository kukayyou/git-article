本文介绍使用gin+go-micro+consul搭建微服务

go-micro git地址：https://github.com/micro/go-micro
安装指令：go get -u github.com/micro/go-micro

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
	
	
	
	
	