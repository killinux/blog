### 安装  
[centos7上consul的安装](http://www.cnblogs.com/wang2650/p/5473881.html)

wget https://releases.hashicorp.com/consul/0.6.4/consul_0.6.4_linux_amd64.zip  
consul -v
json格式化
```shell
yum -y install epel-release  
yum install jq -y  

consul agent -server -bootstrap-expect 2 -data-dir /tmp/consul -node=n1 -bind=192.168.139.194 -dc=dc1  
consul agent -server -bootstrap-expect 2 -data-dir /tmp/consul -node=n2 -bind=192.168.139.218 -dc=dc1  
consul agent -server -bootstrap-expect 2 -data-dir /tmp/consul -node=n2 -bind=192.168.139.193 -dc=dc1  
consul agent -server -bootstrap-expect 2 -data-dir /tmp/consul -node=n2 -bind=192.168.139.161 -dc=dc1  
```
在第一个节点上

consul join 192.168.139.218


#######################################  
如果发现集群有问题，就 rm -rf /tmp/consul 
mkdir /tmp/consul 
#####################################
[consul入门](http://blog.csdn.net/viewcode/article/details/45915179)  
[服务发现系统consul介绍](http://www.codeweblog.com/%E6%9C%8D%E5%8A%A1%E5%8F%91%E7%8E%B0%E7%B3%BB%E7%BB%9Fconsul%E4%BB%8B%E7%BB%8D/)  
# 例子1
mkdir /etc/consul.d/  
echo '{"service": {"name": "web", "tags": ["rails"], "port": 80}}'  >/etc/consul.d/web.json  
consul agent -server -bootstrap-expect 1 -data-dir /tmp/consul  -bind=192.168.139.218 -config-dir /etc/consul.d  

dig @127.0.0.1 -p 8600 web.service.consul 
dig @127.0.0.1 -p 8600 web.service.consul SRV  
curl http://localhost:8500/v1/catalog/service/web  |jq  

curl http://localhost:8500/v1/catalog/nodes |python -m json.tool  
dig @127.0.0.1 -p 8600 mcompute616.node.consul  

健康检查  
echo '{"check": {"name": "ping", "script": "ping -c1 www.baidu.com >/dev/null", "interval": "30s"}}' >/etc/consul.d/ping.json  
echo '{"service": {"name": "web", "tags": ["rails"], "port": 80 ,"check": {"script": "curl localhost:80 >/dev/null 2>&1", "interval": "10s"}}}' >/etc/consul.d/web.json  
curl -s http://localhost:8500/v1/health/state/any | python -m json.tool  

查看所有  
curl -v http://127.0.0.1:8500/v1/kv/?recurse | python -m json.tool  

curl -X PUT -d 'test' http://127.0.0.1:8500/v1/kv/web/key1  
curl -X PUT -d 'test' http://127.0.0.1:8500/v1/kv/web/key2?flags=42  
curl -X PUT -d 'test' http://127.0.0.1:8500/v1/kv/web/web/sub/key3  

查看一个  
curl -s http://127.0.0.1:8500/v1/kv/web/key1|python -m json.tool  
删除所有  
curl -X DELETE http://127.0.0.1:8500/v1/kv/web/sub?recurse  

修改（不好使呀）  
curl -X PUT -d 'newval' http://127.0.0.1:8500/v1/kv/web/key1?cas=106  
curl -X PUT -d 'newval' http://127.0.0.1:8500/v1/kv/web/key1?cas=106  
curl -s http://127.0.0.1:8500/v1/kv/web/key1|python -m json.tool  



### 例子2  
consul agent -server -bootstrap-expect 1  -data-dir /tmp/consul -node=agent-one -bind=192.168.139.218  
#consul agent -data-dir /tmp/consul -node=agent-one -bind=192.168.139.218  
consul agent -server -data-dir /tmp/consul  
consul agent -data-dir /tmp/consul -node=agent-two -bind=192.168.139.194  
consul agent -data-dir /tmp/consul -node=agent-three -bind=192.168.139.193  

第一个节点  
consul join 192.168.139.194  192.168.139.193  
consul members  
consul info  

### 例子3  
consul agent -server -bootstrap-expect 1 -data-dir /tmp/consul  -bind=192.168.139.218 -config-dir /etc/consul.d  
consul agent -data-dir /tmp/consul -join=192.168.139.218 -bind=192.168.139.194  


### 三台机器的测试  
https://blog.coding.net/blog/intro-consul?type=hot  
http://www.bubuko.com/infodetail-800623.html  
consul agent -server -bootstrap -data-dir /tmp/consul -bind=192.168.139.218  
consul agent -server -data-dir /tmp/consul -bind=192.168.139.194  
consul agent -server -data-dir /tmp/consul -bind=192.168.139.193  
第一个节点  
 consul join 192.168.139.194  192.168.139.193  
ctl+c 断开第一个节点，之后
consul agent -server -data-dir /tmp/consul -bind=192.168.139.218  
consul join 192.168.139.194  192.168.139.193  

curl -X PUT -d '{"Datacenter": "dc1", "Node": "mysql-1", "Address": "mysql-1.node.consul","Service": {"Service": "mysql", "tags": ["master","v1"],  "Port": 3306}}' http://127.0.0.1:8500/v1/catalog/register  
curl -X PUT -d '{"Datacenter": "dc1", "Node": "mysql-2", "Address": "mysql-2.node.consul","Service": {"Service": "mysql", "tags": ["slave","v1"], "Port": 3306}}' http://127.0.0.1:8500/v1/catalog/register  

curl http://127.0.0.1:8500/v1/catalog/service/mysql  
curl http://127.0.0.1:8500/v1/catalog/service/mysql|python -m json.tool  
curl 127.0.0.1:8500/v1/catalog/nodes |python -m json.tool  
dig @127.0.0.1 -p 8600 mysql.service.consul SRV  

### 健康检查  
kill掉一个节点，consul members处于fail状态  
curl http://localhost:8500/v1/health/state/critical    

### K/V存储  
curl -v http://localhost:8500/v1/kv/?recurse |python -m json.tool  
curl -X PUT -d 'test' http://localhost:8500/v1/kv/mysql/key2?flags=43  
curl -X DELETE http://localhost:8500/v1/kv/mysql/key2?recurse  
curl -X PUT -d 'newval' http://localhost:8500/v1/kv/mysql/key1?flags=100  

更新index：  
curl "http://localhost:8500/v1/kv/mysql/key1?index=101&wait=5s"  

##################################  

consul agent -atlas-join  -atlas=ATLAS_USERNAME/infrastructure -atlas-token="YOUR_ATLAS_TOKEN"  

curl   https://mysql.service.consul/v1/kv/my-key  

##################################  
{"service": {"name" : "test","port" : 9999,"check":{ "tcp": "127.0.0.1:9999", "interval": "10s" }} }   
#################  
[consul-template入门篇](http://blog.csdn.net/daiyudong2020/article/details/53559008)  

docker run -d --name=consul --net=host gliderlabs/consul-server -bootstrap -bind=192.168.0.149  
docker run -d  --name=registrator     --net=host   --volume=/var/run/docker.sock:/tmp/docker.sock    gliderlabs/registrator:latest consulkv://localhost:8500/hello  

consul-template -consul 127.0.0.1:8500 -template /root/nginx_web.ctmpl:/usr/local/nginx/conf/nginx.conf:"/usr/local/nginx/sbin/nginx -s reload"  


curl -X PUT -d 'test' http://localhost:8500/v1/kv/hello/hehe?flags=43  
curl -v http://localhost:8500/v1/kv/?recurse |python -m json.tool  


### nginx的例子 ###  
模板语言https://book-consul-guide.vnzmi.com/11_consul_template.html    

删除所有
curl -X DELETE http://127.0.0.1:8500/v1/kv/?recurse    
curl --request PUT --data "192.168.139.161" http://localhost:8500/v1/kv/myserver/mcontroller605    
curl --request PUT --data "192.168.139.193" http://localhost:8500/v1/kv/myserver/mcompute605    
curl -s http://localhost:8500/v1/kv/myserver?recurse | jq   

consul-template -consul-addr 127.0.0.1:8500 -template /root/nginx_web.ctmpl:/usr/local/nginx/conf/nginx.conf:"/usr/local/nginx/sbin/nginx -s reload" -once
consul-template -config ./tmpl.json -once  

nginx_web.ctmpl
```go
worker_processes  1;
events {
    worker_connections  1024;
}
http {
    include     mime.types;
    default_type  application/octet-stream;
    sendfile    on;
    keepalive_timeout  65;
    upstream app {
        {{range ls "myserver/" }}
        server {{.Value}} weight=5;{{end}}
    }
    server {
          listen       80;
          server_name  localhost; 
          location / {
            proxy_pass http://app;
          }
    }
}
```
tmpl.json 
```javascript
consul = "127.0.0.1:8500"  
template {  
source = "./nginx_web.ctmpl"  
destination = "/usr/local/nginx/conf/nginx.conf"  
command = "/usr/local/nginx/sbin/nginx -s reload"  
}
```
### consul-template 的helloword:
ls
config.ctmpl   tmpl.json

tmpl.json
```javascript
consul = "127.0.0.1:8500"  
template {  
source = "./config.ctmpl"  
destination = "./config.py"  
command = "python ./config.py"  
}
···
config.ctmpl
```python
#!/usr/bin/python  
#coding:utf-8  
  
#bottle  
iplist = [ {{range service "web"}} "{{.Address}}",{{end}} ]  
port = 8080  
  
for ip in iplist:  
    print ip 
```
consul-template -config ./tmpl.json -once  
生成config.py
```shell
cat /etc/consul.d/web.json 
{"service": {"name": "web", "tags": ["rails"], "port": 80 ,"check": {"script": "curl localhost:80", "interval": "10s"}}}

curl http://127.0.0.1:8500/v1/catalog/service/web|python -m json.tool
```

### kv的例子 ###  
https://python-consul.readthedocs.io/en/latest/#consul-status  

yum install python-virtualenv  
virtualenv mysite  
source mysite/bin/activate  
pip install python-consul  

a.py  
```python
import consul

c = consul.Consul()

# poll a key for updates
index = None
while True:
    index, data = c.kv.get('foo', index=index)
    print data['Value']

# in another process
c.kv.put('foo', 'bar')
```
### 基本使用 ###  
curl -v 是显示详细， -s是只显示结果
设置值
curl --request PUT --data "hello" http://localhost:8500/v1/kv/my-key
查所有值  
curl -v http://localhost:8500/v1/kv/?recurse |python -m json.tool  
查某个值
curl -v http://localhost:8500/v1/kv/my-key |python -m json.tool   
显示值的value
curl -s http://127.0.0.1:8500/v1/kv/my-key| jq -r .[0]'.Value'|base64 -d  
curl -s http://127.0.0.1:8500/v1/kv/foo| jq -r .[0]'.Value'|base64 -d  
删除所有
curl -X DELETE http://127.0.0.1:8500/v1/kv/?recurse


