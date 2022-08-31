---
title: Redis实战经验积累
date: 2022-04-03 22:08:32
categories: Redis
tags: 
   - Redis
   - Experience
urlname: redis-practical-experience
---

## 回顾 Redis 基础篇的一些命令

### redis 主要有以下几种数据类型

- string
- hash
- list
- set
- sorted set

<!--more-->

### string

这是最简单的类型，就是普通的 set 和 get，做简单的 KV 缓存。

```shell
set college szu
```

### hash

这个是类似 map 的一种结构，这个一般就是可以将结构化的数据，比如一个对象（前提是**这个对象没嵌套其他的对象**）给缓存在 redis 里，然后每次读写缓存的时候，可以就操作 hash 里的**某个字段**。

```shell
hset person name bingo
hset person age 20
hset person id 1
hget person name
person = {
    "name": "bingo",
    "age": 20,
    "id": 1
}
```

### list

list 是有序列表，这个可以玩儿出很多花样。

比如可以通过 list 存储一些列表型的数据结构，类似粉丝列表、文章的评论列表之类的东西。

比如可以通过 lrange 命令，读取某个闭区间内的元素，可以基于 list 实现分页查询，这个是很棒的一个功能，基于 redis 实现简单的高性能分页，可以做类似微博那种下拉不断分页的东西，性能高，就一页一页走。

```shell
# 0开始位置，-1结束位置，结束位置为-1时，表示列表的最后一个位置，即查看所有。
lrange mylist 0 -1
```

比如可以搞个简单的消息队列，从 list 头怼进去，从 list 尾巴那里弄出来。

```shell
lpush mylist 1
lpush mylist 2
lpush mylist 3 4 5

# 1
rpop mylist
```

比如微博，某个大 V 的粉丝，就可以使用 list 的格式放在 redis 中去缓存。

key = 某大 V

value = [zhangsan, lisi, wangwu, …]

### set

set 是无序集合，自动去重。

直接基于 set 将系统里需要去重的数据扔进去，自动就给去重了，如果你需要对一些数据进行快速的全局去重，你当然也可以基于 jvm 内存里的 HashSet 进行去重，但是如果你的某个系统部署在多台机器上呢？得基于 redis 进行全局的 set 去重。

可以基于 set 玩儿交集、并集、差集的操作，比如交集吧，可以把两个人的粉丝列表整一个交集，看看俩人的共同好友是谁？对吧。

把两个大 V 的粉丝都放在两个 set 中，对两个 set 做交集。

```shell
#-------操作一个set-------
# 添加元素
sadd mySet 1

# 查看全部元素
smembers mySet

# 判断是否包含某个值
sismember mySet 3

# 删除某个/些元素
srem mySet 1
srem mySet 2 4

# 查看元素个数
scard mySet

# 随机删除一个元素
spop mySet

#-------操作多个set-------
# 将一个set的元素移动到另外一个set
smove yourSet mySet 2

# 求两set的交集
sinter yourSet mySet

# 求两set的并集
sunion yourSet mySet

# 求在yourSet中而不在mySet中的元素
sdiff yourSet mySet
```

### sorted set

sorted set 是排序的 set，去重并可以排序，写进去的时候给一个分数，自动根据分数排序。

```shell
# 排行榜：将每个用户以及其对应的什么分数写入进去，
# 命令：zadd key score value 注：score是sortedset中特有的概念，sortedset因此便可排序，在score位置放入数值，便会以此数值排序。
# 比如：zadd board score username
zadd board 85 zhangsan
zadd board 72 lisi
zadd board 96 wangwu
zadd board 63 zhaoliu

# 获取排名前三的用户（默认是升序，所以需要 rev 改为降序）
# zrange是升序排的，zrevrange是降序排的
zrevrange board 0 2
# 查询结果：
#1) "wangwu"
#2) "zhangsan"
#3) "lisi"
#4) "zhaoliu"

# 查询时附带“score”
zrevrange board 0 3 withscores
# 查询结果
#1) "wangwu"
#2) "96"
#3) "zhangsan"
#4) "85"
#5) "lisi"
#6) "72"
#7) "zhaoliu"
#8) "63"

# 上面 “zrevrange key 起始位置 结束位置” 是按照排位范围查询
# =======================================================

# 按照score范围+offset机制查询
# 命令：ZREVRANGEBYSCORE key MaxScore MinScore LIMIT offset count
ZREVRANGEBYSCORE board 100 0 withscores limit 0 2
# 解析：score在0-100（包含0和100）范围内，查询score排名前二的，第一页的数据
# 按照key为board查询，查询的score范围是0-100，查询结果附加score值，limit分页查，0表示从第一个开始查（左闭右开），count表示每页条数
# 查询结果；
#1) "wangwu"
#2) "96"
#3) "zhangsan"
#4) "85"
ZREVRANGEBYSCORE board 85 0 withscores limit 1 2
# 解析：score在0-85（包含0和85）范围内，查询score排名前二的，第二页的数据
# 查询结果
#1) "lisi"
#2) "72"
#3) "zhaoliu"
#4) "63"
# 上面两条查询语句中，MaxScore和offset变化了。可以看到，第二次查询时，将第一次查询的最小分数作为第二次查询的最大分数；并且offset变成1，原因是：如果还是0的话，那么由于左闭右开，第二次查询时会将85分的人再查一遍，因此从1开始，offset变成1.

zadd board 96 csy
zrevrange board 0 4 withscores
#1)  "wangwu"
#2)  "96"
#3)  "csy"
#4)  "96"
#5)  "zhangsan"
#6)  "85"
#7)  "lisi"
#8)  "72"
#9)  "zhaoliu"
#10) "63"
ZREVRANGEBYSCORE board 100 0 withscores limit 0 2
#1) "wangwu"
#2) "96"
#3) "csy"
#4) "96"
ZREVRANGEBYSCORE board 96 0 withscores limit 1 2 #错
#1) "csy"
#2) "96"
#3) "zhangsan"
#4) "85"
# 显然，重复查询了csy，这不是我们想要的，为何呢？问题出在score有相等的情况，offset需要变化。
#ZREVRANGEBYSCORE key max min          limit offset count
ZREVRANGEBYSCORE board 96 0 withscores limit 2 2 #对
#1) "zhangsan"
#2) "85"
#3) "lisi"
#4) "72"
# 显然，这才是我们想要的，这两个标注了对错的查询语句，很鲜明的体现了问题，在score有相等的情况，offset需要变化。
# 如何变化？offset的值应为：上一次查询的结果中，score最小值出现的次数。
# 比如上上面的例子：96、85，score的最小值为85，出现了一次，那么下次查询的时候offset为1；
# 比如上面的例子：96、96，score的最小值为96，出现了两次，那么下次查询的时候offset为2，这样才不会重复查询。

# 对于使用redis做分页查询，其实zrange和zrangebyscore都是可以的，但是要区分场景，zrange的排序是按照排位角标的传统排序，传统的分页在feed流是不适用的，因为Feed流中的数据会不断更新，所以数据的排位角标也在变化，因此不能采用传统的分页模式。因此Feed流需要使用的是滚动分页。我们需要记录每次操作的最后一条，然后从这个位置开始去读取数据。由此可见，zrangebyscore可以实现滚动分页。

# 获取某用户的排名
zrank board zhaoliu
```

## 短信登录

### 基于 Session 登录业务流程

1、发送短信验证码

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis/1.png)

2、短信验证码登录或注册

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis/2.png)

3、校验登录状态

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis/3.png)

基于 Session 实现登录的原理，其实就是 Session 的原理，Session 的原理为 Cookie。每一个 Session 都会有一个唯一的 sessionid，在你访问 tomcat 的时候，sessionid 就已经自动的被写到你的 cookie 中了，你以后的请求会携带 Cookie，那么 Cookie 中就会有这个 JSESSIONID，就能自动的通过这个 JSESSIONID 找到对应的 Session，进而找到 session 中的数据。程序中 session 都有一个默认的过期时间，其中 tomcat 中的默认时间为 30 分钟。

注：这里要注意的是 Session 的过期时间，Session 默认 30min 内一直无操作的话就过期，如果有操作就会刷新 Session 的过期时间，重新计时 30min。如果 Session 过期的话，那么保存在 Session 中的登录状态自然也就过期了。

### Session 共享问题

Session 在单体项目下尚且可以使用来记录登录状态，但是当我们将服务集群的方式部署的时候，那么多个服务也就意味着多台 tomcat 服务器，多台 tomcat 之间并不共享 session 的存储空间，因此当请求切换到不同的 tomcat 服务器时就会导致数据丢失的问题，也就是登录状态的丢失，比如说用户第一次的请求到达了 tomcat1 做了登录操作，然后后续的操作请求到达了 tomcat2，这时 tomcat2 中的 session 中并没有这个用户的登录状态，那就意味着这个用户需要重新登录，这显然是不合理的。

### 解决方案

#### 使用 Redis + 随机 Token

session 的替代方案应该满足 3 点：数据共享、内存存储、key-value 结构。

##### 原理解析

之前基于 session 来做登录校验，是因为 tomcat 会自动的将你的 sessionid 写到浏览器的 cookie 中，以后每次请求带着 cookie，则带着 JSESSIONID 来了，进而找到 session 进而找到 session 中保存的登录状态，也就是用户信息，因此这里的 sessionid 相当于一个登录凭证，这个 sessionid 是由 tomcat 维护的。但是我们现在不用这套了，我们用 token，这个 token 是我们在服务端通过代码随机生成的，但是浏览器不会自动帮我们将 token 写入到浏览器的 cookie 中，因此需要我们手动将生成的 token 写回前端浏览器，前端人员将 token 保存到请求头，让以后每次请求都携带这个 token，服务端我们通过 token 到 redis 中取用户信息，从而校验登录状态。并且 token 在 redis 中设置过期时间，模拟 session 的过期特性。注，当用户在 token 有效期内进行操作了，要重新刷新 token 的有效时间，从头开始重新计时。

##### 业务流程

1、发送短信验证码

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis/4.png)

2、短信验证码登录或注册

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis/5.png)

3、校验登录状态

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis/6.png)

拦截器的应用。注，单体 SpringBoot 项目这样使用，如果是微服务项目，可以在网关那块拦截。

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis/拦截器的优化.png)

#### 使用 Spring Session + Redis

前面说了，session 是啥？浏览器有个 cookie，在一段时间内这个 cookie 都存在，然后每次发请求过来都带上一个特殊的 `jsessionid cookie`，就根据这个东西，在服务端可以维护一个对应的 session 域，里面可以放点数据。

虽然 session 自动过期时间是 30min，但是一般的话只要你没关掉浏览器，cookie 还在，那么对应的那个 session 就在，因为 session 会自动续期，但是如果 cookie 没了，session 也就没了。常见于什么购物车之类的东西，还有登录状态保存之类的。

##### 原理图

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/fenbushi-session/3.png)

当所有 Tomcat 需要往 Session 中写数据时，都往 Redis 中写，当所有 Tomcat 需要读数据时，都从 Redis 中读。这样，不同的服务就可以使用相同的 Session 数据了。可以使用 Spring Session 来实现这一功能，Spring Session 就是使用 Spring 中的代理过滤器，将所有的 Session 操作拦截下来，自动的将数据同步到 Redis 中，或者自动的从 Redis 中读取数据。

对于开发者来说，所有关于 Session 同步的操作都是透明的，开发者使用 Spring Session，一旦配置完成后，具体的用法就像使用一个普通的 Session 一样。下面列一个简易的 demo。

##### 创建工程

首先，创建一个 Spring Boot 工程，引入 Web、Spring Session 以及 Redis:

![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis/9.png)

创建成功之后，pom.xml 文件如下：

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.session</groupId>
        <artifactId>spring-session-data-redis</artifactId>
    </dependency>
</dependencies>
```

**注意：**

这里我使用的 Spring Boot 版本是 2.1.4 ，如果使用 Spring Boot2.1.5 的话，除了上面这些依赖之外，需要额外添加 Spring Security 依赖（其他操作不受影响，仅仅只是多了一个依赖，当然也多了 Spring Security 的一些默认认证流程）。

##### 配置 Redis

```properties
spring.redis.host=192.168.66.128
spring.redis.port=6379
spring.redis.password=123
spring.redis.database=0

spring.session.store-type=REDIS
server.servlet.session.timeout=30m
```

并且在微服务的启动类上添加 `@EnableRedisHttpSession` 注解，代表整合 redis 作为 session 存储。store-type 的类型除了 REDIS，还有 NONE、JDBC、MONGODB 可供选择，根据公司存储架构和需求选择合适的存储渠道，但是大部分应该使用的是 redis。

##### 使用

配置完成后，就可以使用 Spring Session 了，其实就是使用普通的 HttpSession ，其他的 Session 同步到 Redis 等操作，框架已经自动帮你完成了：

```java
@RestController
public class HelloController {
    @Value("${server.port}")
    Integer port;
    @GetMapping("/set")
    public String set(HttpSession session) {
        session.setAttribute("user", "javaboy");
        return String.valueOf(port);
    }
    @GetMapping("/get")
    public String get(HttpSession session) {
        return session.getAttribute("user") + ":" + port;
    }
}
```

考虑到一会 Spring Boot 将以集群的方式启动 ，为了获取每一个请求到底是哪一个 Spring Boot 提供的服务，需要在每次请求时返回当前服务的端口号，因此这里我注入了 server.port 。

接下来 ，项目打包：

![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis/10.png)

打包之后，启动项目的两个实例：

```shell
java -jar sessionshare-0.0.1-SNAPSHOT.jar --server.port=8080
java -jar sessionshare-0.0.1-SNAPSHOT.jar --server.port=8081
```

然后先访问 `localhost:8080/set` 向 `8080` 这个服务的 `Session` 中保存一个变量，访问完成后，数据就已经自动同步到 `Redis` 中了：

![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis/11.png)

然后，再调用 `localhost:8081/get` 接口，就可以获取到 `8080` 服务的 `session` 中的数据：

![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis/12.png)

此时关于 session 共享的配置就已经全部完成了，session 共享的效果我们已经看到了，但是每次访问都是我自己手动切换服务实例，因此，接下来我们来引入 Nginx ，实现服务实例自动切换。

##### 引入 Nginx

很简单，进入 Nginx 的安装目录的 conf 目录下（默认是在 `/usr/local/nginx/conf`），编辑 nginx.conf 文件：

![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis/13.png)

在这段配置中：

1. upstream 表示配置上游服务器
2. javaboy.org 表示服务器集群的名字，这个可以随意取名字
3. upstream 里边配置的是一个个的单独服务
4. weight 表示服务的权重，意味者将有多少比例的请求从 Nginx 上转发到该服务上
5. location 中的 proxy_pass 表示请求转发的地址，`/` 表示拦截到所有的请求，转发转发到刚刚配置好的服务集群中
6. proxy_redirect 表示设置当发生重定向请求时，nginx 自动修正响应头数据（默认是 Tomcat 返回重定向，此时重定向的地址是 Tomcat 的地址，我们需要将之修改使之成为 Nginx 的地址）。

配置完成后，将本地的 Spring Boot 打包好的 jar 上传到 Linux ，然后在 Linux 上分别启动两个 Spring Boot 实例：

```shell
nohup java -jar sessionshare-0.0.1-SNAPSHOT.jar --server.port=8080 &
nohup java -jar sessionshare-0.0.1-SNAPSHOT.jar --server.port=8081 &
```

其中

- nohup 表示当终端关闭时，Spring Boot 不要停止运行
- & 表示让 Spring Boot 在后台启动

配置完成后，重启 Nginx：

```shell
/usr/local/nginx/sbin/nginx -s reload
```

Nginx 启动成功后，我们首先手动清除 Redis 上的数据，然后访问 `192.168.66.128/set` 表示向 `session` 中保存数据，这个请求首先会到达 `Nginx` 上，再由 `Nginx` 转发给某一个 `Spring Boot` 实例：

![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis/14.png)

如上，表示端口为 `8081` 的 `Spring Boot` 处理了这个 `/set` 请求，再访问 `/get` 请求：

![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis/15.png)

可以看到，`/get` 请求是被端口为 8080 的服务所处理的。

> 本案例我已经上传到码云： https://gitee.com/erdochan/sessionshare。注意配置文件的修改！

**关于 Spring Session 其它的一些问题：**

我们第一次使用 session 时，浏览器会自动帮我们保存一个 cookie，也就是 JSESSIONID，事实上，它就是个 cookie。之后浏览器访问哪个网站就会带上这个网站的 cookie。

域名之间：比如父域 mall.com，它的子域 auth.mall.com，比如我们现在就是在 auth 这个微服务上做鉴权，使用到 session 保存用户的登录状态，那么这个时候 session 的作用域就是当前域，也就是 auth.mall.com，那么如果鉴权服务这里鉴权成功，登录成功到首页的话，首页的域名就是 mall.com，那么则会出现子域 session 的共享问题，针对这个问题，解决方案就是配置放大 session 的作用域。

另外，如果存入 Redis 的 session 是个对象的话，默认是使用 JDK 的方式序列化后存入的，这样在 Redis 中的数据的可读性不高，因此我们可以配置 Spring Session 将 session 存入 redis 的时候使用 JSON 的序列化方式。

配置类如下：

```java
@Configuration
public class MallSessionConfig {
    @Bean
    public CookieSerializer cookieSerializer() {
        DefaultCookieSerializer cookieSerializer = new DefaultCookieSerializer();
        //放大作用域
        cookieSerializer.setDomainName("mall.com");
        cookieSerializer.setCookieName("MALL-SESSION");
        cookieSerializer.setCookieMaxAge(60*60*24*7);
        return cookieSerializer;
    }

