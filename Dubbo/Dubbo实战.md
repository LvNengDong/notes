> [Apache Dubbo 官网](https://dubbo.apache.org/zh/index.html)

# Dubbo3实战

# Dubbo3环境准备

我这里采用 ZooKeeper 作为服务注册中心，所以需要提前安装 ZooKeeper，而 ZooKeeper 又需要 JDK 的支持，安装系统为 CentOS7。

> - JDK版本：jdk-8u151-linux-x64.tar.gz
> - ZooKeeper版本：apache-zookeeper-3.7.0-bin.tar.gz



安装流程：

1. 上传 JDK 和 Zookeeper 的安装包到服务器上；

2. 解压 JDK 和 Zookeeper 至待安装目录；

	```sh
	tar -zxf jdk-8u151-linux-x64.tar.gz -C /opt/module/
	tar -zxvf apache-zookeeper-3.7.0-bin.tar.gz -C /opt/module/
	```



## 安装JDK

1. 复制 `JAVA_HOME` 的路径

	```
	/opt/module/jdk1.8.0_151
	```

2. 修改 `/etc/profile`，

	```sh
	vim /etc/profile
	```

	在末尾追加以下信息：

	```sh
	## JAVA_HOME
	export JAVA_HOME=/opt/module/jdk1.8.0_151
	export PATH=$PATH:$JAVA_HOME/bin
	```

3. 重载 `/etc/profile` 文件，使其生效

	```sh
	source /etc/profile
	```

4. 测试 JDK 安装是否成功

	```sh
	java -version
	```



## 安装ZooKeeper

1. 进入 Zookeeper 根目录

	```sh
	cd /opt/module/apache-zookeeper-3.7.0-bin/
	```

	

2. 修改配置文件.复制出一个 zoo.cfg 文件，这个文件是 Zookeeper 默认读取的配置文件

	```sh
	cp conf/zoo_sample.cfg conf/zoo.cfg
	```

	

3. 编辑 `zoo.cfg`（根据实际需求自行修改）

	![image-20220814005006656](https://raw.githubusercontent.com/LvNengDong/pic-go/main/img/dubbo/image-20220814005006656.png)

	

	

4. 启动 Zookeeper，并使用 JPS 查看 zk 进程

	```sh
	bin/zkServer.sh start
	```

	![image-20220814005804410](https://raw.githubusercontent.com/LvNengDong/pic-go/main/img/dubbo/image-20220814005804410.png)

5. 验证 zk 是否启动成功：执行 `zkCli` 连接服务端，无参数默认查本机的 2181 端口

	```sh
	bin/zkCli.sh
	```

	![image-20220814010054326](https://raw.githubusercontent.com/LvNengDong/pic-go/main/img/dubbo/image-20220814010054326.png)

	如上图，能查到数据就说明 zk 已经正常运行了

6. 注意：如果你的 ZooKeeper 安装在远程服务器上，一定要对外开放 2181 端口。

	```sh
	# 查看防火墙状态 
	systemctl status firewalld
	
	# 开启防火墙
	systemctl start firewalld
	
	# 设置防火墙开机自启
	systemctl enable firewalld.service
	
	# 查询指定端口是否已经开启
	firewall-cmd --query-port=2181/tcp
	
	# 开放指定端口
	firewall-cmd --zone=public --add-port=2181/tcp --permanent
	
	# 重载新添加的端口
	firewall-cmd --reload
	
	# 查询指定端口是否开启成功
	firewall-cmd --query-port=2181/tcp
	```

	







# SpringBoot整合Dubbo

> 版本管理：
>
> - SpringBoot  2.3.X
> - Dubbo 3.X
> - JDK8

## 项目搭建

1. 创建 SpringBoot 基础工程 `dubbo-parent` 作为项目的父工程。作为项目的父工程，主要负责依赖包的管理，不会有任何代码，所以可以删掉 src 目录。

2. 修改 POM 文件

	```xml
		<!--增加packaging类型-->
		<packaging>pom</packaging>
	
		<!--父工程不进行依赖包引入，会对子工程产生影响-->
		<!--
		<dependencies>
			<dependency>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-starter</artifactId>
			</dependency>
	
			<dependency>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-starter-test</artifactId>
				<scope>test</scope>
			</dependency>
		</dependencies>
		-->
	
		<properties>
			<java.version>1.8</java.version>
			<dubbo.version>3.0.1</dubbo.version>
			<curator.version>4.3.0</curator.version>
		</properties>
	
		<!-- 父工程做依赖包的管理 -->
		<dependencyManagement>
			<dependencies>
				<!-- 增加dubbo依赖包管理 -->
				<dependency>
					<groupId>org.apache.dubbo</groupId>
					<artifactId>dubbo</artifactId>
					<version>${dubbo.version}</version>
				</dependency>
				<dependency>
					<groupId>org.apache.dubbo</groupId>
					<artifactId>dubbo-dependencies-zookeeper</artifactId>
					<version>${dubbo.version}</version>
					<type>pom</type>
				</dependency>
				<dependency>
					<groupId>org.apache.zookeeper</groupId>
					<artifactId>zookeeper</artifactId>
					<version>3.7.0</version>
				</dependency>
				<dependency>
					<groupId>org.apache.curator</groupId>
					<artifactId>curator-framework</artifactId>
					<version>${curator.version}</version>
				</dependency>
				<dependency>
					<groupId>org.apache.curator</groupId>
					<artifactId>curator-recipes</artifactId>
					<version>${curator.version}</version>
				</dependency>
			</dependencies>
		</dependencyManagement>
	
		<!-- 移除SpringBoot的打包管理，后续在子工程中进行单独处理 -->
		<!--
		<build>
			<plugins>
				<plugin>
					<groupId>org.springframework.boot</groupId>
					<artifactId>spring-boot-maven-plugin</artifactId>
				</plugin>
			</plugins>
		</build>
		-->
	```

3. 建立子工程 dubbo-consumer、dubbo-provider 和 dubbo-service。其中，dubbo-consumer 和 dubbo-provider 需要是 SpringBoot 工程，而 dubbo-service 只负责保存一些 dubbo-consumer 和 dubbo-provider 通用的接口和类，只需要创建一个普通的 Maven 工程即可，无需启动类，工程运行时也无需启动。

4. 修改父工程和子工程的 POM 配置。

	- 修改子工程的 POM 配置文件（三个子模块都要完成这一步）

		```xml
		    <parent>
		        <!--将SpringBoot的父工程依赖修改为依赖自己的父工程-->
		        <groupId>cn.lnd</groupId>
		        <artifactId>dubbo-parent</artifactId>
		        <version>0.0.1-SNAPSHOT</version>
		        <relativePath>../pom.xml</relativePath>
		    </parent>
		```

	- 修改父工程的 POM 配置文件：增加对子moudle的管理

		```xml
			<modules>
				<module>dubbo-service</module>
				<module>dubbo-consumer</module>
				<module>dubbo-provider</module>
			</modules>
		```

5. dubbo-consumer 和 dubbo-provider 都需要依赖 dubbo-service 模块，需要分别引入对应的依赖。









## 使用XML配置文件的方式集成Dubbo

> 完整代码地址：

### 服务提供者(dubbo-service/dubbo-provider)

#### 1.定义服务接口(dubbo-service)

```java
public interface IHelloService {
    String sayHello(String name);
}
```

#### 2.服务提供方实现接口(dubbo-provider)

```java
public class HelloServiceImpl implements IHelloService {
    @Override
    public String sayHello(String name) {
        System.out.println("Consumer Param：" + name);
        return "Provider：Hello " + name;
    }
}
```

#### 3.使用XML配置声明暴露服务(dubbo-provider)

- `resource/applicationContext-dubbo.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
       xsi:schemaLocation="
            http://www.springframework.org/schema/beans
            http://www.springframework.org/schema/beans/spring-beans-4.3.xsd
            http://dubbo.apache.org/schema/dubbo
            http://dubbo.apache.org/schema/dubbo/dubbo.xsd">

    <!-- 提供方应用信息，name可以随便起名，但是不能重复 -->
    <dubbo:application name="hello-world-producer"/>

    <!-- 使用zookeeper为注册中心，客户端使用curator -->
    <dubbo:registry address="zookeeper://localhost:2181" client="curator"/>
    
	<!-- 对外提供一个 providerService 服务，服务对应的实现 ref="iProviderService" 
	（对外暴露的服务是接口，实际上执行任务的是接口的实现类）-->
    <dubbo:service id="providerService"
                   interface="cn.lnd.dubbo.service.IHelloService"
                   ref="providerServiceImpl"/>


    <bean id="providerServiceImpl" class="cn.lnd.dubbo.provider.service.impl.HelloServiceImpl"/>

</beans>
```



#### 4.让SpringBoot启动时能加载Dubbo的配置文件(dubbo-provider)

让 Spring 启动时能加载 Dubbo 的配置文件，这样对 Dubbo 的配置才能生效。

```java
@ImportResource(locations = {"classpath:applicationContext-dubbo.xml"})
@SpringBootApplication
public class DubboProviderApplication {
    public static void main(String[] args) {
        SpringApplication.run(DubboProviderApplication.class, args);
    }
}
```





### 服务消费者(dubbo-consumer)

#### 1.通过XML配置声明调用远程服务

- `resource/applicationContext-dubbo.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
       xsi:schemaLocation="
            http://www.springframework.org/schema/beans
            http://www.springframework.org/schema/beans/spring-beans-4.3.xsd
            http://dubbo.apache.org/schema/dubbo
            http://dubbo.apache.org/schema/dubbo/dubbo.xsd">

    <!-- 调用方应用信息，name可以随便起名，但是不能重复 -->
    <dubbo:application name="hello-world-consumer">
        <!-- qos默认开启，为了不与producer端口冲突，需要修改此内容 -->
        <dubbo:parameter key="qos.enable" value="true"/>
        <dubbo:parameter key="qos.accept.foreign.ip" value="false"/>
        <dubbo:parameter key="qos.port" value="33333"/>
    </dubbo:application>

    <!-- 使用zookeeper为注册中心，客户端使用curator -->
    <dubbo:registry address="zookeeper://localhost:2181" client="curator"/>


    <!-- 调用远程Provider暴露的服务（通过接口调用） -->
    <dubbo:reference
            id="providerService"
            interface="cn.lnd.dubbo.service.IHelloService"/>
</beans>
```



#### 2.加载Dubbo配置文件，并调用远程服务

```java
@ImportResource(locations = {"classpath:applicationContext-dubbo.xml"})
@SpringBootApplication
public class DubboConsumerApplication {

    public static void main(String[] args) {
        ConfigurableApplicationContext context = SpringApplication.run(DubboConsumerApplication.class, args);
        IHelloService providerService = context.getBean("providerService", IHelloService.class);
        System.out.println(providerService.sayHello("Dubbo"));
    }

}
```





# Dubbo基础架构分析

![Dubbo基本工作原理图](https://dubbo.apache.org/imgs/architecture.png)

# Dubbo和SpringCloud比较

- Dubbo 强调的是服务调用的速度。
- SpringCloud 强调的是微服务的治理。
- 虽然 Dubbo 和 SpringCloud 在一些功能上存在一些重合，但是侧重不同。当前主流的做法就是 SpringCloud 结合 Dubbo 使用。



----



# Dubbo整体架构脉络

![image-20220814175936628](https://raw.githubusercontent.com/LvNengDong/pic-go/main/img/dubbo/image-20220814175936628.png)



### 顶部

- 左半部分蓝色的区域属于 Consumer 区域，右半部分绿色的区域属于 Provider 区域，横跨两个区域的组件是一些共用的公共组件。
- Start：启动
- Inherit：继承
- init：初始化
- Call：回调
- Depend：依赖



### 左侧

- **Business**：业务部分。这块是程序员最常接触到的部分，包含了我们写的接口，以及为接口提供的实现类。
- **RPC**：
- **Remoting**：



### 右侧

- **User API**：如果你只是一个 Dubbo 框架的使用者，那么最重点关注的层面只需要是 Service 和 Config 层就可以了，你可以通过这两层完成 Dubbo 提供的现有的功能
- **Contributor SPI**：如果你想在现有 Dubbo 框架的基础上进行二次开发，那么你就必须了解 Dubbo 更为底层的一些内部实现。



# Dubbo注册中心剖析

> - Dubbo Admin 构建
> - Dubbo3 注册中心工作流程
> - Dubbo3 注册中心选择

## Dubbo3 注册中心数据存储结构概述

![](http://processon.com/chart_image/62f8d9051e08530de5e98a92.png)

### 验证

- 启动 ZK 服务器：`{ZK根目录}/bin/zkServer.cmd`
- 启动 ZK 客户端：`{ZK根目录}/bin/zkCli.cmd`
- 启动一个 Dubbo 微服务实例（以 dubbo-provider 为例）







通过微服务信息的描述找到对应的接口

减少 ZK 在节点数量上的压力