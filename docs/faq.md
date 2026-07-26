# FAQ

---

## Why is my annotation not working?

The most common causes:

1. **`spring-boot-starter-aop` is missing.** Spring LogKit declares `aspectjweaver` as provided, so you need to add this dependency yourself.
2. **The method is not called through a Spring proxy.** Spring AOP only intercepts calls on Spring-managed beans when the method is called from outside the class. A method calling another method in the same class will not trigger the aspect.
3. **The bean is not managed by Spring.** Make sure the class is annotated with `@Service`, `@Component`, or another Spring stereotype.
4. **The application failed to start.** Check your startup logs for `AnnotationConflictException`. If validation failed, the application never became ready.

---

## Do I need AspectJ?

No. Spring LogKit uses Spring AOP, which works with Spring-managed beans and default proxy configurations. You do not need full AspectJ compilation or load-time weaving.

You do need `spring-boot-starter-aop`, which brings in the necessary AspectJ weaver dependency for Spring's proxy-based AOP.

---

## Can I create custom annotations?

Yes. See the [Customization](customization.md) page for a step-by-step guide.

The key requirements are:

1. Annotate your custom annotation with `@SpringLogKitAnnotation`
2. Register it with the `AnnotationRegistry` before the registry freezes
3. Create a corresponding aspect

---

## Can I use multiple annotations on the same method?

Yes, as long as they belong to different families. For example, `@LogBefore` (BEFORE) and `@LogExecutionTime` (AROUND) work together on the same method.

You cannot use two annotations from the same family, like `@LogBefore` and `@LogBeforeWithArguments`, on the same element. The startup validation will catch this and fail.

---

## How does annotation priority work?

At runtime, method-level annotations take precedence over class-level annotations. The resolver checks the method first, then falls back to the class.

Validation is separate: class-level and method-level annotations are validated independently. A class can have `@LogBefore` and a method can have `@LogBeforeWithArguments` without conflict — but at runtime, the method will use its own annotation.

---

## What happens if validation fails?

The application fails to start. An `AnnotationConflictException` is thrown with a message describing which annotations conflict and where they are located. You fix the conflict and restart.

This is intentional — the library prefers to fail at startup rather than behave unpredictably at runtime.

---

## Does the library work with reactive Spring (WebFlux)?

Spring LogKit is built on Spring AOP, which works with both blocking and reactive stacks at the proxy level. However, the aspects are designed for synchronous method interception. Reactive methods returning `Mono` or `Flux` may not produce the expected logging behavior because the advice runs at subscription time, not at the time the method returns.

If you're using WebFlux, test thoroughly before relying on the library for reactive components.

---

## Can I disable logging for a specific method in an annotated class?

If a class has class-level annotations, there's no built-in way to exclude a specific method. The class-level annotation applies to all methods.

Workarounds:

- Move the method to a separate, unannotated class.
- Use method-level annotations to override specific methods with an annotation from a different family (but you can't "un-annotate" a method).

---

## Does the library support Spring Boot 2.x?

No. Spring LogKit requires Spring Boot 3.4.x and Java 17 or later.

---

## How do I change the log level?

The log level is hardcoded in each aspect:

- Most aspects log at INFO level.
- `LogExceptionAspect` logs at ERROR level.

If you need a different log level, you have two options:

1. **Configure your logging framework** to change the level for specific logger names (e.g., set `io.github.abhayrajaryan.springlogkit.aspect.LogBeforeAspect` to DEBUG).
2. **Create a custom annotation and aspect** with your desired log level.

---

## Why is the registry frozen?

The registry is frozen after startup validation to prevent runtime configuration changes. This ensures:

- Consistent behavior throughout the application lifecycle
- Thread safety (no concurrent registration)
- Validation integrity (what was validated at startup is what runs)

If you need to register custom annotations, do it in a `@PostConstruct` method before the validation engine runs.

---

## What happens if I register an annotation after the registry is frozen?

You get a `RegistryAlreadyFrozenException`. This is thrown by `AnnotationRegistry.register()` when the registry has been frozen.

If you encounter this, move your registration code earlier in the application lifecycle.

---

## Does the library work with `@Transactional` or other Spring annotations?

Yes. Spring AOP aspects are ordered by default, and multiple aspects can stack on the same method. `@LogBefore`, `@LogExecutionTime`, and `@Transactional` work together on the same method.

There is no special configuration needed — Spring handles the aspect chain automatically.

---

## How do I contribute?

See the [Contributing section in the README](https://github.com/abhayrajaryan/spring-logkit) for details on adding new annotations and extending the library.