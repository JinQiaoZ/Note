# SpringBoot使用@Async异步注解

在项目中，当访问其他人的接口较慢或者做耗时任务时，不想程序一直卡在耗时任务上，想程序能够并行执行，我们可以使用多线程来并行的处理任务，这里介绍下 SpringBoot 下的 @Async 注解，还有 ApplicationEventPublisher 可以了解下

**代码地址**

* Github: [https://github.com/dolyw/ProjectStudy/tree/master/SpringBoot/AsyncDemo](https://github.com/dolyw/ProjectStudy/tree/master/SpringBoot/AsyncDemo)
* Gitee(码云): [https://gitee.com/dolyw/ProjectStudy/tree/master/SpringBoot/AsyncDemo](https://gitee.com/dolyw/ProjectStudy/tree/master/SpringBoot/AsyncDemo)

## 1. Config

需要一个注解 @EnableAsync 开启 @Async 的功能，SpringBoot 可以放在 Application 上，也可以放其他配置文件上

```java
@EnableAsync
@SpringBootApplication
public class Application { }
```

@Async 配置有两个，一个是执行的线程池，一个是异常处理

执行的线程池默认情况下找唯一的 org.springframework.core.task.TaskExecutor，或者一个 Bean 的 Name 为 taskExecutor 的 java.util.concurrent.Executor 作为执行任务的线程池。如果都没有的话，会创建 SimpleAsyncTaskExecutor 线程池来处理异步方法调用，当然 @Async 注解支持一个 String 参数，来指定一个 Bean 的 Name，类型是 Executor 或 TaskExecutor，表示使用这个指定的线程池来执行这个异步任务

异常处理，@Async 标记的方法只能是 void 或者 Future 返回值，在无返回值的异步调用中，异步处理抛出异常，默认是SimpleAsyncUncaughtExceptionHandler 的 handleUncaughtException() 会捕获指定异常，只是简单的输出了错误日志(一般需要自定义配置异常处理)，原有任务还会继续运行，直到结束(具有 void 返回类型的方法不能将任何异常发送回调用者，默认情况下此类未捕获异常只会输出错误日志)，而在有返回值的异步调用中，异步处理抛出了异常，会直接返回主线程处理，异步任务结束执行，主线程也会被异步方法中的异常中断结束执行

::: tip @Async有两个使用的限制
1. 它必须仅适用于 public 方法
2. 在同一个类中调用异步方法将无法正常工作(self-invocation)
:::

```java
/**
 * AsyncConfig
 *
 * @author wliduo[i@dolyw.com]
 * @date 2020/5/19 17:58
 */
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    /**
     * logger
     */
    private static final Logger logger = LoggerFactory.getLogger(AsyncConfig.class);

    /**
     * 这里不实现了，使用 ThreadPoolConfig 里的线程池即可
     *
     * @param
     * @return java.util.concurrent.Executor
     * @throws
     * @author wliduo[i@dolyw.com]
     * @date 2020/5/19 18:00
     */
    /*@Override
    public Executor getAsyncExecutor() {
        return null;
    }*/

    /**
     * 只能捕获无返回值的异步方法，有返回值的被主线程处理
     *
     * @param
     * @return org.springframework.aop.interceptor.AsyncUncaughtExceptionHandler
     * @throws
     * @author wliduo[i@dolyw.com]
     * @date 2020/5/20 10:16
     */
    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return new CustomAsyncExceptionHandler();
    }

    /***
     * 处理异步方法中未捕获的异常
     *
     * @author wliduo[i@dolyw.com]
     * @date 2020/5/19 19:16
     */
    class CustomAsyncExceptionHandler implements AsyncUncaughtExceptionHandler {

        @Override
        public void handleUncaughtException(Throwable throwable, Method method, Object... obj) {
            logger.info("Exception message - {}", throwable.getMessage());
            logger.info("Method name - {}", method.getName());
            logger.info("Parameter values - {}", Arrays.toString(obj));
            if (throwable instanceof Exception) {
                Exception exception = (Exception) throwable;
                logger.info("exception:{}", exception.getMessage());
            }
            throwable.printStackTrace();
        }
    }
}
```

```java
/**
 * 线程池配置
 *
 * @author wliduo
 * @date 2019/2/15 14:36
 */
@Configuration
public class ThreadPoolConfig {

    /**
     * logger
     */
    private final static Logger logger = LoggerFactory.getLogger(ThreadPoolConfig.class);

    @Value("${asyncThreadPool.corePoolSize}")
    private int corePoolSize;

    @Value("${asyncThreadPool.maxPoolSize}")
    private int maxPoolSize;

    @Value("${asyncThreadPool.queueCapacity}")
    private int queueCapacity;

    @Value("${asyncThreadPool.keepAliveSeconds}")
    private int keepAliveSeconds;

    @Value("${asyncThreadPool.awaitTerminationSeconds}")
    private int awaitTerminationSeconds;

    @Value("${asyncThreadPool.threadNamePrefix}")
    private String threadNamePrefix;

    /**
     * 线程池配置
     * @param
     * @return java.util.concurrent.Executor
     * @author wliduo
     * @date 2019/2/15 14:44
     */
    @Bean(name = "threadPoolTaskExecutor")
    public ThreadPoolTaskExecutor threadPoolTaskExecutor() {
        logger.info("---------- 线程池开始加载 ----------");
        ThreadPoolTaskExecutor threadPoolTaskExecutor = new ThreadPoolTaskExecutor();
        // 核心线程池大小
        threadPoolTaskExecutor.setCorePoolSize(corePoolSize);
        // 最大线程数
        threadPoolTaskExecutor.setMaxPoolSize(maxPoolSize);
        // 队列容量
        threadPoolTaskExecutor.setQueueCapacity(keepAliveSeconds);
        // 活跃时间
        threadPoolTaskExecutor.setKeepAliveSeconds(queueCapacity);
        // 主线程等待子线程执行时间
        threadPoolTaskExecutor.setAwaitTerminationSeconds(awaitTerminationSeconds);
        // 线程名字前缀
        threadPoolTaskExecutor.setThreadNamePrefix(threadNamePrefix);
        // RejectedExecutionHandler:当pool已经达到max-size的时候，如何处理新任务
        // CallerRunsPolicy:不在新线程中执行任务，而是由调用者所在的线程来执行
        threadPoolTaskExecutor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        // 初始化
        threadPoolTaskExecutor.initialize();
        logger.info("---------- 线程池加载完成 ----------");
        return threadPoolTaskExecutor;
    }
}
```

## 2. Code

简单的使用，异步异常处理，Service 方法异步，多个 Service 方法异步，工具类异步

### 2.1. Controller

```java
/**
 * AsyncController
 *
 * @author wliduo[i@dolyw.com]
 * @date 2020/5/19 14:46
 */
@RestController
@RequestMapping("/async")
public class AsyncController {

    /**
     * logger
     */
    private final static Logger logger = LoggerFactory.getLogger(AsyncController.class);

    @Autowired
    private AsyncService asyncService;

    @Autowired
    private SmsUtil smsUtil;

    /**
     * 可以看到无返回值异步方法出现异常，主线程还是继续执行完成
     *
     * @param
     * @return void
     * @throws
     * @author wliduo[i@dolyw.com]
     * @date 2020/5/20 9:53
     */
    @GetMapping("/run1")
    public String run1() throws Exception {
        asyncService.task1();
        logger.info("run1开始执行");
        Thread.sleep(5000);
        logger.info("run1执行完成");
        return "run1 success";
    }

    /**
     * 可以看到有返回值异步方法出现异常，异常抛给主线程处理，导致主线程也被中断执行
     *
     * @param
     * @return java.lang.String
     * @throws
     * @author wliduo[i@dolyw.com]
     * @date 2020/5/20 10:15
     */
    @GetMapping("/run2")
    public String run2() throws Exception {
        Future<String> future = asyncService.task2();
        // get()方法阻塞主线程，直到执行完成
        String result = future.get();
        logger.info("run2开始执行");
        Thread.sleep(5000);
        logger.info("run2执行完成");
        return result;
    }

    /**
     * 多个异步执行
     *
     * @param
     * @return java.lang.String
     * @throws
     * @author wliduo[i@dolyw.com]
     * @date 2020/5/20 10:26
     */
    @GetMapping("/run3")
    public String run3() throws Exception {
        logger.info("run3开始执行");
        long start = System.currentTimeMillis();
        Future<String> future3 = asyncService.task3();
        Future<String> future4 = asyncService.task4();
        // 这样与下面是一样的
        logger.info(future3.get());
        logger.info(future4.get());
        // 先判断是否执行完成
        boolean run3Done = Boolean.FALSE;
        while (true) {
            if (future3.isDone() && future4.isDone()) {
                // 执行完成
                run3Done = Boolean.TRUE;
                break;
            }
            if (future3.isCancelled() || future4.isCancelled()) {
                // 取消情况
                break;
            }
        }
        if (run3Done) {
            logger.info(future3.get());
            logger.info(future4.get());
        } else {
            // 其他异常情况
        }
        long end = System.currentTimeMillis();
        logger.info("run3执行完成，执行时间: {}", end - start);
        return "run3 success";
    }

    /**
     * 工具类异步
     *
     * @param
     * @return java.lang.String
     * @throws
     * @author wliduo[i@dolyw.com]
     * @date 2020/5/20 10:59
     */
    @GetMapping("/sms")
    public String sms() throws Exception {
        logger.info("run1开始执行");
        smsUtil.sendCode("15912347896", "135333");
        logger.info("run1执行完成");
        return "send sms success";
    }
}
```

### 2.2. ServiceImpl

```java
/**
 * AsyncServiceImpl
 *
 * @author wliduo[i@dolyw.com]
 * @date 2020/5/19 14:24
 */
@Service
public class AsyncServiceImpl implements AsyncService {

    /**
     * logger
     */
    private final static Logger logger = LoggerFactory.getLogger(AsyncServiceImpl.class);

    @Override
    @Async("threadPoolTaskExecutor")
    public void task1() throws Exception {
        logger.info("task1开始执行");
        Thread.sleep(3000);
        logger.info("task1执行结束");
        throw new RuntimeException("出现异常");
    }

    @Override
    @Async("threadPoolTaskExecutor")
    public Future<String> task2() throws Exception {
        logger.info("task2开始执行");
        Thread.sleep(3000);
        logger.info("task2执行结束");
        throw new RuntimeException("出现异常");
        // return new AsyncResult<String>("task2 success");
    }

    @Override
    @Async("threadPoolTaskExecutor")
    public Future<String> task3() throws Exception {
        logger.info("task3开始执行");
        Thread.sleep(3000);
        logger.info("task3执行结束");
        return new AsyncResult<String>("task3 success");
    }

    @Override
    @Async("threadPoolTaskExecutor")
    public Future<String> task4() throws Exception {
        logger.info("task4开始执行");
        Thread.sleep(3000);
        logger.info("task4执行结束");
        return new AsyncResult<String>("task4 success");
    }
}
```

### 2.3. SmsUtil

```java
/**
 * SmsUtil
 *
 * @author wliduo[i@dolyw.com]
 * @date 2020/5/20 10:50
 */
@Component
public class SmsUtil {

    private static final Logger logger = LoggerFactory.getLogger(SmsUtil.class);

    /**
     * 异步发送短信
     *
     * @param phone
	 * @param code
     * @return void
     * @throws
     * @author wliduo[i@dolyw.com]
     * @date 2020/5/20 10:53
     */
    @Async
    public void sendCode(String phone, String code) {
        logger.info("开始发送验证码...");
        // 模拟调用接口发验证码的耗时
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        logger.info("发送成功: {}", phone);
    }
    
}
```

**参考**

* [Spring Boot 异步任务 -- @EnableAsync 详解](https://www.jianshu.com/p/c76269c68cb0)
* [springboot异步调用@Async](https://segmentfault.com/a/1190000010142962)
* [springboot之@Async实现异步](https://my.oschina.net/u/2277632/blog/1942500)
* [Spring异步任务配置、执行@EnableAsync和@Async](https://www.jianshu.com/p/ec57c52f47ad)