---
title: Hystrix-MDC跟踪
date: 2020-03-20
categories:
  - Java
tags:
  - Java
---
一、实现HystrixConcurrencyStrategy
```java
public class MdcHystrixConcurrencyStrategy extends HystrixConcurrencyStrategy {

    @Override
    public <T> Callable<T> wrapCallable(Callable<T> callable) {
        return new MdcAwareCallable(callable, MDC.getCopyOfContextMap());
    }

    private class MdcAwareCallable<T> implements Callable<T> {
        private final Callable<T> delegate;

        private final Map<String, String> contextMap;

        public MdcAwareCallable(Callable<T> callable, Map<String, String> contextMap) {
            this.delegate = callable;
            this.contextMap = contextMap != null ? contextMap : new HashMap();
        }

        @Override
        public T call() throws Exception {
            try {
                MDC.setContextMap(contextMap);
                return delegate.call();
            } finally {
                MDC.clear();
            }
        }
    }
}

```

二、注册一个插件
```java
@Configuration
public class HystrixConfig {
    //用来拦截处理HystrixCommand注解
    @Bean
    public HystrixCommandAspect hystrixAspect() {
        return new HystrixCommandAspect();
    }

    @PostConstruct
    public void init() {
        HystrixPlugins.getInstance().registerConcurrencyStrategy(new MdcHystrixConcurrencyStrategy());
    }
}
```

三、mq producer 和 consumer之间的trace跟踪
> 正常情况下、用msgId就可以了、有些数据分析需要trace来全局跟踪

1. 在生产者的消息属性中加入traceId
```java 
       SendResult result = null;
        Message msg = new Message(topic, tag, key, val.getBytes());
        msg.putUserProperty("traceId", MDC.get(TraceConst.TRACE_KEY));
```

2. 在消费者中取出traceId、放入MDC
```java
MessageExt msgExt = msgs.get(i);
                        String msgId = msgExt.getMsgId();
                        String msgBody = new String(msgExt.getBody());
                        MDC.put(TraceConst.TRACE_KEY, msgExt.getUserProperty("traceId"));
                        logger.info("msgs-size={}, msgId={}, msgBody={}", msgs.size(), msgId, msgBody);
```

3. 部分源码
![image.png](https://upload-images.jianshu.io/upload_images/14027542-44bfdc7e5e6f2949.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image.png](https://upload-images.jianshu.io/upload_images/14027542-87639bf3513c1838.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

文章参考
[https://my.oschina.net/u/3748347/blog/1823392](https://my.oschina.net/u/3748347/blog/1823392)
[https://wanghaolistening.github.io/2019/01/26/Hystrix-%E5%9C%A8%E9%A1%B9%E7%9B%AE%E4%B8%AD%E7%9A%84%E5%BA%94%E7%94%A8/](https://wanghaolistening.github.io/2019/01/26/Hystrix-%E5%9C%A8%E9%A1%B9%E7%9B%AE%E4%B8%AD%E7%9A%84%E5%BA%94%E7%94%A8/)
