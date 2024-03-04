---

---

# Sentinel

## Sentinel解决的是什么问题

解决的是微服务会出现的雪崩问题。

雪崩的危害，当其中一个异常比率或者是慢调用过大，就会导致其他服务受到影响，最终全局崩溃。

![image-20240207221631300](C:\Users\geekyang\AppData\Roaming\Typora\typora-user-images\image-20240207221631300.png)

## 雪崩问题的解决方案

### 1.超时处理

超时处理：给服务响应时间设置一个超时时间，，请求超过一定时间没有响应就返回错误信息，不会无休止等待。

### 2.舱壁模式

![image-20240207222118522](C:\Users\geekyang\AppData\Roaming\Typora\typora-user-images\image-20240207222118522.png)

给每一个业务规定对应的线程数，避免消耗掉Tomcat的所用资源。

![image-20240207222302491](C:\Users\geekyang\AppData\Roaming\Typora\typora-user-images\image-20240207222302491.png)

### 3.断路器（降级熔断）

统计异常比率或者是慢调比率，如果该服务异常比率或者是慢调比率过大，请求访问该服务是直接返回异常，不在接受请求。

![image-20240207225334466](C:\Users\geekyang\AppData\Roaming\Typora\typora-user-images\image-20240207225334466.png)

### 4.限流（控制限流）

**流量控制**：限制业务访问的QPS，避免服务因流量的突增而故障。

### 总结

**超时处理**，**舱壁模式**，**降级熔断**是在部分服务故障时，将故障控制在一定范围，避免雪崩，是一种**补救措施**。

**超时处理**：并不能解决问题，只是拖延了服务器挂掉的时间。

**舱壁模式**：能解决问题，但是会浪费一定资源，增大服务器压力。

**控制流量**可以防是对服务的保护，避免因瞬间高并发流量而导致服务故障，进而避免雪崩，是一种**预防措施**。

## 技术对比

![image-20240207230046919](C:\Users\geekyang\AppData\Roaming\Typora\typora-user-images\image-20240207230046919.png)

## 流量控制

### 1.簇点链路

​		进入微服务之后，首先会进入DispacherServlet，然后会调用Controller，Service，Map等，这个调用链路就叫做**簇点链路**，簇点链路中每一个接口就对应一个**资源**。

​		默认情况下sentinel会监控SpringMVC的每一个端点（Endpoint，也就是controller中的方法），因此SpringMVC的每一个端点（Endpoint）就是调用链路中的一个资源。

​	值得注意的是，接口需要被调用过一次之后才能在UI界面上显示。

![image-20240207230838173](C:\Users\geekyang\AppData\Roaming\Typora\typora-user-images\image-20240207230838173.png)

### 2.流控模式

在添加限流规则时，点击高级选项，可以选择三种**流控模式**：

- 直接：统计当前资源的请求，触发阈值时对当前资源直接限流，也是默认的模式。
- 关联：当前资源与其他资源绑定，当前资源到达阈值后，当前资源限流，关联资源正常执行。
- 链路模式：多条链路访问本资源，当链路到达阈值时，对该链路限流。

#### 关联模式

谁关联，谁限流。

![image-20240207231436873](C:\Users\geekyang\AppData\Roaming\Typora\typora-user-images\image-20240207231436873.png)

**使用场景**：比如用户支付时需要修改订单状态，同时用户要查询订单。查询和修改操作会争抢数据库锁，产生竞争。业务需求是优先支付和更新订单的业务，因此当修改订单业务触发阈值时，需要对查询订单业务限流。

**总结**

满足一下条件可以使用关联模式：

- 两个有竞争关系的资源
- 两个资源有优先级，一个优先级高，一个优先级低

#### 链路模式

**实战案例**

需求：有查询订单和创建订单业务，两者都需要查询商品。针对从查询订单进入到查询商品的请求统计，并设置限流

## 流控效果

### 1.快速失败：

达到阈值后，新的请求会被立即拒绝并抛出FlowException异常。是默认的处理方式。

### 2.warm up（预热模式）：

预热模式，对超出阈值的请求同样是拒绝并抛出异常。但这种模式阈值会动态变化，从一个较小值逐渐增加到最大阈值。

​		warm up也叫预热模式，是应对服务冷启动的一种方案。请求阈值初始值是 threshold / coldFactor，持续指定时长后，逐渐提高到threshold值。而coldFactor的默认值是3

需要关注的值是 **threshold**  和 **预热时间**，两个值。

![image-20240208202602323](C:\Users\geekyang\AppData\Roaming\Typora\typora-user-images\image-20240208202602323.png)

### 3.排队等待：

让所有的请求按照先后次序排队执行，两个请求的间隔不能小于指定时长。

对于**快速失败**，**warm up** 对于超过阈值的请求直接返回**异常**。

而**排队等待**是，将所有请求放到一个排队队列中去，然后根据阈值允许的时间间隔去依次执行，后一个请求必须等前一个请求执行完毕才能执行，请求的预期时间超过了最大时间将会被拒绝。

