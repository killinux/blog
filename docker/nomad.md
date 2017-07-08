```shell
docker
yum install docker -y
systemctl start docker
systemctl enable docker
docker pull fedora/apache 

 docker run -i -t fedora/apache /bin/bash 

docker run -p 192.168.139.204:80:80 -d -i -t fedora/apache /bin/bash 
不好用
docker run -i -t  fedora/apache /run-apache.sh 

docker run -d -i -t -p 192.168.139.204:80:80  fedora/apache /run-apache.sh 
mkdir /opt/share
echo "i am haoning" >a.txt
docker run -v /opt/share:/share -i -t fedora/apache /bin/bash 

```

``` shell
docker run -v /opt/share:/share  -d -i -t -p 192.168.139.204:80:80  fedora/apache /run-apache.sh 
docker run -v /opt/share/hello:/var/log/httpd  -d -i -t -p 192.168.139.204:80:80  fedora/apache /run-apache.sh 

```
进入运行的容器
#docker attach 2d6d02d99d0e
docker ps
docker exec -it 61e0dd477d28 /bin/sh

可以看到docker中的/var/log/httpd 映射到本地的/opt/share/hello 
可以查看apache的log了

docker ps
docker port 61e0dd477d28 80
docker logs -f web  
docker top web  
docker inspect web  
docker start web  
run启动过的，之后直接start就行了，端口和挂载的目录都还在
docker stop web 



yum install golang -y

安装nomad
https://www.hashicorp.com/products/nomad/
直接下载可执行文件
 go run hello.go
```C
package main

import "fmt"

func main() {
   fmt.Println("Hello, World!")
}
```


1.简介
Nomad是一个分布式的，高可用的，数据中心相关的，可以在任何基础设施上部署任何规模应用程序的机群管理器和调度器。
   它将机器和其上的应用程序抽象分离，代替允许用户声明想运行什么，并且让nomad来处理在什么地方以及怎么运行。
   主要特性
Docker Support，docker支持
Operationally Simple，操作简单
Multi-Datacenter and Multi-Region Aware，多数据中心多区域相关
Flexible Workloads，灵活负载
Built for Scale，规模可伸缩
2.运行agent
         agent可以server或者client模式运行，建议每个机器至少3-5个server，其他的均以client模式运行。
   client是一个轻量级的进程，用于处理注册主机，执行心跳检测，执行server分配的各种任务。
启动agent，nomad agent -dev，参数dev指以开发者身份运行，此时agent同时具有server和client模式。
查看集群中节点，nomad node-status，nomad server-members
停止agent，ctrl-c

3.工作job
运行job
产生job文件，nomad init，产生example.nomad文件
运行job，nomad run example.nomad
检查job状态，nomad status example
nomad alloc-status allocation-id
修改job
修改job文件，再执行 nomad plan example.nomad
nomad run -check-index modify-index example.nomad
停止job，nomad stop example
4.集群cluster

```go
# server.hcl
# Increase log verbosity
log_level = "DEBUG"
# Setup data dir
data_dir = "/tmp/server1"
# Enable the server
server {
    enabled = true
    # Self-elect, should be 3 or 5 for production
    bootstrap_expect = 1
}
# Advertise must be set to a non-loopback address.
# Defaults to the resolving the local hostname.
advertise {
    http = "192.168.139.204"
    rpc  = "192.168.139.204"
    serf = "192.168.139.204"
}
```
启动server
nomad agent -config server.hcl

配置文件并启动agent client1
```go
# client1.hcl
# Increase log verbosity
log_level = "DEBUG"
# Setup data dir
data_dir = "/tmp/client1"
enable_debug = true
name = "client1"
# Enable the client
client {
    enabled = true
    # no_host_uuid = true
    # For demo assume we are talking to server1. For production,
    # this should be like "nomad.service.consul:4647" and a system
    # like Consul used for service discovery.
    servers = ["192.168.139.204:4647"]
    node_class = "foo"
    options {
        "driver.raw_exec.enable" = "1"
    }
    reserved {
       cpu = 500
    }
}
# Modify our port to avoid a collision with server1
ports {
    http = 5656
}
advertise {
  http = "192.168.139.204"
  rpc = "192.168.139.204"
  serf = "192.168.139.204"
}
```
启动client1
nomad agent -config client1.hcl
