---
layout: post
title: "Redis-接口的幂等性之Redis解决方案"
date: 2019-08-15
description: "Redis，分布式锁"
tag: JAVA进阶学习

---

&emsp;&emsp;最近工作的过程中，遇到了解决接口幂等性的需求，最终使用了Redis分布式锁 + token + spring拦截器的方案解决，在这里记录下。

### 什么是接口幂等性

&emsp;&emsp;接口幂等性就是用户对于同一操作发起的一次请求或者多次请求的结果是一致的，不会因为多次点击而产生了副作用。举个最简单的例子，那就是支付，用户购买商品后支付，支付扣款成功，但是返回结果的时候网络异常，此时钱已经扣了，用户再次点击按钮，此时会进行第二次扣款，返回结果成功，用户查询余额返发现多扣钱了，流水记录也变成了两条。这就没有保证接口的幂等性。

&emsp;&emsp;声明为幂等的服务会认为外部调用失败是常态，并且失败之后必然会有重试。

### 什么情况下需要接口幂等

&emsp;&emsp;以SQL为例：

```sql
UPDATE tab1 SET col1=1 WHERE col2=2;  #无论执行成功多少次状态都是一致的，因此是幂等操作。
UPDATE tab1 SET col1=col1+1 WHERE col2=2; #每次执行的结果都会发生变化，这种不是幂等的。
```

### 解决接口幂等性的方案

&emsp;&emsp;其实发生接口幂等性问题的本质就是多次请求了，解决问题首先从如何避免多次请求，或者判断请求是不是多次请求来出发。

&emsp;&emsp;其实我们通过token和拦截器就可以简单的实现一个解决接口幂等的方案。即：当客户端请求服务端的某个服务涉及到接口幂等问题时，先去发一个请求去服务端获取token，这时服务端会生成一个token放在缓存里面，然后把这个token返回给客户端，客户端之后带着这个token去访问涉及接口幂等的服务，服务判断token是否存在于缓存中，存在的话那就代表是第一次请求，然后删除这个token，之后进行业务操作，如果token不存在，那么就代表是重复请求，向客户端提示即可。

&emsp;&emsp;流程图如下：

