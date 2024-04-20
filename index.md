Thank you for visiting the site!

Hello, I am Yangfan. I am a Software Developer with a background in designing/implementing scalable cloud applications & systems. I have a lot of deep hands-on experience with building a modern end to end web based product. This covers a wide range of topics include but not limited to:
* API Gateway (AuthN/AuthN, throttling, etc)
* Load Balancer
* Microservices (Containers/Autoscaling)
* Communication method (RESTful/GraphQL)
* Error handling (Retry, Timeout, CircuitBreaker)
* Request Optimization (Caching/Queues)
* Database
* Data Ingestion (Async/Batch processing)
* Monitoring (Distributed tracing, Dashboards, Alerting)
* CICD (Version Control/Artifactory/Rollback plan)

Much of my experience is in the Java/SpringBoot/Docker/Kubernetes/AWS ecosystem. This very much resembles the tech stack at companies like Netflix (which is predominantly a Java shop). A lot of what I use during day-to-day job is very much language/framework specific, but I spent a lot of time in gaining advanced knowledge about each subject, even if they may not directly be used at work. For example, some of my favorite topics to explore are:
* Java for things like JVM (JIT, GC, Class Loader..), Compiler (JSR-269/Annotation Processing, AST/Lombok), Lambda (FI, Vavr, function composition over inheritance), Profiling (Java Instrumentation API)..
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


## General Approaches/Techniques
I'd like to share a collection of approaches/techniques I've come across and adapted over the years. They show some glimpse into the work I do, w.r.t how I write code and what I share with other developers. Some of which are technology/framework specific, but most should serve a general purpose in software development. Hope you can enjoy them too! 

