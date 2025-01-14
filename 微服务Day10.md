# 微服务Day10

3.线程隔离

当商品服务出现阻塞或故障时，调用商品服务的购物车服务可能因此被拖慢，甚至资源耗尽

所以必须限制购物车服务中查询商品这个业务的可用线程数，实现线程隔离

（1）配置方法

在sentinel控制台中，会出现Feign接口的簇点资源，点击后面的流控按钮，即可配置线程隔离（和限流的操作类似）

注意区分QPS和并发线程数：并发线程数指的是可以利用的资源的数量，如果有5个并发线程，且单个线程的QPS为2，则5个线程的QPS为10

4.Fallback

添加查询商品fallback逻辑，当请求失败时会走fallback逻辑，提高用户体验

（1）将FeignClient作为Sentinel的簇点资源

仅针对远程调用做限流，而不是对整个服务做限流

修改cart-service模块中application文件的配置

feign:sentinel:enabled:true

（2）配置FeignClient的Fallback逻辑

步骤一：自定义类，实现FallbackFactory，编写对某个FeignClient的fallback逻辑

在hm-api模块中定义ItemClientFallbackFactory类，实现ItemClient的fallback逻辑

步骤二：在DefaultFeignConfig中把刚才定义的ItemClientFallbackFactory注册成一个Bean（这里用的是匿名内部类，也可以新写一个配置类）

步骤三：在ItemClient接口中使用ItemClientFallbackFactory

在@FeignClient中添加fallbackFactory属性，并使用ItemClientFallbackFactory.class作为属性值

5.服务熔断

（1）待解决的问题

在配置了fallback逻辑后，可以在请求失败时有效提升用户体验（请求失败走fallback逻辑，比如打印日志、返回空集合等等，而不是直接报异常）

但是现在还存在一个问题，就是fallback逻辑是在发送请求失败后才会采用的，也就是说在明知道发送请求会失败的情况下，仍然需要给问题服务不断的发送请求，这就造成了资源浪费

这时可以采用熔断的方法，当满足一定条件（熔断策略）时，拒绝掉给问题服务发送的所有请求，直接走fallback逻辑，避免发送无用请求

（2）熔断

熔断是解决雪崩问题的重要手段，思路是由断路器统计服务调用的异常比例、慢请求比例，如果超出阈值则会熔断该服务，即拦截访问该服务的一切请求；

而当该服务恢复时，断路器会放行访问该服务的请求

（3）断路器

断路器内部有Closed、Open、Half-Open三个状态

Closed：默认状态，不拦截请求

Open：拦截所有请求，直接走fallback逻辑

Half-Open：Open状态到期后进入Half-Open状态，放行一次请求，如果请求失败则切回为Open状态，请求成功则进入Closed状态

（4）配置方法

点击Sentinel控制台中簇点资源后的熔断按钮，即可配置熔断策略

熔断策略：有慢调用比例、异常比例、异常数等三种策略