    // 使用 JSON 的序列化方式
    @Bean
    public RedisSerializer<Object> springSessionDefaultRedisSerializer() {
        return new GenericJackson2JsonRedisSerializer();
    }
}
```

最后，一级域名也就是顶级域名不同的话，是不能共用 cookie 的，就比如说，一个公司有多款产品，但是顶级域名并不相同，例如：sina.cn.com 和 weibo.com，这种虽然都是新浪公司的但是父域名不同，这样就不能使用 spring session 解决共享登录状态的问题。那么在这种情况下，我想要保证共享登录状态的话，就需要使用单点登录了，一处登录，处处可用，[单点登录案例](https://gitee.com/xuxueli0323/xxl-sso?_from=gitee_search)。单点登录核心：三个系统即使域名不一样，想办法给三个系统同步同一个用户的票据；

- 中央认证服务器，ssoserver.com；
- 其他系统，想要登录去 ssoserver.com 登录，登录成功跳转回来；
- 只要有一个系统已经登录，其它系统都不需要登录；
- 所有系统统一一个 sso-sessionid；所有系统可能域名都不相同。

总的来说，使用 Spring Session 解决分布式 session 问题，实现共享登录状态，是指在父子域名的情况下共享，也就是不同的微服务之间共享和相同的微服务（可能是微服务集群部署）之间共享，而使用单点登录实现共享登录状态，是在父域名不同的情况下共享。

最后介绍一下，有一种基于 JWT 的无状态登录方案，就是不使用 session，使用 token 的一种方式，因为不涉及 cookie 和 session，因此称为无状态。JWT，全称是Json Web Token， 是JSON风格轻量级的授权和身份认证规范，可实现无状态、分布式的Web应用授权；[官网](https://jwt.io)。之前写过一篇博客，就是大致介绍了一下 JWT，以及在项目中的简单使用，博客地址：https://chanservy.github.io/2021/04/09/auth-center.html。

##### 总结

我们写了一些代码，也做了一些配置，但是基本上都和 Spring Session 无关，配置是配置 Redis，代码就是普通的 HttpSession，和 Spring Session 没有任何关系！和 Spring Session 相关的，就是一开始引入了 Spring Session 的依赖，添加了 session 存储类型的配置以及 `@EnableRedisHttpSession` 注解。其它的就是正常使用 HttpSession ，只不过在没配置 Spring Session 之前，我们的 session 是基于 web 服务器存储的，配置了之后 session 就是基于 Redis 存储了，对于 session 的操作也是一样的。

> Spring Session 核心原理：
>
> 我们加入了 @EnableRedisHttpSession 注解，@EnableRedisHttpSession 导入了 RedisHttpSessionConfiguration 配置，那么这个配置做了什么？
>
> 1. 给容器中添加了一个组件：SessionRepository ---> RedisOperationsSessionRepository ---> redis 操作 session。session 的增删改查封装类。
> 2. SessionRepositoryFilter ---> Filter：session 存储过滤器；每个请求过来都必须经过 filter。
>    1. 创建的时候，就自动从容器中获取到了 SessionRepository
>    2. 原始的 request，response 都被包装。SessionRepositoryRequestWrapper，SessionRepositoryResponseWrapper
>    3. 以后获取 session 时，依然是 request.getSession();  注：这里是 SessionRepositoryRequestWrapper 的 request，也就是被包装后的 request，并非原生的 HttpServletRequest。
>    4. wrappedRequest.getSession(); ---> SessionRepository 中获取到的。
>
> 这便是装饰者模式。

#### 使用 ip_hash 指令

在负载均衡系统中，如果客户端已经在某台服务器中登陆，如果我们在访问系统，Nginx 会给客户端重新分配一台服务器，这台服务器很有可能不是原先的那台服务器，这显然是不妥的，因为这样就意味着客户端又要重新登陆一次系统。所以需要通过 ip_hash 指令来解决这个问题。

##### ip_hash 指令的原理

Nginx 通过哈希算法（键值对）给每个客户端指定一个对应的服务器，当一个用户已经在一台服务器上登陆，当它再次访问 Nginx 服务器时，Nginx 会从哈希集合中拿到用户上次登陆的那个服务器，然后跳转到相应的服务器。从而解决 session 丢失的问题，ip_hash 会自动运算对应的 ip 访问的地址，可以提高命中率，Nginx 可以自动发现上游服务是不是挂了，如果挂了 Nginx 会自动分配其他的，当挂掉的服务重新又启动起来了，又自动的回去找原来的服务了。  

##### 使用 ip_hash 解决 session 共享

保证一个 ip 地址永远的访问一台 web 服务器，就不存在 session 共享的问题了。在 Nginx 的配置文件中的 upstream 中添加 ip_hash。

## 商户查询缓存

用缓存，主要有两个用途：**高性能**、**高并发**。

### 缓存更新策略

其实这主要是说`数据库和缓存的数据双写不一致的问题`，该怎样解决。

|          | 内存淘汰                                                     | 超时剔除                                                     | 主动更新                                     |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ | -------------------------------------------- |
| 说明     | 不用自己维护，利用Redis的内存淘汰机制，当内存不足时自动淘汰部分数据。下次查询时更新缓存。 | 给缓存数据添加TTL时间，到期后自动删除缓存。下次查询时更新缓存。 | 编写业务逻辑，在修改数据库的同时，更新缓存。 |
| 一致性   | 差                                                           | 一般                                                         | 好                                           |
| 维护成本 | 无                                                           | 低                                                           | 高                                           |

业务场景：

- 低一致性需求：使用`内存淘汰机制`。例如店铺类型的查询缓存
- 高一致性需求：主动更新，并以超时剔除作为兜底方案。例如店铺详情查询的缓存

#### 关于内存淘汰机制

- 往 redis 写入的数据怎么没了？

啥叫缓存？用内存当缓存。内存是无限的吗，内存是很宝贵而且是有限的，磁盘是廉价而且是大量的。可能一台机器就几十个 G 的内存，但是可以有几个 T 的硬盘空间。redis 主要是基于内存来进行高性能、高并发的读写操作的。

那既然内存是有限的，比如 redis 就只能用 10G，你要是往里面写了 20G 的数据，会咋办？当然会干掉 10G 的数据，然后就保留 10G 的数据了。那干掉哪些数据？保留哪些数据？当然是干掉不常用的数据，保留常用的数据了。触发了内存淘汰机制。

- 数据明明过期了，怎么还占用着内存？

这是由 redis 的过期策略来决定。定期删除时随机抽取的设置了过期时间的那些 key 可能有一部分暂时并没有过期，当然过期的会被删除掉，但是并不是所有的过期的 key 都会被抽取到，没抽取到的过期的 key 如果没有被查询，那就不能触发惰性删除，因此可能会出现数据明明过期了但是内存依然被占用着这种情况。

##### redis 过期策略

redis 过期策略是：**定期删除+惰性删除**。

所谓**定期删除**，指的是 redis 默认是每隔 100ms 就随机`抽取一些设置了过期时间的 key，检查其是否过期，如果过期就删除`。

假设 redis 里放了 10w 个 key，都设置了过期时间，你每隔几百毫秒，就检查 10w 个 key，那 redis 基本上就死了，cpu 负载会很高的，消耗在你的检查过期 key 上了。注意，这里可不是每隔 100ms 就遍历所有的设置过期时间的 key，那样就是一场性能上的**灾难**。实际上 redis 是每隔 100ms **随机抽取**一些 key 来检查和删除的。

但是问题是，定期删除可能会导致很多过期 key 到了时间并没有被删除掉，那咋整呢？所以就是惰性删除了。这就是说，在你获取某个 key 的时候，redis 会检查一下 ，这个 key 如果设置了过期时间那么是否过期了？如果过期了此时就会删除，不会给你返回任何东西。

> 获取 key 的时候，如果此时 key 已经过期，就删除，不会返回任何东西。

但是实际上这还是有问题的，如果定期删除漏掉了很多过期 key，然后你也没及时去查，也就没走惰性删除，此时会怎么样？如果大量过期 key 堆积在内存里，导致 redis 内存块耗尽了，咋整？

答案是：**走内存淘汰机制**。

##### 内存淘汰机制

redis 内存淘汰机制有以下几个：

- noeviction: 当内存不足以容纳新写入数据时，新写入操作会报错，这个一般没人用吧，实在是太恶心了。
- **allkeys-lru**：当内存不足以容纳新写入数据时，在**键空间**中，移除最近最少使用的 key（这个是**最常用**的）。
- allkeys-random：当内存不足以容纳新写入数据时，在**键空间**中，随机移除某个 key，这个一般没人用吧，为啥要随机，肯定是把最近最少使用的 key 给干掉啊。
- volatile-lru：当内存不足以容纳新写入数据时，在**设置了过期时间的键空间**中，移除最近最少使用的 key（这个一般不太合适）。
- volatile-random：当内存不足以容纳新写入数据时，在**设置了过期时间的键空间**中，**随机移除**某个 key。
- volatile-ttl：当内存不足以容纳新写入数据时，在**设置了过期时间的键空间**中，有**更早过期时间**的 key 优先移除。

##### 手写一个 LRU 算法

不求自己纯手工从底层开始打造出自己的 LRU，但是起码要知道如何利用已有的 JDK 数据结构实现一个 Java 版的 LRU。

```java
class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int CACHE_SIZE;

    /**
     * 传递进来的参数是最多能缓存多少数据
     *
     * @param cacheSize 缓存大小
     */
    public LRUCache(int cacheSize) {
        // 这块就是设置一个 hashmap 的初始大小，最后的 true 表示让 linkedHashMap 按照访问顺序来进行排序，最近访问的放在头部，最老访问的放在尾部。
        super((int) Math.ceil(cacheSize / 0.75) + 1, 0.75f, true);
        CACHE_SIZE = cacheSize;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        // 当 map中的数据量大于指定的缓存个数的时候，就自动删除最老的数据。
        return size() > CACHE_SIZE;
    }
}
```

#### 主动更新策略

##### Cache Aside Pattern

由缓存的调用者，在更新数据库的同时更新缓存。

**操作缓存和数据库时有三个问题需要考虑：**

1. 删除缓存还是更新缓存？
   - 更新缓存：每次更新数据库都更新缓存，无效写操作较多
   - 删除缓存：更新数据库时让缓存失效，查询时再更新缓存
   - 举个栗子：比如一个缓存涉及到的表的字段，在 1 分钟内就修改了 20 次，或者是 100 次，那么缓存更新 20 次、100 次；但是这个缓存在 1 分钟内只被读取了 1 次，对于缓存的无效写操作较多。实际上，如果你只是删除缓存的话，那么在 1 分钟内，这个缓存不过就重新计算一次而已，开销大幅度降低。用到缓存才去算缓存。其实`删除缓存，而不是更新缓存`，就是一个 lazy 加载的思想，是让它到需要被使用的时候再重新加载。
2. 如何保证缓存与数据库的操作的同时成功或失败？
   - 单体系统，将缓存与数据库操作放在一个事务
   - 分布式系统，利用 TCC 等分布式事务方案
3. 先操作缓存还是先操作数据库？
   - 先删除缓存，再更新数据库
     - 分析：先删除缓存，再更新数据库，如果更新数据库失败了，那么再查的时候缓存中没有，进而到数据库中取到旧数据放入缓存，可能有脏数据的情况，不会造成缓存数据库中的数据不一致。但是在`对一个数据并发的进行读写`的时候，可能会造成缓存数据库中的数据不一致的情况。众所周知，数据库的操作相比缓存的操作效率要低很多，耗时更长；另外如果数据库是主从架构部署，主写从读，数据库的数据修改之后，数据从主节点同步到从节点也需要耗时。如果写请求到来，先删除了缓存，更新数据库的操作还未完成时，读请求到来，这时缓存中无数据，查询数据库从节点，然后将数据写回缓存，在这之后数据库才更新并同步成功，这个时候缓存中和数据库中的数据不一致。
     - 解决：更新数据的时候，根据**数据的唯一标识**，比如商品的 id ，对这个 id 进行 hash 取值，然后得到的值再对内存队列的数量进行取模，这样针对每个商品操作都可以路由到某一个内存队列中，也就是 hash 算法。将这个 id 的商品更新操作路由之后，发送到一个 jvm 内部队列中。读取这个 id 的商品的数据的时候，如果发现数据不在缓存中，根据唯一标识路由之后，也发送同一个 jvm 内部队列中。一个队列对应一个工作线程，每个工作线程**串行**拿到对应的操作，然后一条一条的执行。这样的话，一个数据变更的操作，先删除缓存，然后再去更新数据库，但是还没完成更新。此时如果一个读请求过来，如果没有读到缓存，那么可以先将查询数据库并更新缓存的操作发送到队列中，此时会在队列中积压，然后同步等待数据库数据更新完成之后再进行查询数据库。
     - 优化点：一个队列中，其实**多个查询请求（包含查库+更新缓存）串在一起是没意义的**，因此可以做过滤，如果发现队列中已经有一个查询的请求了，那么就不用再放个查询请求操作进去了，直接等待前面的更新操作请求和查询请求完成即可。如果请求还在等待时间范围内，不断轮询发现缓存中可以取到值了，那么就直接返回；如果请求等待的时间超过一定时长，那么这一次直接从数据库中读取当前的旧值。
     - 高并发场景下该解决方案要考虑的问题：
       1. 读请求长时阻塞
       2. 读请求并发量过高
       3. 服务集群式部署的请求路由
       4. 热点商品的路由问题，导致请求的倾斜
   - 先更新数据库，再删除缓存
     - 分析：这种情况，如果数据库更新成功，但是缓存更新失败，那么缓存中就是旧数据，而数据库中是新数据，这样造成了数据库和缓存的数据不一致。
     - 解决：首先一定要**确保操作数据库与操作缓存两个动作的原子性，要么一起成功，要么一起失败**！单体系统，将缓存与数据库操作放在一个事务中；分布式系统，利用 TCC 等分布式事务方案。其次缓存中写入数据的时候设置一个过期淘汰时间作为兜底方案，即使事务操作失误了，数据库更新了，缓存删除失败了，那缓存中的数据到期之后也会过期淘汰掉，下次在查询的时候先查数据库然后将数据写入缓存。

综上，我觉得选择“先更新数据库，再删除缓存”方案更好。

单体项目中：

```java
@Transactional
public Result update(Shop shop) {
    Long id = shop.getId();
    if (id == null) {
        return Result.fail("店铺id不能为空");
    }
    // 1.更新数据库
    updateById(shop);
    // 2.删除缓存
    stringRedisTemplate.delete(CACHE_SHOP_KEY + id);
    return Result.ok();
}
```

##### 总结

缓存更新策略的最佳实践方案：

1. 低一致性需求：使用 Redis 自带的内存淘汰机制
2. 高一致性需求：主动更新，并以超时剔除作为兜底方案
   - 读操作：缓存命中则直接返回；缓存未命中则查询数据库，并写入缓存，设定超时时间，最后返回
   - 写操作：先更新数据库，然后再删除缓存，一定要确保操作数据库与操作缓存两个动作的原子性，要么一起成功，要么一起失败！
     - 单体系统，将缓存与数据库操作放在一个事务中
     - 分布式系统，利用 TCC 等分布式事务方案

### 缓存雪崩

- 对于系统 A，假设每天高峰期的时候每秒 5000 个请求，本来缓存在高峰期可以扛住每秒 4000 个请求，但是缓存机器意外发生了全盘宕机。`缓存挂了`，缓存查不到任何数据了，那么此时 1 秒 5000 个请求在缓存中都查不到数据，自然全部落到数据库，数据库必然扛不住，它会报一下警，然后就挂了。
- 或者是指`在同一时段大量的缓存 key 同时失效`，导致大量请求到达数据库，带来巨大压力。

其实都差不多，这就是缓存雪崩。

[![redis-caching-avalanche](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis-problems/7.png)](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis-problems/7.png)

缓存雪崩的事前事中事后的解决方案如下：

- 事前：redis 高可用，主从+哨兵，redis cluster，避免全盘崩溃。
- 事中：系统 A 中开启本地 ehcache 缓存 + hystrix 限流&降级，避免 MySQL 被打死。给不同的 Key 的 TTL 添加随机值。
- 事后：redis 持久化，一旦重启，自动从磁盘上加载数据，快速恢复缓存数据。

[![redis-caching-avalanche-solution](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis-problems/8.png)](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis-problems/8.png)

用户发送一个请求，系统 A 收到请求后，先查本地 ehcache 缓存，如果没查到再查 redis。如果 ehcache 和 redis 都没有，限流之后再查数据库，将数据库中的结果，写入 ehcache 和 redis 中。

限流组件，可以设置每秒的请求，有多少能通过组件，剩余的未通过的请求，怎么办？**走降级**！可以返回一些默认的值，或者友情提示，或者空白的值。

好处：

- 数据库绝对不会死，限流组件确保了每秒只有多少个请求能通过。
- 只要数据库不死，就是说，对用户来说，2/5 的请求都是可以被处理的。
- 只要有 2/5 的请求可以被处理，就意味着你的系统没死，对用户来说，可能就是点击几次刷不出来页面，但是多点几次，就可以刷出来一次。

### 缓存穿透

缓存穿透是指客户端请求的数据在缓存中和数据库中都不存在，这样缓存永远不会生效，这些请求都会打到数据库。

图解：

[![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis-problems/1.png)](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis-problems/1.png)

解决方案：

1. 以数据库和缓存同步主键的方式来解决缓存穿透

   首先，主键会分为两种情况：自增和非自增。

   - 自增：如果数据库中的数据都是自增的主键，则将最大值的主键存放到 Redis 中。在高并发请求到来的时候，缓存中没有数据的情况下，先判断查询的主键是否小于等于缓存中主键的最大值，如果小于，则继续查询数据库，如果不小于，那么就没必要查询数据库了，因为数据肯定不在数据库中，从而减小数据库的压力。

   - 非自增：如果数据库中的主键非自增的话，则将数据库中所有的主键都取出来放到 Redis 的 set 中，按照的步骤，缓存中没有数据的时候先判断当前查询的主键在不在 set 中，在的话则代表数据在数据库中，进而查询数据库，不在的话就证明数据不在数据库中，就不用查询数据库了。

     [![img](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis-problems/2.png)](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis-problems/2.png)

2. 以缓存空值的方式来解决缓存穿透

   - 解析：如果请求来了，先查询缓存，缓存没查到，然后查询数据库，数据库中也查不到，这个时候向缓存中存入一个空值，并且设置一个过期时间，然后返回 null 或者错误信息提示，按业务需求，下次相同的请求就能到缓存中查到空值。

   - 优点：实现简单，维护方便。

   - 缺点：额外的内存消耗，另外如果前端发来不重复的请求，请求数据库不存在数据，那这样设置 null 值方案就等于失效的，这些请求还是会打到数据库，请求并发量过高使数据库宕机。

   - 流程图：以查询商铺为例：

     ![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis/设置空值解决缓存穿透业务逻辑图.png)

     ```java
     public Shop queryByIdWithPassThroughBySetNil(Long id) {
         //从redis中查询商铺缓存
         String shopJson = stringRedisTemplate.opsForValue().get(RedisConstants.CACHE_SHOP_KEY + id);
         //判断是否存在
         if (StrUtil.isNotBlank(shopJson)) {
             //存在，直接返回
             return JSONUtil.toBean(shopJson, Shop.class);
         }
         //有效解决缓存穿透
         if (Objects.equals(shopJson, "")) {
             return null;
         }
         //缓存中不存在，根据id查询数据库
         Shop shop = shopMapper.selectById(id);
         //数据库中不存在，在redis中给这个键设置一个空值，防止缓存穿透，并返回错误
         if (shop == null) {
             stringRedisTemplate.opsForValue().set(RedisConstants.CACHE_SHOP_KEY + id, "", RedisConstants.CACHE_NULL_TTL, TimeUnit.MINUTES);
             return null;
         }
         //数据库中存在，将数据写入redis，这里设置一个超时时间，是为双写一致性方案可能会出现的纰漏兜底
         //即使极端情况发生导致数据库和缓存的数据不一致，那么到达超时时间之后缓存会清空，数据再被访问时会同步新数据
         stringRedisTemplate.opsForValue().set(RedisConstants.CACHE_SHOP_KEY + id, JSONUtil.toJsonStr(shop), RedisConstants.CACHE_SHOP_TTL, TimeUnit.MINUTES);
         //返回
         return shop;
     }
     ```

3. 以布隆过滤器的方式来解决缓存穿透

   - 优点：内存占用较少，没有多余 key
   - 缺点：实现复杂、存在误判的可能。

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis/缓存穿透解决方案.png)

#### 总结

缓存穿透产生的原因是什么？

用户请求的数据在缓存中和数据库中都不存在，不断发起这样的请求，给数据库带来巨大压力。

缓存穿透的解决方案有哪些？

1. 数据库和缓存同步主键
2. 缓存null值
3. 布隆过滤
4. 增强id的复杂度，避免被猜测id规律
5. 做好数据的基础格式校验
6. 加强用户权限校验
7. 做好热点参数的限流

### 缓存击穿

缓存击穿也叫热点 Key 问题，就是说某个 key 非常热点，访问非常频繁，处于集中式高并发访问的情况，大量请求高并发查询某个高热度 key 的数据，大量请求到达应用服务器集群，首先查询 Redis 集群，但是这个 key 突然失效了，大量的访问请求会在瞬间给数据库带来巨大的冲击。

解决方式：

- 可以将热点数据设置为永远不过期；
- 或者基于 redis or zookeeper 实现互斥锁，等待第一个请求查询完数据库得到数据并且构建完缓存之后，再释放锁，进而其它请求才能通过该 key 访问数据，这样一来，释放锁之后，后面的请求就通过 key 直接去缓存中取数据
- 在将数据存入缓存的时候不设置 TTL，并且多存入一个字段，用它来表示 key 的过期时间。那么实质上，缓存中的这条数据就永远不会过期，当查询到这条缓存的时候，我们在程序中通过逻辑，根据这个字段来决定是否需要缓存重建。

| 解决方案 | 优点                             | 缺点                                       |
| -------- | -------------------------------- | ------------------------------------------ |
| 互斥锁   | 没有额外的内存消耗；保证一致性   | 线程需要等待，性能受影响；可能有死锁的风险 |
| 逻辑过期 | 线程无需等待，性能较好；高可用性 | 一致性稍差；有额外内存消耗；实现较复杂     |

#### 基于互斥锁方式解决缓存击穿问题

核心在于加锁，加锁的时机就是访问数据库之前，加锁的效果就是在缓存中查不到热点 key 的数据的情况下，需要访问数据库时不让这高并发的请求都打到数据库，只放进去一个线程，查询数据库、构建缓存，然后释放锁。剩下的请求去缓存中取数据，保护数据库。这里说的就是分布式锁。

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis/互斥锁解决缓存击穿业务逻辑图.png)

```java
public Shop queryByIdWithBreakDownByMutex(Long id) {
    String shopJson = stringRedisTemplate.opsForValue().get(RedisConstants.CACHE_SHOP_KEY + id);
    if (StrUtil.isNotBlank(shopJson)) {
        return JSONUtil.toBean(shopJson, Shop.class);
    }
    if (Objects.equals(shopJson, "")) {
        return null;
    }
    //缓存中拿不到数据，需要查询数据库，涉及到数据要考虑缓存穿透和缓存击穿的问题
    try {
        if (!getLock(id)) {
            TimeUnit.SECONDS.sleep(1);
            //递归调用，重新尝试到缓存中取数据，没有数据则尝试获取锁，拿不到锁就再次进到这个代码块
            queryByIdWithBreakDownByMutex(id);
        } else {
            Shop shop = shopMapper.selectById(id);
            //模拟查询数据库的延时
            TimeUnit.MILLISECONDS.sleep(200);
            if (shop == null) {
                stringRedisTemplate.opsForValue().set(RedisConstants.CACHE_SHOP_KEY + id, "", RedisConstants.CACHE_NULL_TTL, TimeUnit.MINUTES);
                return null;
            }
            stringRedisTemplate.opsForValue().set(RedisConstants.CACHE_SHOP_KEY + id, JSONUtil.toJsonStr(shop), RedisConstants.CACHE_SHOP_TTL, TimeUnit.MINUTES);
            return shop;
        }
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        releaseLock(id);
    }
    return null;
}
```

#### 基于逻辑过期方式解决缓存击穿问题

互斥锁方案的数据一致性更强，是因为一个线程拿到锁，其它线程都阻塞，同步等待到这个线程查询数据库、构建缓存完成之后。

逻辑过期方案的一致性显然差一些，即使逻辑判断过期也不等待，而是直接返回旧数据，查询数据库、构建缓存操作是在获取互斥锁后开启一个独立线程异步去执行的，这大大提升了性能，可用性提高。可以看到，此方案在开启独立线程之前，也就是操作数据库之前依然获取了互斥锁，目的就是不让所有请求的线程都开启独立线程去操作，最主要也起到了保护数据库的作用。

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis/逻辑过期解决缓存击穿业务逻辑图.png)

```java
public Shop queryByIdWithBreakDownByLogicalExpiration(Long id) {
    String redisDataJson = stringRedisTemplate.opsForValue().get(RedisConstants.CACHE_SHOP_KEY + id);
    if (StrUtil.isNotBlank(redisDataJson)) {
        RedisData redisData = JSONUtil.toBean(redisDataJson, RedisData.class);
        Object data = redisData.getData();
        LocalDateTime expireTime = redisData.getExpireTime();
        Shop shop = JSONUtil.toBean((JSONObject) data, Shop.class);
        if (LocalDateTime.now().isBefore(expireTime)) {
            //返回的数据没过期
            return shop;
        } else {
            //过期了，需要缓存重建，要先从数据库中查询，加一个互斥锁保护mysql的安全
            if (getLock(id)) {
                //异步去刷新redis的数据
                CACHE_REBUILD_POOL.submit(() -> {
                    try {
                        saveShopToRedis(id, 20L);
                        TimeUnit.MILLISECONDS.sleep(200);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } finally {
                        releaseLock(id);
                    }
                });
            }
            //没拿到锁。不等，直接返回一个过期的数据，但这个数据不一定就是错的
            return shop;
        }
    }
    if (Objects.equals(redisDataJson, "")) {
        return null;
    }
    //缓存查不到的话
    Shop shop = shopMapper.selectById(id);
    if (shop == null) {
        stringRedisTemplate.opsForValue().set(RedisConstants.CACHE_SHOP_KEY + id, "", RedisConstants.CACHE_NULL_TTL, TimeUnit.MINUTES);
        return null;
    }
    RedisData redisData = new RedisData();
    redisData.setData(shop);
    redisData.setExpireTime(LocalDateTime.now().plusSeconds(20L));
    stringRedisTemplate.opsForValue().set(RedisConstants.CACHE_SHOP_KEY + id, JSONUtil.toJsonStr(redisData));
    return shop;
}
```

其实说了这么多，无论是缓存雪崩、缓存穿透还是缓存击穿，都是在某种情况下大量请求打到数据库上，导致数据库宕机，遇到这些问题主要还是围绕如何让数据库不被高并发的请求冲击考虑。

### 缓存倾斜

引发原因：大量请求高并发访问高热度数据 key1，请求到达应用服务器集群，请求查询 Redis 集群，缓存中存在，但是因为高并发访问某一条数据它的 key 是固定的，导致请求全部落入固定的某台机器，最终这台缓存机器崩溃。

解决方式：添加节点的方式。热点数据所在的服务器一定是主从，我们可以做读写分离，将查询的请求都分发到 Redis 的从机上面，进行平分请求，这样一来，每个 master node 都有 slave node，主从节点，主节点主要负责增删改，从节点们主要负责查询。可使用注册中心来存取集群中的从机节点信息。其实，既然是 redis 集群的话，那么就是 redis cluster 模式，自身就已经包含了 replication+sentinel 的优点了。

### 缓存工具类的封装

- 将任意 Java 对象序列化为 json 并存储在 string 类型的 key 中，并且可以设置 TTL 过期时间
- 将任意 Java 对象序列化为 json 并存储在 string 类型的 key 中，并且可以设置逻辑过期时间，用于处理缓存击穿问题
- 根据指定的 key 查询缓存，并反序列化为指定类型，利用缓存空值的方式解决缓存穿透问题
- 根据指定的 key 查询缓存，并反序列化为指定类型，需要利用逻辑过期解决查询热点数据时可能发生的缓存击穿问题
- 根据指定的 key 查询缓存，并反序列化为指定类型，需要利用互斥锁解决查询热点数据时可能发生的缓存击穿问题

```java
import cn.hutool.core.util.BooleanUtil;
import cn.hutool.core.util.StrUtil;
import cn.hutool.json.JSONObject;
import cn.hutool.json.JSONUtil;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;
import java.util.Objects;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;
import java.util.function.Function;