* [AOP](#aop)
* [Declarative Programming](#declarative-programming)
* [Parallelism in Java](#parallelism)
* [Design Pattern - Chain of Responsibility](#design-pattern---chain-of-responsibility)
* [Extensibility](#extensibility-in-oop)

* [Cloud](./)

### AOP
When it comes to aspect-oriented programming, one could greatly reduce amount of boilerplate/redundant code and create greater modularity. Logging exemplifies this in that much of non application/business logic executions should have their logs automated. 

Performance aside, a naive approach in writing a log statement might be as follows:
```
// log GET user(37)
@GetMapping("api/v1/users/{id}")
public getUser(@PathVariable String id) {
  log.info("GET user({})", id);
}

// log 
// PATCH user(37), body(email=a@b.c)
// Time taken: 778ms
@PatchMapping("api/v1/users/{id}")
public getUser(@PathVariable String id, @Validated @RequestBody UserRequest userReq) {
  val start = Instant.now();
  log.info("PATCH user({}), body({})", id, userReq);
  val end = Instant.now();
  log.info("Time taken: {}", end - start);
}
```

Using Aspect, one could delegate logging concern and achieve above without explicit logging:
```java
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
Above snippet shows that no only we can automate logs for each http request to our API, but also we can log execution time using custom annotation: 
> 2024-04-11T13:00:39.720-04:00  INFO [core, 661aba370bf972c054d12916af39f3fa, 54d12916af39f3fa] 56182 --- [nio-8080-exec-6] io.github.yangfan.core.CoreController    : GET /api/users?id=123

And:
> 2024-04-11T13:00:39.798-04:00  INFO [core, 661aba370bf972c054d12916af39f3fa, 54d12916af39f3fa] 56182 --- [nio-8080-exec-6] i.g.y.core.common.logging.LogConfig      : getFoo([123]) -> FooBarResponse[value=a, b] - time taken: 70ms

### Declarative Programming
Maintenance and readability become critical aspects of any code base that is evolving to be increasingly large and complex. That's why having a declarative style can be very beneficial. AssertJ, for example, has become a popular way in writing unit tests in java due to its declarative-ness. Let's look at a naive approach in object comparison:

```
if (coffee.equals(drink)
        || latte.equals(drink)
        || cap.equals(drink)) {
    ...
}

if (weight.comparesTo(AVG_WEIGHT) < 0 || weight.comparesTo(AVG_WEIGHT) == 0) {
    ...
}
```
Instead, we can make the code more intuitive and readable by applying a declarative style.

Given that the object is of a class that extends Comparable:

```
if (ComparableObj.of(drink).isAnyOf(coffee, latte, cap)) {..}
if (ComparableObj.of(weight).isLessThanOrEqual(AVG_WEIGHT)) {..}
```
We can provide such construct as such:

```java
public class ComparableObj<T extends Comparable<T>> {
    private T val;

    private ComparableObj(T t) {this.val = t;}

    public static <T extends Comparable<T>> ComparableObj<T> of(T t) {
        Assert.notNull(t, "Arg cannot be null");
        return new ComparableObj<>(t);
    }

    @SafeVarargs
    public final boolean isAnyOf(T... values) {
        return Arrays.asList(values).contains(this.val);
    }

    public final boolean isLessThanOrEqualTo(T operand) {
        return isLessThan.or(isEqualTo).test(operand);
    }

    public final boolean isGreaterThan(T operand) {
        return isGreaterThan.test(operand);
    }

    private final Predicate<T> isLessThan = arg -> this.val.compareTo(arg) < 0;

    private final Predicate<T> isEqualTo = arg -> this.val.compareTo(arg) == 0;

    private final Predicate<T> isGreaterThan = arg -> this.val.compareTo(arg) > 0;

}
```

### Parallelism
Prior to Stream API, writing parallel processing or concurrency code in Java is likely to open a can of worms. And doing such optimization without knowing common pitfalls and executing thorough testing can lead to behaviors such as unexpected output and OOM errors. Stream API's `parallelStream` greatly simplifies and overcome such issues, albeit coming with pitfalls of its own. Let's look at how one would process a large (CSV) data file and upsert each entry into a data store.
```
public sychronized Future<Integer> syncByFile(MultipartFile file) {
    try (BufferedReader br = new BufferedReader(
            new InputStreamReader(file.getInputStream()))
    ) {
        int numUpserted =
                (int) br.lines()
                        .parallel()
                        .filter(line -> !StringUtils.isEmpty(line))
                        .filter(Product::isValid)
                        .map(Product::convertToProduct)
                        .map(productRepo::save)
                        .peek(product -> log.info("Product(id={}) upserted", product.getId()))
                        .count();
        return new AsyncResult<>(numUpserted);
    } catch (IOException e) {
        log.error("Error reading file: {}", file.getName(), e);
        return new AsyncResult<>(-1);
    }
}
```


### Design Pattern - Chain of Responsibility
Let's visit one of the popular design patterns - Chain of Responsibility. With Lambda's syntactic sugar since Java 8, we could write much concise code using such pattern. Say we want to implement a search functionality in a file system, and this search has various criteria by which resulting files are filtered. We could define a Filter interface as such:
```java

@FunctionalInterface
public interface Filter {
    boolean is(File resource, Params params);

    default Filter and(Filter next) {
        return (resource, params) ->
                is(resource, params) && next.is(resource, params);
    }

    static Filter nameFilter() {
        return (resource, params) -> {
            if (params.name != null) {
                return resource.getName().contains(params.name);
            }
            return true;
        };
    }

    static Filter extensionFilter() {
        return (resource, params) -> {
            if (params.extension != null) {
                return params.extension.startsWith(".") && resource.getName().endsWith(params.extension);
            }
            return true;
        };
    }

    static Filter maxFileSizeFilter() {
        return (resource, params) -> {
            if (params.size != null) {
                return params.size > resource.getSize();
            }
            return true;
        };
    }
}

@AllArgsConstructor
@NoArgsConstructor
class Params {
    String name;
    Integer size;
    String extension;
}

```
And `Resource` interface which represents a file or a directory:

```java
public interface Resource {

    String getName();

    boolean isDirectory();

    void setName(String name);

    int getSize();

    Instant getLastModified();
}

public abstract class AbstractResource implements Resource {

    protected String name;
    protected Instant lastModified;

    public AbstractResource(String name) {
        this.name = name;
        this.lastModified = Instant.now();
    }

    @Override
    public String getName() {
        return this.name;
    }

    @Override
    public void setName(String name) {
        this.name = name;
    }

    @Override
    public Instant getLastModified() {
        return this.lastModified;
    }
}


public class Directory extends AbstractResource {

    private final List<Resource> resources;

    public Directory(String name) {
        super(name);
        this.resources = new ArrayList<>();
    }


    @Override
    public boolean isDirectory() {
        return true;
    }

    @Override
    public int getSize() {
        return resources.stream().map(Resource::getSize).reduce(0, Integer::sum);
    }

    public boolean addResource(Resource resource) {
        return this.resources.add(resource);
    }

    public List<Resource> getResources() {
        return resources;
    }
}

```

Given different `Filter`s are defined as above, we could now compose a different chain of file filters such that each filter performs its function in isolation. Such approach is similar to Stream API provides since Java 8:
```
Streamable.of(coffee, latte, cappuccino)
    .filter(Props::hasCaffeine)
    .filter(Props::hasMilk)
    .filter(Props::hasExpresso)
    ...
```
Now if we want to compose our file filter chain, we simply concatenate using `and(filter)`. Say we want to create a filter chain that would return a list of MP3 files containing some artist name, we could define a simple BFS search function as such:
```java
public class FileSearch {
    private static final Filter MP3ByArtistFilter = Filter.nameFilter().and(extensionFilter());

    public List<Resource> getMP3FilesContainingName(Resource directory, String name) {
        var params = new Params();
        params.name = name;
        params.extension = ".mp3";
        
        var result = new ArrayList<>();
        var queue = new LinkedList<>();
        
        queue.add(directory);
        while (!queue.isEmpty()) {
            var resource = queue.poll();
            
            if (resource.isDirectory()) { // If resource is folder, then add its resources to the queue
                var dir = (Directory) resource;
                dir.getResources().forEach(queue::offer);
            } else { // otherwise resource is a file, then apply desired filter
                var file = (File) resource;
                if (MP3ByArtistFilter.is(file, params)) {
                    result.add(file);
                }
            }
        }
    
        return result;
    }
}
```

### Extensibility in OOP
Extensibility in OOP can allow for great flexibility & creativity in problem. Ever coming cross a need to build key value pairs in a HashMap but not wanting to add entries with `null` values? Here is a typical solution:
```
void addHeaderIfNotNull(Map<String, String> headers, String key, String val) {
    if (val != null) {
        header.add(key, val);
    }
}
..
Map<String, String> headers = new HashMap<>();
addHeaderIfNotNull(headers, "x-client-id", params._0)
addHeaderIfNotNull(headers, "x-request-id", params._1)
```
What if we want an immutable Map. Due to `Map.of()` not accepting null values. Below would result in an error:
```
Map.of("x-client-id", params._0, "x-request-id", params._1);
```
How can we overcome this shortcoming? Extend & customize `Map`! Below is an example using Guava's ImmutableMap (which is a quite popular alternative esp. before Java 9)
```java
public static class CustomImmutableMapBuilder<K, V> extends ImmutableMap.Builder<K, V> {
    public CustomImmutableMapBuilder<K, V> putIfValNotNull(K key, V val) {
        if (val != null) super.put(key, val);
        return this;
    }

    @Override
    public CustomImmutableMapBuilder<K, V> put(@NonNull K key, @NonNull V val) {
        super.put(key, val);
        return this;
    }
}
```
