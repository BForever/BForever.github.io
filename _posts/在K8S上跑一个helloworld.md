# 在K8S上跑一个helloworld
## 建立docker镜像
为了方便起见，这里直接使用一个js网页作为应用，以此创建镜像
### hello world网页
创建server.js，输入以下代码创建helloworld网页：

```
var http = require('http');

var handleRequest = function(request, response) {
  console.log('Received request for URL: ' + request.url);
  response.writeHead(200);
  response.end('Hello World!');
};
var www = http.createServer(handleRequest);
www.listen(8080);
```

### Dockerfile
创建Dockerfile文件，配置镜像：

```
FROM node:6.9.2
EXPOSE 8080
COPY server.js .
CMD node server.js
```
其中，FROM是从官方镜像库取得node的镜像，EXPOSE表示暴露本容器的8080端口，COPY将server.js加入容器，CMD为容器中执行的命令

>更详细的Dockerfile写法见官方文档

### 创建镜像
配置好Dockerfile后，就可以使用docker的build命令根据Dockerfile的内容创建一个镜像：
`docker build -t hello-node:v1 .`
这里注意不要遗漏最后的`.`

## 在Kubernetes上以该镜像创建一个POD
在K8S集群配置完毕后，执行

`kubectl run hello-node --image=hello-node:v1 --port=8080 --image-pull-policy=Never`

即可在K8S上建立一个新的运行刚刚建立的镜像的POD

但是此时由于POD默认不暴露在外部，因此我们无法观察到node的输出，为此需要创建一个service将该POD的端口暴露出来

## 访问该POD
### Kubernetes Service
Service有几个种类，默认的是cluster-ip，即只能通过pod在集群内部的IP地址进行对service的访问

第二种是nodeport，即在每一个Node上暴露出一个端口：nodePort，外部网络可以通过（任一Node）[NodeIP]:[NodePort]访问到后端的Service。

第三种是loadbalancer，请求底层云平台创建一个负载均衡器，将每个Node作为后端，进行服务分发。该模式需要底层云平台（例如GCE）支持

最简单的就是直接开启一个默认的clusterip的service，在master上直接通过集群内部IP访问helloworld：

`kubectl expose deployment hello-node --port=8080 --target-port=8080`

其他方式比如NodePort可以通过`--type=NodePort`参数指定

### 进行访问
在创建好service之后，就可以使用`kubectl get service`查看当前运行的service，其中可以看到hello-node的ClusterIP，在浏览器中访问该IP：8080即可看到hello world！
