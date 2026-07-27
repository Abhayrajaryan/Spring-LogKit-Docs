# Spring LogKit

Spring LogKit is an annotation-driven logging library for Spring Boot applications. It uses Spring AOP to intercept method executions and log before, after, around, exception, and execution-time information based on declarative annotations placed on classes or methods.

The library validates annotation configuration at application startup, catching conflicts before any request is processed.

---

## Features

- **Annotation-driven logging** — seven built-in annotations for common logging scenarios
- **Spring AOP-based interception** — no manual proxy configuration needed
- **Method-level and class-level support** — annotate a whole class or individual methods
- **Automatic annotation resolution** — method-level annotations take precedence over class-level ones
- **Startup validation** — detects annotation family conflicts before the application becomes ready
- **Annotation registry** — freezes after initialization to prevent runtime configuration changes
- **Custom annotation support** — extend the library with your own annotations via the registry API
- **SLF4J-based output** — integrates with your existing logging framework
- **Zero configuration** — auto-configuration registers all framework beans

---

## Maven Dependency

```xml
<dependency>
    <groupId>io.github.abhayrajaryan</groupId>
    <artifactId>spring-logkit</artifactId>
    <version>1.0.1</version>
</dependency>
```

You also need `spring-boot-starter-aop` in your project:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

---

## Quick Example

Annotate a service class or method with a logging annotation. The library intercepts the method call and logs the appropriate information.

```java
import io.github.abhayrajaryan.springlogkit.annotation.LogBefore;
import io.github.abhayrajaryan.springlogkit.annotation.LogExecutionTime;
import org.springframework.stereotype.Service;

@Service
@LogBefore
@LogExecutionTime
public class OrderService {

    public String createOrder(String userId, double amount) {
        // business logic
        return "ORDER-123";
    }
}
```

When `createOrder` is invoked, the following output is produced:

```
INFO  [LogBeforeAspect] - Entering OrderService.createOrder(..)
INFO  [LogExecutionTimeAspect] - Method execution | START | method=OrderService.createOrder(..)
INFO  [LogExecutionTimeAspect] - Method execution | END | method=OrderService.createOrder(..) | executionTime=42.0 ms | returnValue=ORDER-123
```

---

## Next Steps

- [Getting Started](getting-started.md) — install, configure, and write your first annotated service
- [Annotations](annotations.md) — detailed reference for every annotation
- [Customization](customization.md) — create custom annotations and extend the library
- [Internals](internals.md) — how the library works under the hood
- [Examples](examples.md) — complete, runnable examples
- [FAQ](faq.md) — common questions and troubleshooting
- [Behind Spring LogKit](behind-spring-logkit.md) — the story of why this library exists