![image-20240208203131673](C:\Users\geekyang\AppData\Roaming\Typora\typora-user-images\image-20240208203131673.png)

## 热点参数限流

热点参数限流在SpringMVC中不起作用，需要增加@SentinelResource注解，标记资源。

### 1.全局热点参数控制

![image-20240213105631060](C:\Users\geekyang\AppData\Roaming\Typora\typora-user-images\image-20240213105631060.png)

热点参数控制就是对**某一资源接口内参数的全部请求进行参数控制**，只需要给资源名，QPS就可以进行全局热点参数限流。

![image-20240213105649940](C:\Users\geekyang\AppData\Roaming\Typora\typora-user-images\image-20240213105649940.png)

### 2.热点参数限流

对于秒杀业务中，针对特定商品可能会出现多访问的情况，所以就需要对指定商品进行限流。

我们希望这些商品的QPS与其他商品的QPS不一样，所以就需要在参数限流中配置热点参数限流。

![image-20240213110212521](C:\Users\geekyang\AppData\Roaming\Typora\typora-user-images\image-20240213110212521.png)

## 隔离熔断

流量控制是一种**预防**手段，内部还是可能出现故障。

所以我们需要 线程隔离  熔断降级 来确保故障出现不会导致整个服务雪崩。他俩是出现问题的**解决**措施。

### 1.Feign整合Sentinel

​		因为不论是降级熔断，还是舱壁模式都是在客户端发生的，请求在服务端的传递是Feign实现的，需要Feign去实现Sentinel的部分API。

​		1.首先导入依赖

```yaml
feign:
  sentinel:
    enabled: true # 开启feign对sentinel的支持
```

​		2.创建FallbackFactory，可以对远程调用的异常做处理。

​		在这个接口里面可以编写失败降级策略。

```java
package cn.itcast.feign.clients.fallback;

import cn.itcast.feign.clients.UserClient;
import cn.itcast.feign.pojo.User;
import feign.hystrix.FallbackFactory;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@Compant
public class UserClientFallbackFactory implements FallbackFactory<UserClient> {
    @Override
    public UserClient create(Throwable throwable) {
        return new UserClient() {
            @Override
            public User findById(Long id) {
                log.error("查询用户异常", throwable);
                return new User();
            }
        };
    }
}

```

​		3.将FallbackFactory配置到FeignClient

```java
import cn.itcast.feign.clients.fallback.UserClientFallbackFactory;
import cn.itcast.feign.pojo.User;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;

@FeignClient(value = "userservice", fallbackFactory = UserClientFallbackFactory.class)
public interface UserClient {

    @GetMapping("/user/{id}")
    User findById(@PathVariable("id") Long id);
}
```

### 线程隔离（舱壁模式）

线程隔离有两种模式：

- ​	线程池隔离
- ​    信号量计数（Sentinel采用方式）

线程池隔离：

**优势**  ：可以异步调用，可以主动超时

**缺点**  ： 线程开销大

**场景**  ： 低扇出 （扇出，直接调用下级模块的数量。数量过多就以为这难以控制和协调下级模块）

信号量计数：

**优势**  ：轻量级，没有额外开销。

**缺点**  ：不支持主动超时，不支持异步调用

**场景**  ：高扇出

![image-20240213122116581](C:\Users\geekyang\AppData\Roaming\Typora\typora-user-images\image-20240213122116581.png)

### 熔断降级

​		熔断降级**是解决**雪崩问题**的重要手段。他里面状态机统计慢调用，异常比例超出阈值的就会**熔断**该服务。即拦截访问该服务的一切请求，而当服务恢复时就会恢复。

![image-20240213122839540](C:\Users\geekyang\AppData\Roaming\Typora\typora-user-images\image-20240213122839540.png)

状态机有三种状态：

- Closed：关闭态，请求过来直接通过。但是会统计慢调用比例，异常请求比例。超过阈值转为open态
- Open：开启态，请求过来直接失败，Open状态5秒后会转为Half-Open状态
- Half-Open：半开启状态，会放行一次请求。

请求成功：状态恢复为Closed态，请求放行

请求失败：回到Open，继续等待放行



断路器熔断策略有三种：慢调用、异常比例、异常数

#### 慢调用

慢调用：业务响应时间（RT 		ResponseTime）大于指定时间。在指定时间内，如果请求数量超过设定的最小数量，慢调用比例大于设定的阈值，则触发熔断。

![image-20240213182447402](C:\Users\geekyang\AppData\Roaming\Typora\typora-user-images\image-20240213182447402.png)

#### 异常比率，异常数

异常比率：就是在规定时间内调用次数超过最小请求数，并且异常比例 或者异常数超过阈值则触发熔断。

![image-20240213182715450](C:\Users\geekyang\AppData\Roaming\Typora\typora-user-images\image-20240213182715450.png)