import static com.hmdp.utils.RedisConstants.CACHE_NULL_TTL;
import static com.hmdp.utils.RedisConstants.LOCK_SHOP_KEY;

/**
 * 利用泛型和函数式接口
 * @author CHAN
 * @since 2022-03-28
 */
@Slf4j
@Component
public class CacheClient {
    private final StringRedisTemplate stringRedisTemplate;
    private static final ExecutorService CACHE_REBUILD_POOL = Executors.newFixedThreadPool(10);

    public CacheClient(StringRedisTemplate stringRedisTemplate) {
        this.stringRedisTemplate = stringRedisTemplate;
    }

    /**
     * 放入缓存，并设置TTL过期
     * @param key 键
     * @param value 值
     * @param time 过期时间
     * @param unit 时间单位
     */
    public void set(String key, Object value, Long time, TimeUnit unit) {
        stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(value), time, unit);
    }

    /**
     * 存入缓存，带逻辑过期字段
     * @param key 键
     * @param value 值
     * @param time 过期时间
     * @param unit 时间单位
     */
    public void setWithLogicalExpire(String key, Object value, Long time, TimeUnit unit) {
        //设置逻辑过期
        RedisData redisData = new RedisData();
        redisData.setData(value);
        redisData.setExpireTime(LocalDateTime.now().plusSeconds(unit.toSeconds(time)));
        //写入redis
        stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(redisData));
    }

    /**
     * 使用设置空值的方式解决缓存穿透问题
     * @param keyPrefix 键前缀
     * @param id 根据id查询
     * @param type 查询的数据类型
     * @param dbFallback 查询数据库函数式接口
     * @param time 时间
     * @param unit 单位
     * @param <R> 数据类型泛型
     * @param <ID> id类型
     * @return R
     */
    public <R, ID> R queryByIdWithPassThroughBySetNil(String keyPrefix, ID id, Class<R> type, Function<ID, R> dbFallback, Long time, TimeUnit unit) {
        String key = keyPrefix + id;
        //从redis中查询缓存
        String json = stringRedisTemplate.opsForValue().get(key);
        //判断是否存在
        if (StrUtil.isNotBlank(json)) {
            //存在，直接返回
            return JSONUtil.toBean(json, type);
        }
        //有效解决缓存穿透
        if (Objects.equals(json, "")) {
            return null;
        }
        //缓存中不存在，根据id查询数据库
        R r = dbFallback.apply(id);
        //数据库中不存在，在redis中给这个键设置一个空值，防止缓存穿透，并返回错误
        if (r == null) {
            this.set(key, "", time, unit);
            return null;
        }
        //数据库中存在，将数据写入redis，这里设置一个超时时间，是为双写一致性方案可能会出现的纰漏兜底
        //即使极端情况发生导致数据库和缓存的数据不一致，那么到达超时时间之后缓存会清空，数据再被访问时会同步新数据
        this.set(key, r, time, unit);
        //返回
        return r;
    }

    /**
     * 使用逻辑过期的方式解决查询热点数据时可能发生的缓存击穿问题
     * @param keyPrefix 键前缀
     * @param id 根据id查询
     * @param type 查询的数据类型
     * @param dbFallback 查询数据库函数式接口
     * @param time 时间
     * @param unit 单位
     * @param <R> 数据类型泛型
     * @param <ID> id类型泛型
     * @return R
     */
    public <R, ID> R queryWithLogicalExpire(
            String keyPrefix, ID id, Class<R> type, Function<ID, R> dbFallback, Long time, TimeUnit unit) {
        String key = keyPrefix + id;
        // 1.从redis查询缓存
        String json = stringRedisTemplate.opsForValue().get(key);
        // 2.判断是否存在
        if (StrUtil.isBlank(json)) {
            // 3.存在，直接返回
            return null;
        }
        // 4.命中，需要先把json反序列化为对象
        RedisData redisData = JSONUtil.toBean(json, RedisData.class);
        R r = JSONUtil.toBean((JSONObject) redisData.getData(), type);
        LocalDateTime expireTime = redisData.getExpireTime();
        // 5.判断是否过期
        if(expireTime.isAfter(LocalDateTime.now())) {
            // 5.1.未过期，直接返回信息
            return r;
        }
        // 5.2.已过期，需要缓存重建
        // 6.缓存重建
        // 6.1.获取互斥锁
        String lockKey = LOCK_SHOP_KEY + id;
        boolean isLock = tryLock(lockKey);
        // 6.2.判断是否获取锁成功
        if (isLock){
            // 6.3.成功，开启独立线程，实现缓存重建
            CACHE_REBUILD_POOL.submit(() -> {
                try {
                    // 查询数据库
                    R newR = dbFallback.apply(id);
                    // 重建缓存
                    this.setWithLogicalExpire(key, newR, time, unit);
                } catch (Exception e) {
                    throw new RuntimeException(e);
                }finally {
                    // 释放锁
                    unlock(lockKey);
                }
            });
        }
        // 6.4.返回过期的信息
        return r;
    }

    /**
     * 使用互斥锁的方式解决查询热点数据时可能发生的缓存击穿问题
     * @param keyPrefix 键前缀
     * @param id 根据id查询
     * @param type 查询的数据类型
     * @param dbFallback 查询数据库函数式接口
     * @param time 时间
     * @param unit 单位
     * @param <R> 数据类型泛型
     * @param <ID> id类型泛型
     * @return R
     */
    public <R, ID> R queryWithMutex(
            String keyPrefix, ID id, Class<R> type, Function<ID, R> dbFallback, Long time, TimeUnit unit) {
        String key = keyPrefix + id;
        // 1.从redis查询缓存
        String shopJson = stringRedisTemplate.opsForValue().get(key);
        // 2.判断是否存在
        if (StrUtil.isNotBlank(shopJson)) {
            // 3.存在，直接返回
            return JSONUtil.toBean(shopJson, type);
        }
        // 判断命中的是否是空值
        if (shopJson != null) {
            // 返回一个错误信息
            return null;
        }

        // 4.实现缓存重建
        // 4.1.获取互斥锁
        String lockKey = LOCK_SHOP_KEY + id;
        R r = null;
        try {
            boolean isLock = tryLock(lockKey);
            // 4.2.判断是否获取成功
            if (!isLock) {
                // 4.3.获取锁失败，休眠并重试
                Thread.sleep(50);
                return queryWithMutex(keyPrefix, id, type, dbFallback, time, unit);
            }
            // 4.4.获取锁成功，根据id查询数据库
            r = dbFallback.apply(id);
            // 5.不存在，返回错误
            if (r == null) {
                // 将空值写入redis
                stringRedisTemplate.opsForValue().set(key, "", CACHE_NULL_TTL, TimeUnit.MINUTES);
                // 返回错误信息
                return null;
            }
            // 6.存在，写入redis
            this.set(key, r, time, unit);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }finally {
            // 7.释放锁
            unlock(lockKey);
        }
        // 8.返回
        return r;
    }

    /**
     * 获取互斥锁
     * @param key 键
     * @return boolean
     */
    private boolean tryLock(String key) {
        Boolean flag = stringRedisTemplate.opsForValue().setIfAbsent(key, "1", 10, TimeUnit.SECONDS);
        return BooleanUtil.isTrue(flag);
    }

    /**
     * 释放互斥锁
     * @param key 键
     */
    private void unlock(String key) {
        stringRedisTemplate.delete(key);
    }
}
```

## 优惠券秒杀

Redis 的计数器、Lua 脚本 Redis、分布式锁、Redis 的三种消息队列。

### 全局唯一ID方案

全局唯一ID生成策略：

- UUID
- Redis自增
- snowflake算法
- 数据库自增

使用 Redis 自增方案生成全局唯一 ID 的时候，为了增加 ID 的安全性，我们可以不直接使用 Redis 自增的数值，而是拼接一些其它信息。

比如 ID 的组成部分为：

- 符号位：1bit，永远为0
- 时间戳：31bit，以秒为单位，可以使用69年
- 序列号：32bit，秒内的计数器（Redis 的自增计数器），支持每秒产生2^32个不同ID

```java
@Component
public class RedisIdWorker {

