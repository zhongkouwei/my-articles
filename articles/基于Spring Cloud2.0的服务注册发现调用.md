# 基于Spring boot2的服务注册发现与调用
> 最近在尝试使用Spring cloud构建微服务组件，参考了许多文档与博客，但在这个过程中发现了一个问题，网络上的文档等大多是基于Spring boot1.5的，Spring boot在升级到2.*之后，Spring Cloud组件相关配置方式发生了许多变化，按照原有的配置方式会出现很多错误，所以记录下基于Spring boot2的配置过程。并对调用过程进行分析。

### 1、构建Eureka服务
众所周知，在微服务里面，业务系统之间有复杂的调用关系，他们之间需要有一个注册中心来管理。也就是服务发现代理，它应该能起到如下作用：

 - 普通服务能使用服务发现代理进行注册。
 - 服务客户端能通过服务发现代理查找所需的服务信息。
 - 多个服务发现代理间能共享服务的注册信息。
 - 服务发现代理能检测服务的健康信息。

 在Spring Cloud中，有多个组件可以起到服务发现代理的作用，但一般比较常用、较专业、可用性较高的是Eureka，Eureka分为Server和Client，Server是注册中心，Client是注册/调用方，接下来我们构建Eureka Server来启动一个服务发现代理。

#### 1.1 Idea创建Eureka项目
1. 新建项目，选择Spring boot项目。
[![APhbOs.md.png](https://s2.ax1x.com/2019/03/12/APhbOs.md.png)](https://imgchr.com/i/APhbOs)

2. 选择Eureka Server。
[![APhOwq.md.png](https://s2.ax1x.com/2019/03/12/APhOwq.md.png)](https://imgchr.com/i/APhOwq)
如图所示，Idea会自动帮你创建Eureka项目并生成pom文件，如果不想用自动生成的话，可以手动设置pom文件，引入如下包。

```JAVA
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

#### 1.2 添加配置文件
接下来，需要创建src/main/resources/application.yml文件（或application.properties，两者作用是一样的，但.yml文件为树形结构，更容易理解）。添加如下配置。

```JAVA
server:
  port: 8761 #Eureka服务监听端口

eureka:
  client:
    registerWithEureka: false #不要使用Eureka服务注册
    fetchRegistry: false #不要在本地缓存注册表信息
  #server:
  #  waitTimeInMsWhenSyncEmpty: 5 #在服务器接收请求之前等待的初始时间
```
registerWithEureka属性的含义是通过Eureka服务来注册自身，这里设置为false时因为它本身就是Eureka服务，不需要进行注册。fetchRegistry是在客户端本地缓存注册表信息。**waitTimeInMsWhenSyncEmpty**属性比较重要。Eureka会等待5min才会通告任何通过它注册的服务，所以本地运行可以注释此行，以加快速度。

每次服务注册时，需要等待30s，才会成功显示在Eureka服务中，因为Eureka需要从服务接收3次连续的心跳包ping，才能确认此服务是健康的。

#### 1.3 通过注解启动
最后一步是在启动类中添加@EnableEurekaServer注解，就可以让此应用程序成为一个Eureka服务。

```
@SpringBootApplication
@EnableEurekaServer
public class EurekaserverApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaserverApplication.class, args);
    }

}
```
启动后，访问http://localhost:8761/便可看到Eureka界面。
[![AP594f.md.png](https://s2.ax1x.com/2019/03/12/AP594f.md.png)](https://imgchr.com/i/AP594f)

### 2 创建服务提供方
#### 2.1 创建Spring Web、Eureka Client应用
1. 通过Idea创建项目
选择Spring Web和Eureka Discovery,Idea会自动创建该类型的项目。
[![APIQot.md.png](https://s2.ax1x.com/2019/03/12/APIQot.md.png)](https://imgchr.com/i/APIQot)

2. 通过pom创建

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

#### 2.2 创建配置文件
在src/main/java/resources/application.yml创建如下配置。

```
spring:
  application:
    name: mailservice #在Eureka中注册的服务名称，将在服务查找时使用
server:
  port: 8080
eureka:
  instance:
    prefer-ip-address: true #注册服务的ip
  client:
    register-with-eureka: true #使用Eureka注册服务
    fetch-registry: true #在本地缓存注册表
    service-url:
      defaultZone: http://localhost:8761/eureka #Eureka服务位置
```
spring.application.name将用来在Eureka中注册本服务，作为应用程序的ID，不管该服务有多少个实例，它们的应用ID是唯一的。而实例ID是一个随机数。

eureka.instance.prefer-ip-address的值设置为true，是将本服务的Ip注册到Eureka，而不是域名，一般建议使用ip，因为在使用容器部署时，容器本身不具有DNS记录，服务就没办法正确解析地址。基于云的微服务本身是短暂和无状态的，所以使用ip地址更适合。
service-url属性的值是一个map，因为单个Eureka并不能保证高可用，所以一般会为客户端提供一个Eureka服务列表，在单个Eureka服务宕机时选择另外的进行使用。

#### 2.3 启动服务
在这个demo中，我创建了一个名叫mailservice的服务，在启动后，可以在Eureka管理界面看到该服务。
[![APIXTI.md.png](https://s2.ax1x.com/2019/03/12/APIXTI.md.png)](https://imgchr.com/i/APIXTI)
调用/eureka/apps/MAILSERVICE也可以看到该服务的详细信息。
[![APopp8.md.png](https://s2.ax1x.com/2019/03/12/APopp8.md.png)](https://imgchr.com/i/APopp8)
可以看到，该服务已经成功注册到Eureka中了。

### 3 使用服务发现来查找服务
服务注册成功后，我们可以使用另外的服务客户端来查找和调用该服务mailservice，而不用知道该服务的位置。现在我们创建另一个服务，让这两个服务互相调用。

为了实现这个目的，我们需要引入客户端库。能够实现服务查找和调用功能的在Spring Cloud中有三个客户端，分别为：

 - Spring DiscoveryClient.
 - RestTemplate
 - Netflix Feign。

下面我们使用Feign来实现服务调用。

#### 3.1 创建Feign项目
和上面创建的mailservice相比，多了一个feign依赖。

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```
#### 3.2 创建配置文件
```
server:
  port: 6001
eureka:
  client:
    service-url:
      defaultZone : http://localhost:8761/eureka
spring:
  application:
    name: demoService
```

#### 3.3 通过注解启动Feign客户端
在启动类添加一个新注解@EnableFeignClients

```
@SpringBootApplication
@EnableFeignClients
@EnableEurekaClient
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

}
```

#### 3.4 定义一个用于调用mailservice的Feign接口。
在上面的mailservice中， 我定义了一个接口/v1/send

```
@RestController
public class MailController {

    @RequestMapping(value = "v1/send/${msg}")
    public String send(@PathVariable("msg") String msg) {
        return msg;
    }
}
```
在调用方服务demoService中，需要定义一个JAVA接口来调用REST接口，如下：

```
@FeignClient(name = "mailservice")
public interface MailClient {

    @RequestMapping("/v1/send/{message}")
    String send(@PathVariable("message") String message);
}
```
我们通过@FeignClient注解来标志服务，其中的name为在Eureka中注册的应用ID。在这个接口中的send方法中，我们加上@RequestMapping注解来将此方法的实现映射到url。参数的传递通过@PathVariable注解将接口参数映射到url参数。远程接口的返回值将自动映射到接口返回值。

下面，我们启动demoService来尝试调用mailService服务。
[![APH91K.md.png](https://s2.ax1x.com/2019/03/12/APH91K.md.png)](https://imgchr.com/i/APH91K)
我们调用位于6001的demo，成功返回结果。

#### 3.5 使用Feign客户端时的错误处理
在通过Feign客户端调用服务时，任何被调用的服务返回的HTTP状态码4xx~5xx都将映射为FeignException，其中包含可以被解析为特定错误消息的JSON结构。可以自定义错误解析编码类来映射自定义的异常类。

### 4 服务发现架构如何保证可用性
在上面的过程中，我们构建了一整个服务注册发现调用流程，在其中可以发现很多问题，比如Eureka挂掉怎么办，客户端联系不上Eureka怎么办，似乎这样的架构是非常脆弱的，但Spring Cloud提供了一系列的机制来保证可用性，下面来进行分析。

#### 4.1 服务发现集群
[![APLPXT.md.png](https://s2.ax1x.com/2019/03/12/APLPXT.md.png)](https://imgchr.com/i/APLPXT)
每个服务实例启动时会通过一个或多个服务发现代理进行注册它们的IP等信息，但即使只在一个服务发现节点注册。由于服务发现代理间使用了数据传播的点对点模型，每个节点的数据都会同步到服务发现集群中的所有节点。
在注册成功后，每个服务实例会按时间间隔发送心跳包，来推送自身的状态，一段时间没有发送心跳包时，会被从服务实例池中移除。
通过这样的方式，即使单个服务发现代理节点宕机，其他节点也能继续提供同样的服务。

#### 4.2 客户端负载均衡
[![APLjKK.md.png](https://s2.ax1x.com/2019/03/12/APLjKK.md.png)](https://imgchr.com/i/APLjKK)

上图是加入了客户端缓存和负载均衡，和直接请求服务发现代理相比，进行了如下优化。

1. 首先请求服务发现代理，获取它请求的服务的所有实例，并在本地进行缓存。
2. 当需要调用该服务时，首先从缓存中查找所有实例信息，再使用负载均衡算法，选择一个实例进行调用。
3. 客户端将定期请求服务发现代理，刷新缓存数据。
4. 如果本地缓存数据错误，请求到了错误的实例，本地缓存将会失效，并重新请求服务发现代理的数据。

通过以上的机制，能最大程度地保证客户端能够正确地查找到所需服务的实例信息。




