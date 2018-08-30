## 在集群中安装influxdb
influxdb提供了官方镜像，因此在集群中安装influxdb十分方便，只需要指定镜像名为influxdb即可自动下载运行，只需要配置环境变量就可以进行初始化设置

以下是官方文档提供的可配置环境变量
#### `INFLUXDB_DB`
以这个环境变量作为名字自动初始化一个数据库

#### `INFLUXDB_HTTP_AUTH_ENABLED`
允许身份验证。必须设置此选项或必须在配置文件中设置`auth-enabled = true`才能使后续的任何身份验证相关选项生效。

#### `INFLUXDB_ADMIN_USER`
以这个环境变量作为名字自动初始化一个管理员用户

#### `INFLUXDB_ADMIN_PASSWORD`
以这个环境变量作为管理员用户的密码

#### `INFLUXDB_USER`
以这个环境变量作为名字自动初始化一个普通用户，如果初始化了数据库，则自动获得该数据库的读写权限

#### `INFLUXDB_USER_PASSWORD`
使用INFLUXDB_USER配置的用户的密码。如果未设置，则生成随机密码并打印到标准输出。

#### `INFLUXDB_READ_USER`
要在`INFLUXDB_DB`上使用读取权限创建的用户的名称。如果未设置`INFLUXDB_DB`，则此用户将没有授予权限。

####`INFLUXDB_READ_USER_PASSWORD`
使用`INFLUXDB_READ_USER`配置的用户的密码。如果未设置，则生成随机密码并打印到标准输出。

####`INFLUXDB_WRITE_USER`
要在`INFLUXDB_DB`上使用写权限创建的用户的名称。如果未设置`INFLUXDB_DB`，则此用户将没有授予权限。

####`INFLUXDB_WRITE_USER_PASSWORD`
使用`INFLUXDB_WRITE_USER`配置的用户的密码。如果未设置，则生成随机密码并打印到标准输出。

## 使用HTTP API与数据库通信
influxDB自带HTTP接口，可以轻松编写代码读写数据库

首先观察[官方文档](https://docs.influxdata.com/influxdb/v1.6/guides/writing_data/)中使用curl操作数据库的样例：

#### Create your first database

```
curl -XPOST "http://localhost:8086/query" --data-urlencode "q=CREATE DATABASE mydb"
```

#### Insert some data

```
curl -XPOST "http://localhost:8086/write?db=mydb" \
-d 'cpu,host=server01,region=uswest load=42 1434055562000000000'

curl -XPOST "http://localhost:8086/write?db=mydb" \
-d 'cpu,host=server02,region=uswest load=78 1434055562000000000'

curl -XPOST "http://localhost:8086/write?db=mydb" \
-d 'cpu,host=server03,region=useast load=15.4 1434055562000000000'
```

#### Query for the data

```
curl -G "http://localhost:8086/query?pretty=true" --data-urlencode "db=mydb" \
--data-urlencode "q=SELECT * FROM cpu WHERE host='server01' AND time < now() - 1d"
```

#### Analyze the data

```
curl -G "http://localhost:8086/query?pretty=true" --data-urlencode "db=mydb" \
--data-urlencode "q=SELECT mean(load) FROM cpu WHERE region='uswest'"
```

看起来有一些麻烦，但是influxDB官方同时给出了HTTP client代码库

# 使用GO Client
## 连接客户端
```
c, err := client_v2.NewHTTPClient(client_v2.HTTPConfig{
		Addr:     "http://192.168.102.238:8086",
		Username: "kubespy",
		Password: "kubespy",
	})
	if err != nil {
		log.Fatal(err)
	}
	defer c.Close()
```