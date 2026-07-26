# Internals

This page explains how Spring LogKit works under the hood. You don't need to know this to use the library, but it helps when things don't work as expected or when you want to extend it.

---

## Architecture Overview

The library has four main components:

1. **Annotations** — The seven logging annotations and the `@SpringLogKitAnnotation` meta-annotation
2. **Aspects** — One AspectJ aspect per annotation that intercepts method calls
3. **Registry** — Maps annotation types to their families
4. **Validation Engine** — Scans beans at startup and validates annotation configuration

These components are wired together by Spring auto-configuration.

---

## Startup Validation

When your Spring Boot application starts, the following sequence happens:

```
Spring Boot Starts
        |
        v
Create AnnotationRegistry (DefaultAnnotationRegistry)
        |
        v
Register Default Annotations (@PostConstruct)
   - BEFORE: LogBefore, LogBeforeWithArguments
   - AFTER: LogAfter, LogAfterWithReturnValue
   - AROUND: LogAround, LogExecutionTime
   - EXCEPTION: LogException
        |
        v
StartupValidationEngine (SmartInitializingSingleton)
        |
        v
Scan All Spring Beans for LogKit Annotations
        |
        v
Validate Annotation Family Conflicts
        |
        v
Freeze Annotation Registry
        |
        v
Application Ready
```

### Why Validation Exists

The validation engine exists to catch annotation conflicts **before** the application starts serving requests. Without it, a misconfigured annotation (like having both `@LogBefore` and `@LogBeforeWithArguments` on the same method) would silently pick one and ignore the other, or behave unpredictably.

The library takes a fail-fast approach: if your configuration is wrong, the application doesn't start. You fix it, then restart.

### How Validation Works

1. Spring calls `afterSingletonsInstantiated()` on the `StartupValidationEngine` (which implements `SmartInitializingSingleton`).
2. The engine iterates all bean definitions and inspects their class and method annotations.
3. For each annotated class, it checks that no two annotations from the same family appear together on the class-level element.
4. For each method, it checks the same — no two annotations from the same family on the same method.
5. Class-level and method-level annotations are **never compared against each other**. This is intentional: a class can have `@LogBefore` and a method can have `@LogBeforeWithArguments` without conflict.
6. If a conflict is found, an `AnnotationConflictException` is thrown and the application fails to start.

---

## Annotation Resolution

At runtime, when a method is invoked, the `AnnotationResolver` determines which annotation applies.

### Resolution Rules

1. **Check the method** for the annotation. If found, use it.
2. **Fall back to the class** if the method has no annotation.

### Resolution Flow

```
Method Invocation
        |
        v
Method has @LogBefore?  ---YES--> Use method-level annotation
        |
        NO
        v
Class has @LogBefore?  ---YES--> Use class-level annotation
        |
        NO
        v
No annotation -- no logging
```

### Why Method Takes Priority

Method-level annotations are more specific. If you annotate a class with `@LogBefore` but want one method to use `@LogBeforeWithArguments`, you put the more specific annotation on the method. The resolver respects that specificity.

### How the Resolver Works with Families

The resolver doesn't look for a specific annotation type. Instead, it:

1. Queries the registry for all annotation types in the requested family (e.g., BEFORE).
2. Checks if any of those types are present on the method.
3. If found, returns the first match.
4. If not found, checks the class level.
5. Returns `null` if no annotation from that family is found on either level.

This is why validation is important: if two BEFORE annotations were on the same method, the resolver would return whichever one it finds first, which is unpredictable.

---

## Method vs Class Annotation Priority

This is a common point of confusion, so let's be explicit.

**Validation is per-element.** Class-level annotations are validated independently of method-level annotations. This means:

```java
@LogBefore                          // class-level
public class UserService {

    @LogBeforeWithArguments          // method-level — VALID, validated separately
    public void save(String name) { }
}
```

This is valid because the class-level annotations are checked against each other (only `@LogBefore` — no conflict), and the method-level annotations are checked against each other (only `@LogBeforeWithArguments` — no conflict).

**Resolution is hierarchical.** At runtime, the method-level annotation wins:

- `save()` resolves to `@LogBeforeWithArguments` (method-level), not `@LogBefore` (class-level).

---

## Why the Registry Freezes

After validation passes, the registry is frozen. This prevents:

- Accidental registration of annotations at runtime
- Inconsistent state where some beans were validated with one set of annotations and later beans see a different set
- Thread-safety issues from concurrent registration

If you need to register custom annotations, do it before the validation engine runs — typically in a `@PostConstruct` method.

---

## Aspect Design

Each annotation has its own aspect class. For example:

- `@LogBefore` → `LogBeforeAspect`
- `@LogExecutionTime` → `LogExecutionTimeAspect`
- `@LogException` → `LogExceptionAspect`

Each aspect:

1. Defines a pointcut that matches the annotation (using `@annotation` for method-level and `@within` for class-level).
2. Injects the `AnnotationResolver` via constructor injection.
3. Calls `resolver.resolve(joinPoint, family)` to get the effective annotation.
4. Uses the annotation's `value()` to construct the log message.
5. Logs via SLF4J at the appropriate level (INFO for most, ERROR for exceptions).

The aspects are `@Component`-annotated and picked up by Spring's component scan, so they are automatically registered as beans.

---

## Auto-Configuration

The library provides a Spring Boot auto-configuration class that:

1. Creates the `DefaultAnnotationRegistry` bean
2. Creates the `AnnotationResolver` bean
3. Creates the `StartupValidationEngine` bean
4. Ensures all aspect beans are registered

No `@EnableAspectJAutoProxy` is needed — Spring Boot enables it automatically when `spring-boot-starter-aop` is on the classpath.

---

## Package Structure

```
io.github.abhayrajaryan.springlogkit/
├── annotation/       # @SpringLogKitAnnotation meta-annotation + 7 logging annotations
├── aspect/           # One aspect per annotation
├── config/           # Auto-configuration
├── exception/        # AnnotationConflictException, RegistryAlreadyFrozenException
├── metadata/         # AnnotationFamily enum
├── registry/         # AnnotationRegistry interface + DefaultAnnotationRegistry
├── resolver/         # AnnotationResolver
└── validation/       # StartupValidationEngine
```

Each package has a single responsibility. The separation makes it easy to understand, test, and extend.