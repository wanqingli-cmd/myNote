## 基于 Hystrix 信号量机制实现资源隔离
Hystrix 里面核心的一项功能，其实就是所谓的**资源隔离**，要解决的最最核心的问题，就是将多个依赖服务的调用分别隔离到各自的资源池内。避免说对某一个依赖服务的调用，因为依赖服务的接口调用的延迟或者失败，导致服务所有的线程资源全部耗费在这个服务的接口调用上。一旦说某个服务的线程资源全部耗尽的话，就可能导致服务崩溃，甚至说这种故障会不断蔓延。

Hystrix 实现资源隔离，主要有两种技术：

- 线程池
- 信号量

默认情况下，Hystrix 使用线程池模式。

前面已经说过线程池技术了，这一小节就来说说信号量机制实现资源隔离，以及这两种技术的区别与具体应用场景。

### 信号量机制
信号量的资源隔离只是起到一个开关的作用，比如，服务 A 的信号量大小为 10，那么就是说它同时只允许有 10 个 tomcat 线程来访问服务 A，其它的请求都会被拒绝，从而达到资源隔离和限流保护的作用。

![hystrix-semphore](/images/hystrix-semphore.png)

### 线程池与信号量区别
线程池隔离技术，并不是说去控制类似 tomcat 这种 web 容器的线程。更加严格的意义上来说，Hystrix 的线程池隔离技术，控制的是 tomcat 线程的执行。Hystrix 线程池满后，会确保说，tomcat 的线程不会因为依赖服务的接口调用延迟或故障而被 hang 住，tomcat 其它的线程不会卡死，可以快速返回，然后支撑其它的事情。

线程池隔离技术，是用 Hystrix 自己的线程去执行调用；而信号量隔离技术，是直接让 tomcat 线程去调用依赖服务。信号量隔离，只是一道关卡，信号量有多少，就允许多少个 tomcat 线程通过它，然后去执行。

![hystrix-semphore-thread-pool](/images/hystrix-semphore-thread-pool.png)

**适用场景**：
- **线程池技术**，适合绝大多数场景，比如说我们对依赖服务的网络请求的调用和访问、需要对调用的 timeout 进行控制（捕捉 timeout 超时异常）。
- **信号量技术**，适合说你的访问不是对外部依赖的访问，而是对内部的一些比较复杂的业务逻辑的访问，并且系统内部的代码，其实不涉及任何的网络请求，那么只要做信号量的普通限流就可以了，因为不需要去捕获 timeout 类似的问题。


Hystrix 信号量和线程池隔离的差异
信号量线程池隔离差异
信号量隔离适应非网络请求，因为是同步的请求，无法支持超时，只能依靠协议本身
线程池隔离，即，每个实例都增加个线程池进行隔离
                         线程池隔离	                                                       信号量隔离
是否支持熔断	          支持，当线程池到达MaxSize后，再请求会触发fallback接口进行熔断	       支持，当信号量达到maxConcurrentRequest后，再请求会触发fallback
是否支持超时	          支持，可直接返回	                                                    不支持，如果阻塞，只能通过调用协议
隔离原理	           每个服务单独用线程池	                                                通过信号量的计数器
是否支持异步调用	    可以是异步，也可以是同步。看调用的方法	                                 同步调用，不支持异步
资源消耗	           大，大量线程的上下文切换，容易造成机器负载高	                         小，只是个计数器



网关是通过线程池隔离 同步的路由方式，适当的线程池大小配置能够防止网关负载过大
 

hystrixCommand线程
   线程池隔离：
      1、调用线程和hystrixCommand线程不是同一个线程，并发请求数受到线程池（不是容器tomcat的线程池，而是hystrixCommand所属于线程组的线程池）中的线程数限制，默认是10。
      2、这个是默认的隔离机制
      3、hystrixCommand线程无法获取到调用线程中的ThreadLocal中的值
   信号量隔离：
      1、调用线程和hystrixCommand线程是同一个线程，默认最大并发请求数是10
      2、调用数度快，开销小，由于和调用线程是处于同一个线程，所以必须确保调用的微服务可用性足够高并且返回快才用

注意：如果发生找不到上下文的运行时异常，可考虑将隔离策略设置为SEMAPHONE。

　　
  
  
  信号量模式:
　　在该模式下，接收请求和执行下游依赖在同一个线程内完成，不存在线程上下文切换所带来的性能开销，所以大部分场景应该选择信号量模式，但是在下面这种情况下，信号量模式并非是一个好的选择。

　　比如一个接口中依赖了3个下游：serviceA、serviceB、serviceC，且这3个服务返回的数据互相不依赖，这种情况下如果针对A、B、C的熔断降级使用信号量模式，那么接口耗时就等于请求A、B、C服务耗时的总和，无疑这不是好的方案。

　　另外，为了限制对下游依赖的并发调用量，可以配置Hystrix的execution.isolation.semaphore.maxConcurrentRequests，当并发请求数达到阈值时，请求线程可以快速失败，执行降级