    /*LocalDateTime time = LocalDateTime.of(2022, 1, 1, 0, 0, 0);
    long second = time.toEpochSecond(ZoneOffset.UTC);
    second = 1640995200*/
    private static final long BEGIN_TIMESTAMP = 1640995200L;
    private static final int COUNT_BITS = 32;

    private final StringRedisTemplate stringRedisTemplate;

    public RedisIdWorker(StringRedisTemplate stringRedisTemplate) {
        this.stringRedisTemplate = stringRedisTemplate;
    }

    public long nextId(String keyPrefix) {
        //1.生成时间戳
        LocalDateTime now = LocalDateTime.now();
        long nowSecond = now.toEpochSecond(ZoneOffset.UTC);
        long timestamp = nowSecond - BEGIN_TIMESTAMP;
        //2.生成序列号
        String date = now.format(DateTimeFormatter.ofPattern("yyyy:MM:dd"));
        //将redis中"increment:" + keyPrefix + ":" + date键对应的值自增并返回自增之后的值，起初没有这个键的话，incr后会新建并返回1
        //increment方法参数就是要自增的key，效果就是redis中这个key对应的值自增
        //count 就是自增之后的计数值，也就是序列号
        Long count = stringRedisTemplate.opsForValue().increment("incr:" + keyPrefix + ":" + date);//redis的incr为原子操作，key拼接date效果就是每天一个key，也便于统计日流量。
        //3.拼接并返回
        return timestamp << COUNT_BITS | count;//时间戳左移运算后再以或运算的方式拼接计数器
    }

    public static void main(String[] args) {
        LocalDateTime time = LocalDateTime.of(2022, 1, 1, 0, 0, 0);
        long second = time.toEpochSecond(ZoneOffset.UTC);
        System.out.println("second = " + second);
    }
}
```

Redis自增ID策略：

- 每天一个key，方便统计订单量
- ID构造是时间戳 + 计数器

```java
/**
 * 测试IdWorker，生成30000个ID计时
 */
@Test
public void testIdWorker() throws InterruptedException {
    CountDownLatch latch = new CountDownLatch(300);//设置从300次开始递减
    long begin = System.currentTimeMillis();//开始时间
    for (int i = 0; i < 300; i++) {
        //x300
        POOL.submit(() -> {
            for (int j = 0; j < 100; j++) {
                //x100
                long orderId = idWorker.nextId("order");
                System.out.println("id = " + orderId);
            }
            //每次-1
            latch.countDown();
        });
    }
    latch.await();//等300次执行完
    long end = System.currentTimeMillis();//结束时间
    System.out.println("time = " + (end - begin));//300次总用时
}
```

### 实现优惠券秒杀下单

初步流程图

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis/秒杀下单初步流程.png)

客户发送请求到我们的系统，每个请求都对应着一个线程，我们的库存数据是存在数据库的，数据库中的所有数据对于每个线程来说都是共享资源。因此需要考虑多线程并发的情况下的线程安全问题。

#### 超卖问题

上面说的线程安全问题，也就是业务中优惠券的超卖问题。超卖问题是典型的多线程安全问题，针对这一问题的常见解决方案就是加锁。

##### 悲观锁

认为线程安全问题一定会发生，因此在操作数据之前先获取锁，确保线程串行执行。例如Synchronized，Lock 在某种意义上也属于悲观锁。

##### 乐观锁

认为线程安全问题不一定会发生，因此不加锁，只是在更新数据时去判断有没有其它线程对数据做了修改。

- 如果没有修改则认为是安全的，自己才更新数据。
- 如果已经被其它线程修改说明发生了安全问题，此时可以重试或异常。

1. 版本号法

   其实 Mybatis plus 里面乐观锁方案的实现就是版本号法，需要在数据库多加一个 version 字段。

   - 取出记录时，获取当前 version
   - 更新时，带上这个 version 一起更新
   - 执行更新时， set version = newVersion where version = oldVersion
   - 如果 version 不对，就更新失败
   
   ```sql
   update ... set stock = stock - 1, version = version + 1 where id = ... and version = ...;
   ```

2. CAS 法

   ```sql
   update ... set stock = stock - 1 where id = ... and stock = ...;
   ```

   stock 的值为 ... 时，才会将 stock 的值改为 stock - 1，否则就不做更改，典型的比较并置换的思想。但是如果高并发的请求来，判断完只要不等就不改，很有可能会存在成功率低的问题，因为不会自动重试。

所以最终：

   ```sql
   update ... set stock = stock - 1 where id = ... and stock > 0;
   ```

##### 总结

超卖这样的线程安全问题，解决方案有哪些？

1. 悲观锁：添加同步锁，让线程串行执行
   - 优点：简单粗暴
   - 缺点：性能一般
2. 乐观锁：不加锁，在更新时判断是否有其它线程在修改
   - 优点：性能好
   - 缺点：存在成功率低的问题

#### 一人一单

如果一个人发起高并发的秒杀下单请求，优惠券的数量有限，如果一个人高并发重复下单的话，那所有的优惠券都被他一个人抢购了，这不是黄牛是啥？这显然是不公平的，因此有一人一单的业务需求。

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis/一人一单.png)

在根据用户 ID 和优惠券 ID 查询订单之前，需要获取锁。也就是说，只能让一个线程去查询订单。因为如果一个人发起高并发秒杀请求，这个人之前没有下过单，如果不加锁的话，多个请求并发去查询数据库订单表，都发现订单不存在，那么这些个请求就可以重复下单了，不符合一人一单。

```java
@Transactional
public VoucherOrder updateStockAndSaveOrder(Long voucherId) {
    Long userId = UserHolder.getUser().getId();
    synchronized (userId.toString().intern()) {
        QueryWrapper<VoucherOrder> queryWrapper = new QueryWrapper<>();
        queryWrapper.eq("user_id", userId);
        queryWrapper.eq("voucher_id", voucherId);
        Integer count = voucherOrderMapper.selectCount(queryWrapper);
        if (count > 0) {
            return null;
        }
        UpdateWrapper<SeckillVoucher> updateWrapper = new UpdateWrapper<>();
        updateWrapper.setSql("stock = stock - 1").eq("voucher_id", voucherId).gt("stock", 0);
        // mysql连接驱动的更新操作默认返回的并不是受影响的行数，如果想设置返回值是受影响的行数，修改数据库链接配置：增加useAffectedRows=true
        int modified = seckillVoucherMapper.update(null, updateWrapper);
        if (modified != 1) {
            return null;
        }
        VoucherOrder voucherOrder = new VoucherOrder();
        voucherOrder.setId(idWorker.nextId("order"));
        voucherOrder.setUserId(userId);
        voucherOrder.setVoucherId(voucherId);
        voucherOrderMapper.insert(voucherOrder);
        return voucherOrder;
    }
}
```

##### 分布式锁

通过加锁可以解决在单机情况下的一人一单安全问题，但是在集群模式下就不行了。我们将服务启动两份，端口分别为8081和8082。然后修改nginx的conf目录下的nginx.conf文件，配置反向代理和负载均衡：

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis/集群式部署服务nginx负载均衡.png)

现在，用户请求会在这两个节点上负载均衡，再次测试下是否存在线程安全问题。当然存在！

这时就需要用到分布式锁了，什么是分布式锁？满足分布式系统或集群模式下多进程都可见并且互斥的锁。

##### 分布式锁的实现

分布式锁的核心是实现多进程之间互斥，而满足这一点的方式有很多，常见的有三种：

|        | MySQL                     | Redis                    | Zookeeper                        |
| ------ | ------------------------- | ------------------------ | -------------------------------- |
| 互斥   | 利用mysql本身的互斥锁机制 | 利用setnx这样的互斥命令  | 利用节点的唯一性和有序性实现互斥 |
| 高可用 | 好                        | 好                       | 好                               |
| 高性能 | 一般                      | 好                       | 一般                             |
| 安全性 | 断开连接，自动释放锁      | 利用锁超时时间，到期释放 | 临时节点，断开连接自动释放       |

##### 基于 Redis 的分布式锁

实现分布式锁时需要实现的两个基本方法：

- 获取锁

  - 互斥：确保只能有一个线程获取锁

  - 非阻塞：尝试一次，成功返回 true，失败返回 false

    ```sh
    # 添加锁，NX是互斥，EX是设置超时时间，要添加超时时间，避免服务宕机引起的死锁，将EX写在一条命令里，保证原子性
    SET lock thread01 NX EX 10;
    ```

- 释放锁

  - 手动释放

  - 超时释放

    ```sh
    DEL lock;
    ```

###### 细节分析1

加了 Redis 分布式锁之后，业务流程大致长这样：

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis/可能误删.png)

看起来好像没什么问题，但是如果是下图中的情况，线程1执行的时候业务阻塞了，阻塞时间超过了锁的 key 的过期时间，那么超时就会自动释放锁，这时线程2一看锁被释放了，它抢到锁了，开始执行线程2中的业务，这时线程1不阻塞了，线程1业务完成之后释放锁了，这个时候释放的是线程2持有的锁，线程2的锁被线程1释放了，线程3一看锁被释放了，它又抢到了锁开始执行线程3的业务，但是此时线程2还未执行完，线程3也开始执行了，此时线程2、3并行运行了，就很有可能出现线程安全问题。

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis/细节1.png)

> 解决方案：

添加线程标识，任何线程在释放锁之前都要去判断当前业务的锁此刻被哪个线程持有。在一个线程获得锁的时候，设置键为锁前缀加当前业务名，设置值为当前线程标识。判断之后，如果从缓存中取出的线程标识和自己当前获取的线程标识一致，那么就释放锁，否则不操作。如下图：图中的“锁标识”，就是上面说的线程标识，用来标识当前锁被哪个线程持有。

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis/细节2.png)

业务流程变化为下面这样子：

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis/误删解决.png)

这样就避免细节分析1中所说的隐患。

部分业务代码：

```java
/**
 * @author CHAN
 * @since 2022-03-31
 */
public interface ILock {
    boolean getLock(long timeoutSec);
    void releaseLock();
}
/*------------------------------------------------------------------------------*/
/**
 * 基于redis实现分布式锁
 * @author CHAN
 * @since 2022-03-31
 */
public class SimpleLock implements ILock{

    //业务名称
    private final String businessName;
    private final StringRedisTemplate redisTemplate;
    private static final String KEY_PREFIX = "lock:";
    private static final String ID_PREFIX = UUID.randomUUID().toString(true) + "-";
    public SimpleLock(String businessName, StringRedisTemplate redisTemplate) {
        this.businessName = businessName;
        this.redisTemplate = redisTemplate;
    }


    /**
     * 添加分布式锁
     * @param timeoutSec 分布式锁未释放，超时自动释放
     * @return boolean
     */
    @Override
    public boolean getLock(long timeoutSec) {
        //获取当前线程标识
        String threadId = ID_PREFIX + Thread.currentThread().getId();
        //获取锁
        Boolean success = redisTemplate.opsForValue().setIfAbsent(KEY_PREFIX + businessName, threadId, timeoutSec, TimeUnit.SECONDS);
        return Boolean.TRUE.equals(success);
    }

    /**
     * 释放分布式锁
     */
    @Override
    public void releaseLock() {
        //获取当前线程标识
        String threadId = ID_PREFIX + Thread.currentThread().getId();
        //获取redis锁中的标识
        String id = redisTemplate.opsForValue().get(KEY_PREFIX + businessName);
        //判断标识是否一致
        if (Objects.equals(threadId, id)){
            //一致则释放锁
            redisTemplate.delete(KEY_PREFIX + businessName);
        }
    }
}
/*------------------------------------------------------------------------------*/
@Transactional
public VoucherOrder updateStockAndSaveOrder2(Long voucherId) {
    Long userId = UserHolder.getUser().getId();
    //以userId为键，锁user，降低锁粒度，提升效率
    SimpleLock simpleLock = new SimpleLock("order:" + userId, stringRedisTemplate);
    boolean lock = simpleLock.getLock(10);
    if (!lock) {
        return null;
    } else {
        try {
            //这里加锁的意义在于只能有一个线程去查询数据库中的订单，确保一人一单
            QueryWrapper<VoucherOrder> queryWrapper = new QueryWrapper<>();
            queryWrapper.eq("user_id", userId);
            queryWrapper.eq("voucher_id", voucherId);
            Integer count = voucherOrderMapper.selectCount(queryWrapper);
            if (count != 0) {
                //证明之前购买过
                return null;
            }
            //扣减库存
            UpdateWrapper<SeckillVoucher> updateWrapper = new UpdateWrapper<>();
            updateWrapper.setSql("stock = stock - 1");
            updateWrapper.eq("voucher_id", voucherId);
            updateWrapper.gt("stock", 0);
            int modified = seckillVoucherMapper.update(null, updateWrapper);
            if (modified != 1) {
                return null;
            }
            VoucherOrder voucherOrder = new VoucherOrder();
            voucherOrder.setVoucherId(voucherId);
            voucherOrder.setUserId(userId);
            voucherOrder.setId(idWorker.nextId("order"));
            voucherOrderMapper.insert(voucherOrder);
            return voucherOrder;
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            simpleLock.releaseLock();
        }
    }
    return null;
}
```



###### 细节分析2

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis/细节3.png)

这种情况是一种很极端的情况。线程1执行完业务逻辑，要释放锁，从缓存中获取线程标识并判断是否一致，结果一致，可以释放锁。可是就在快要释放锁还没释放的时候，线程1阻塞了，这时的阻塞不是因为业务阻塞，而很可能是 JVM 恰好此时进行垃圾回收了，可能是回收老年代，这时整个服务器都会 STW，从而造成阻塞，然后阻塞期间，分布式锁到达过期时间，超时释放了锁。然后因为服务是集群式部署，线程2在另一个服务器上，这时线程2一看，锁被释放了，然后线程2去抢锁并且成功抢到，然后线程2开始执行自己的业务代码。然鹅过一会，线程1所在的服务器垃圾清理完，阻塞解除恢复运行，线程1啪嚓释放锁，因为线程1在前面判断过线程标识一致了，不会重复判断，直接把此时线程2持有的锁释放了。线程3一看锁被释放了，线程3去抢锁并成功抢到，线程3开始执行自己的业务代码，此时线程2、3并行执行，就会出现线程安全问题。

> 解决方案：

思路主要是围绕：确保`判断锁标识`的动作和`释放锁`的动作，两个动作的`原子性`，一起执行成功或失败。一聊到原子性，那么就很容易想到事务，没错，Redis 中也有事务，但是 Redis 中的事务和我们所了解的 MySQL 的事务是有很大差别的，因此这里不介绍 Redis 的事务方案了。我们使用 Redis 的 Lua 脚本来实现这两个动作的“原子性”。

###### Redis 的 Lua 脚本

Redis 提供了 Lua 脚本功能，在一个脚本中编写多条 Redis 命令，确保多条命令执行时的原子性。Lua 是一种编程语言，它的基本语法大家可以参考网站：https://www.runoob.com/lua/lua-tutorial.html。这里重点介绍Redis提供的调用函数，语法如下：

```lua
-- 执行redis命令
redis.call('命令名称', 'key', '其它参数', ...)
```

例如，我们要执行set name jack，则脚本是这样：

```lua
-- 执行 set name jack
redis.call('set', 'name', 'jack')
```

例如，我们要先执行set name Rose，再执行get name，保证这两个动作的原子性，就把两条命令写入一个脚本，以脚本为单位执行。则脚本如下：

```lua
-- 先执行 set name jack
redis.call('set', 'name', 'jack')
-- 再执行 get name
local name = redis.call('get', 'name')
-- 返回
return name
```

写好脚本以后，需要用Redis命令来调用脚本，调用脚本的常见命令如下：

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis/redis调用lua脚本.png)

例如，我们要执行 `redis.call('set', 'name', 'jack')` 这个脚本，语法如下：EVAL "脚本"

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis/redis调用lua脚本案例1.png)

注：Redis 里面的参数分为两类

- key 类型的参数，比如上面案例中的 name；
- 另一个就是其它参数，比如案例中的 jack；

如果脚本中的 key、value 不想写死，可以作为参数传递。key 类型参数会放入 KEYS 数组，其它参数会放入 ARGV 数组，在脚本中可以从 KEYS 和 ARGV 数组获取这些参数：

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis/redis调用lua脚本案例2.png)

注：与 Java 语言不同的是，在 Lua 语言中，索引下标是从1开始的，因此这里的 KEYS[1]、ARGV[1] 都是数组中的第一个元素。

练习：

```lua
127.0.0.1:6379> keys *
 1) "zs"
 2) "bbbbb"
 3) "cache:shop:types"
 4) "aaaaa"
 5) "user:1"
 6) "ls"
 7) "cache:shop:1"
 8) "user:2"
 9) "stu"
10) "incr:order:2022:03:31"
11) "incr:order:2022:03:30"
12) "exam"
127.0.0.1:6379> get name
(nil)
----------------------------------------------------------------

127.0.0.1:6379> EVAL "return redis.call('set','name','chan')" 0
OK
127.0.0.1:6379> get name
"chan"
----------------------------------------------------------------

-- 使用redis调用lua脚本的方式来存入string类型的value
-- 在key、value不想写死，想作为参数传递时，KEYS和ARGV不能小写，否则报错！
127.0.0.1:6379> eval "return redis.call('set', keys[1], argv[1])" 1 name chanservy
(error) ERR Error running script (call to f_a3d2706d73040d2c4e982bb2bb686692d05f34a7): @enable_strict_lua:15: user_script:1: Script attempted to access nonexistent global variable 'keys'
127.0.0.1:6379> eval "return redis.call('set', KEYS[1], ARGV[1])" 1 name chanservy
OK
127.0.0.1:6379> get name
"chanservy"
----------------------------------------------------------------

