# Examples

This page shows complete, runnable examples of Spring LogKit in action.

---

## Basic Usage

A service with class-level annotations. All methods inherit the logging behavior.

```java
import io.github.abhayrajaryan.springlogkit.annotation.LogBefore;
import io.github.abhayrajaryan.springlogkit.annotation.LogExecutionTime;
import org.springframework.stereotype.Service;

@Service
@LogBefore
@LogExecutionTime
public class ProductService {

    public String findById(Long id) {
        return "Product-" + id;
    }

    public List<String> findAll() {
        return List.of("Product-1", "Product-2");
    }
}
```

**Output when `findById(1L)` is called:**

```
INFO  [LogBeforeAspect] - Entering ProductService.findById(..)
INFO  [LogExecutionTimeAspect] - Method execution | START | method=ProductService.findById(..)
INFO  [LogExecutionTimeAspect] - Method execution | END | method=ProductService.findById(..) | executionTime=1.2 ms | returnValue=Product-1
```

---

## Multiple Annotations

Using annotations from different families on the same method.

```java
import io.github.abhayrajaryan.springlogkit.annotation.*;
import org.springframework.stereotype.Service;

@Service
public class OrderService {

    @LogBefore("Processing order")
    @LogAfterWithReturnValue("Order created")
    @LogExecutionTime("Order creation")
    public String createOrder(String userId, double amount) {
        // business logic
        return "ORDER-456";
    }
}
```

**Output:**

```
INFO  [LogBeforeAspect] - Processing order
INFO  [LogExecutionTimeAspect] - Order creation | START | method=OrderService.createOrder(..)
INFO  [LogExecutionTimeAspect] - Order creation | END | method=OrderService.createOrder(..) | executionTime=5.0 ms | returnValue=ORDER-456
INFO  [LogAfterWithReturnValueAspect] - Order created | returnValue=ORDER-456
```

All three annotations work together because they belong to different families (BEFORE, AROUND, AFTER).

---

## Exception Logging

Logging exceptions automatically.

```java
import io.github.abhayrajaryan.springlogkit.annotation.LogBefore;
import io.github.abhayrajaryan.springlogkit.annotation.LogException;
import org.springframework.stereotype.Service;

@Service
@LogBefore
@LogException
public class PaymentService {

    public void processPayment(String transactionId) {
        if (transactionId == null) {
            throw new IllegalArgumentException("Transaction ID must not be null");
        }
        // payment logic
    }
}
```

**Output when `processPayment(null)` is called:**

```
INFO  [LogBeforeAspect] - Entering PaymentService.processPayment(..)
ERROR [LogExceptionAspect] - Exception occurred | exception=java.lang.IllegalArgumentException: Transaction ID must not be null | method=PaymentService.processPayment(..)
```

The exception is logged at ERROR level with the full exception details.

---

## Execution Time Logging

Measuring how long a method takes.

```java
import io.github.abhayrajaryan.springlogkit.annotation.LogExecutionTime;
import org.springframework.stereotype.Service;

@Service
public class ReportService {

    @LogExecutionTime("Report generation")
    public byte[] generateMonthlyReport() {
        // simulate slow operation
        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        return new byte[]{1, 2, 3};
    }
}
```

**Output:**

```
INFO  [LogExecutionTimeAspect] - Report generation | START | method=ReportService.generateMonthlyReport(..)
INFO  [LogExecutionTimeAspect] - Report generation | END | method=ReportService.generateMonthlyReport(..) | executionTime=502.0 ms
```

---

## Custom Annotation

A complete example of creating and using a custom annotation.

### 1. Create the annotation

```java
import io.github.abhayrajaryan.springlogkit.annotation.SpringLogKitAnnotation;
import java.lang.annotation.*;

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@SpringLogKitAnnotation
public @interface LogAudit {
    String value() default "Audit log";
}
```

### 2. Register the annotation

```java
import io.github.abhayrajaryan.springlogkit.metadata.AnnotationFamily;
import io.github.abhayrajaryan.springlogkit.registry.AnnotationRegistry;
import jakarta.annotation.PostConstruct;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AuditConfig {

    private final AnnotationRegistry registry;

    public AuditConfig(AnnotationRegistry registry) {
        this.registry = registry;
    }

    @PostConstruct
    public void register() {
        registry.register(LogAudit.class, AnnotationFamily.BEFORE);
    }
}
```

### 3. Create the aspect

```java
import io.github.abhayrajaryan.springlogkit.metadata.AnnotationFamily;
import io.github.abhayrajaryan.springlogkit.resolver.AnnotationResolver;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class LogAuditAspect {

    private static final Logger log = LoggerFactory.getLogger(LogAuditAspect.class);
    private final AnnotationResolver resolver;

    public LogAuditAspect(AnnotationResolver resolver) {
        this.resolver = resolver;
    }

    @Before("@annotation(LogAudit) || @within(LogAudit)")
    public void logAudit(JoinPoint joinPoint) {
        LogAudit annotation = resolver.resolve(joinPoint, AnnotationFamily.BEFORE);
        String message = annotation != null ? annotation.value() : "Audit log";
        log.info("AUDIT: {} | method={} | args={}",
                message,
                joinPoint.getSignature().toShortString(),
                joinPoint.getArgs());
    }
}
```

### 4. Use the custom annotation

```java
import org.springframework.stereotype.Service;

@Service
public class AccountService {

    @LogAudit("Account deletion requested")
    public void deleteAccount(Long accountId) {
        // delete logic
    }
}
```

**Output when `deleteAccount(42L)` is called:**

```
INFO  [LogAuditAspect] - AUDIT: Account deletion requested | method=AccountService.deleteAccount(..) | args=[42]
```

---

## Method-Level Override

A class-level annotation overridden by a method-level annotation from a different family.

```java
import io.github.abhayrajaryan.springlogkit.annotation.*;
import org.springframework.stereotype.Service;

@LogBefore("Default before")
@LogExecutionTime
@Service
public class UserService {

    public void findAll() {
        // inherits class-level annotations
    }

    @LogException("User lookup failed")
    public User findById(Long id) {
        // inherits @LogBefore and @LogExecutionTime from class
        // adds @LogException from method
        return new User(id, "John");
    }

    @LogAfterWithReturnValue("User creation result")
    public User save(String name) {
        // uses @LogAfterWithReturnValue instead of @LogBefore
        // still inherits @LogExecutionTime from class
        return new User(1L, name);
    }
}
```

**Resolution at runtime:**

| Method | BEFORE | AROUND | AFTER | EXCEPTION |
|---|---|---|---|---|
| `findAll()` | Class-level `@LogBefore` | Class-level `@LogExecutionTime` | — | — |
| `findById()` | Class-level `@LogBefore` | Class-level `@LogExecutionTime` | — | Method-level `@LogException` |
| `save()` | Class-level `@LogBefore` | Class-level `@LogExecutionTime` | Method-level `@LogAfterWithReturnValue` | — |

---

## Without Spring Boot (Manual Setup)

If you're not using Spring Boot auto-configuration, you can manually register the beans:

```java
import io.github.abhayrajaryan.springlogkit.config.LogKitAutoConfiguration;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Import;

@Configuration
@Import(LogKitAutoConfiguration.class)
public class AppConfig {
    // your other beans
}
```

Or import individual components if you need fine-grained control.