　　实现也很简单，一个简单的计数器，当请求进入熔断器时，执行tryAcquire(),计数器加1，结果大于阈值的话，就返回false，发生信号量拒绝事件，执行降级逻辑。当请求离开熔断器时，执行release()，计数器减1。

　线程池模式:
　　在该模式下，用户请求会被提交到各自的线程池中执行，把执行每个下游服务的线程分离，从而达到资源隔离的作用。当线程池来不及处理并且请求队列塞满时，新进来的请求将快速失败，可以避免依赖问题扩散。

　　在信号量模式提到的问题，对所依赖的多个下游服务，通过线程池的异步执行，可以有效的提高接口性能。

优势

减少所依赖服务发生故障时的影响面，比如ServiceA服务发生异常，导致请求大量超时，对应的线程池被打满，这时并不影响ServiceB、ServiceC的调用。
如果接口性能有变动，可以方便的动态调整线程池的参数或者是超时时间，前提是Hystrix参数实现了动态调整。
缺点

请求在线程池中执行，肯定会带来任务调度、排队和上下文切换带来的开销。
因为涉及到跨线程，那么就存在ThreadLocal数据的传递问题，比如在主线程初始化的ThreadLocal变量，在线程池线程中无法获取
三 Zuul中的实现
　　Zuul默认是使用信号量隔离，并且信号量的大小是100，请求的并发线程超过100就会报错

　　可以调大该信号量的最大值来提高性能，配置如下：

zuul:
  semaphore:
    max-semaphores: 5000
　　也可以改为使用线程隔离，调大hystrix线程池线程大小，该线程池默认10个线程，配置如下：

复制代码
zuul:
  ribbonIsolationStrategy: THREAD

hystrix:
  threadpool:
    default:
      coreSize: 100
      maximumSize: 400
      allowMaximumSizeToDivergeFromCoreSize: true
      maxQueueSize: -1
复制代码
maximumSize：最大线程数量

allowMaximumSizeToDivergeFromCoreSize：是否让maximumSize生效，false的话则只有coreSize会生效
maxQueueSize：线程池的队列大小，-1代表使用SynchronousQueue队列




### 信号量简单 Demo
业务背景里，比较适合信号量的是什么场景呢？

比如说，我们一般来说，缓存服务，可能会将一些量特别少、访问又特别频繁的数据，放在自己的纯内存中。

举个栗子。一般我们在获取到商品数据之后，都要去获取商品是属于哪个地理位置、省、市、卖家等，可能在自己的纯内存中，比如就一个 Map 去获取。对于这种直接访问本地内存的逻辑，比较适合用信号量做一下简单的隔离。

优点在于，不用自己管理线程池啦，不用 care timeout 超时啦，也不需要进行线程的上下文切换啦。信号量做隔离的话，性能相对来说会高一些。

假如这是本地缓存，我们可以通过 cityId，拿到 cityName。
```java
public class LocationCache {
    private static Map<Long, String> cityMap = new HashMap<>();

    static {
        cityMap.put(1L, "北京");
    }

    /**
     * 通过cityId 获取 cityName
     *
     * @param cityId 城市id
     * @return 城市名
     */
    public static String getCityName(Long cityId) {
        return cityMap.get(cityId);
    }
}
```

写一个 GetCityNameCommand，策略设置为**信号量**。run() 方法中获取本地缓存。我们目的就是对获取本地缓存的代码进行资源隔离。
```java
public class GetCityNameCommand extends HystrixCommand<String> {

    private Long cityId;

    public GetCityNameCommand(Long cityId) {
        // 设置信号量隔离策略
        super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("GetCityNameGroup"))
                .andCommandPropertiesDefaults(HystrixCommandProperties.Setter()
                        .withExecutionIsolationStrategy(HystrixCommandProperties.ExecutionIsolationStrategy.SEMAPHORE)));

        this.cityId = cityId;
    }

    @Override
    protected String run() {
        // 需要进行信号量隔离的代码
        return LocationCache.getCityName(cityId);
    }
}
```

在接口层，通过创建 GetCityNameCommand，传入 cityId，执行 execute() 方法，那么获取本地 cityName 缓存的代码将会进行信号量的资源隔离。
```java
@RequestMapping("/getProductInfo")
@ResponseBody
public String getProductInfo(Long productId) {
    HystrixCommand<ProductInfo> getProductInfoCommand = new GetProductInfoCommand(productId);

    // 通过command执行，获取最新商品数据
    ProductInfo productInfo = getProductInfoCommand.execute();

    Long cityId = productInfo.getCityId();

    GetCityNameCommand getCityNameCommand = new GetCityNameCommand(cityId);
    // 获取本地内存(cityName)的代码会被信号量进行资源隔离
    String cityName = getCityNameCommand.execute();

    productInfo.setCityName(cityName);

    System.out.println(productInfo);
    return "success";
}
```
