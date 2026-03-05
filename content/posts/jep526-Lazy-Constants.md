---
title: "Java 26 - part 8: JEP 526 - Lazy Constants"
date: 2026-03-01
draft: false
tags: ["Java", "JEP", "Java 26"]
cover:
  image: "/images/jep526-lazy-constants.png"
  alt: "Lazy Constants"
series: ["Java 26"]
---

**Bye-Bye Boilerplate: Say Hello to Java’s New "Lazy Constants"**
If you’ve ever had to initialize a heavy object only when it’s actually needed, you know the drill. 
You probably reached for the "Double-Checked Locking" pattern (which is easy to mess up) or the "Initialization-on-demand Holder" idiom (which is verbose).

JEP 526 (Lazy Constants), coming as a second preview in JDK 26, is here to retire those old hacks. 
It introduces a first-class way to handle deferred initialization that is thread-safe, easy to read, and—best of all—lightning fast.

**What’s the big deal?**
In the past, you had two choices:
- `final` fields: Great for performance (the JVM loves constants), but they must be initialized immediately.
- Mutable fields: Flexible, but the JVM can’t optimize them as well because it thinks they might change.

Lazy Constants give you the best of both worlds. 
They are initialized lazily (on demand) but, once set, the JVM treats them as "stable" constants. 
This allows the JIT compiler to perform "constant folding," making access almost as fast as a hard-coded value.

**The New API in Action**
The star of the show is `java.lang.LazyConstant<T>`. 
You define it with a supplier (the logic to create the value), and it handles the rest.

1. Basic Usage
   No more `if (logger == null) synchronized....` Just declare it and call `.get()`.

```Java
import java.lang.LazyConstant;

public class OrderService {
// 1. Define the lazy constant with a factory/supplier
private static final LazyConstant<Logger> LOGGER =
LazyConstant.of(() -> Logger.getLogger("OrderService"));

    public void processOrder(String id) {
        // 2. The first thread to call .get() triggers initialization.
        // Subsequent calls return the cached value instantly.
        LOGGER.get().info("Processing order: " + id);
    }
}
```

2. Lazy Collections (The "Cool" Part)
   JEP 526 doesn’t just stop at single values. It introduces Lazy Lists and Lazy Maps. This is perfect for when you have a massive collection where calculating each element is expensive.

```Java
import java.util.List;

public class ConfigManager {
// Creates a list of 1000 items, but NONE are created yet.
private final List<ComplexConfig> configs =
List.ofLazy(1000, index -> loadConfigFromDB(index));

    public void useConfig(int index) {
        // Only the config at 'index' is initialized here. 
        // The other 999 remain uninitialized.
        var config = configs.get(index);
    }
}
```

**Why you’ll love it:**
- Thread Safety for Free: You don't need synchronized or volatile. The API guarantees the supplier runs at most once.
- Startup Speed: By deferring heavy object creation until they are actually used, your app starts up much faster.
- No Nulls allowed: Unlike traditional lazy loading where null might mean "not initialized," LazyConstant forbids null values. This keeps the logic clean and avoids ambiguity.
- JVM Optimizations: Because these are "stable," the JVM can optimize the code path as if you had used a static final field.

**What changes in JEP 526?**

JEP 526 is the second preview of what was previously known as Stable Values (introduced in JEP 502). 
While the core mission remains the same—providing thread-safe, lazy initialization that the JVM can optimize like a constant—the API underwent a significant "philosophical shift" to make it simpler and harder to misuse.

Here are the four key changes compared to its predecessor:

1. **The Name Change (Rebranding)**  
The initial JEP was called Stable Values (StableValue), which is not renaed to Lazy Constants (LazyConstant).
The new name is more intuitive for everyday developers. 
"Stable" was a low-level JVM term; "Lazy Constant" clearly describes the high-level intent: it's lazy to start but permanent once set.

2. **Disallowing `null`**  
In the first preview, a stable value could be initialized to null. 
In JEP 526, `null` is strictly forbidden as a computed value.
This removes ambiguity (is it null because it failed, or because null is the value?). 
It also allows the JVM to skip "null-check" branches in the code, making it even faster. 
If you need a null-like state, the JEP now encourages you to use `LazyConstant<Optional<T>>`.

3. **API Simplification (Removal of "Manual" Setters)**  
The first preview had low-level methods like `setOrThrow()`, `trySet()`, and `orElseSet()`.
This new JEP centers almost entirely on a factory-based approach using `LazyConstant.of(Supplier)`.
The previous version felt like a synchronization primitive (similar to a Future or AtomicReference). 
The new version feels like a declarative configuration. 
It pushes you to define how to build the value upfront rather than managing the state transitions yourself.

4. **Moving Lazy Collections**  
The factory methods for creating lazy lists and maps were moved directly into the standard `java.util.List`and `java.util.Map` interfaces (e.g., `List.ofLazy(...)`).
This makes the feature much more "discoverable." 
Most developers look at the List interface when they want a collection, so putting the lazy variant there makes it a first-class citizen of the Collections Framework.

In short: JEP 526 moved away from being a "low-level tool for library experts" to a "high-level tool for all Java developers."

*Wait time:* This is a Preview feature in JDK 26, so you'll need to use the --enable-preview flag to take it for a spin!