-- 使用redis调用lua脚本的方式来存入多组string类型的value
-- 在使用mset时，设置几组键值对，key类型的参数个数就设置几个。
-- 当前面顺序是：KEYS[1], KEYS[2], ARGV[1], ARGV[2]时，后面设置值的顺序如果也是键键值值，那么设置结果会错乱。
127.0.0.1:6379> eval "return redis.call('mset', KEYS[1], KEYS[2], ARGV[1], ARGV[2])" 2 age sport 22 run
OK
127.0.0.1:6379> keys *
 1) "zs"
 2) "bbbbb"
 3) "cache:shop:types"
 4) "aaaaa"
 5) "user:1"
 6) "ls"
 7) "cache:shop:1"
 8) "user:2"
 9) "age"
10) "stu"
11) "incr:order:2022:03:31"
12) "name"
13) "incr:order:2022:03:30"
14) "22"
15) "exam"
-- 结果错乱
127.0.0.1:6379> get age
"sport"
127.0.0.1:6379> get 22
"run"
127.0.0.1:6379> del age 22
(integer) 2
-------------------------------------------------------------------

-- 当前面顺序是：KEYS[1], KEYS[2], ARGV[1], ARGV[2]时，后面设置值的顺序必须是键值键值，这样设置的结果就不会错乱。
127.0.0.1:6379> eval "return redis.call('mset', KEYS[1], KEYS[2], ARGV[1], ARGV[2])" 2 age 22 sport run
OK
127.0.0.1:6379> keys *
 1) "zs"
 2) "bbbbb"
 3) "cache:shop:types"
 4) "aaaaa"
 5) "user:1"
 6) "sport"
 7) "ls"
 8) "cache:shop:1"
 9) "user:2"
10) "age"
11) "stu"
12) "incr:order:2022:03:31"
13) "name"
14) "incr:order:2022:03:30"
15) "exam"
127.0.0.1:6379> get age
"22"
127.0.0.1:6379> get sport
"run"
--------------------------------------------------------------------------

-- 当前面顺序是：KEYS[1], ARGV[1], KEYS[2], ARGV[2]时，后面设置值的顺序如果也是键值键值，那么设置结果会错乱
127.0.0.1:6379> eval "return redis.call('mset', KEYS[1], ARGV[1], KEYS[2], ARGV[2])" 2 age 23 sport running
OK
-- 结果错乱
127.0.0.1:6379> get age
"sport"
127.0.0.1:6379> get 23
"running"
--------------------------------------------------------------------------

-- 当前面顺序是：KEYS[1], ARGV[1], KEYS[2], ARGV[2]时，后面设置值的顺序必须是键键值值，这样设置的结果就不会错乱。
127.0.0.1:6379> eval "return redis.call('mset', KEYS[1], ARGV[1], KEYS[2], ARGV[2])" 2 age sport 25 running
OK
127.0.0.1:6379> get age
"25"
127.0.0.1:6379> get sport
"running"
---------------------------------------------------------------------------

-- 使用redis调用lua脚本的方式来存入hash类型的value
127.0.0.1:6379> eval "return redis.call('hset', KEYS[1], ARGV[1], ARGV[2])" 1 classroom stuNum 30
(integer) 1
127.0.0.1:6379> hget classroom stuNum
"30"
127.0.0.1:6379> 
```

现在回到原来的问题，我们要保证判断锁标识和释放锁两个动作的原子性。基于 Redis 的分布式锁，回顾我们前面研究过的，在释放锁的时候，整体的业务流程是这样的：

1. 从 Redis 中获取锁中的线程标识
2. 判断是否与指定的标识（当前线程标识）一致
3. 如果一致则释放锁（删除）
4. 如果不一致则什么都不做

我们前面是在 Java 代码中实现上述的逻辑的，现在用 Lua 语言逻辑实现一下：

```lua
-- redis锁的key
local key = "lock:order:2";
-- 当前线程标识
local threadId = "hcuisjfoehnsdkfcjnseuf-33";
-- 获取锁中的线程标识
local id = redis.call('get', key);
-- 比较当前线程标识与锁中的线程标识是否一致
if id == threadId then
    -- 一致，释放锁
    return redis.call('del', key);
end
-- 不一致，什么都不做
return 0;
```

我们稍作改进，不把脚本中的 key、value 写死，可以作为参数传递：

```lua
-- redis锁的key，这里作为参数在java程序中传过来
local key = KEYS[1];
-- 当前线程标识，这里作为参数在java程序中获取到传过来
local threadId = ARGV[1];
-- 获取锁中的线程标识，redis中的
local id = redis.call('get', key);
-- 比较当前线程标识与锁中的线程标识是否一致
if id == threadId then
    -- 一致，释放锁
    return redis.call('del', key);
end
-- 不一致，什么都不做
return 0;
```

最终也可以简化为：

```lua
-- 这里的 KEYS[1] 就是锁的key，这里的ARGV[1] 就是当前线程标示
-- 获取锁中的标示，判断是否与当前线程标示一致
if (redis.call('GET', KEYS[1]) == ARGV[1]) then
  -- 一致，则删除锁
  return redis.call('DEL', KEYS[1])
end
-- 不一致，则直接返回
return 0
```

###### 再次改进 Redis 的分布式锁

基于 Lua 脚本实现 Redis 分布式锁的释放锁逻辑，RedisTemplate 调用 Lua 脚本的 API 如下：

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis/RedisTemplate调用Lua脚本.png)

代码改动如下：

```java
/**
 * 基于redis实现分布式锁
 * @author CHAN
 * @since 2022-03-31
 */
public class SimpleLock implements ILock{

    //业务名称
    private final String businessName;
    private final StringRedisTemplate redisTemplate;

    public SimpleLock(String businessName, StringRedisTemplate redisTemplate) {
        this.businessName = businessName;
        this.redisTemplate = redisTemplate;
    }

    private static final String KEY_PREFIX = "lock:";
    private static final String ID_PREFIX = UUID.randomUUID().toString(true) + "-";
    private static final DefaultRedisScript<Long> UNLOCK_SCRIPT;
    static {
        // 静态的，在类的初始化阶段执行，初始化只在类加载的时候执行一次，这样不用每次释放锁都去加载lua脚本文件，减少IO，提升效率
        UNLOCK_SCRIPT = new DefaultRedisScript<>();
        UNLOCK_SCRIPT.setLocation(new ClassPathResource("unlock.lua"));
        UNLOCK_SCRIPT.setResultType(Long.class);
    }


    /**
     * 添加分布式锁
     * @param timeoutSec 分布式锁未释放，超时自动释放
     * @return boolean
     */
    @Override
    public boolean getLock(long timeoutSec) {
        //获取当前线程标识
        String threadId = ID_PREFIX + Thread.currentThread().getId();
        //获取锁
        Boolean success = redisTemplate.opsForValue().setIfAbsent(KEY_PREFIX + businessName, threadId, timeoutSec, TimeUnit.SECONDS);
        return Boolean.TRUE.equals(success);
    }

    /**
     * 释放分布式锁，使用lua脚本的方式
     */
    @Override
    public void releaseLock() {
        List<String> key = Collections.singletonList(KEY_PREFIX + businessName);
        String threadID = ID_PREFIX + Thread.currentThread().getId();
        //调用Lua脚本，判断线程标识和释放锁放在一个脚本中，保证原子性
        redisTemplate.execute(UNLOCK_SCRIPT, key, threadID);
    }

    /*
    之前的实现方式，可以看到判断线程标识和释放锁是两句代码，不能保证原子性
    @Override
    public void releaseLock() {
        //获取当前线程标识
        String threadId = ID_PREFIX + Thread.currentThread().getId();
        //获取redis锁中的标识
        String id = redisTemplate.opsForValue().get(KEY_PREFIX + businessName);
        //判断标识是否一致
        if (Objects.equals(threadId, id)){
            //一致则释放锁
            redisTemplate.delete(KEY_PREFIX + businessName);
        }
    }*/
}
```

###### 总结

大致思路：在使用 Redis 的分布式锁时：在获取锁的时候，为了防止后期死锁的问题，我们一般需要加一个锁的过期时间，并且注意`获取锁和设置锁的过期时间两个操作的原子性`；在释放锁的时候，需要注意，如果当前线程执行业务的时间过长，可能比我们设置的锁的过期时间还要长，导致锁到了过期时间自己释放了，这时这把分布式锁可能被其它线程获取到，然后当前线程执行到最后释放锁，这时相当于把其它线程的锁释放了，因此在获取锁的时候，我们需要`把 Redis 中代表锁的 key 对应的 value 设置一个标识`，这个 value 可以是 UUID + 当前线程的 id，然后在当前线程释放锁之前先获取一下 value，判断一下这把锁是否仍然被当前线程所持有，是则释放锁，否则不操作。最后注意一点：`保证判断线程标识和释放锁两个动作的原子性`。原因很简单：当前线程根据 key 获取到了 value，判断确实是当前线程在持有，准备释放锁，可是就在释放锁之前，恰好这时锁到期了自动释放了，而且又被其它线程获取到了。然后当前线程开始执行释放锁的代码，那就相当于又是释放了其它线程的锁。

基于 Redis 的分布式锁实现思路：

- 利用 set nx ex 获取锁，并根据业务估算设置过期时间，并保存线程标识
- 释放锁的时候先判断 Redis 锁中的线程标识与当前自己的线程标识是否一致，一致则删除锁，防止误删。
- 保证判断线程标识和释放锁两个动作的原子性

特性：

- 利用 set nx 满足互斥性
- 利用 set ex 保证程序故障时锁依然能释放，避免死锁，提高安全性
- 利用 Redis 集群保证高可用和高并发特性

至此，我们自己实现的 Redis 分布式锁，已经具备线上生产可用的能力。

#### Redis 的分布式锁优化

虽然我们实现了一个生产可用的基于 Redis 的分布式锁，但是基于 setnx 实现的分布式锁存在下面的问题：

- 不可重入：同一个线程无法多次获取同一把锁。
- 不可重试：获取锁只尝试一次就返回 false，没有重试机制。
- 超时释放：锁超时释放虽然可以避免死锁，但如果是业务执行耗时较长，也会导致锁释放，存在安全隐患。
- 主从一致性：如果 Redis 是主从架构，主从同步存在延迟，当主宕机时，如果从节点没有同步主节点中的锁数据，则会出现锁失效。

这个时候虽然可以参考 ReentrantLock 的底层实现，但是自己实现起来很繁琐。所以我们可以借助 Redisson。

#### Redisson

Redisson 是一个在 Redis 的基础上实现的 Java 驻内存数据网格（In-Memory Data Grid）。它不仅提供了一系列的分布式的 Java 常用对象，还提供了许多分布式服务，其中就包含了各种分布式锁的实现。

通俗点讲，Redisson 是一个在 Redis 的基础上实现的一个分布式工具的集合，也就是说，在分布式系统下需要用到的各种各样的工具它都有，分布式锁只是它的一个子集，其中就包含了各种分布式锁的实现：

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis/Redisson.png)

官网地址： https://redisson.org；

GitHub地址： https://github.com/redisson/redisson。

[Redisson 项目介绍](https://github.com/redisson/redisson/wiki/Redisson%E9%A1%B9%E7%9B%AE%E4%BB%8B%E7%BB%8D)

##### Redisson 入门

1、引入依赖，注：虽然 springboot 提供了 starter 的方式引入，并且可以以 boot 的方式将配置项配置到 yml 中，但是这里不建议那样引入和配置。因为那样的话会覆盖掉 springboot 对于 redis 原有的默认配置。

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson</artifactId>
    <version>3.17.0</version>
</dependency>
```

2、配置 Redisson 客户端

```java
@Configuration
public class RedisConfig {
    @Bean
    public RedissonClient redisClient(){
        //配置类
        Config config = new Config();
        // 添加redis地址，这里添加了单点的地址，也可以使用config.useClusterServers()添加集群地址
        config.useSingleServer().setAddress("redis://192.168.1.107:6379").setPassword("123456");
        //创建客户端
        return Redisson.create(config);
    }
}
```

3、使用Redisson的分布式锁

```java
@Test
public void testRedisson() throws InterruptedException {
    // 获取锁（可重入），指定锁的名称
    RLock lock = redissonClient.getLock("testLock");
    // 尝试获取锁，参数分别是：获取锁的最大等待时间（期间会重试，不写会默认-1，不等待），锁自动释放时间（不写默认30s），时间单位
    // true代表获取到分布式锁，存在redis中的格式 key:testLock  field:bd32e60a-8e52-455f-a4a9-6262d39c976f:1  value:1
    boolean isLock = lock.tryLock(1, 10, TimeUnit.SECONDS);
    // 判断释放获取成功
    if (isLock) {
        try {
            System.out.println("执行业务逻辑...");
        } finally {
            // 释放锁
            lock.unlock();
        }
    }
}
```

##### Redisson 分布式锁可重入

其实 Redisson 可重入的原理和 ReentrantLock 的可重入原理是一样的。可以看看 ReentrantLock 的可重入原理源码解析：

[ReentrantLock 的可重入原理](https://chanservy.vercel.app/posts/20211124/java-concurrent-6.html#%E5%8F%AF%E9%87%8D%E5%85%A5%E5%8E%9F%E7%90%86)

在 ReentrantLock 中，加锁时，如果发现 state 不为 0，并且 owner 是当前线程的话，就代表锁重入，让 state 自增；解锁时，让 state 自减，直到减为 0，才真正解开。所以在 Redisson 的分布式锁的可重入实现中也有一个计数，用来表示当前线程获取了几次分布式锁。前面我们自己实现的 Redis 分布式锁使用的是 String 类型的 value，但是 Redisson 的分布式锁中不仅要记录锁的线程标识还要记录重入的数值。因此，显而易见，在 Redisson 分布式锁的底层使用的是 redis 的 hash 类型的 value。

```java
// Redisson分布式锁是可重入的
RLock lock = redissonClient.getLock("lock");
@Test
public void method1() throws InterruptedException {
    boolean isLock = lock.tryLock();
    if (!isLock) {
        log.error("获取锁失败，1");
    }
    try {
        log.info("获取锁成功，1");
        method2();
    } finally {
        log.info("释放锁，1");
        lock.unlock();
    }
}
public void method2() {
    boolean isLock = lock.tryLock();
    if (!isLock) {
        log.error("获取锁失败，2");
    }
    try {
        log.info("获取锁成功，2");
    } finally {
        log.info("释放锁，2");
        lock.unlock();
    }
}
```

###### 可重入原理图

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis/redisson可重入锁原理图.png)

###### 获取锁的 Lua 脚本

```lua
local key = KEYS[1]; -- 锁的key
local threadId = ARGV[1]; -- 线程唯一标识
local releaseTime = ARGV[2]; -- 锁的自动释放时间
-- 判断是否存在
if(redis.call('exists', key) == 0) then
    -- 不存在, 获取锁
    redis.call('hset', key, threadId, '1');
    -- 设置有效期
    redis.call('expire', key, releaseTime);
    return 1; -- 返回结果
end;
-- 锁已经存在，判断threadId是否是自己
if(redis.call('hexists', key, threadId) == 1) then
    -- 存在(threadId是自己), 获取锁，重入次数+1
    redis.call('hincrby', key, threadId, '1');
    -- 重置有效期
    redis.call('expire', key, releaseTime);
    return 1; -- 返回结果
end;
return 0; -- 代码走到这里,说明持有锁的不是自己，获取锁失败
```

###### 释放锁的 Lua 脚本

```lua
local key = KEYS[1];-- 锁的key
local threadId = ARGV[1]; -- 线程唯一标识
local releaseTime = ARGV[2]; -- 锁的自动释放时间
-- 判断当前锁是否还是被自己持有
if (redis.call('HEXISTS', key, threadId) == 0) then
    return nil; -- 如果已经不是自己，则直接返回
end;
-- 是自己的锁，则重入次数-1
local count = redis.call('HINCRBY', key, threadId, -1);
-- 判断是否重入次数是否已经为0 
if (count > 0) then
    -- 大于0说明不能释放锁，重置有效期然后返回
    redis.call('EXPIRE', key, releaseTime);
    return nil;
else  -- 等于0说明可以释放锁，直接删除
    redis.call('DEL', key);
    return nil;
end;
```

##### Redisson 分布式锁原理图

这张图是根据 Redisson 的底层源码逻辑流程画的，其中能充分表现 Redisson 分布式锁的“重入、可重试、超时释放机制”这几个重要的特性。

![](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/CDN2/redis/redisson原理图.png)

Redisson 分布式锁原理总结：

- 可重入：利用 redis 的 hash 结构记录线程 id 和重入次数。
- 可重试：利用信号量和 PubSub 功能实现等待、唤醒，就是获取锁失败的重试机制。
- 超时续约：利用 watchDog 看门狗机制，每隔一段时间（releaseTime / 3 = 10s），重置超时时间为 30s。具体原理看图。
- 主从一致性：如果 Redis 是主从架构，主从同步存在延迟，当主宕机时，如果从节点没有同步主节点中的锁数据，则会出现锁失效，其它线程可以获取锁，造成线程安全问题。为了避免上述的问题，Redisson 对于这个问题的解决方案简单粗暴，不采用一主多从模式，而是多主多从模式。也就是说多个独立的 Redis 节点，必须在所有节点都获取重入锁，才算获取锁成功。

#### Redisson 之分布式锁和同步器

##### 可重入锁（Reentrant Lock）

基于Redis的Redisson分布式可重入锁[`RLock`](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RLock.html) Java对象实现了`java.util.concurrent.locks.Lock`接口。同时还提供了[异步（Async）](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RLockAsync.html)、[反射式（Reactive）](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RLockReactive.html)和[RxJava2标准](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RLockRx.html)的接口。

```java
RLock lock = redisson.getLock("anyLock");
// 最常见的使用方法
lock.lock();// 获取锁失败一直等待
```

