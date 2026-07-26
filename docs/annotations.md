# Annotations

Spring LogKit provides seven built-in annotations. Each annotation belongs to one of four families, and annotations from the same family cannot be used together on the same class or method.

All annotations accept an optional `value()` parameter for a custom log message. When the value is blank, a default message is used.

---

## Annotation Families

| Family | Annotations |
|---|---|
| BEFORE | `@LogBefore`, `@LogBeforeWithArguments` |
| AFTER | `@LogAfter`, `@LogAfterWithReturnValue` |
| AROUND | `@LogAround`, `@LogExecutionTime` |
| EXCEPTION | `@LogException` |

---

## @LogBefore

Logs a message **before** a method executes.

**Purpose:** Trace method entry points. Useful for understanding the flow of execution through your services.

**Syntax:**

```java
@LogBefore
@LogBefore("Custom message before {method}")
```

**Example:**

```java
import io.github.abhayrajaryan.springlogkit.annotation.LogBefore;
import org.springframework.stereotype.Service;

@Service
public class PaymentService {

    @LogBefore("Processing payment in {method}")
    public void processPayment(String orderId) {
        // payment logic
    }
}
```

**Output:**

```
INFO  [LogBeforeAspect] - Processing payment in PaymentService.processPayment(..)
```

**Best practices:**

- Use `@LogBefore` on service-layer methods where you want to trace entry points.
- Add a custom message with `{method}` placeholder to make logs more readable.
- Avoid using on high-frequency methods in tight loops — every invocation produces a log statement.

**Common mistakes:**

- Using `@LogBefore` together with `@LogBeforeWithArguments` on the same method or class. They belong to the same family (BEFORE) and will cause an `AnnotationConflictException` at startup.

---

## @LogBeforeWithArguments

Logs a message **before** a method executes, including the method arguments.

**Purpose:** Trace method entry points with input values. Useful for debugging and auditing.

**Syntax:**

```java
@LogBeforeWithArguments
@LogBeforeWithArguments("Custom message")
```

**Example:**

```java
import io.github.abhayrajaryan.springlogkit.annotation.LogBeforeWithArguments;
import org.springframework.stereotype.Service;

@Service
public class UserService {

    @LogBeforeWithArguments("Creating user with args")
    public User createUser(String name, String email) {
        return new User(name, email);
    }
}
```

**Output:**

```
INFO  [LogBeforeWithArgumentsAspect] - Creating user with args | args=[name=John, email=john@example.com]
```

**Best practices:**

- Be careful with sensitive data. Arguments are logged as-is, so avoid logging passwords or personal information.
- Use on methods where input values are important for debugging.

**Common mistakes:**

- Combining with `@LogBefore` on the same element — they conflict because both are in the BEFORE family.

---

## @LogAfter

Logs a message **after** a method returns successfully.

**Purpose:** Trace method exit points. Useful for confirming that a method completed without throwing.

**Syntax:**

```java
@LogAfter
@LogAfter("Custom message after {method}")
```

**Example:**

```java
import io.github.abhayrajaryan.springlogkit.annotation.LogAfter;
import org.springframework.stereotype.Service;

@Service
public class EmailService {

    @LogAfter("Email sent from {method}")
    public void sendEmail(String to, String subject) {
        // send logic
    }
}
```

**Output:**

```
INFO  [LogAfterAspect] - Email sent from EmailService.sendEmail(..)
```

**Best practices:**

- Pair with `@LogBefore` on the same method to trace full entry/exit flow (they're in different families, so it's valid).
- Use on void methods where you want to confirm completion.

**Common mistakes:**

- Using with `@LogAfterWithReturnValue` on the same element — they conflict (both AFTER family).

---

## @LogAfterWithReturnValue

Logs a message **after** a method returns successfully, including the return value.

**Purpose:** Trace method exit points with the returned result. Useful for understanding what a method produced.

**Syntax:**

```java
@LogAfterWithReturnValue
@LogAfterWithReturnValue("Custom message")
```

**Example:**

```java
import io.github.abhayrajaryan.springlogkit.annotation.LogAfterWithReturnValue;
import org.springframework.stereotype.Service;

@Service
public class InventoryService {

    @LogAfterWithReturnValue("Stock check completed")
    public int checkStock(String productId) {
        return 42;
    }
}
```

**Output:**

```
INFO  [LogAfterWithReturnValueAspect] - Stock check completed | returnValue=42
```

**Best practices:**

- Use on query methods where the return value is meaningful.
- Avoid on methods returning large objects or collections — the entire return value is serialized to the log.

**Common mistakes:**

- Combining with `@LogAfter` on the same element (same family conflict).

---

## @LogAround

