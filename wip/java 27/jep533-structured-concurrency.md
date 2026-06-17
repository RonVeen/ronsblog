---
title: "JEP 533 - Structured Concurrency Gets Sharper in JDK 27"
date: 2026-06-22
draft: true
tags: ["java", "java27", "jdk27", "structured concurrency", "jep533"]
cover:
  image: "/images/jep533-structured-concurrency.png"
  alt: "Structured Concurrency in JDK 27"
series: ["Java 27"]
series_order: 5
categories: ["java"]
---
If you've been following the evolution of Project Loom, you'll know that Structured Concurrency has been on quite a journey. It started as an incubator module in JDK 19, became a preview API in JDK 21, and has now re-previewed five more times since — JDK 22, 23, 24, 25, and 26. JEP 533 is the seventh preview, targeting JDK 27, and this time the changes are meaningful enough that it's worth taking a proper look.

Let me walk you through where things stood, what felt awkward, and exactly what changed.

---

## The Story So Far

The core idea behind Structured Concurrency is elegant: if a task spawns subtasks, those subtasks should live and die within the same lexical scope as the parent. No rogue threads running after the parent method returns. No exceptions swallowed in the background. Clean, predictable lifetimes.


> To use some shameless self-promotion, my book on the subject can be found [here](https://www.amazon.com/Virtual-Threads-Structured-Concurrency-Scoped/dp/B0D5TTFSJ1). Though I must admit, it is slowly becoming a bit outdated. But I'll wait to update it until Structured Concurrency becomes a final feature.


The API that delivers this is `StructuredTaskScope`. You open a scope, fork subtasks into it, join them, and then process the results — all inside a `try`-with-resources block:

```java
try (var scope = StructuredTaskScope.open(Joiner.allSuccessfulOrThrow())) {
    Subtask<String> user    = scope.fork(() -> fetchUser(id));
    Subtask<Order>  order   = scope.fork(() -> fetchOrder(id));
    scope.join();
    return new Response(user.get(), order.get());
}
```

Simple. Readable. Exactly the kind of code that makes a conference talk slide look good.

But as with most preview features, the devil is in the details. Previous iterations had a few rough edges.

---

## What Felt Rough Before

### The exception type was invisible in the signature

The `join()` method has always been able to throw. But *what* it throws depended entirely on the `Joiner` you plugged in. In earlier previews, a joiner like `allSuccessfulOrThrow()` would surface failures as a `FailedException` — but that type lived in the Javadoc, not in the method signature. The compiler couldn't tell you any more than "this throws Exception."

If you wanted to catch a specific exception type from `join()`, you were on your own.

### Timeout behaviour was imprecise

The old `Joiner.onTimeout()` method cancelled the scope when the timeout elapsed, but it wasn't entirely clear — in the API itself — whether that would produce a result or blow up. The method name didn't tell you much, the Javadoc told you the rest, and it all felt a bit informal for something that will inevitably be used in production systems handling request deadlines.

### `awaitAll()` did too little

`Joiner.awaitAll()` waited for everything but gave you no way to handle failures. It was the "I just want all subtasks to complete, no matter what" factory method. Useful for fire-and-forget style work, but not really a join *policy* in any meaningful sense. It ended up being a footgun for people who wanted to inspect results but forgot it wouldn't surface failures.

---

## What Changed in JEP 533

### A New Type Parameter: `R_X`

This is the most structurally significant change. `StructuredTaskScope` and `Joiner` now have a third type parameter: `R_X`. Where the signature used to be `Joiner<T, R>`, it's now `Joiner<T, R, R_X>` — and `R_X` captures the exception type that `join()` is declared to throw.

Before:
```java
// join() just throws Exception — you had no type information
public T join() throws InterruptedException, Exception
```

After:
```java
// join() now throws exactly what the Joiner declares
public T join() throws InterruptedException, R_X
```

For application code that just uses the supplied joiners through `open()`, the compiler infers everything and your source looks the same as before. The difference shows up for library authors writing custom joiners — the `throws` clause is now part of the type itself, not something the implementation declares separately on the side.

Here's what it looks like in a full type-aware example:

```java
// The compiler knows join() throws ExecutionException here
try (var scope = StructuredTaskScope.open(Joiner.<String, ExecutionException>allSuccessfulOrThrow())) {
    Subtask<String> task1 = scope.fork(() -> callServiceA());
    Subtask<String> task2 = scope.fork(() -> callServiceB());

    try {
        scope.join();
    } catch (ExecutionException e) {
        // Compiler-verified — no guessing required
        throw new RuntimeException("A subtask failed", e.getCause());
    }

    return task1.get() + task2.get();
}
```

Much better. The contract is now in the signature, not buried in the documentation.

---

### A New Static `open` Method with `UnaryOperator`

`StructuredTaskScope` gains a new static `open` overload that accepts both a `Joiner` *and* a `UnaryOperator<StructuredTaskScope.Config>` to configure the scope. This lets you set a thread factory, a name, or other configuration options in a fluent, composable way — without needing to subclass `StructuredTaskScope`.

```java
try (var scope = StructuredTaskScope.open(
        Joiner.allSuccessfulOrThrow(),
        config -> config
            .withName("payment-scope")
            .withThreadFactory(Thread.ofVirtual().factory()))) {

    Subtask<PaymentResult> payment = scope.fork(() -> processPayment(order));
    Subtask<Receipt>       receipt = scope.fork(() -> generateReceipt(order));

    scope.join();

    return new Confirmation(payment.get(), receipt.get());
}
```

Previously you'd need to extend `StructuredTaskScope` to get at the configuration. Now you can do it inline, which is a lot more practical for application code where you don't want to litter your codebase with one-off subclasses.

---

### Updated `Joiner` Factory Methods — and Custom Exception Types

The three main factory methods — `allSuccessfulOrThrow()`, `anySuccessfulOrThrow()`, and `awaitAllSuccessfulOrThrow()` — now throw `ExecutionException` when the outcome of the scope is a failed subtask. If you're migrating from an earlier preview, this is the breaking change to watch for: code that used to catch `FailedException` needs to catch `ExecutionException` instead.

But there's more. Each of these three methods now has an overload that accepts a `Function` to map the failure into any exception type you choose. This is great for application code where you want failures from your structured scope to surface as domain exceptions rather than raw `ExecutionException`.

```java
// Default: throws ExecutionException on failure
try (var scope = StructuredTaskScope.open(Joiner.allSuccessfulOrThrow())) {
    // ...
    scope.join();
}

// Custom: map to your own exception type
try (var scope = StructuredTaskScope.open(
        Joiner.allSuccessfulOrThrow(cause -> new PaymentServiceException("Subtask failed", cause)))) {

    Subtask<PaymentResult> payment = scope.fork(() -> processPayment(order));
    scope.join(); // throws PaymentServiceException if the subtask fails
}
```

Your callers can now catch meaningful, domain-specific exceptions rather than catching `ExecutionException` and then digging into `.getCause()` to figure out what actually went wrong.

---

### `awaitAll()` Is Gone

`Joiner.awaitAll()` has been removed. This factory method waited for all subtasks to complete but never surfaced failures — it always returned normally, even if every subtask threw an exception. The result was that developers who used it thinking "I'll check the subtask outcomes after join()" would find that `join()` happily returned, and then discover the failures when they tried to call `subtask.get()`.

That's a surprising API. JEP 533 removes it entirely. If you want to wait for all subtasks to complete and then handle failures yourself, you now have better-defined options.

---

### `onTimeout()` Is Replaced by `timeout()`

The old `Joiner.onTimeout()` has been replaced with a new `timeout()` method. The difference is important: `timeout()` is explicit about what it does when the scope is cancelled by the timeout — it either returns a result or throws a `CancelledByTimeoutException` as the cause.

Before, `onTimeout()` would cancel the scope and return whatever partial result was available (or nothing), and you'd have to check whether the result was valid. Now the failure path is modelled as an exception, which is far more composable with standard Java error handling.

```java
// Fail fast: throw if the whole operation takes longer than 2 seconds
try (var scope = StructuredTaskScope.open(
        Joiner.anySuccessfulOrThrow(cause -> new TimeoutException("Operation timed out"))
              .timeout(Duration.ofSeconds(2)))) {

    Subtask<String> primary  = scope.fork(() -> fetchFromPrimary());
    Subtask<String> fallback = scope.fork(() -> fetchFromFallback());

    try {
        String result = scope.join();
        return result;
    } catch (TimeoutException e) {
        // Explicit, typed, caught here
        return defaultValue();
    }
}
```

If the timeout fires, `CancelledByTimeoutException` is the underlying cause, and your function maps it to whatever exception type makes sense for your context. This is much cleaner than the old approach of interrogating the result and guessing whether it was populated before the timeout hit.

---

## A Worked Example: Order Fulfillment

Here's a fuller example that combines the new features — the `UnaryOperator` config, the custom exception mapping, and the typed `join()` — into a pattern you might actually write in a Spring Boot service:

```java
@Service
public class OrderFulfillmentService {

    public FulfillmentResult fulfill(Order order) throws OrderFulfillmentException {
        try (var scope = StructuredTaskScope.open(
                Joiner.<FulfillmentResult, OrderFulfillmentException>allSuccessfulOrThrow(
                        cause -> new OrderFulfillmentException("Fulfillment subtask failed", cause)),
                config -> config
                    .withName("fulfillment-scope")
                    .withThreadFactory(Thread.ofVirtual().factory()))) {

            Subtask<PaymentResult>  payment  = scope.fork(() -> paymentService.charge(order));
            Subtask<InventoryResult> inventory = scope.fork(() -> inventoryService.reserve(order));
            Subtask<ShippingLabel>  shipping  = scope.fork(() -> shippingService.label(order));

            scope.join(); // throws OrderFulfillmentException if any subtask fails

            return new FulfillmentResult(payment.get(), inventory.get(), shipping.get());

        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new OrderFulfillmentException("Fulfillment interrupted", e);
        }
    }
}
```

Three subtasks, all running concurrently on virtual threads, scoped to the method body, with a domain exception surfaced on failure. Clean, readable, and now fully type-safe from `join()` to the caller's catch block.

---

## When Will You See This?

JEP 533 targets JDK 27, which is not an LTS release. That said, this is the seventh preview of an API that's been iterating since JDK 19 — six rounds of feedback is a lot of mileage, and the surface area is converging rather than churning. If you're on JDK 25 or 26, you can still use the current preview version of the API — just enable preview features in your build. The one thing to actually budget time for when you upgrade: hunt down every `catch (FailedException e)` in your codebase and change it to `ExecutionException`. That's a mechanical fix, but the compiler won't do it for you until you recompile against 27.

---

## Final Thoughts

Six previews in, and I'll admit I was starting to wonder if Structured Concurrency would ever sit still long enough to call final. JEP 533 doesn't settle that question, but it does make the wait feel worthwhile — the API finally tells you, in the type signature, what you're supposed to catch. That's not a flashy feature. It's the kind of change nobody puts on a slide. But it's exactly the kind of change that decides whether an API survives contact with a real codebase or just looks nice in a blog post.

Seems to me that the Structured Concurrency is quite close to being final now. I suspect one more round of reviews in Java 28 before it becomes final in Java 29. Which is the next LTS.
Seven previews might seem like a lot, but thats nothing compared to the Vector API.

So it looks like I can finally start working on the next edition of my book.
