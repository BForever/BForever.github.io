## 镜像的创建

1. 更换镜像至ustc（为了测试时的速度）
2. 安装musl-dev make gcc linux-headers bison flex以使TC可以编译 
3. 拷贝进TC的源代码
4. 进入源代码文件夹进行编译
5. 运行top（或任何不自动退出的程序）以便exec进入容器

Dockerfile如下：

```
FROM alpine

RUN rm /etc/apk/repositories
RUN echo -e "http://mirrors.ustc.edu.cn/alpine/v3.8/main\nhttp://mirrors.ustc.edu.cn/alpine/v3.8/community" > /etc/apk/repositories

RUN set -ex \
        && apk update && apk add --no-cache --virtual .build-deps \
                musl-dev \
                make \
                gcc \
                linux-headers \
                bison \
                flex

ADD iproute /code/iproute
RUN cd code/iproute && make
CMD top
```

## 部署用yaml的设置

### TC内核通信权限设置
由于TC与内核通信并更改网络设置，需要提高容器权限，这需要在YAML文件中进行设置：

```
spec:
  containers:
    securityContext:
      privileged: true
```
这样容器内的TC才有权限通过netlink与内核进行通信

### 网络设置

TC所在的POD需要能够设置该Node的所有网卡，因此需要设置为hostNetwork网络，另外需要设置DNS以使Pod能以POD的身份访问service

```
    spec:
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
```

最后的部署用的YAML如下：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-deployment
spec:
  selector:
    matchLabels:
      app: test
  replicas: 1
  template:
    metadata:
      labels:
        app: test
    spec:
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      containers:
      - name: test
        image: foo
        imagePullPolicy: Never
        securityContext:
          privileged: true
```