Logs a message **before and after** a method executes, wrapping the entire invocation.

**Purpose:** Trace the full lifecycle of a method call. Useful for understanding duration and flow in a single aspect.

**Syntax:**

```java
@LogAround
@LogAround("Custom message")
```

**Example:**

```java
import io.github.abhayrajaryan.springlogkit.annotation.LogAround;
import org.springframework.stereotype.Service;

@Service
public class ReportService {

    @LogAround("Report generation")
    public byte[] generateReport(String reportId) {
        // generate report
        return new byte[]{1, 2, 3};
    }
}
```

**Output:**

```
INFO  [LogAroundAspect] - Report generation | START | method=ReportService.generateReport(..)
INFO  [LogAroundAspect] - Report generation | END | method=ReportService.generateReport(..) | executionTime=15.0 ms
```

**Best practices:**

- Use when you want both start and end logging from a single annotation.
- The execution time is included automatically in the end message.

**Common mistakes:**

- Using with `@LogExecutionTime` on the same element — they conflict (both AROUND family). Since they produce similar output, pick one.

---

## @LogExecutionTime

Logs the execution time of a method, with start and end markers.

**Purpose:** Measure and log how long a method takes to execute. Useful for performance monitoring.

**Syntax:**

```java
@LogExecutionTime
@LogExecutionTime("Custom message")
```

**Example:**

```java
import io.github.abhayrajaryan.springlogkit.annotation.LogExecutionTime;
import org.springframework.stereotype.Service;

@Service
public class DataService {

    @LogExecutionTime("Data export")
    public void exportData() {
        // export logic
    }
}
```

**Output:**

```
INFO  [LogExecutionTimeAspect] - Data export | START | method=DataService.exportData(..)
INFO  [LogExecutionTimeAspect] - Data export | END | method=DataService.exportData(..) | executionTime=120.0 ms
```

**Best practices:**

- Use on methods where performance is a concern.
- Combine with `@LogBefore` or `@LogAfter` (different families) for richer tracing.
- The execution time is measured in milliseconds.

**Common mistakes:**

- Using with `@LogAround` on the same element — they conflict (both AROUND family). They produce similar output, so choose the one that fits your needs.

---

## @LogException

Logs a message **when a method throws an exception**.

**Purpose:** Automatically log exceptions with context. Useful for ensuring every exception is logged without writing try-catch blocks.

**Syntax:**

```java
@LogException
@LogException("Custom error message")
```

**Example:**

```java
import io.github.abhayrajaryan.springlogkit.annotation.LogException;
import org.springframework.stereotype.Service;

@Service
public class RiskService {

    @LogException("Risk assessment failed")
    public void assessRisk(String transactionId) {
        throw new RuntimeException("High risk transaction");
    }
}
```

**Output:**

```
ERROR [LogExceptionAspect] - Risk assessment failed | exception=java.lang.RuntimeException: High risk transaction | method=RiskService.assessRisk(..)
```

**Best practices:**

- Use on methods where exceptions are expected and you want automatic logging.
- Combine with other annotations from different families — for example, `@LogBefore` + `@LogException` gives you entry tracing and error logging.
- The exception is logged at ERROR level, so it stands out in your logs.

**Common mistakes:**

- `@LogException` is the only annotation in the EXCEPTION family, so it cannot conflict with any other built-in annotation. But if you create a custom EXCEPTION annotation, they would conflict.

---

## Annotation Placement

All annotations can be placed on:

- **Class level** (`ElementType.TYPE`) — applies to all methods in the class
- **Method level** (`ElementType.METHOD`) — applies only to that method

When both a class-level and method-level annotation exist, the method-level one takes precedence. See [Internals](internals.md) for the resolution logic.

---

## Custom Messages

Every annotation accepts a `value()` parameter for a custom log message:

```java
@LogBefore("Checking user permissions in {method}")
public void checkAccess(String userId) { }
```

The `{method}` placeholder is replaced with the fully qualified method signature at runtime. Not all annotations support the `{method}` placeholder — refer to each annotation's output format above.

---

## Summary Table

| Annotation | Family | When It Logs | Default Message |
|---|---|---|---|
| `@LogBefore` | BEFORE | Before execution | `Entering {method}` |
| `@LogBeforeWithArguments` | BEFORE | Before execution | `Method execution started` |
| `@LogAfter` | AFTER | After successful return | `Exiting {method}` |
| `@LogAfterWithReturnValue` | AFTER | After successful return | `Method execution completed` |
| `@LogAround` | AROUND | Before and after execution | `Method execution` |
| `@LogExecutionTime` | AROUND | Before and after execution | `Method execution` |
| `@LogException` | EXCEPTION | On exception | `Exception occurred` |