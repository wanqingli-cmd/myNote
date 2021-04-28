
**zk，一般来说还好，服务注册和发现，都是很快的**

**eureka，必须优化参数**


更新写缓存到读缓存时间
**eureka.server.responseCacheUpdateIntervalMs = 3000**


默认值为30秒，即每30秒去Eureka Server上获取服务并缓存默认值：30
**eureka.client.registryFetchIntervalSeconds = 30000**


向Eureka Server发送心跳的间隔时间，单位为秒，用于服务续约默认值：30
**eureka.client.leaseRenewalIntervalInSeconds = 30**


**eureka.server.evictionIntervalTimerInMs = 60000**


定义服务失效时间，即Eureka Server检测到Eureka Client木有心跳后（客户端意外下线）多少秒将其剔除默认值：90
**eureka.instance.leaseExpirationDurationInSeconds = 90**

**服务发现的时效性变成秒级，几秒钟可以感知服务的上线和下线**







eureka底层实现是使用concurrentHashMap,图中为eureka的实现原理可以很清楚的理解，其中有个多级缓存，服务每隔30s发送一个心跳。

注册到服务注册中心，接着立马同步到readwith缓存中，接着30s同步到readonly缓存中，然后服务每隔30s拉取注册表即可调用注册中的服务。

服务下线是在eureka中有个每隔60s的定时检查，然后从readwith中剔除，30s后再从readonly中剔除，再会去被拉取。

从中可以看出时间还是比较长的，当在生产环境中还是要优化一下的，服务的发现还是比较慢的。

服务的实例是如何从服务中心剔除的：eureka server 要求client端定时进行续约，也就是发送心跳，来证明该服务实例还存活，是健康的，是可以调用的。

如果租约超过一定的时间没有进行续约操作，eureka server端会主动的剔除，这一点即心跳模式。

所以我们要对参数进行一些优化，来达到服务注册发现的及时。

下面我们总结一下在Eureka中常用的配置选项及代表的含义：

配置

含义

默认值

eureka.client.enabled

是否启用Eureka Client    默认值：true

true

eureka.client.register-with-eureka

表示是否将自己注册到Eureka Server   默认值：true

true

eureka.client.fetch-registry

表示是否从Eureka Server获取注册的服务信息默认值：true

true

eureka.client.serviceUrl.defaultZone

配置Eureka Server地址，用于注册服务和获取服务默认值：http://localhost:8761/eureka

http://localhost:8761/eureka

eureka.client.registry-fetch-interval-seconds

默认值为30秒，即每30秒去Eureka Server上获取服务并缓存默认值：30

30

eureka.instance.lease-renewal-interval-in-seconds

向Eureka Server发送心跳的间隔时间，单位为秒，用于服务续约默认值：30

30

eureka.instance.lease-expiration-duration-in-seconds

定义服务失效时间，即Eureka Server检测到Eureka Client木有心跳后（客户端意外下线）多少秒将其剔除默认值：90

90

eureka.server.enable-self-preservation

用于开启Eureka Server自我保护功能默认值：true

true

eureka.client.instance-info-replication-interval-seconds

更新实例信息的变化到Eureka服务端的间隔时间，单位为秒默认值：30

30

eureka.client.eureka-service-url-poll-interval-seconds

轮询Eureka服务端地址更改的间隔时间，单位为秒。默认值：300

300

eureka.instance.prefer-ip-address

表示使用IP进行配置为不是域名默认值：false

false

eureka.client.healthcheck.enabled

默认Erueka Server是通过心跳来检测Eureka Client的健康状况的，通过置为true改变Eeureka Server对客户端健康检测的方式，改用Actuator的/health端点来检测。默认值：false

false

再贴下一下eureka server的配置：

