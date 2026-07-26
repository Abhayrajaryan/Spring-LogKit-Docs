# Customization

Spring LogKit is designed to be extensible. You can create your own logging annotations and register them with the framework without modifying the library source code.

---

## Custom Annotations

You can create a custom annotation and have it recognized by Spring LogKit. The annotation must be annotated with `@SpringLogKitAnnotation` and registered with the `AnnotationRegistry`.

### Step 1: Create the Annotation

```java
import io.github.abhayrajaryan.springlogkit.annotation.SpringLogKitAnnotation;
import java.lang.annotation.*;

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@SpringLogKitAnnotation
public @interface LogSlowExecution {
    String value() default "Slow execution detected";
}
```

The `@SpringLogKitAnnotation` meta-annotation tells the framework that this is a logging annotation. Without it, the validation engine and resolver will not recognize it.

### Step 2: Register the Annotation

You need to register your annotation with the `AnnotationRegistry` before the registry is frozen. The freeze happens after startup validation completes, so registration must happen during bean initialization.

Create a configuration class that injects the `AnnotationRegistry` and registers your annotation:

```java
import io.github.abhayrajaryan.springlogkit.metadata.AnnotationFamily;
import io.github.abhayrajaryan.springlogkit.registry.AnnotationRegistry;
import jakarta.annotation.PostConstruct;
import org.springframework.context.annotation.Configuration;

@Configuration
public class LogKitCustomConfig {

    private final AnnotationRegistry registry;

    public LogKitCustomConfig(AnnotationRegistry registry) {
        this.registry = registry;
    }

    @PostConstruct
    public void registerCustomAnnotations() {
        registry.register(LogSlowExecution.class, AnnotationFamily.AROUND);
    }
}
```

### Step 3: Create the Aspect

Each annotation needs a corresponding aspect. The aspect defines the pointcut and the advice logic.

```java
import io.github.abhayrajaryan.springlogkit.metadata.AnnotationFamily;
import io.github.abhayrajaryan.springlogkit.resolver.AnnotationResolver;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class LogSlowExecutionAspect {

    private static final Logger log = LoggerFactory.getLogger(LogSlowExecutionAspect.class);
    private final AnnotationResolver resolver;

    public LogSlowExecutionAspect(AnnotationResolver resolver) {
        this.resolver = resolver;
    }

    @Around("@annotation(LogSlowExecution) || @within(LogSlowExecution)")
    public Object logSlowExecution(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            return joinPoint.proceed();
        } finally {
            long duration = System.currentTimeMillis() - start;
            if (duration > 1000) { // 1 second threshold
                LogSlowExecution annotation = resolver.resolve(joinPoint, AnnotationFamily.AROUND);
                String message = annotation != null ? annotation.value() : "Slow execution detected";
                log.warn("{} | method={} | duration={} ms", message, joinPoint.getSignature().toShortString(), duration);
            }
        }
    }
}
```

### Step 4: Use the Custom Annotation

```java
@Service
public class HeavyComputationService {

    @LogSlowExecution("Heavy computation took too long")
    public Result compute() {
        // slow logic
        return new Result();
    }
}
```

---

## Annotation Registry

The `AnnotationRegistry` is the central store that maps annotation types to their families. It is available as a Spring bean and can be injected anywhere in your application.

### Interface

```java
public interface AnnotationRegistry {
    void register(Class<? extends Annotation> annotationType, AnnotationFamily family);
    AnnotationFamily getFamily(Class<? extends Annotation> annotationType);
    Set<Class<? extends Annotation>> getAnnotations(AnnotationFamily family);
    boolean isFrozen();
}
```

### Key Methods

- **`register(annotationType, family)`** — Registers an annotation type under a family. Throws `RegistryAlreadyFrozenException` if called after the registry is frozen.
- **`getFamily(annotationType)`** — Returns the family for a given annotation type.
- **`getAnnotations(family)`** — Returns all annotation types registered under a family.
- **`isFrozen()`** — Returns `true` if the registry has been frozen.

### Freeze Behavior

The registry is frozen automatically by the `StartupValidationEngine` after successful validation. Once frozen, no new annotations can be registered. This prevents runtime configuration changes that could lead to inconsistent behavior.

If you need to register custom annotations, do it in a `@PostConstruct` method or a `BeanPostProcessor` — anything that runs before `SmartInitializingSingleton.afterSingletonsInstantiated()`.

---

## Extension Points

| Extension Point | What You Can Do |
|---|---|
| Custom annotation + aspect | Add entirely new logging behavior |
| Annotation registry injection | Register custom annotations programmatically |
| Custom message formatting | Override how log messages are constructed (by writing your own aspect) |

---

## When Customization Is Useful

- **Domain-specific logging** — You want annotations like `@LogAudit` or `@LogSecurityCheck` that carry domain meaning.
- **Conditional logging** — You want to log only when certain conditions are met (e.g., execution time exceeds a threshold).
- **Integration with monitoring** — You want to send metrics to Micrometer or push data to an external system alongside logging.
- **Custom log formats** — You want different log output formats (JSON, structured logging, etc.).

---

## Limitations

- Custom annotations must be registered before the registry freezes. There is no way to register annotations at runtime after the application has started.
- The validation engine validates all beans for annotation conflicts. Custom annotations are validated the same way as built-in ones.
- If you create a custom annotation in an existing family (e.g., a new BEFORE annotation), it will conflict with other BEFORE annotations on the same element.