> 主要来自于青空の霞光学习的笔记，代码示例全部都成功 ---- [青空の霞光](https://www.bilibili.com/video/BV1AL4y1j7RY?p=9)

## LoadBalancer 负载均衡

前面我们讲解了如何对服务进行拆分、如何通过Eureka服务器进行服务注册与发现，那么现在我们来看看，它的负载均衡到底是如何实现的，实际上之前演示的负载均衡是依靠LoadBalancer实现的。

在2020年前的SpringCloud版本是采用Ribbon作为负载均衡实现，但是2020年的版本之后SpringCloud把Ribbon移除了，进而用自己编写的LoadBalancer替代。

那么，负载均衡是如何进行的呢？

### 负载均衡

实际上，在添加`@LoadBalanced`注解之后，会启用拦截器对我们发起的服务调用请求进行拦截（注意这里是针对我们发起的请求进行拦截），叫做`LoadBalancerInterceptor`，它实现`ClientHttpRequestInterceptor`接口：

```java
@FunctionalInterface
public interface ClientHttpRequestInterceptor {
    // 经过一系列的拦截器过滤，返回响应
    ClientHttpResponse intercept(HttpRequest request, byte[] body, ClientHttpRequestExecution execution) throws IOException;
}
```

主要是对`intercept`方法的实现：

```java
public ClientHttpResponse intercept(final HttpRequest request, final byte[] body, final ClientHttpRequestExecution execution) throws IOException {
    URI originalUri = request.getURI();
    String serviceName = originalUri.getHost();
    Assert.state(serviceName != null, "Request URI does not contain a valid hostname: " + originalUri);
    // 主要就是就这个 execute 方法
    return (ClientHttpResponse)this.loadBalancer.execute(serviceName, this.requestFactory.createRequest(request, body, execution));
}
```

我们可以打个断点看看实际是怎么在执行的，可以看到：

![image-20220323220519463](SpringClouid LoadBalancer 学习.assets/e6c9d24ely1h0k61wb6pxj222y0cm77n.jpg)

![image-20220323220548051](SpringClouid LoadBalancer 学习.assets/e6c9d24ely1h0k62dm3knj21yi0fgwiz.jpg)

服务端会在发起请求时执行这些拦截器。

那么这个拦截器做了什么事情呢，首先我们要明确，我们给过来的请求地址，并不是一个有效的主机名称，而是服务名称，那么怎么才能得到真正需要访问的主机名称呢，肯定是得找Eureka获取的。

我们来看看`loadBalancer.execute()`做了什么，它的具体实现为`BlockingLoadBalancerClient`：

```java
//从上面给进来了服务的名称和具体的请求实体
public <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException {
    String hint = this.getHint(serviceId);
    LoadBalancerRequestAdapter<T, DefaultRequestContext> lbRequest = new LoadBalancerRequestAdapter(request, new DefaultRequestContext(request, hint));
    Set<LoadBalancerLifecycle> supportedLifecycleProcessors = this.getSupportedLifecycleProcessors(serviceId);
    supportedLifecycleProcessors.forEach((lifecycle) -> {
        lifecycle.onStart(lbRequest);
    });
  	//可以看到在这里会调用choose方法自动获取对应的服务实例信息
    ServiceInstance serviceInstance = this.choose(serviceId, lbRequest);
    if (serviceInstance == null) {
        supportedLifecycleProcessors.forEach((lifecycle) -> {
            lifecycle.onComplete(new CompletionContext(Status.DISCARD, lbRequest, new EmptyResponse()));
        });
      	//没有发现任何此服务的实例就抛异常（之前的测试中可能已经遇到了）
        throw new IllegalStateException("No instances available for " + serviceId);
    } else {
      	//成功获取到对应服务的实例，这时就可以发起HTTP请求获取信息了
        return this.execute(serviceId, serviceInstance, lbRequest);
    }
}
```

所以，实际上在进行负载均衡的时候，会向Eureka发起请求，选择一个可用的对应服务，然后会返回此服务的主机地址等信息：

![image-20220324120741736](SpringClouid LoadBalancer 学习.assets/e6c9d24ely1h0kuedkhinj221e0jin2y.jpg)

### 自定义负载均衡策略

LoadBalancer默认提供了两种负载均衡策略：

* RandomLoadBalancer  -  随机分配策略
* **(默认)** RoundRobinLoadBalancer  -  轮询分配策略

现在我们希望修改默认的负载均衡策略，可以进行指定，比如我们现在希望用户服务采用随机分配策略，我们需要先创建随机分配策略的配置类（不用加`@Configuration`）：

```java
public class LoadBalancerConfig {
  	//将官方提供的 RandomLoadBalancer 注册为Bean
    @Bean
    public ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(Environment environment, LoadBalancerClientFactory loadBalancerClientFactory){
        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RandomLoadBalancer(loadBalancerClientFactory.getLazyProvider(name, ServiceInstanceListSupplier.class), name);
    }
}
```

接着我们需要为对应的服务指定负载均衡策略，直接使用注解即可：

```java
@Configuration
@LoadBalancerClient(value = "userservice",      //指定为 userservice 服务，只要是调用此服务都会使用我们指定的策略
                    configuration = LoadBalancerConfig.class)   //指定我们刚刚定义好的配置类
public class BeanConfig {
    @Bean
    @LoadBalanced
    RestTemplate template(){
        return new RestTemplate();
    }
}
```

接着我们在`BlockingLoadBalancerClient`中添加断点，观察是否采用我们指定的策略进行请求：

![image-20220324221750289](SpringClouid LoadBalancer 学习.assets/e6c9d24ely1h0lc17or9aj221y07swhq.jpg)

![image-20220324221713964](SpringClouid LoadBalancer 学习.assets/e6c9d24ely1h0lc0mbsmqj21ye07yjuh.jpg)

发现访问userservice服务的策略已经更改为我们指定的策略了。

### OpenFeign实现负载均衡

官方文档：https://docs.spring.io/spring-cloud-openfeign/docs/current/reference/html/

Feign和RestTemplate一样，也是HTTP客户端请求工具，但是它的使用方式更加便捷。首先是依赖：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

接着在启动类添加`@EnableFeignClients`注解：

```java
@SpringBootApplication
@EnableFeignClients
public class BorrowApplication {
    public static void main(String[] args) {
        SpringApplication.run(BorrowApplication.class, args);
    }
}
```

那么现在我们需要调用其他微服务提供的接口，该怎么做呢？我们直接创建一个对应服务的接口类即可：

```java
@FeignClient("userservice")   //声明为userservice服务的HTTP请求客户端
public interface UserClient {
}
```

接着我们直接创建所需类型的方法，比如我们之前的：

```java
RestTemplate template = new RestTemplate();
User user = template.getForObject("http://userservice/user/"+uid, User.class);
```

现在可以直接写成这样：

```java
@FeignClient("userservice")
public interface UserClient {

  	//路径保证和其他微服务提供的一致即可
    @RequestMapping("/user/{uid}")
    User getUserById(@PathVariable("uid") int uid);  //参数和返回值也保持一致
}
```

接着我们直接注入使用（有Mybatis那味了）：

```java
@Resource
UserClient userClient;

@Override
public UserBorrowDetail getUserBorrowDetailByUid(int uid) {
    List<Borrow> borrow = mapper.getBorrowsByUid(uid);
    
    User user = userClient.getUserById(uid);
    //这里不用再写IP，直接写服务名称bookservice
    List<Book> bookList = borrow
            .stream()
            .map(b -> template.getForObject("http://bookservice/book/"+b.getBid(), Book.class))
            .collect(Collectors.toList());
    return new UserBorrowDetail(user, bookList);
}
```

访问，可以看到结果依然是正确的：

![image-20220324181614387](SpringClouid LoadBalancer 学习.assets/e6c9d24ely1h0l51tto72j229e080dhe.jpg)

并且我们可以观察一下两个用户微服务的调用情况，也是以负载均衡的形式进行的。

按照同样的方法，我们接着将图书管理服务的调用也改成接口形式：

![image-20220324181740566](SpringClouid LoadBalancer 学习.assets/e6c9d24ely1h0l53boxmlj21j60bgq51.jpg)

最后我们的Service代码就变成了：

```java
@Service
public class BorrowServiceImpl implements BorrowService {

    @Resource
    BorrowMapper mapper;

    @Resource
    UserClient userClient;
    
    @Resource
    BookClient bookClient;

    @Override
    public UserBorrowDetail getUserBorrowDetailByUid(int uid) {
        List<Borrow> borrow = mapper.getBorrowsByUid(uid);

        User user = userClient.getUserById(uid);
        List<Book> bookList = borrow
                .stream()
                .map(b -> bookClient.getBookById(b.getBid()))
                .collect(Collectors.toList());
        return new UserBorrowDetail(user, bookList);
    }
}
```

继续访问进行测试：

![image-20220324181910173](SpringClouid LoadBalancer 学习.assets/e6c9d24ely1h0l54vuecvj226206igmz.jpg)

OK，正常。

当然，Feign也有很多的其他配置选项，这里就不多做介绍了，详细请查阅官方文档。