![](https://images.weserv.nl/?url=https://i0.hdslb.com/bfs/article/01a374bf79e76fcda7bcb5fbdec8b641d1ace2c2.png)

&emsp;&emsp;这里会有一个问题，就是先删除token再执行业务，还是先执行业务再删除token？

&emsp;&emsp;如果使用上面的方案，先删除token，那么如果之后执行业务的过程中执行失败了，客户端那边也没有获取到明确的结果，这时客户端再去请求，token已经被删除了，服务端判断是重复请求，就直接返回了，不进行业务处理。

&emsp;&emsp;如果先处理业务呢？也会有问题。如果处理完业务删除token失败了，那客户端重复发请求，token还是存在的，这样就会导致业务数据错误。

&emsp;&emsp;这两种方式具体选择哪种方式，这就视业务而定了，我们采用的是先删除token的方案，保证业务数据是对的，当出现上面的问题，再次由调用方发起重试请求就可以了。

&emsp;&emsp;那么如果有大量的请求并发的发带token参数的请求进来，我们怎么解决呢？这个时候就需要用到分布式锁，分布式锁可以基于Redis的setnx来实现，具体实现看下面的代码。

### 代码实现

&emsp;&emsp;我这里使用SpringBoot和Jedis的方式实现。这里只贴服务端的代码。

首先是获取token的接口

```java
@RestController
public class TokenController {
    @Autowired
    private RedisService redisService;
    @GetMapping("/users-anon/gettoken")
    public Map getToken(@RequestParam("url") String url) {
        Map<String,String> tokenMap = new HashMap();
        String tokenValue = UUID.randomUUID().toString();
        tokenMap.put(url + tokenValue, tokenValue);
        //把token放到redis中，使用分布式锁的方式
        redisService.set(url + tokenValue, tokenValue);
        return tokenMap;
    }
}
```

涉及接口幂等的服务的拦截器

```java
@Slf4j
@Component
public class TokenInterceptor implements HandlerInterceptor {
    @Autowired
    private RedisService redisService;
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String tokenName = request.getRequestURI() + request.getParameter("token_value");
        String tokenValue = request.getParameter("token_value");
        if (tokenValue != null && !tokenValue.equals("")) {
            log.info("tokenName:{},tokenValue:{}",tokenName,tokenValue);
            return handleToken(request,response,handler);
        }
        return false;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, @Nullable ModelAndView modelAndView) throws Exception {
        if (redisService.exists(request.getParameter("token_value"))) {
            RedisTool.releaseDistributedLock(redisService, request.getParameter("token_value"), request.getParameter("token_value"));
        }
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, @Nullable Exception ex) throws Exception {

    }

    /**
     * 分布式锁处理
     * @param request
     * @param response
     * @param handler
     * @return
     * @throws Exception
     */
    private boolean handleToken(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        //当大量高并发下所有带token参数的请求进来时，进行分布式锁定,允许某一台服务器的一个线程进入，锁定时间3分钟
        if (RedisTool.tryGetDistributedLock(redisService,request.getParameter("token_value"),request.getParameter("token_value"),180)) {
            if (redisService.exists(request.getRequestURI() + request.getParameter("token_value"))) {
                //当请求的url与token与redis中的存储相同时
                if (redisService.get(request.getRequestURI() + request.getParameter("token_value")).equals(request.getParameter("token_value"))) {
                    //放行的该线程删除redis中存储的token
                    redisService.del(request.getRequestURI() + request.getParameter("token_value"));
                    //放行
                    return true;
                }
            }
            //当请求的url与token与redis中的存储不相同时，解除锁定
            RedisTool.releaseDistributedLock(redisService,request.getParameter("token_value"),request.getParameter("token_value"));
            //进行拦截
            return false;
        }
        return false;
    }
```

分布式锁的实现：

```java
public class RedisTool {
    private static final String LOCK_SUCCESS = "OK";
    private static final Long RELEASE_SUCCESS = 1L;
    /**
     * 尝试获取分布式锁
     * @param lockKey 锁
     * @param requestId 请求标识
     * @param expireTime 超期时间
     * @return 是否获取成功
     */
    public static boolean tryGetDistributedLock(RedisService redisService, String lockKey, String requestId, int expireTime) {

        String result = redisService.set(lockKey, requestId, expireTime);

        if (LOCK_SUCCESS.equals(result)) {
            return true;
        }
        return false;
    }
    /**
     * 释放分布式锁
     * @param lockKey 锁
     * @param requestId 请求标识
     * @return 是否释放成功
     */
    public static boolean releaseDistributedLock(RedisService redisService, String lockKey, String requestId) {
        Object result = redisService.eval(lockKey,requestId);

        if (RELEASE_SUCCESS.equals(result)) {
            return true;
        }
        return false;

    }
}
```

redisServiceImpl的实现：

```java
private static final String SET_IF_NOT_EXIST = "NX";
private static final String SET_WITH_EXPIRE_TIME = "EX";
@Autowired
private JedisPool jedisPool;

public <T> T execute(RedisFunction<T, Jedis> fun) {
    Jedis jedis = null;
    try {
        jedis = jedisPool.getResource();
        return (T)fun.callback(jedis);
    }catch (Exception e) {
        logger.error(e.getMessage());
        return null;
    }finally {
        if (jedis != null) {
            jedis.close();
        }
    }
}
```

```java
@Override
public String set(String lockKey, String requestId, int expireTime) {
    return execute(new RedisFunction<String, Jedis>() {

        @Override
        public String callback(Jedis jedis) {
            return jedis.set(lockKey,requestId,SET_IF_NOT_EXIST,SET_WITH_EXPIRE_TIME,expireTime);
        }

    });
}
```

```java
@Override
public Object eval(String lockKey, String requestId) {
    return execute(new RedisFunction<String, Jedis>() {
        @Override
        public Object callback(Jedis jedis) {
            String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
            return jedis.eval(script, Collections.singletonList(lockKey),Collections.singletonList(requestId));
        }
    });
}
```

```java
public interface RedisFunction<T, E> {
    Object callback(E jedis);
}
```

### 解决接口幂等的其他方式

&emsp;&emsp;除了用Redis分布式锁的方式解决接口幂等的问题，还有以下几种方式也可以解决接口幂等问题，可以根据业务选择使用。

### 1. 乐观锁机制

&emsp;&emsp;乐观锁这里解决了计算赋值型的修改场景。比如说一条修改的sql，可以这样写：

```mysql
update user set point = point + 20, version = version + 1 where userid=1 and version=1;
```

&emsp;&emsp;加上了版本号后，就让此计算赋值型业务，具备了幂等性。

### 2. 唯一主键机制

&emsp;&emsp;这个机制是利用了数据库的主键唯一约束的特性，解决了在insert场景时幂等问题。但主键的要求不是自增的主键，这样就需要业务生成全局唯一的主键，之前老顾的文章也介绍过分布式唯一主键ID的生成，可自行查阅。

&emsp;&emsp;如果是分库分表场景下，路由规则要保证相同请求下，落地在同一个数据库和同一表中，要不然数据库主键约束就不起效果了，因为是不同的数据库和表主键不相关。

&emsp;&emsp;因为对主键有一定的要求，这个方案就跟业务有点耦合了，无法用自增主键了。

### 总结
上面介绍了一些幂等方案，小伙伴们根据自身的业务进行选择，尽量不要让系统变的复杂，所以推荐唯一主键和乐观锁方式，因为实现比较简单。

----------
<font color="blue">笔者水平有限，若有错漏，欢迎指正，如果转载以及CV操作，请务必注明出处，谢谢！</font>


----------


<font color="red">版权声明：本文为博主原创文章，未经博主允许不得转载。</font>