eureka:
  server:
    wait-time-in-ms-when-sync-empty: 0   #在eureka服务器获取不到集群里对等服务器上的实例时，需要等待的时间，单机默认0
    shouldUseReadOnlyResponseCache: true #eureka是CAP理论种基于AP策略，为了保证强一致性关闭此切换CP 默认不关闭 false关闭
    enable-self-preservation: false    #关闭服务器自我保护，客户端心跳检测15分钟内错误达到80%服务会保护，导致别人还认为是好用的服务
    eviction-interval-timer-in-ms: 6000 #清理间隔（单位毫秒，默认是60*1000）5秒将客户端剔除的服务在服务注册列表中剔除#
    response-cache-update-interval-ms: 3000  #eureka server刷新readCacheMap的时间，注意，client读取的是readCacheMap，这个时间决定了多久会把readWriteCacheMap的缓存更新到readCacheMap上 #eureka server刷新readCacheMap的时间，注意，client读取的是readCacheMap，这个时间决定了多久会把readWriteCacheMap的缓存更新到readCacheMap上默认30s
    response-cache-auto-expiration-in-seconds: 180   #eureka server缓存readWriteCacheMap失效时间，这个只有在这个时间过去后缓存才会失效，失效前不会更新，过期后从registry重新读取注册服务信息，registry是一个ConcurrentHashMap。
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${spring.cloud.client.ip-address}:${spring.application.instance_id:${server.port}}
    hostname: 127.0.0.1
    lease-renewal-interval-in-seconds: 30    # 续约更新时间间隔（默认30秒），eureka客户端向服务端发送心跳的时间间隔
    lease-expiration-duration-in-seconds: 90 # 续约到期时间（默认90秒）
  client:
    #注册到其他eureka
    registerWithEureka: false
    fetchRegistry: false #为true时，可以启动，但报异常：Cannot execute request on any known server ，是否从eureka服务端获取注册信息，消费者需要配置true
    register-with-eureka: false  #表示是否将服务注册到Eureka服务端，由于自身就是Eureka服务端，所以设置为false；
    fetch-registry: false #表示是否从Eureka服务端获取服务信息，因为这里只搭建了一个Eureka服务端，并不需要从别的Eureka服务端同步服务信息，所以这里设置为false；
    instance-info-replication-interval-seconds: 10  #更新实例信息的变化到Eureka服务端的间隔时间，单位为秒
    registry-fetch-interval-seconds: 30  #从eureka服务端获取注册信息的间隔时间
    eureka-service-url-poll-interval-seconds: 300 #轮询Eureka服务端地址更改的间隔时间，单位为秒。
    service-url:
      defaultZone: http://lee:lee@${eureka.instance.hostname}:${server.port}/eureka/
eureka，必须优化参数

eureka.server.responseCacheUpdateIntervalMs = 3000 #eureka server刷新readCacheMap的时间，注意，client读取的是readCacheMap，这个时间决定了多久会把readWriteCacheMap的缓存更新到readCacheMap上 默认30s
eureka.client.registryFetchIntervalSeconds = 30000 #从eureka服务端获取注册信息的间隔时间
eureka.client.leaseRenewalIntervalInSeconds = 30 # 续约更新时间间隔（默认30秒），eureka客户端向服务端发送心跳的时间间隔
eureka.server.evictionIntervalTimerInMs = 60000 #清理间隔（单位毫秒，默认是60*1000）5秒将客户端剔除的服务在服务注册列表中剔除#
eureka.instance.leaseExpirationDurationInSeconds = 90 90 # 续约到期时间（默认90秒）
服务发现的时效性变成秒级，几秒钟可以感知服务的上线和下线 

这样eurek server差不多就优化好了，我们再优化下client端。

贴一下我gateway的参数配置：

spring:
  application:
    name: api-gateway
  cloud:
    config:
      discovery:
        enabled: true
        service-id: CONFIG
      profile: dev
eureka:
  client:
    service-url:
      defaultZone: http://lee:lee@127.0.0.1:8080/eureka
    registry-fetch-interval-seconds: 5 #eureka client刷新本地缓存时间，默认30
  instance:
    hostname: 127.0.0.1
    instance-id: 127.0.0.1:9000
    lease-expiration-duration-in-seconds: 7  #Eureka服务端在收到最后一次心跳之后等待的时间上限，单位为秒，超过则剔除（客户端告诉服务端按照此规则等待自己），默认90
    lease-renewal-interval-in-seconds: 5   #Eureka客户端向服务端发送心跳的时间间隔，单位为秒（客户端告诉服务端自己会按照该规则），默认30
这样配置号以后服务的注册发现几乎可以达到秒级的程度，再结合我上次写的

eureka服务下线通知
李依金，公众号：技术从心EUREKA服务下线邮件通知

就可以很好的实现服务的及时下线通知指定邮箱。

接下来再说下zookeeper和eureka的区别：

CAP,C是一致性，A是可用性，P是分区容错性。

zk是有一个leader节点会接受数据，然后同步到其他的节点，一但leader挂了，要重新选举leader，这个过程为了保证C，就牺牲了A，不接用一段时间，但是一个leader选举好了，就可以继续写数据了，保证一致性，即CP

eureka是peer模式，可能还没同步数据过去，结果自己就死了，此时还是可以继续从别的eureka机器上拉取注册表，但是可能看到的就不是最新的数据了，但是保证了可用性A，即AP。
