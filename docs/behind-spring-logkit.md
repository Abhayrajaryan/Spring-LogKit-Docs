# Behind Spring LogKit

Let me tell you the story of why this library exists.

---

## The Problem

I was working on a Spring Boot project, and like every project, we needed logging. Not just for debugging — we needed consistent, structured logging across all our services so we could trace requests, measure performance, and catch exceptions.

The standard approach was the same everywhere: inject a Logger, write `log.info("Entering methodX")`, `log.info("Exiting methodX")`, wrap things in try-catch to log exceptions, measure time with `System.currentTimeMillis()`. It worked, but it was repetitive. Every new service class meant copying the same patterns. And because it was manual, people forgot. Some methods had logging, others didn't. The format was inconsistent.

I wanted something better.

---

## Why Not Use Existing Solutions?

There are logging libraries out there. Aspect-oriented logging is not a new idea. But most solutions I found were either:

- **Too heavy** — full AOP frameworks with complex configuration
- **Too rigid** — fixed behavior that you couldn't extend
- **Too magical** — things happened, but you couldn't understand why or how to control them

I also looked at Spring's own `@Log` annotation from Lombok. It's fine for simple cases, but it doesn't give you control over when and what to log. You can't log before, after, around, and exceptions separately with different messages.

I wanted something that felt like Spring itself — annotation-driven, declarative, and easy to understand. Something where you look at a class and immediately see what gets logged.

---

## Learning Spring AOP

Honestly, the main reason I built Spring LogKit was to learn Spring AOP deeply.

I had used `@Transactional` and `@Cacheable` in projects. I understood the basics of proxies and pointcuts. But I had never built an AOP-based library from scratch. I wanted to understand:

- How do pointcut expressions really work?
- How does `@annotation` differ from `@within`?
- How do you resolve annotations at runtime from a join point?
- How does Spring's auto-configuration work for custom libraries?
- What does it take to publish a library to Maven Central?

Spring LogKit became my learning project. I set a rule: no shortcuts. I would build every piece myself, understand every line, and only use what Spring provides out of the box.

---

## Design Decisions

### Why Annotations?

Annotations are declarative. You put `@LogBefore` on a method, and logging happens. No configuration files, no programmatic setup, no boilerplate. It's the Spring way.

### Why Annotation Families?

The family system came from a practical problem. If you have `@LogBefore` and `@LogBeforeWithArguments`, they both want to run before the method. If you put both on the same method, which one wins? The answer is: neither should be allowed. They conflict.

So I grouped annotations into families. Same family on the same element = conflict. Different families = fine. This makes the rules clear and catches mistakes early.

### Why Startup Validation?

This was one of the best decisions I made. Early in development, I had a bug where two conflicting annotations silently picked one over the other. The behavior was unpredictable. I spent hours debugging.

I decided then: if the configuration is wrong, the application should not start. Fail fast. The `StartupValidationEngine` runs after all beans are initialized but before the application is ready. If there's a conflict, you see it immediately, not in production at 3 AM.

### Why Freeze the Registry?

The registry freeze is a direct consequence of startup validation. If validation passes with a certain set of annotations, those should be the annotations that run. If someone registers a new annotation later, the validation state becomes stale. Freezing prevents that.

It also makes the library thread-safe by design. No concurrent registration, no race conditions.

### Why One Aspect Per Annotation?

I considered having a single aspect that handles all annotations. It would be more DRY. But it would also be harder to understand, test, and extend.

Each aspect has one job. `LogBeforeAspect` handles `@LogBefore`. `LogExceptionAspect` handles `@LogException`. If you want to know how `@LogBefore` works, you read one file. If you want to add a new annotation, you create one aspect. Simple.

---

## Challenges

### Annotation Resolution

The hardest part was resolving annotations correctly at runtime. Spring AOP proxies can obscure annotations — a method on a proxy might not have the same annotations as the target method. I had to use Spring's `AnnotationUtils` and understand how `@annotation` and `@within` pointcuts work together.

The two-step resolution (method first, then class) seems simple in retrospect, but getting it right with proxies took several iterations.

### Validation Without Class Loading

The validation engine needs to inspect annotations on all beans. But at startup, not all classes are loaded yet. I had to work with bean definitions and use Spring's metadata reading capabilities to inspect annotations without forcing class loading.

### Testing with AOP

Testing AOP aspects is tricky. The aspect only works when the method is called through a Spring proxy. Unit tests that directly instantiate the class won't trigger the aspect. I had to write integration tests with `@SpringBootTest` and verify logging output, which is itself a challenge.

---

## Publishing to Maven Central

This was a whole separate journey. Setting up GPG keys, configuring the POM for Maven Central requirements, setting up the OSSRH repository, understanding the staging and release process. The first successful publish felt like a bigger achievement than writing the library itself.

The lesson: publishing a library is 30% code and 70% infrastructure. Documentation, licensing, group IDs, artifact names, signing — there's a lot that has nothing to do with the actual code.

---

## What I Learned

- **Spring AOP is powerful but has sharp edges.** Proxy-based AOP has limitations (self-invocation, final methods, private methods). Understanding these limitations is essential.
- **Fail fast is worth the effort.** The validation engine catches mistakes that would otherwise be silent runtime bugs.
- **Simple is better.** I could have made the library more configurable, more flexible, more abstract. But the current design is easy to understand, easy to test, and easy to extend. That's worth more than flexibility.
- **Documentation is part of the product.** Writing this documentation forced me to clarify my own thinking about the design.

---

## Future Plans

- **Configuration properties** — let users enable or disable specific annotation families via `application.properties`
- **Micrometer integration** — expose execution time as metrics automatically
- **Correlation IDs** — add support for tracing context in log messages
- **Better reactive support** — make the library work well with WebFlux

But honestly, the library does what I built it to do. Everything else is a bonus.

---

If you've read this far, thank you. I hope Spring LogKit makes your logging a little easier. If you have questions or ideas, the GitHub repository is open for issues and contributions.