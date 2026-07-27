# Getting Started

This page walks you through installing Spring LogKit, setting it up in a Spring Boot project, and writing your first annotated service.

---

## Requirements

- Java 17 or later
- Spring Boot 3.4.x
- SLF4J logging implementation on the classpath (provided by Spring Boot)
- `spring-boot-starter-aop` in your project

---

## Installation

Add the Spring LogKit dependency to your `pom.xml`:

```xml
<dependency>
    <groupId>io.github.abhayrajaryan</groupId>
    <artifactId>spring-logkit</artifactId>
    <version>1.0.1</version>
</dependency>
```

Spring LogKit uses AspectJ under the hood, but it declares `aspectjweaver` as **provided** — meaning it won't be pulled transitively. You need to add `spring-boot-starter-aop` yourself:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

---

## Spring Boot Setup

No additional configuration is required. Spring LogKit provides auto-configuration that registers all framework beans automatically. As long as the dependency is on the classpath, the library is active.

Make sure your main application class has `@SpringBootApplication` or `@EnableAutoConfiguration`:

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

That's it. The library's auto-configuration picks up the component scan and registers:

- All seven logging aspects
- The `AnnotationResolver`
- The `AnnotationRegistry`
- The `StartupValidationEngine`

---

## Your First Annotated Service

Create a service and annotate it with `@LogBefore` and `@LogExecutionTime`:

```java
import io.github.abhayrajaryan.springlogkit.annotation.LogBefore;
import io.github.abhayrajaryan.springlogkit.annotation.LogExecutionTime;
import org.springframework.stereotype.Service;

@Service
@LogBefore
@LogExecutionTime
public class OrderService {

    public String createOrder(String userId, double amount) {
        // simulate business logic
        return "ORDER-123";
    }
}
```

Now call this service from a controller or a test:

```java
import org.springframework.web.bind.annotation.*;

@RestController
public class OrderController {

    private final OrderService orderService;

    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @GetMapping("/orders")
    public String createOrder() {
        return orderService.createOrder("user-1", 99.99);
    }
}
```

When `createOrder` is invoked, you'll see output like this:

```
INFO  [LogBeforeAspect] - Entering OrderService.createOrder(..)
INFO  [LogExecutionTimeAspect] - Method execution | START | method=OrderService.createOrder(..)
INFO  [LogExecutionTimeAspect] - Method execution | END | method=OrderService.createOrder(..) | executionTime=42.0 ms | returnValue=ORDER-123
```

---

## Verifying It Works

1. Start your Spring Boot application.
2. If there are annotation conflicts, the application **will not start** and you'll see an `AnnotationConflictException` in the logs.
3. If the application starts successfully, invoke a method on an annotated bean.
4. Check your logs for the output shown above.

---

## What's Next

- [Annotations](annotations.md) — learn about all seven annotations and how to use them
- [Customization](customization.md) — create your own annotations
- [Examples](examples.md) — complete, runnable code samples