大家都知道，如果负责储存这个分布式锁的Redisson节点宕机以后，而且这个锁正好处于锁住的状态时，这个锁会出现锁死的状态。为了避免这种情况的发生，Redisson内部提供了一个监控锁的看门狗，它的作用是在Redisson实例被关闭前，不断的延长锁的有效期。默认情况下，看门狗的检查锁的超时时间是30秒钟，也可以通过修改[Config.lockWatchdogTimeout](https://github.com/redisson/redisson/wiki/2.-配置方法#lockwatchdogtimeout监控锁的看门狗超时单位毫秒)来另行指定。

另外Redisson还通过加锁的方法提供了`leaseTime`的参数来指定加锁的时间。超过这个时间后锁便自动解开了。

```java
// 加锁以后10秒钟自动解锁
// 无需调用unlock方法手动解锁
lock.lock(10, TimeUnit.SECONDS);

// 尝试加锁，失败最多等待100秒，上锁以后10秒自动解锁
boolean res = lock.tryLock(100, 10, TimeUnit.SECONDS);
if (res) {
   try {
     ...
   } finally {
       lock.unlock();
   }
}
```

Redisson同时还为分布式锁提供了异步执行的相关方法：

```java
RLock lock = redisson.getLock("anyLock");
lock.lockAsync();
lock.lockAsync(10, TimeUnit.SECONDS);
Future<Boolean> res = lock.tryLockAsync(100, 10, TimeUnit.SECONDS);
```

`RLock`对象完全符合Java的Lock规范。也就是说只有拥有锁的进程才能解锁，其他进程解锁则会抛出`IllegalMonitorStateException`错误。但是如果遇到需要其他进程也能解锁的情况，请使用[分布式信号量`Semaphore`](https://github.com/redisson/redisson/wiki/8.-分布式锁和同步器#86-信号量semaphore) 对象.

##### 公平锁（Fair Lock）

基于Redis的Redisson分布式可重入公平锁也是实现了`java.util.concurrent.locks.Lock`接口的一种`RLock`对象。同时还提供了[异步（Async）](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RLockAsync.html)、[反射式（Reactive）](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RLockReactive.html)和[RxJava2标准](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RLockRx.html)的接口。它保证了当多个Redisson客户端线程同时请求加锁时，优先分配给先发出请求的线程。所有请求线程会在一个队列中排队，当某个线程出现宕机时，Redisson会等待5秒后继续下一个线程，也就是说如果前面有5个线程都处于等待状态，那么后面的线程会等待至少25秒。

```java
RLock fairLock = redisson.getFairLock("anyLock");
// 最常见的使用方法
fairLock.lock();
```

大家都知道，如果负责储存这个分布式锁的Redis节点宕机以后，而且这个锁正好处于锁住的状态时，这个锁会出现锁死的状态。为了避免这种情况的发生，Redisson内部提供了一个监控锁的看门狗，它的作用是在Redisson实例被关闭前，不断的延长锁的有效期。默认情况下，看门狗的检查锁的超时时间是30秒钟，也可以通过修改[Config.lockWatchdogTimeout](https://github.com/redisson/redisson/wiki/2.-配置方法#lockwatchdogtimeout监控锁的看门狗超时单位毫秒)来另行指定。

另外Redisson还通过加锁的方法提供了`leaseTime`的参数来指定加锁的时间。超过这个时间后锁便自动解开了。

```java
// 10秒钟以后自动解锁
// 无需调用unlock方法手动解锁
fairLock.lock(10, TimeUnit.SECONDS);

// 尝试加锁，最多等待100秒，上锁以后10秒自动解锁
boolean res = fairLock.tryLock(100, 10, TimeUnit.SECONDS);
...
fairLock.unlock();
```

Redisson同时还为分布式可重入公平锁提供了异步执行的相关方法：

```java
RLock fairLock = redisson.getFairLock("anyLock");
fairLock.lockAsync();
fairLock.lockAsync(10, TimeUnit.SECONDS);
Future<Boolean> res = fairLock.tryLockAsync(100, 10, TimeUnit.SECONDS);
```

##### 联锁（MultiLock）

基于Redis的Redisson分布式联锁[`RedissonMultiLock`](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/RedissonMultiLock.html)对象可以将多个`RLock`对象关联为一个联锁，每个`RLock`对象实例可以来自于不同的Redisson实例。

```java
RLock lock1 = redissonInstance1.getLock("lock1");
RLock lock2 = redissonInstance2.getLock("lock2");
RLock lock3 = redissonInstance3.getLock("lock3");

RedissonMultiLock lock = new RedissonMultiLock(lock1, lock2, lock3);
// 同时加锁：lock1 lock2 lock3
// 所有的锁都上锁成功才算成功。
lock.lock();
...
lock.unlock();
```

大家都知道，如果负责储存某些分布式锁的某些Redis节点宕机以后，而且这些锁正好处于锁住的状态时，这些锁会出现锁死的状态。为了避免这种情况的发生，Redisson内部提供了一个监控锁的看门狗，它的作用是在Redisson实例被关闭前，不断的延长锁的有效期。默认情况下，看门狗的检查锁的超时时间是30秒钟，也可以通过修改[Config.lockWatchdogTimeout](https://github.com/redisson/redisson/wiki/2.-配置方法#lockwatchdogtimeout监控锁的看门狗超时单位毫秒)来另行指定。

另外Redisson还通过加锁的方法提供了`leaseTime`的参数来指定加锁的时间。超过这个时间后锁便自动解开了。

```java
RedissonMultiLock lock = new RedissonMultiLock(lock1, lock2, lock3);
// 给lock1，lock2，lock3加锁，如果没有手动解开的话，10秒钟后将会自动解开
lock.lock(10, TimeUnit.SECONDS);

// 为加锁等待100秒时间，并在加锁成功10秒钟后自动解开
boolean res = lock.tryLock(100, 10, TimeUnit.SECONDS);
...
lock.unlock();
```

##### 红锁（RedLock）

基于Redis的Redisson红锁`RedissonRedLock`对象实现了[Redlock](http://redis.cn/topics/distlock.html)介绍的加锁算法。该对象也可以用来将多个`RLock`对象关联为一个红锁，每个`RLock`对象实例可以来自于不同的Redisson实例。

```java
RLock lock1 = redissonInstance1.getLock("lock1");
RLock lock2 = redissonInstance2.getLock("lock2");
RLock lock3 = redissonInstance3.getLock("lock3");

RedissonRedLock lock = new RedissonRedLock(lock1, lock2, lock3);
// 同时加锁：lock1 lock2 lock3
// 红锁在大部分节点上加锁成功就算成功。
lock.lock();
...
lock.unlock();
```

大家都知道，如果负责储存某些分布式锁的某些Redis节点宕机以后，而且这些锁正好处于锁住的状态时，这些锁会出现锁死的状态。为了避免这种情况的发生，Redisson内部提供了一个监控锁的看门狗，它的作用是在Redisson实例被关闭前，不断的延长锁的有效期。默认情况下，看门狗的检查锁的超时时间是30秒钟，也可以通过修改[Config.lockWatchdogTimeout](https://github.com/redisson/redisson/wiki/2.-配置方法#lockwatchdogtimeout监控锁的看门狗超时单位毫秒)来另行指定。

另外Redisson还通过加锁的方法提供了`leaseTime`的参数来指定加锁的时间。超过这个时间后锁便自动解开了。

```java
RedissonRedLock lock = new RedissonRedLock(lock1, lock2, lock3);
// 给lock1，lock2，lock3加锁，如果没有手动解开的话，10秒钟后将会自动解开
lock.lock(10, TimeUnit.SECONDS);

// 为加锁等待100秒时间，并在加锁成功10秒钟后自动解开
boolean res = lock.tryLock(100, 10, TimeUnit.SECONDS);
...
lock.unlock();
```

##### 读写锁（ReadWriteLock）

对比 JUC 中的读写锁：[ReadWriteLock](https://chanservy.vercel.app/posts/20211124/java-concurrent-6.html#%E8%AF%BB%E5%86%99%E9%94%81)

基于Redis的Redisson分布式可重入读写锁[`RReadWriteLock`](http://static.javadoc.io/org.redisson/redisson/3.4.3/org/redisson/api/RReadWriteLock.html) Java对象实现了`java.util.concurrent.locks.ReadWriteLock`接口。其中读锁和写锁都继承了[RLock](https://github.com/redisson/redisson/wiki/8.-分布式锁和同步器#81-可重入锁reentrant-lock)接口。

分布式可重入读写锁允许同时有多个读锁和一个写锁处于加锁状态。

```java
RReadWriteLock rwlock = redisson.getReadWriteLock("anyRWLock");
// 最常见的使用方法
rwlock.readLock().lock();
// 或
rwlock.writeLock().lock();
```

大家都知道，如果负责储存这个分布式锁的Redis节点宕机以后，而且这个锁正好处于锁住的状态时，这个锁会出现锁死的状态。为了避免这种情况的发生，Redisson内部提供了一个监控锁的看门狗，它的作用是在Redisson实例被关闭前，不断的延长锁的有效期。默认情况下，看门狗的检查锁的超时时间是30秒钟，也可以通过修改[Config.lockWatchdogTimeout](https://github.com/redisson/redisson/wiki/2.-配置方法#lockwatchdogtimeout监控锁的看门狗超时单位毫秒)来另行指定。

另外Redisson还通过加锁的方法提供了`leaseTime`的参数来指定加锁的时间。超过这个时间后锁便自动解开了。

```java
// 10秒钟以后自动解锁
// 无需调用unlock方法手动解锁
rwlock.readLock().lock(10, TimeUnit.SECONDS);
// 或
rwlock.writeLock().lock(10, TimeUnit.SECONDS);

// 尝试加锁，最多等待100秒，上锁以后10秒自动解锁
boolean res = rwlock.readLock().tryLock(100, 10, TimeUnit.SECONDS);
// 或
boolean res = rwlock.writeLock().tryLock(100, 10, TimeUnit.SECONDS);
...
lock.unlock();
```

案例：

```java
/**
 * 读写锁的用法：改数据加写锁，读数据加读锁
 */
@GetMapping("/read")
@ResponseBody
public String read() {
    RReadWriteLock lock = redissonClient.getReadWriteLock("rw-lock");
    RLock rLock = lock.readLock();
    String s = "";
    rLock.lock();
    try {
        System.out.println("读锁加锁" + Thread.currentThread().getId());
        Thread.sleep(5000);
        s = redisTemplate.opsForValue().get("writeValue");
    } catch (InterruptedException e) {
        e.printStackTrace();
    } finally {
        rLock.unlock();
    }
    return "读取完成:" + s;
}

@GetMapping("/write")
@ResponseBody
public String write() {
    String s = "";
    RReadWriteLock lock = redissonClient.getReadWriteLock("rw-lock");
    RLock wLock = lock.writeLock();
    wLock.lock();
    try {
        System.out.println("写锁加锁" + Thread.currentThread().getId());
        s = UUID.randomUUID().toString();
        Thread.sleep(10000);
        redisTemplate.opsForValue().set("writeValue", s);
    } catch (InterruptedException e) {
        e.printStackTrace();
    } finally {
        wLock.unlock();
    }
    return "写入完成:" + s;
}
```

注：写锁是一个排他锁（互斥锁、独占锁）；读锁是一个共享锁。

- 读 + 读：相当于无锁，并发读，只会在 redis 中记录好所有当前的读锁，它们都会同时加锁成功。
- 写 + 读：获取读锁要等待写锁释放。
- 写 + 写：阻塞、互斥。
- 读 + 写：获取写锁要等待读锁释放。

##### 信号量（Semaphore）

对比 JUC 中的信号量：[Semaphore](https://chanservy.vercel.app/posts/20211124/java-concurrent-6.html#Semaphore)

基于Redis的Redisson的分布式信号量（[Semaphore](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RSemaphore.html)）Java对象`RSemaphore`采用了与`java.util.concurrent.Semaphore`相似的接口和用法。同时还提供了[异步（Async）](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RSemaphoreAsync.html)、[反射式（Reactive）](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RSemaphoreReactive.html)和[RxJava2标准](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RSemaphoreRx.html)的接口。

```java
RSemaphore semaphore = redisson.getSemaphore("semaphore");
semaphore.acquire();
//或
semaphore.acquireAsync();
semaphore.acquire(23);
semaphore.tryAcquire();
//或
semaphore.tryAcquireAsync();
semaphore.tryAcquire(23, TimeUnit.SECONDS);
//或
semaphore.tryAcquireAsync(23, TimeUnit.SECONDS);
semaphore.release(10);
semaphore.release();
//或
semaphore.releaseAsync();
```

案例：车库停车，一共 3 个车位。

模拟，手动将名为 park 的信号量在 redis 中放入一份，key 为 park，类型为 String，车位有三个，value 设为 3。

```java
@GetMapping("/park")
@ResponseBody
public String park() {
    RSemaphore park = redissonClient.getSemaphore("park");
    try {
        park.acquire();// 获取一个信号，占一个车位
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return "ok";
}

@GetMapping("/go")
@ResponseBody
public String go() {
    RSemaphore park = redissonClient.getSemaphore("park");
    park.release();// 释放一个信号，腾出来一个车位
    return "ok";
}
```

上面代码中使用的是`park.acquire();`，当三个车位全都占满，第四台车来了就得等着，一直等到有释放的车位，再将车停进去。如果使用`park.tryAcquire();`时，就是停车的时候车主瞅一眼，有空车位就停，没空车位不等直接溜了。就是尝试获取一下，不行就算了。

这个 Redisson 的分布式信号量也可以做分布式限流，和车库停车一个思想。

##### 可过期性信号量（PermitExpirableSemaphore）

基于Redis的Redisson可过期性信号量（[PermitExpirableSemaphore](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RPermitExpirableSemaphore.html)）是在`RSemaphore`对象的基础上，为每个信号增加了一个过期时间。每个信号可以通过独立的ID来辨识，释放时只能通过提交这个ID才能释放。它提供了[异步（Async）](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RPermitExpirableSemaphoreAsync.html)、[反射式（Reactive）](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RPermitExpirableSemaphoreReactive.html)和[RxJava2标准](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RPermitExpirableSemaphoreRx.html)的接口。

```java
RPermitExpirableSemaphore semaphore = redisson.getPermitExpirableSemaphore("mySemaphore");
String permitId = semaphore.acquire();
// 获取一个信号，有效期只有2秒钟。
String permitId = semaphore.acquire(2, TimeUnit.SECONDS);
// ...
semaphore.release(permitId);
```

##### 闭锁（CountDownLatch）

基于Redisson的Redisson分布式闭锁（[CountDownLatch](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RCountDownLatch.html)）Java对象`RCountDownLatch`采用了与`java.util.concurrent.CountDownLatch`相似的接口和用法。

用法：在多线程任务调度的时候使用 CountDownLatch，比如有十个线程，这十个线程都给我把任务做完了，那才算全部完成，才能往下继续执行代码，否则要等待。

场景：英雄联盟十位玩家全部加载完成才能进入游戏。

Redisson 的 CountDownLatch 和 JUC 中的思想是一样的，只不过一个是单体应用中，一个是分布式场景下。

对比 JUC 中的 CountDownLatch：[CountDownLatch](https://chanservy.vercel.app/posts/20211124/java-concurrent-6.html#CountdownLatch)

```java
RCountDownLatch latch = redisson.getCountDownLatch("anyCountDownLatch");
latch.trySetCount(10);
latch.await();

// 在其他线程或其他JVM里
RCountDownLatch latch = redisson.getCountDownLatch("anyCountDownLatch");
latch.countDown();
```

案例：

学校一共 5 个班，放学，锁门，学校的门卫要等 5 个班全部走完，才可以锁大门。

```java
@GetMapping("/lockDoor")
@ResponseBody
public String lockDoor() {
    RCountDownLatch latch = redissonClient.getCountDownLatch("CountDownLatch");
    try {
        latch.trySetCount(5);
        latch.await();// 等待闭锁都完成
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return "锁大门！";
}

@GetMapping("/leaveSchool/{id}")
@ResponseBody
public String leaveSchool(@PathVariable("id") Long id) {
    RCountDownLatch latch = redissonClient.getCountDownLatch("CountDownLatch");
    latch.countDown();// 计数-1
    return id + "班的人都走了。";
}
```



#### 总结

1. 不可重入 Redis 分布式锁：
   - 原理：利用 setnx 的互斥性；利用 ex 避免死锁；释放锁时判断线程标示
   - 缺陷：不可重入、无法重试、锁超时失效
2. 可重入的 Redis 分布式锁——Redisson：
   - 原理：利用 hash 结构，记录线程标示和重入次数；利用 watchDog 延续锁时间；利用信号量控制锁重试等待
   - 缺陷：redis 宕机引起锁失效问题
3. Redisson 的 multiLock：
   - 原理：多个独立的 Redis 节点，必须在所有节点都获取重入锁，才算获取锁成功
   - 缺陷：运维成本高、实现复杂

#### Redis优化秒杀

前面几种方案，在判断完有购买资格，然后需要操作数据库扣减库存并且将订单数据入库之后，才会返回结果，这一系列的操作都是同步的，也就是说，最终是在一系列对数据库的操作之后才会返回结果给前端，进而才会给客户响应，这样的耗时可能很长。如果我们让数据库的一系列操作异步去执行，也就是在判断完购买资格之后就返回结果，然后扣减库存和订单信息入库操作异步去执行，那样用户体验感会更佳。

并且前面我们是使用了锁，在同步代码块中判断“一人一单”的逻辑，然后进行扣减库存最后释放锁，这样加锁的操作固然能避免线程安全问题，但是也同样降低了效率。我们这次将这块逻辑放在一个 lua 脚本中完成，redis 在调用脚本运行的时候，是以脚本为单位的，也就是说整个脚本中的逻辑可以保证原子性，并且可以保证线程安全。

**Redis 调用 Lua 脚本的执行方式：**

`Redis 可以保证以原子方式执行脚本：执行脚本时不会执行其他脚本或 Redis 命令。类似于给 lua 脚本中的这段代码加了锁` ，可能 redis 内部实现和我们在程序中加锁会有一定的差异，反正大致意思就是这样。

`Redis 使用同一个 Lua 解释器来执行所有命令，同时，Redis 保证以一种原子性的方式来执行脚本：当 lua 脚本在执行的时候，不会有其他脚本和命令同时执行，这种语义类似于 MULTI（互斥）/EXEC。`从别的客户端的视角来看，一个 lua 脚本要么不可见，要么已经执行完。

##### 优化方式一：

Lua 脚本、阻塞队列 + 线程池 + 任务。

在 lua 脚本中判断并扣减库存、判断一人一单。lua 判断后有资格，才会异步去数据库中扣减库存、订单信息入库。前提是，添加优惠券信息的时候进行缓存预热操作，将优惠券的初始库存放入缓存中，使用 String 类型的 value 就可以，因为还需要在脚本中进行一人一单的判断，我们将每次在判断完有资格的用户 id 存入缓存中，因为一人一单，所以对一个优惠券下单的用户 id 只能有一个，因此不能重复，我们使用 redis 的 set 类型的 value 。脚本代码如下： （注：不要忘记库存的数据预热，否则在执行脚本代码时会出现空指针异常！）

```lua
--- 参数列表
--- 优惠券id
local voucherId = ARGV[1];
--- 用户id
local userId = ARGV[2];
--- 数据key
--- 库存key：id为voucherId的优惠券库存，注意lua脚本语言中拼接字符串用..
local stockKey = "seckill:stock:"..voucherId;
--- 订单key：id为voucherId的优惠券被哪些userId下了订单
local orderKey = "seckill:order:"..voucherId;
--- 脚本业务
--- 判断库存是否充足，拿到的库存值是一个string，lua中的tonumber方法转成数字。
if (tonumber(redis.call('get', stockKey)) <= 0) then
    --- 库存不足，返回1
    return 1;
end
--- 判断用户是否下过单
if (redis.call('sismember', orderKey, userId) == 1) then
    --- 存在，说明是重复下单，返回2
    return 2;
end
--- 符合下单条件
--- 扣减库存
redis.call('incrby', stockKey, -1);
--- 保存下单的用户id
redis.call('sadd', orderKey, userId)
return 0;
```

业务代码改造：

```java
@Service
@Slf4j
public class VoucherOrderServiceImpl extends ServiceImpl<VoucherOrderMapper, VoucherOrder> implements IVoucherOrderService {

    @Resource
    private SeckillVoucherMapper seckillVoucherMapper;

    @Resource
    private VoucherOrderMapper voucherOrderMapper;

    @Resource
    private RedisIdWorker idWorker;

    @Resource
    private StringRedisTemplate stringRedisTemplate;

    @Resource
    private RedissonClient redissonClient;

    // 加载lua脚本
    private static final DefaultRedisScript<Long> SECKILL_SCRIPT;

    static {
        SECKILL_SCRIPT = new DefaultRedisScript<>();
        SECKILL_SCRIPT.setLocation(new ClassPathResource("secondKill.lua"));
        SECKILL_SCRIPT.setResultType(Long.class);
    }

    // 创建阻塞队列
    private final BlockingQueue<VoucherOrder> orderTasksQueue = new ArrayBlockingQueue<>(1024 * 1024);
    // 单线程线程池
    private static final ExecutorService SECKILL_ORDER_POOL = Executors.newSingleThreadExecutor();
    // 创建任务
    private class VoucherOrderHandler implements Runnable {
        @Override
        public void run() {
            for (; ; ) {
                try {
                    // 获取队列中的订单信息
                    // take方法，获取和删除该队列的头部，如果队列中为空，则阻塞等待；也就是说队列中没有元素会卡在这里，有元素才会继续，不用担心死循环浪费CPU
                    VoucherOrder voucherOrder = orderTasksQueue.take();
                    // 创建订单
                    createVoucherOrder(voucherOrder);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                    log.error("处理订单异常", e);
                    return;
                }
            }
        }
    }

    // 项目一启动，用户随时都可能会来秒杀，因此我们要让类一初始化完就来执行这个任务
    @PostConstruct//在当前类初始化完毕以后就来执行
    private void init() {
        SECKILL_ORDER_POOL.submit(new VoucherOrderHandler());
    }

    /**
     * 秒杀优惠券
     *
     * @param voucherId 优惠券id
     * @return Result
     */
    @Override
    // @Transactional
    public Result seckill(Long voucherId) {
        //根据id查询秒杀优惠券信息
        SeckillVoucher seckillVoucher = seckillVoucherMapper.selectById(voucherId);
        //判断秒杀活动是否已经开始
        LocalDateTime beginTime = seckillVoucher.getBeginTime();
        if (LocalDateTime.now().isBefore(beginTime)) {
            return Result.fail("秒杀活动未开始！");
        }
        //判断秒杀活动是否已经结束
        LocalDateTime endTime = seckillVoucher.getEndTime();
        if (LocalDateTime.now().isAfter(endTime)) {
            return Result.fail("秒杀活动已经结束！");
        }
        //判断是否还有库存
        Integer stock = seckillVoucher.getStock();
        if (stock < 1) {
            return Result.fail("库存不足！");
        }
        
        // 使用redis调用lua脚本的方式实现下单业务
        Long userId = UserHolder.getUser().getId();
        // 执行lua脚本
        Long result = stringRedisTemplate.execute(
                SECKILL_SCRIPT,
                Collections.emptyList(),
                voucherId.toString(),
                userId.toString()
        );
        // 判断结果是否为0
        assert result != null;
        if (result.intValue() != 0) {
            // 不为0，根据lua脚本中我们自己写的逻辑，代表没有购买资格
            return Result.fail(result.intValue() == 1 ? "库存不足！" : "不能重复下单！");
        }
        // 为0了，有购买资格，把下订单信息保存到阻塞队列中
        VoucherOrder voucherOrder = new VoucherOrder();
        long orderId = idWorker.nextId("order");
        voucherOrder.setId(orderId);
        voucherOrder.setVoucherId(voucherId);
        voucherOrder.setUserId(userId);

        // 放入阻塞队列，由线程池异步处理减库存和入库任务
        orderTasksQueue.add(voucherOrder);

        // 返回
        return Result.ok(orderId);
    }
    @Transactional
    public void createVoucherOrder(VoucherOrder voucherOrder) {
        Long voucherId = voucherOrder.getVoucherId();
        // 这块不用加锁，因为在lua脚本判断过“一人一单”，并且判断过“库存充足”，库存充足且符合一人一单，程序才会跑到这里
        // 扣减库存
        UpdateWrapper<SeckillVoucher> updateWrapper = new UpdateWrapper<>();
        updateWrapper.setSql("stock = stock - 1");
        updateWrapper.eq("voucher_id", voucherId);
        updateWrapper.gt("stock", 0);
        seckillVoucherMapper.update(null, updateWrapper);
        // 订单入数据库
        voucherOrderMapper.insert(voucherOrder);
    }
}
```

但是这样实现，放入阻塞队列中，如果订单的数据量过大，那么会对 JVM 的内存造成很大的压力；另外如果这个服务宕机重启，那么阻塞队列中的数据就丢失了。如果我们的数据量如果在可控范围，并且 JVM 的内存参数设置加以考量，这种方案也是可行的，因为这块业务是对代金券的秒杀，因此总数量也没必要太大，但是如果是对于某个热点商品的特价秒杀，结果就可想而知了，因此基于阻塞队列这种异步处理的思想，我们引入 RabbitMQ 消息队列。

##### 优化方式二：

Lua 脚本、RabbitMQ 消息队列。

在 lua 脚本中判断并扣减库存、判断一人一单，解释如上。

在原基础上：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

```yml
  # rabbitmq配置
spring:
  rabbitmq:
    host: 127.0.0.1
    username: root
    password: 123456
    virtual-host: hmdp
    publisher-confirms: true # 开启发送确认
    publisher-returns: true # 开启发送失败退回
    listener:
      simple:
        acknowledge-mode: manual # 全局开启ACK
```

```java
package com.hmdp.utils;

/**
 * @author CHAN
 * @since 2022/4/9
 */
public class RabbitMQConstants {
    // 异步创建订单，消息队列
    public static final String ASYNC_ORDER_QUEUE = "create.order.queue";
    // 异步创建订单，消息交换机
    public static final String ASYNC_ORDER_EXCHANGE = "create.order.exchange";
    // 异步创建订单，绑定标识
    public static final String ASYNC_CREATE_ORDER_KEY = "update.stock.create.order";
}
```

 异步监听消息队列中的消息：

```java
/**
 * 异步监听消息队列中的消息
 * @author CHAN
 * @since 2022/4/9
 */
@Component
@Slf4j
public class CreateOrderListener {

    @Resource
    private SeckillVoucherMapper seckillVoucherMapper;

    @Resource
    private VoucherOrderMapper voucherOrderMapper;

    @RabbitListener(
            bindings = @QueueBinding(
                    value = @Queue(value = ASYNC_ORDER_QUEUE, durable = "true"),
                    exchange = @Exchange(
                            value = ASYNC_ORDER_EXCHANGE,
                            ignoreDeclarationExceptions = "true"
                            // type = ExchangeTypes.DIRECT//默认就是direct类型
                    ),
                    key = ASYNC_CREATE_ORDER_KEY
            )
    )
    public void listen(VoucherOrder voucherOrder, Channel channel, Message message) throws IOException {
        try {
            /*
             * 无异常就确认消息
             * basicAck(long deliveryTag, boolean multiple)
             * deliveryTag:取出来当前消息在队列中的的索引;
             * multiple:为true的话就是批量确认,如果当前deliveryTag为5,那么就会确认
             * deliveryTag为5及其以下的消息;一般设置为false
             */
            channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
            log.info("接收信息成功，开始处理业务！");
            Long voucherId = voucherOrder.getVoucherId();
            log.debug("voucherId: {}, voucherOrder: {}", voucherId, voucherOrder);
            // 这块不用加锁，因为在lua脚本判断过“一人一单”，并且判断过“库存充足”，库存充足且符合一人一单，程序才会跑到这里

            // 扣减库存
            UpdateWrapper<SeckillVoucher> updateWrapper = new UpdateWrapper<>();
            updateWrapper.setSql("stock = stock - 1");
            updateWrapper.eq("voucher_id", voucherId);
            updateWrapper.gt("stock", 0);
            seckillVoucherMapper.update(null, updateWrapper);

            // 订单入数据库
            voucherOrderMapper.insert(voucherOrder);

        } catch (Exception e) {
            e.printStackTrace();
            /*
             * 有异常就拒收消息
             * basicNack(long deliveryTag, boolean multiple, boolean requeue)
             * requeue:true:将消息重返当前消息队列,还可以重新发送给消费者;
             *         false:将消息丢弃
             */
            channel.basicNack(message.getMessageProperties().getDeliveryTag(), false, true);
            log.info("接收信息异常！");
        }
    }
}
```

业务代码优化：

```java
@Service
@Slf4j
public class VoucherOrderServiceImpl extends ServiceImpl<VoucherOrderMapper, VoucherOrder> implements IVoucherOrderService {

    @Resource
    private SeckillVoucherMapper seckillVoucherMapper;

    @Resource
    private VoucherOrderMapper voucherOrderMapper;

    @Resource
    private RedisIdWorker idWorker;

    @Resource
    private StringRedisTemplate stringRedisTemplate;

    @Resource
    private RedissonClient redissonClient;

    @Resource
    private RabbitTemplate rabbitTemplate;


    // 加载lua脚本
    private static final DefaultRedisScript<Long> SECKILL_SCRIPT;

    static {
        SECKILL_SCRIPT = new DefaultRedisScript<>();
        SECKILL_SCRIPT.setLocation(new ClassPathResource("secondKill.lua"));
        SECKILL_SCRIPT.setResultType(Long.class);
    }
    
    /**
     * 秒杀优惠券
     *
     * @param voucherId 优惠券id
     * @return Result
     */
    @Override
    // @Transactional
    public Result seckill(Long voucherId) {
        //根据id查询秒杀优惠券信息
        SeckillVoucher seckillVoucher = seckillVoucherMapper.selectById(voucherId);
        //判断秒杀活动是否已经开始
        LocalDateTime beginTime = seckillVoucher.getBeginTime();
        if (LocalDateTime.now().isBefore(beginTime)) {
            return Result.fail("秒杀活动未开始！");
        }
        //判断秒杀活动是否已经结束
        LocalDateTime endTime = seckillVoucher.getEndTime();
        if (LocalDateTime.now().isAfter(endTime)) {
            return Result.fail("秒杀活动已经结束！");
        }
        //判断是否还有库存
        Integer stock = seckillVoucher.getStock();
        if (stock < 1) {
            return Result.fail("库存不足！");
        }

        // 使用redis调用lua脚本的方式实现下单业务
        Long userId = UserHolder.getUser().getId();
        // 执行lua脚本
        Long result = stringRedisTemplate.execute(
                SECKILL_SCRIPT,
                Collections.emptyList(),
                voucherId.toString(),
                userId.toString()
        );
        // 判断结果是否为0
        assert result != null;
        if (result.intValue() != 0) {
            // 不为0，根据lua脚本中我们自己写的逻辑，代表没有购买资格
            return Result.fail(result.intValue() == 1 ? "库存不足！" : "不能重复下单！");
        }
        // 为0了，有购买资格，把下订单信息保存到队列中
        VoucherOrder voucherOrder = new VoucherOrder();
        long orderId = idWorker.nextId("order");
        voucherOrder.setId(orderId);
        voucherOrder.setVoucherId(voucherId);
        voucherOrder.setUserId(userId);

        // 放入阻塞队列中，如果数据量过大，那么会对JVM的内存造成很大的压力；另外如果这个服务宕机重启，那么阻塞队列中的数据就丢失了
        // 因此基于阻塞队列这种异步处理的思想，我们引入RabbitMQ消息队列
        log.info("send voucherOrder message: {}", voucherOrder);
        // 生产者发送消息到exchange后没有绑定的queue时将消息退回
        rabbitTemplate.setReturnCallback((message, replyCode, replyText, exchange, routingKey) ->
                log.info("发送优惠券订单消息被退回！exchange: {}, routingKey: {}", exchange, routingKey));
        // 生产者发送消息confirm检测
        this.rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> {
            if (!ack) {
                log.info("消息发送失败！cause：{}，correlationData：{}", cause, correlationData);
            } else {
                log.info("消息发送成功！");
            }
        });
        rabbitTemplate.convertAndSend(ASYNC_ORDER_EXCHANGE, ASYNC_CREATE_ORDER_KEY, voucherOrder);

        // 返回
        return Result.ok(orderId);
    }
}
```

> 注：不推荐在程序中大量使用 Lua 脚本，Lua 脚本如果很多就不方便管理，我们一般使用 Lua 脚本来实现高并发的业务流程，类似上面的对于优惠券的秒杀业务。Over！

## Feed流实现方案

场景：

当我们关注了用户后，这个用户发了动态，那么我们应该把这些数据推送给用户，这个需求，其实我们又把他叫做Feed流，关注推送也叫做Feed流，直译为投喂。为用户持续的提供“沉浸式”的体验，通过无限下拉刷新获取新的信息。

对于传统的模式的内容解锁：我们是需要用户去通过搜索引擎或者是其他的方式去解锁想要看的内容

![1653808641260](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/1653808641260.png)



对于新型的Feed流的的效果：不需要我们用户再去推送信息，而是系统分析用户到底想要什么，然后直接把内容推送给用户，从而使用户能够更加的节约时间，不用主动去寻找。

![1653808993693](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/1653808993693.png)

Feed流的实现有两种模式：

Feed流产品有两种常见模式：

Timeline：不做内容筛选，简单的按照内容发布时间排序，常用于好友或关注。例如朋友圈

* 优点：信息全面，不会有缺失。并且实现也相对简单
* 缺点：信息噪音较多，用户不一定感兴趣，内容获取效率低

智能排序：利用智能算法屏蔽掉违规的、用户不感兴趣的内容。推送用户感兴趣信息来吸引用户。例如抖音。

* 优点：投喂用户感兴趣信息，用户粘度很高，容易沉迷
* 缺点：如果算法不精准，可能起到反作用

针对好友的操作，基于关注的好友来做Feed流，采用的就是Timeline的方式，只需要拿到我们关注的用户发送的动态，然后按照时间排序即可。该模式的实现方案有三种：

* 拉模式
* 推模式
* 推拉结合

**拉模式**：也叫做读扩散

该模式的核心含义就是：当张三和李四和王五发了消息后，都会保存在自己的邮箱中，假设赵六要读取信息，那么他会从读取他自己的收件箱，此时系统会从他关注的人群中，把他关注人的信息全部都进行拉取，然后在进行排序

优点：比较节约空间，因为赵六在读信息时，并没有重复读取，而且读取完之后可以把他的收件箱进行清楚。

缺点：比较延迟，当用户读取数据时才去关注的人里边去读取数据，假设用户关注了大量的用户，那么此时就会拉取海量的内容，对服务器压力巨大。

![1653809450816](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/1653809450816.png)



**推模式**：也叫做写扩散。

推模式是没有写邮箱的，当张三写了一个内容，此时会主动的把张三写的内容发送到他的粉丝收件箱中去，假设此时李四再来读取，就不用再去临时拉取了

优点：时效快，不用临时拉取

缺点：内存压力大，假设一个大V写信息，很多人关注他， 就会写很多分数据到粉丝那边去

![1653809875208](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/1653809875208.png)

**推拉结合模式**：也叫做读写混合，兼具推和拉两种模式的优点。

推拉模式是一个折中的方案，站在发件人这一段，如果是个普通的人，那么我们采用写扩散的方式，直接把数据写入到他的粉丝中去，因为普通的人他的粉丝关注量比较小，所以这样做没有压力，如果是大V，那么他是直接将数据先写入到一份到发件箱里边去，然后再直接写一份到活跃粉丝收件箱里边去，现在站在收件人这端来看，如果是活跃粉丝，那么大V和普通的人发的都会直接写入到自己收件箱里边来，而如果是普通的粉丝，由于他们上线不是很频繁，所以等他们上线时，再从发件箱里边去拉信息。

![1653812346852](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/1653812346852.png)

我关注的好友们，他们发送动态的时候，会保存数据到数据库的表中，并且推送到我的“收件箱”。收件箱满足可以根据时间戳排序，所谓的收件箱，其实就是Redis，必须用Redis的数据结构实现。查询Redis中保存的数据时，可以实现分页查询。

Feed流中的数据会不断更新，所以数据的角标也在变化，因此不能采用传统的分页模式。

传统的分页在feed流是不适用的，因为我们的数据会随时发生变化

假设在t1 时刻，我们去读取第一页，此时page = 1 ，size = 5 ，那么我们拿到的就是10~6 这几条记录，假设现在t2时候又发布了一条记录，此时t3 时刻，我们来读取第二页，读取第二页传入的参数是page=2 ，size=5 ，那么此时读取到的第二页实际上是从6 开始，然后是6~2 ，那么我们就读取到了重复的数据，所以feed流的分页，不能采用原始方案来做。

![1653813047671](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/1653813047671.png)

Feed流的滚动分页

我们需要记录每次操作的最后一条，然后从这个位置开始去读取数据

举个例子：我们从t1时刻开始，拿第一页数据，拿到了10~6，然后记录下当前最后一次拿取的记录，就是6，t2时刻发布了新的记录，此时这个11放到最顶上，但是不会影响我们之前记录的6，此时t3时刻来拿第二页，第二页这个时候拿数据，还是从6后一点的5去拿，就拿到了5-1的记录。我们这个地方可以采用sortedSet来做，可以进行范围查询，并且还可以记录当前获取数据时间戳最小值，就可以实现滚动分页了

![1653813462834](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/1653813462834.png)

核心的意思：就是在保存完当前用户发布的动态后，获得到当前用户的粉丝，然后把数据推送到redis（其他用户的id作为key，其实当前用户的粉丝也就是其他用户）中去。

## 附近的商户

Redis 的 GeoHash 的应用。

GEO就是Geolocation的简写形式，代表地理坐标。Redis在3.2版本中加入了对GEO的支持，允许存储地理坐标信息，帮助我们根据经纬度来检索数据。常见的命令有：

* GEOADD：添加一个地理空间信息，包含：经度（longitude）、纬度（latitude）、值（member）
* GEODIST：计算指定的两个点之间的距离并返回
* GEOHASH：将指定member的坐标转为hash字符串形式并返回
* GEOPOS：返回指定member的坐标
* GEORADIUS：指定圆心、半径，找到该圆内包含的所有member，并按照与圆心之间的距离排序后返回。6.以后已废弃
* GEOSEARCH：在指定范围内搜索member，并按照与指定点之间的距离排序后返回。范围可以是圆形或矩形。6.2.新功能
* GEOSEARCHSTORE：与GEOSEARCH功能一致，不过可以把结果存储到一个指定的key。 6.2.新功能

比如：

当我们点击美食之后，会出现一系列的商家，商家中可以按照多种排序方式，我们此时关注的是距离，这个地方就需要使用到我们的GEO，向后台传入当前app收集的地址(我们此处是写死的) ，以当前坐标作为圆心，同时绑定相同的店家类型type，以及分页信息，把这几个条件传入后台，后台查询出对应的数据再返回。

![1653822021827](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/1653822021827.png)

思路：

我们要做的事情是：将数据库表中的数据导入到redis中去，redis中的GEO，GEO在redis中就一个menber和一个经纬度，我们把x和y轴传入到redis做的经纬度位置去，但我们不能把所有的数据都放入到menber中去，毕竟作为redis是一个内存级数据库，如果存海量数据，redis还是力不从心，所以我们在这个地方存储他的id即可。

但是这个时候还有一个问题，就是在redis中并没有存储type，所以我们无法根据type来对数据进行筛选，所以我们可以按照商户类型做分组，类型相同的商户作为同一组，以typeId为key存入同一个GEO集合中即可。

## UV 统计

Redis 的 HyperLogLog 的统计功能。

### UV统计-HyperLogLog

首先我们搞懂两个概念：

* UV：全称Unique Visitor，也叫独立访客量，是指通过互联网访问、浏览这个网页的自然人。1天内同一个用户多次访问该网站，只记录1次。
* PV：全称Page View，也叫页面访问量或点击量，用户每访问网站的一个页面，记录1次PV，用户多次打开页面，则记录多次PV。往往用来衡量网站的流量。

通常来说PV会比UV大很多，所以衡量同一个网站的访问量，我们需要综合考虑很多因素，所以我们只是单纯的把这两个值作为一个参考值

UV统计在服务端做会比较麻烦，因为要判断该用户是否已经统计过了，需要将统计过的用户信息保存。但是如果每个访问的用户都保存到Redis中，数据量会非常恐怖，那怎么处理呢？

Hyperloglog(HLL)是从Loglog算法派生的概率算法，用于确定非常大的集合的基数，而不需要存储其所有值。相关算法原理大家可以参考：https://juejin.cn/post/6844903785744056333#heading-0
Redis中的HLL是基于string结构实现的，单个HLL的内存**永远小于16kb**，**内存占用低**的令人发指！作为代价，其测量结果是概率性的，**有小于0.81％的误差**。不过对于UV统计来说，这完全可以忽略。

![1653837988985](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/1653837988985.png)

### UV统计-测试百万数据的统计

测试思路：我们直接利用单元测试，向HyperLogLog中添加100万条数据，看看内存占用和统计效果如何

![1653838053608](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/1653838053608.png)

经过测试：我们会发生他的误差是在允许范围内，并且内存占用极小。

## 用户签到

Redis 的 BitMap 数据统计功能。

### BitMap功能演示

我们针对签到功能完全可以通过mysql来完成，比如说以下这张表

![1653823145495](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/1653823145495.png)

用户一次签到，就是一条记录，假如有1000万用户，平均每人每年签到次数为10次，则这张表一年的数据量为 1亿条，显然不可取。

每签到一次需要使用（8 + 8 + 1 + 1 + 3 + 1）共22 字节的内存，一个月则最多需要600多字节，不可取。

我们如何能够简化一点呢？其实可以考虑小时候一个挺常见的方案，就是小时候，咱们准备一张小小的卡片，你只要签到就打上一个勾，我最后判断你是否签到，其实只需要到小卡片上看一看就知道了，我们可以采用类似这样的方案来实现我们的签到需求。

我们按月来统计用户签到信息，签到记录为1，未签到则记录为0.

把每一个bit位对应当月的每一天，形成了映射关系。用0和1标示业务状态，这种思路就称为位图（BitMap）。这样我们就用极小的空间，来实现了大量数据的表示，Redis中是利用string类型数据结构实现BitMap，因此最大上限是512M，转换为bit则是 2^32个bit位。

![1653824498278](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/1653824498278.png)

BitMap的操作命令有：

* SETBIT：向指定位置（offset）存入一个0或1
* GETBIT ：获取指定位置（offset）的bit值
* BITCOUNT ：统计BitMap中值为1的bit位的数量
* BITFIELD ：操作（查询、修改、自增）BitMap中bit数组中的指定位置（offset）的值
* BITFIELD_RO ：获取BitMap中bit数组，并以十进制形式返回
* BITOP ：将多个BitMap的结果做位运算（与 、或、异或）
* BITPOS ：查找bit数组中指定范围内第一个0或1出现的位置

### 签到功能

实现签到功能思路：我们可以把年和月作为bitMap的key，然后保存到一个bitMap中，每次签到就到对应的位上把数字从0变成1，只要对应是1，就表明说明这一天已经签到了，反之则没有签到。

因为 Redis 中的 BitMap 底层是基于 Redis 中 String 的数据结构，因此其操作也都封装在字符串相关的操作中了。

案例代码：

```java
@Override
public Result sign() {
    // 1.获取当前登录用户
    Long userId = UserHolder.getUser().getId();
    // 2.获取日期
    LocalDateTime now = LocalDateTime.now();
    // 3.拼接key
    String keySuffix = now.format(DateTimeFormatter.ofPattern(":yyyyMM"));
    String key = USER_SIGN_KEY + userId + keySuffix;
    // 4.获取今天是本月的第几天
    int dayOfMonth = now.getDayOfMonth();
    // 5.写入Redis SETBIT key offset 1
    stringRedisTemplate.opsForValue().setBit(key, dayOfMonth - 1, true);
    return Result.ok();
}
```



### 签到统计

**问题1：**什么叫做连续签到天数？
从最后一次签到开始向前统计，直到遇到第一次未签到为止，计算总的签到次数，就是连续签到天数。

![1653834455899](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/1653834455899.png)

Java逻辑代码：获得当前这个月的最后一次签到数据，定义一个计数器，然后不停的向前统计，直到获得第一个非0的数字即可，每得到一个非0的数字计数器+1，直到遍历完所有的数据，就可以获得当前月的签到总天数了

**问题2：**如何得到本月到今天为止的所有签到数据？

  BITFIELD key GET u[dayOfMonth] 0：表示从角标0开始，查多少个。

假设今天是10号，那么我们就可以从当前月的第一天开始，获得到当前这一天的位数，是10号，那么就是10位，去拿这段时间的数据，就能拿到所有的数据了，那么这10天里边签到了多少次呢？统计有多少个1即可。

**问题3：如何从后向前遍历每个bit位？**

注意：bitMap返回的数据是10进制，哪假如说返回一个数字8，那么我哪儿知道到底哪些是0，哪些是1呢？我们只需要让得到的10进制数字和1做与运算就可以了，因为1只有遇见1 才是1，其他数字都是0 ，我们把签到结果和1进行与操作，每与一次，就把签到结果向右移动一位，依次内推，我们就能完成逐个遍历的效果了。

案例代码：

```java
@Override
public Result signCount() {
    // 1.获取当前登录用户
    Long userId = UserHolder.getUser().getId();
    // 2.获取日期
    LocalDateTime now = LocalDateTime.now();
    // 3.拼接key
    String keySuffix = now.format(DateTimeFormatter.ofPattern(":yyyyMM"));
    String key = USER_SIGN_KEY + userId + keySuffix;
    // 4.获取今天是本月的第几天
    int dayOfMonth = now.getDayOfMonth();
    // 5.获取本月截止今天为止的所有的签到记录，返回的是一个十进制的数字 BITFIELD sign:5:202203 GET u14 0
    List<Long> result = stringRedisTemplate.opsForValue().bitField(
            key,
            BitFieldSubCommands.create()
                    .get(BitFieldSubCommands.BitFieldType.unsigned(dayOfMonth)).valueAt(0)
    );
    if (result == null || result.isEmpty()) {
        // 没有任何签到结果
        return Result.ok(0);
    }
    Long num = result.get(0);
    if (num == null || num == 0) {
        return Result.ok(0);
    }
    // 6.循环遍历
    int count = 0;
    while (true) {
        // 6.1.让这个数字与1做与运算，得到数字的最后一个bit位  // 判断这个bit位是否为0
        if ((num & 1) == 0) {
            // 如果为0，说明未签到，结束
            break;
        }else {
            // 如果不为0，说明已签到，计数器+1
            count++;
        }
        // 把数字右移一位，抛弃最后一个bit位，继续下一个bit位
        num >>>= 1;
    }
    return Result.ok(count);
}
```



## 关于使用bitmap来解决缓存穿透的方案

回顾**缓存穿透**：

发起了一个数据库不存在的，redis里边也不存在的数据，通常你可以把他看成一个攻击

解决方案：

* 判断id<0

* 如果数据库是空，那么就可以直接往redis里边把这个空数据缓存起来

第一种解决方案：遇到的问题是如果用户访问的是id不存在的数据，则此时就无法生效

第二种解决方案：遇到的问题是：如果是不同的id那就可以防止下次过来直击数据

所以我们如何解决呢？

我们可以将数据库的数据，所对应的id写入到一个list集合中，当用户过来访问的时候，我们直接去判断list中是否包含当前的要查询的数据，如果说用户要查询的id数据并不在list集合中，则直接返回，如果list中包含对应查询的id数据，则说明不是一次缓存穿透数据，则直接放行。

![1653836416586](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/1653836416586.png)

现在的问题是这个主键其实并没有那么短，而是很长的一个 主键

哪怕你单独去提取这个主键，但是在11年左右，淘宝的商品总量就已经超过10亿个

所以如果采用以上方案，这个list也会很大，所以我们可以使用bitmap来减少list的存储空间

我们可以把list数据抽象成一个非常大的bitmap，我们不再使用list，而是将db中的id数据利用哈希思想，比如：

id % bitmap.size  = 算出当前这个id对应应该落在bitmap的哪个索引上，然后将这个值从0变成1，然后当用户来查询数据时，此时已经没有了list，让用户用他查询的id去用相同的哈希算法， 算出来当前这个id应当落在bitmap的哪一位，然后判断这一位是0，还是1，如果是0则表明这一位上的数据一定不存在，  采用这种方式来处理，需要重点考虑一个事情，就是误差率，所谓的误差率就是指当发生哈希冲突的时候，产生的误差。



![1653836578970](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/1653836578970.png)



## 好友关注

基于 Set 集合的关注、取关、共同关注、消息推送等功能。

关注、取关、共同关注功能，结合 Redis，其实思路很简单：比如 A 关注了 B、C、D。那么可以将 A 的 id 作为 key，value 就是一个 set，存入 A 关注的所有人的 id；E 关注了 B、D、F，然后将 E 的 id 作为 key，value 也是一个 set，存入 E 关注的所有人的 id。利用Redis中恰当的数据结构，实现共同关注功能。在set集合中，有交集并集补集的api，我们可以把两人的关注的人分别放入到一个set集合中，然后再通过api去查看这两个set集合中的交集数据。用户关注了某位用户后，需要将数据放入到set集合中，同时当取消关注时，也需要从set集合中进行删除。，

消息推送：比如 A 关注了 B，那么 B 发布的动态 A 能看到，相当于 B 发布的动态推送给了 A，相同的场景就是微信朋友圈，加了好友的两个人发了朋友圈，那么两个人互相都能刷到对方的动态，并且动态按照时间排序。这其实就是 Feed 流的经典案例。关于 Feed 流，可以看看前面的段落。可以使用 redis 的 sorted set 来实现存储，核心的意思：就是我发布的动态在保存完之后，再获取到我有哪些粉丝，然后把这条动态推送到粉丝的 redis 中去。

## 点赞功能

点赞功能需求：

* 同一个用户只能点赞一次，再次点击则取消点赞
* 如果当前用户已经点赞，则点赞按钮高亮显示（前端实现，判断字段 isLike 的属性值）

假设用户是要给一个 BLOG 点赞，可以利用 Redis 的 set 集合判断是否点赞过。思路大概就是：BLOG 的 id 作为 key，value 是一个 set，set 中存放点过赞的用户的 id。点赞的时候获取到当前的用户 id，判断一下 redis 中有无即可。

案例代码：

```java
@Override
public Result likeBlog(Long id){
    // 1.获取登录用户
    Long userId = UserHolder.getUser().getId();
    // 2.判断当前登录用户是否已经点赞
    String key = BLOG_LIKED_KEY + id;
    Boolean isMember = stringRedisTemplate.opsForSet().isMember(key, userId.toString());
    if(BooleanUtil.isFalse(isMember)){
        //3.如果未点赞，可以点赞
        //3.1 数据库点赞数+1
        boolean isSuccess = update().setSql("liked = liked + 1").eq("id", id).update();
        //3.2 保存用户到Redis的set集合
        if(isSuccess){
            stringRedisTemplate.opsForSet().add(key,userId.toString());
        }
    }else{
        //4.如果已点赞，取消点赞
        //4.1 数据库点赞数-1
        boolean isSuccess = update().setSql("liked = liked - 1").eq("id", id).update();
        //4.2 把用户从Redis的set集合移除
        if(isSuccess){
            stringRedisTemplate.opsForSet().remove(key,userId.toString());
        }
    }
}
```

需求拓展：

- 点赞排行榜

点赞是放到set集合，但是set集合是不能排序的，所以这个时候，咱们可以采用一个可以排序的set集合，就是咱们的sortedSet。

我们接下来来对比一下这些集合的区别是什么

所有点赞的人，需要是唯一的，所以我们应当使用set或者是sortedSet

其次我们需要排序，就可以直接锁定使用sortedSet啦

![1653805203758](https://blog-images-erdochan.oss-cn-beijing.aliyuncs.com/img/1653805203758.png)

修改案例代码

```java
@Override
public Result likeBlog(Long id) {
    // 1.获取登录用户
    Long userId = UserHolder.getUser().getId();
    // 2.判断当前登录用户是否已经点赞
    String key = BLOG_LIKED_KEY + id;
    Double score = stringRedisTemplate.opsForZSet().score(key, userId.toString());
    if (score == null) {
        // 3.如果未点赞，可以点赞
        // 3.1.数据库点赞数 + 1
        boolean isSuccess = update().setSql("liked = liked + 1").eq("id", id).update();
        // 3.2.保存用户到Redis的set集合  zadd key value score
        if (isSuccess) {
            stringRedisTemplate.opsForZSet().add(key, userId.toString(), System.currentTimeMillis());
        }
    } else {
        // 4.如果已点赞，取消点赞
        // 4.1.数据库点赞数 -1
        boolean isSuccess = update().setSql("liked = liked - 1").eq("id", id).update();
        // 4.2.把用户从Redis的set集合移除
        if (isSuccess) {
            stringRedisTemplate.opsForZSet().remove(key, userId.toString());
        }
    }
    return Result.ok();
}


private void isBlogLiked(Blog blog) {
    // 1.获取登录用户
    UserDTO user = UserHolder.getUser();
    if (user == null) {
        // 用户未登录，无需查询是否点赞
        return;
    }
    Long userId = user.getId();
    // 2.判断当前登录用户是否已经点赞
    String key = "blog:liked:" + blog.getId();
    Double score = stringRedisTemplate.opsForZSet().score(key, userId.toString());
    blog.setIsLike(score != null);
}
```

