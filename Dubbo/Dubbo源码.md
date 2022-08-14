> - 下载Dubbo源码：https://github.com/apache/dubbo
> - 下载Dubbo-Admin源码：https://github.com/apache/dubbo-admin

# Dubbo Admin

## 了解 Dubbo Admin

Dubbo Admin 是一款为 Dubbo 定制的监控运维平台

## Dubbo Admin编译部署

### 1.修改配置文件

dubbo-admin 是一个 JavaWeb 项目，所以可以直接在 dubbo-admin 源码的根目录下找到 `dubbo-admin-server\src\main\resources\application.properties` 配置文件，并根据自己的需求修改其中的配置信息。常见的配置选项如下：

- Dubbo Admin 依赖于注册中心，所以必须要求注册中心处于运行状态。注册中心默认使用ZooKeeper（可根据实际需求自行更改）。

	```properties
	# 注册中心地址/配置中心地址/保存元数据信息的地址
	admin.registry.address=zookeeper://127.0.0.1:2181
	admin.config-center=zookeeper://127.0.0.1:2181
	admin.metadata-report.address=zookeeper://127.0.0.1:2181
	```

	

- Dubbo Admin 是一个运维监控平台，提供了可视化界面，可视化界面默认的账号密码是写在配置文件中的，可根据需求自行修改。

	```properties
	# 登录监控平台时使用的初始用户名和密码
	admin.root.user.name=root
	admin.root.user.password=root
	```

	

- Dubbo Admin 作为一个独立的项目，需要占用一个端口号用于访问，并且其没有显式指定项目启动的端口号，如果跟你的其它项目端口号发生了冲突，可以自行修改。

	```properties
	# Dubbo Admin没有显式设置项目启动的端口号，采用了默认的8080，如果跟你的其它项目端口号发生了冲突，可以自行修改
	server.port=10010
	```



### 2.打包dubbo-admin项目

1. 在 dubbo-admin 项目的根目录下执行 `mvn clean package` 命令打包项目。

	```
	mvn clean package
	```

	

2. 找到编译后的 jar 包，目录：`项目根目录\dubbo-admin-distribution\target`

3. 运行 jar

	```sh
	java -jar xxx.jar
	```

	

4. 访问地址：http://localhost:10010

	![image-20220814173709538](https://raw.githubusercontent.com/LvNengDong/pic-go/main/img/dubbo/image-20220814173709538.png)

5. 此时，再去启动一个注册到当前 ZooKeeper 上的 Dubbo 微服务，就可以在这个页面查看服务状态了。

	![image-20220814174117781](https://raw.githubusercontent.com/LvNengDong/pic-go/main/img/dubbo/image-20220814174117781.png)

6. 并且可以直接在网页上对 Dubbo 提供的接口进行测试。







