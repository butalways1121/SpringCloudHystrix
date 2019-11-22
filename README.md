# SpringCloudHystrix
SpringCloud之断路器(Hystrix)和断路器监控(Dashboard)
---
&emsp;&emsp;**本篇主要介绍的是SpringCloud中的断路器(Hystrix)和断路器指标看板(Dashboard)的相关使用知识，项目实例也非常简单，戳[这里](https://github.com/butalways1121/SpringCloudHystrix)下载源码。**
<!-- more -->
## 一、SpringCloud Hystrix
### 1.SpringCloud Hystrix简介
&emsp;&emsp;Netflix创建了一个名为Hystrix的库，它实现了断路器模式，主要是为了解决服务雪崩效应的一个组件，是保护服务高可用的最后一道防线。在分布式环境中，许多服务依赖关系中的一些必然会失败。Hystrix是一个库，它通过添加延迟容忍和容错逻辑来帮助您控制这些分布式服务之间的交互。Hystrix通过隔离服务之间的访问点、停止跨服务的级联故障并提供回退选项来实现这一点，所有这些选项都提高了系统的总体弹性。

Hystrix的设计目的如下:
* 为通过第三方客户端库访问的依赖项(通常通过网络)提供保护和控制延迟和故障；
* 停止复杂分布式系统中的级联故障；
* 故障快速恢复；
* 在可能的情况下，后退并优雅地降级；
* 启用近实时监视、警报和操作控制。
### 2.SpringCloud Hystrix示例
#### （1）服务端
&emsp;&emsp;同之前一样首先创建一个注册中心springcloud-hystrix-eureka，pom.xml完整配置如下：
```bash
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>1.0.0</groupId>
	<artifactId>springcloud-hystrix-eureka</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>
	<name>springcloud-eureka</name>
	<url>http://maven.apache.org</url>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.5.9.RELEASE</version>
		<relativePath />
	</parent>
	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<java.version>1.8</java.version>
		<spring-cloud.version>Dalston.RELEASE</spring-cloud.version>
	</properties>

	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-eureka-server</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
		<!-- feign依赖 -->
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-feign</artifactId>
			<!-- <artifactId>spring-cloud-starter-openfeign</artifactId> -->
		</dependency>
		<!-- hystrix依赖 -->
		<dependency>
	        <groupId>org.springframework.cloud</groupId>
	        <artifactId>spring-cloud-starter-hystrix</artifactId>
    	</dependency>
        <!-- Spring Boot 热部署 class文件之后会自动重启 -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-devtools</artifactId>
			<optional>true</optional>
		</dependency>
	</dependencies>
	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>${spring-cloud.version}</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>
	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>
</project>
```
application.properties文件配置如下：
```bash
spring.application.name=springcloud-hystrix-eureka-server
server.port=8002
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
eureka.client.serviceUrl.defaultZone=http://localhost:8002/eureka/
```
启动类代码：
```bash
@SpringBootApplication
@EnableEurekaServer
public class App 
{
    public static void main( String[] args )
    {
    	SpringApplication.run(App.class, args);
        System.out.println( "hystrix注册中心服务启动..." );
    }
}
```
#### （2）客户端
&emsp;&emsp;这次同样是新建两个项目springcloud-hystrix-consumer和springcloud-hystrix-consumer2，前者使用Feign做转发，同时新增一个回调类，用于处理断路的情况，后者只是提供一个简单的接口用于打印信息即可，这两个项目的依赖同springcloud-hystrix-eureka即可。
#### 1）springcloud-hystrix-consumer
application.properties配置如下，其中`feign.hystrix.enabled`配置表示是否启用熔断机制：
```bash
spring.application.name=springcloud-hystrix-consumer
server.port=9004
eureka.client.serviceUrl.defaultZone=http://localhost:8002/eureka/
feign.hystrix.enabled=true
```
启动类代码：
```bash
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
public class App 
{
    public static void main( String[] args )
    {
    	SpringApplication.run(App.class, args);
        System.out.println( "hystrix第一个消费者服务启动..." );
    }
}
```
接着定义一个转发类把请求转发到服务springcloud-hystrix-consumer2上，同时在@FeignClient注解上添加一个回调方法fallback，表示服务熔断的时候返回fallback类中的内容：
```bash
@FeignClient(name= "springcloud-hystrix-consumer2",fallback = HelloRemoteHystrix.class)
public interface HelloRemote {
    @RequestMapping(value = "/hello")
    public String hello(@RequestParam(value = "name") String name);
}
```
新增一个回调类，用于处理断路的情况，就是定义上述转发类的实现类，对应的方法就是转发类熔断时调用的方法：
```bash
@Component
public class HelloRemoteHystrix implements HelloRemote{

    @Override
    public String hello(@RequestParam(value = "name") String name) {
        return name+", 请求另一个服务失败!";
    }
}
```
最后，在提供一个入口供外部调用，然后调用上述的方法将请求进行转发：
```bash
@RestController
public class ConsumerController {
	@Autowired
    HelloRemote helloRemote;

    @RequestMapping("/hello/{name}")
    public String index(@PathVariable("name") String name) {
    	System.out.println("接受到请求参数:"+name+",进行转发到其他服务");
        return helloRemote.hello(name);
    }
}
```
#### 2）springcloud-hystrix-consumer2
application.properties配置：
```bash
spring.application.name=springcloud-hystrix-consumer2
server.port=9005
eureka.client.serviceUrl.defaultZone=http://localhost:8002/eureka/
```
启动类代码：
```bash
@SpringBootApplication
@EnableEurekaClient
public class App 
{
    public static void main( String[] args )
    {
    	SpringApplication.run(App.class, args);
        System.out.println( "hystrix第二个消费者服务启动..." );
    }
}
```
提供打印信息的控制层:
```bash
@RestController
public class ConsumerController {

	@RequestMapping("/hello")
    public String index(@RequestParam String name) {
        return name+",Hello World";
    }
}
```
#### （3）测试
&emsp;&emsp;完成如上的工程开发之后，依次启动服务端和客户端的springcloud-hystrix-eureka、springcloud-hystrix-consumer和springcloud-hystrix-consumer2这三个程序，然后在浏览器界面输入`http://localhost:8002/`，即可查看注册中心的信息：
![](https://raw.githubusercontent.com/butalways1121/img-Blog/master/80.png)

接着，在浏览器中输入`http://localhost:9005/hello?name=butalways`，浏览器若返回如下则说明springcloud-hystrix-consumer2服务正常：
![](https://raw.githubusercontent.com/butalways1121/img-Blog/master/81.png)

然后在浏览器中输入`http://localhost:9004/hello/bualways`，控制台打印:
`接受到请求参数:pancm,进行转发到其他服务`，同时浏览器显示如下则说明springcloud-hystrix-consumer的Feign调用也是好的：
![](https://raw.githubusercontent.com/butalways1121/img-Blog/master/82.png)

接下来进行短路测试，首先停止springcloud-hystrix-consumer2的服务，然后在浏览器中输入`http://localhost:9004/hello/bualways`，此时控制台应打印`接受到请求参数:butalways,进行转发到其他服务`，浏览器则返回如下：
![](https://raw.githubusercontent.com/butalways1121/img-Blog/master/83.png)

出现以上结果则说明短路的功能已经实现了。
## 二、SpringCloud Hystrix-Dashboard
### 1.Hystrix-Dashboard介绍
&emsp;&emsp;Hystrix-dashboard是一款针对Hystrix进行实时监控的工具，通过Hystrix Dashboard我们可以在直观地看到各Hystrix Command的请求响应时间、请求成功率等数据。

&emsp;&emsp;Hystrix提供了对于微服务调用状态的监控信息，但是需要结合spring-boot-actuator模块一起使用。
Hystrix Dashboard主要用来实时监控Hystrix的各项指标信息。通过Hystrix Dashboard反馈的实时信息，可以帮助我们快速发现系统中存在的问题。
### 2.SpringCloud Hystrix-Dashboard 示例
&emsp;&emsp;这里直接把上面的springcloud-hystrix-consumer项目简单修改下，项目名称改为springcloud-hystrix-dashboard-consumer，在启动类上添加两个注解：
```bash
@EnableCircuitBreaker:表示启用hystrix功能。
@EnableHystrixDashboard:启用HystrixDashboard 断路器看板相关配置。
```
启动类完整配置如下：
```
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
@EnableCircuitBreaker
@EnableHystrixDashboard
public class App 
{
    public static void main( String[] args )
    {
    	SpringApplication.run(App.class, args);
        System.out.println( "hystrix dashboard 服务启动..." );
    }
}
```
然后在application.properties配置文件中新增配置如下，该配置的意思是指定hystrixDashboard的访问路径，SpringBoot2.x以上必须指定，不然是无法进行访问的，访问会出现 Unable to connect to Command Metric Stream 错误，在修改一下端口，完整的配置如下:
```bash
spring.application.name=springcloud-hystrix-dashboard-consumer
server.port=9010
eureka.client.serviceUrl.defaultZone=http://localhost:8002/eureka/
feign.hystrix.enabled=true
management.endpoints.web.exposure.include=hystrix.stream
management.endpoints.web.base-path=/
```
接着，就可以进行测试了，启动springcloud-hystrix-eureka、springcloud-hystrix-dashboard-consumer和springcloud-hystrix-consumer2三个程序，然后在浏览器中输入`http://localhost:9010/hystrix`，出现如下界面，可以通过该界面监控使用hystrix-dashboard的项目：
![](https://raw.githubusercontent.com/butalways1121/img-Blog/master/84.png)

接着依照提示在中间的输入框输入:
`http://localhost:9010/hystrix.stream`,就会出现如下界面：
![](https://raw.githubusercontent.com/butalways1121/img-Blog/master/85.png)

该界面就是Hystrix Dashboard监控的界面了，通过这个界面我们可以很详细的看到程序的信息，图中一些信息的简单说明如下：
![](https://raw.githubusercontent.com/butalways1121/img-Blog/master/86.png)

**注: 如果界面一直提示loading，那么是因为没有进行请求访问，只需在到浏览器上输入:`http://localhost:9010/hello/butalways`，然后刷新该界面就可以进行查看了。**
