本文基于compose管理镜像，对此不熟悉的，可以先了解下如何使用。
####安装compose
下载compose,使用下面的指令下载compose
```
sudo curl -L "https://github.com/docker/compose/releases/download/1.24.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```
将可执行权限应用于二进制文件：
```
sudo chmod +x /usr/local/bin/docker-compose
```

创建软链：
```
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```
测试是否安装成功：
```
docker-compose --version
cker-compose version 1.24.1, build 4667896b
```
####拉取etcd官方镜像
```
docker pull quay.io/coreos/etcd
```
在一个文件夹下创建 etcd-compose.yml文件，用于管理etcd容器
在文件中贴如如下内容：
```
version: '3'
services:
  etcd-node1:
    image: "quay.io/coreos/etcd"
    container_name: "etcd-node1"
    ports:
      - "12379:2379"
      - "12380:2380"
    command: 'etcd -name etcd-node1 -advertise-client-urls http://0.0.0.0:2379 -listen-client-urls http://0.0.0.0:2379 -listen-peer-urls http://0.0.0.0:2380 -initial-cluster-token etcd-cluster -initial-cluster "etcd-node1=http://etcd-node1:2380,etcd-node2=http://etcd-node2:2380,etcd-node3=http://etcd-node3:2380" -initial-cluster-state new'
    networks:
      - "etcd"

  etcd-node2:
    image: "quay.io/coreos/etcd"
    container_name: "etcd-node2"
    ports:
      - "22379:2379"
      - "22380:2380"
    command: 'etcd -name etcd-node2 -advertise-client-urls http://0.0.0.0:2379 -listen-client-urls http://0.0.0.0:2379 -listen-peer-urls http://0.0.0.0:2380 -initial-cluster-token etcd-cluster -initial-cluster "etcd-node1=http://etcd-node1:2380,etcd-node2=http://etcd-node2:2380,etcd-node3=http://etcd-node3:2380" -initial-cluster-state new'
    networks:
      - "etcd"

  etcd-node3:
    image: "quay.io/coreos/etcd"
    container_name: "etcd-node3"
    ports:
      - "32379:2379"
      - "32380:2380"
    command: 'etcd -name etcd-node3 -advertise-client-urls http://0.0.0.0:2379 -listen-client-urls http://0.0.0.0:2379 -listen-peer-urls http://0.0.0.0:2380 -initial-cluster-token etcd-cluster -initial-cluster "etcd-node1=http://etcd-node1:2380,etcd-node2=http://etcd-node2:2380,etcd-node3=http://etcd-node3:2380" -initial-cluster-state new'
    networks:
      - "etcd"

networks:
  etcd:
```
启动yml文件
```
docker-compose -f etcd-compose.yml up -d
```
使用指令，查看容器启动情况
```
docker ps -a 
```

可以看到如下内容
![图片.png](https://upload-images.jianshu.io/upload_images/13833591-5ceb6422f1003ee8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

至此，etcd容器集群搭建完毕.

查看注册到etcd的服务有两种方法，一种是进入etcd容器，使用etcdctl指令查看，这里不再介绍。

介绍一种使用go-micro工具查看注册到etcd服务的方法

在goland中执行如下指令
```
set MICRO_REGISTRY=etcd
set MICRO_REGISTRY_ADDRESS=192.168.109.131:12379
set MICRO_API_NAMESPACE=api.tutor.com
micro web
```
192.168.109.131是我的虚拟机ip
12379这是上面创建的etcd容器代理端口

然后在浏览器输入：http://localhost:8082/registry

即可看到如下效果：
![图片.png](https://upload-images.jianshu.io/upload_images/13833591-f9b963419fddce16.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 红框内的 是我注册到etcd的服务
