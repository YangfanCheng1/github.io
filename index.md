Hello, I am Yangfan, and I am a Software Developer with a background in designing/implementing scalable cloud system. I have a lot of deep hands-on experience with bulding a mordern end to end product. This covers a wide range of topics include but not limited to:
* API Gateway (AuthN/AuthN, throttling, etc)
* Load Balancer
* Microservices (Containers/Autoscaling)
* Communication method (RESTful/GraphQL)
* Error handling (Retry, Timeout, Distributed logging/monitoring/alerting)
* Request Optimization (Caching/Queues)
* Database
* Data Ingestion (Async/Batch processing)
* CICD (Version Control/Artifactory/Rollback plan)

Much of my experience is in the Java/SpringBoot/Docker/Kubernetes/AWS ecosystem. This very much resembles the tech stack at companies like Netflix (which is predomantly a Java shop). A lot of what I use during day-to-day job is very much language/framework specific, but I spent a lot of time in gaining advanced knowledge about each subject, even if they may not directly be used at work. For example, some of my favorite topics to explore are:
* Java for things like JVM (JIT, GC, Class Loader..), Complier (JSR-269/Annotation Processing, AST/Lombok), Lambda (FI, Vavr, function composition over inheritance), Profiling (Java Instrumentation API)..
* Spring/Boot -- IoC, Template, AutoConfiguration, Die hard fan :)
* MongoDB -- Model (Subset pattern, no ACID (transaction)), Aggregation, Change Stream



## Github
[Github Profile](https://github.com/YangfanCheng1)


## StackOverflow
[SO Profile](https://stackoverflow.com/users/4493352/%e5%a4%a2%e3%81%ae%e3%81%ae%e5%a4%a2?tab=profile)


## Projects:
* [College Souk](https://www.collegesouk.com/)
* [Chat App](http://yangfanchat-env.gp2axbmhet.us-east-2.elasticbeanstalk.com)
* [Introduction](http://yangfanc.org/about-me)
* [Bus Locator](http://yangfanchat-env.gp2axbmhet.us-east-2.elasticbeanstalk.com/bus-locator)


### Java/SpringBoot
When it comes to aspect-oriented programming, one could greatly reduce amount of boilerplate/redundant code and create greater modularity. Logging exemplifies this in that much of non application/business logic executions should have their logs automated. 

An example as follows:
```
@Aspect
@Slf4j
@Configuration
public class LogConfig {

    private static final Map<Class<?>, Logger> LOGGERS = Maps.newConcurrentMap();

    private Logger logger(JoinPoint joinPoint) {
        val clazz = joinPoint.getTarget().getClass();
        return LOGGERS.computeIfAbsent(clazz, LoggerFactory::getLogger);
    }

    @Pointcut("within(@org.springframework.web.bind.annotation.RestController *)")
    void restController() {
        // noop
    }

    @Before("restController()")
    void beforeController(JoinPoint joinPoint) {
        val log = logger(joinPoint);
        val attributes = RequestContextHolder.currentRequestAttributes();
        if (attributes instanceof ServletRequestAttributes) {
            val request = ((ServletRequestAttributes) attributes).getRequest();
            val name = joinPoint.getSignature().getName();
            log.info("{} {}{}",
                    request.getMethod(),
                    request.getServletPath(),
                    Optional.ofNullable(request.getQueryString()).map(q -> "?" + q).orElse(""));
            log.debug("{}() - args:  {}", name, joinPoint.getArgs());
        }
    }

    @SneakyThrows
    @Around("@annotation(LogExecutionTime)")
    public Object logExecutionTime(ProceedingJoinPoint point) {
        val start = System.currentTimeMillis();
        val result = point.proceed();
        val end = System.currentTimeMillis();
        log.info("{}({}) -> {} - time taken: {}ms", point.getSignature().getName(), point.getArgs(), result, end - start);
        return result;
    }
}
```
This snippet shows that no only we can automate logs for each http request to our API, but also we can log extraction time using custom annotation. 
> 2024-04-11T13:00:39.720-04:00  INFO [core, 661aba370bf972c054d12916af39f3fa, 54d12916af39f3fa] 56182 --- [nio-8080-exec-6] io.github.yangfan.core.CoreController    : GET /api/users?id=123
> 2024-04-11T13:00:39.798-04:00  INFO [core, 661aba370bf972c054d12916af39f3fa, 54d12916af39f3fa] 56182 --- [nio-8080-exec-6] i.g.y.core.common.logging.LogConfig      : getFoo([123]) -> FooBarResponse[value=a, b] - time taken: 70ms
