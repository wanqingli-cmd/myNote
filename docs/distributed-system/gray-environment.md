
#### 准备一个数据库和一个表（也可以用Apollo配置中心、Redis、ZooKeeper，其实都可以），放一个灰度发布启用表

```
id service_id path enable_gray_release

CREATE TABLE `gray_release_config` (
   `id` int(11) NOT NULL AUTO_INCREMENT,
   `service_id` varchar(255) DEFAULT NULL,
   `path` varchar(255) DEFAULT NULL,
   `enable_gray_release` int(11) DEFAULT NULL,
   PRIMARY KEY (`id`)
 ) ENGINE=InnoDB DEFAULT CHARSET=utf8
 


在zuul里面加入下面的filter，可以在zuul的filter里定制ribbon的负载均衡策略

<dependency>
			<groupId>io.jmnarloch</groupId>
			<artifactId>ribbon-discovery-filter-spring-cloud-starter</artifactId>
			<version>2.1.0</version>
		</dependency>

写一个zuul的filter，对每个请求，zuul都会调用这个filter

@Configuration
public class GrayReleaseFilter extends ZuulFilter {

@Autowired
private JdbcTemplate jdbcTemplate;

    @Override
    public int filterOrder() {
        return PRE_DECORATION_FILTER_ORDER - 1;
    }
 
    @Override
    public String filterType() {
        return PRE_TYPE;
    }
 
    @Override
    public boolean shouldFilter() {
    	
    }
 
    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();

Random random = new Random();
int seed = random.getInt() * 100;

        if (seed = 50) {
            // put the serviceId in `RequestContext`
            RibbonFilterContextHolder.getCurrentContext()
                    .add("version", "new");
        }  else {
            RibbonFilterContextHolder.getCurrentContext()
                    .add("version", "old");
        }
        
        return null;
    }
}
 ```

eureka:
instance:
metadata-map:
version: new














https://github.com/














参考资料
灰度发布、蓝绿发布、金丝雀发布各是什么意思，可以看这篇http://www.appadhoc.com/blog/product-release-strategy/。

基于eureka、ribbon实现灰度发布，是这一篇要讲的知识。

我们要发布版本了，在不确定正确性的情况下，我们选择先部分节点升级，然后让一些特定的流量进入到这些新节点，完成测试后再全量发布。



我们知道，在eureka中注册各个服务后，如果一个服务有多个实例，那么默认会走ribbon的软负载均衡来进行分发请求。

我们要完成灰度发布，要做的就是修改ribbon的负载策略（rule），通过一些特定的标识，譬如我们可以选择header里带有foo=1的全部路由到金丝雀服务上，其他的还走原来的老版本。或者可以设置个比重，虽然roll个小于4的正数，将等于1的路由到金丝雀，这样就会有1/4的请求到达金丝雀。诸如此类，我们可以定制各种规则来进行灰度测试。

在SpringCloud体系中，完成这件事，模式比较固定，就是根据eureka的metadata进行自定义元数据，然后修改ribbon的Rule规则。

使用很简单，我们直接上例子，注意我这里只发出来目标服务和zuul的代码，eureka的就不放了。eureka很简单，就是一个eureka server项目，什么也没有。

我们的目标服务是User，在User的application.yml里，由于我要启动2个，所以使用不同的端口

application.yml

server:
  port: 8888
eureka:
  instance:
    prefer-ip-address: true
    metadata-map:
      lancher: 2
  client:
    service-url:
      defaultZone: http://localhost:10000/eureka/
application-dev.yml
server:
  port: 8889
eureka:
  instance:
    metadata-map:
      lancher: 1
就是那个metadata-map元数据，这是一个map，里面就自定义一些key-value键值对。将来匹配时就用这个键值对。
然后分别启动这两个实例，启动后就有两个user注册到了eureka。

zuul配置:

在zuul项目里添加依赖，https://github.com/jmnarloch/ribbon-discovery-filter-spring-cloud-starter

<dependency>
			<groupId>io.jmnarloch</groupId>
			<artifactId>ribbon-discovery-filter-spring-cloud-starter</artifactId>
			<version>2.1.0</version>
		</dependency>
这个就是做ribbon的Rule的。
io.jmnarloch ribbon - discovery - filter - spring - cloud - starter 2.1.0这个就是做ribbon的Rule的。package com.example.zuul_route;

import com.netflix.zuul.ZuulFilter;
import com.netflix.zuul.context.RequestContext;
import io.jmnarloch.spring.cloud.ribbon.support.RibbonFilterContextHolder;
import org.springframework.context.annotation.Configuration;

import javax.servlet.http.HttpServletRequest;

import static org.springframework.cloud.netflix.zuul.filters.support.FilterConstants. * ;

/**

    @author wuweifeng wrote on 2018/1/17. */
@Configuration public class PreFilter extends ZuulFilter {@Override public int filterOrder() {
        return PRE_DECORATION_FILTER_ORDER - 1;
    }

    @Override public String filterType() {
        return PRE_TYPE;
    }

    @Override public boolean shouldFilter() {
        RequestContext ctx = RequestContext.getCurrentContext(); // a filter has already forwarded // a filter has already determined serviceId return !ctx.containsKey(FORWARD_TO_KEY) && !ctx.containsKey(SERVICE_ID_KEY); }
        @Override public Object run() {
            RequestContext ctx = RequestContext.getCurrentContext();
            HttpServletRequest request = ctx.getRequest();
            if (request.getParameter("foo") != null) { // put the serviceId in RequestContext RibbonFilterContextHolder.getCurrentContext() .add("lancher", "1"); } else { RibbonFilterContextHolder.getCurrentContext() .add("lancher", "2"); }
                return null;

            }
        }
这个是zuul的filter，别的无所谓，注意看run方法，RibbonFilterContextHolder.getCurrentContext() .add("lancher", "1");这句话就代表将请求路由到metadata-map里lancher为1的那个服务。
so，很简单，我们就可以在这里定制各种规则了，把符合什么条件的请求，只发送到某个实例。



