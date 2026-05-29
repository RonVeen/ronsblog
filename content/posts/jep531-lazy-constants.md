---
title: "Java 27 - part 3: JEP 531 - Lazy Constants, Round Three"
date: 2026-05-27
draft: false
tags: ["java", "java27", "jdk27", "lazy constants", "jvm", "jep531"]
cover:
  image: "/images/jpe531-lazy-constants.png"
  alt: "Lazy Constants"
series: ["Java 27"]
series_order: 4
---
**Third Time's the Charm: Lazy Constants Get a Trim and a New Trick**

Back in the Java 26 series, I walked you through JEP 526 and the (then) second preview of the Lazy Constants API. If you missed it, the short version is this: Java finally got a clean way to do deferred immutability — fields that initialize on demand but, once set, are treated by the JVM as if they'd been hard-coded constants. No more double-checked locking. No more holder idioms. Just `LazyConstant.of(...)` and a `.get()`.

JEP 531, targeting JDK 27, is the third preview. It doesn't reinvent anything — it sharpens what we already had and adds one nice little extension to the collections family. Let's take a look.

**A quick recap of why this exists**

The whole point of Lazy Constants is to stop forcing you to choose between two bad options. Either you `final` everything and pay the startup cost up front, or you go mutable and the JIT can't optimize your hot path because the field "might change." Lazy Constants give you the best of both: deferred initialization, thread-safe by default, and once initialized the JVM treats them as proper constants (which means constant folding — the same trick the JDK has been using internally for years).

If you want the full backstory, jump over to my [JEP 526 post](https://ronveen.com/posts/jep526-lazy-constants/). Otherwise — let's see what changed.

**What's new in JEP 531?**

Two changes. One subtraction, one addition.

**1. Goodbye, `isInitialized()` and `orElse()`**

These two methods are gone. The JDK team noticed that developers were using them in ways that fought against the design of the API — basically reintroducing the "is it null yet?" branching that Lazy Constants were created to eliminate. So they pulled them.

```java
// Java 26 / JEP 526 — no longer compiles in Java 27
if (!myLazyConstant.isInitialized()) {
    System.out.println("Not ready yet!");
}
Logger logger = myLazyConstant.orElse(defaultLogger);

// Java 27 / JEP 531 — just call .get()
Logger logger = myLazyConstant.get();
```

If you were relying on `isInitialized()` for diagnostics, you're going to need to rethink that. The philosophy is sharper now: define the supplier up front, call `.get()`, let the API do its job.

**2. Hello, `Set.ofLazy(...)`**

This is the fun one. The lazy collections family now includes Sets, joining `List.ofLazy(...)` and `Map.ofLazy(...)`. The trifecta is complete.

The idea is the same as before: you hand it a set of candidate elements and a predicate, and membership gets evaluated on demand. Perfect for configuration flags, feature toggles, or anywhere you have a fixed universe of options but figuring out which ones are "in" is expensive.

```java
class Application {
    enum Option { VERBOSE, DRY_RUN, STRICT }

    private static boolean isEnabled(Option option) {
        // Parse the command line, hit a config file, query the database…
        return true;
    }

    // None of the predicates have run yet.
    static final Set<Option> OPTIONS =
            Set.ofLazy(EnumSet.allOf(Option.class), Application::isEnabled);

    public static void process() {
        // First call to contains() for DRY_RUN triggers isEnabled(DRY_RUN).
        // Subsequent calls return the cached answer instantly.
        if (OPTIONS.contains(Option.DRY_RUN)) {
            return;
        }
        // … actual processing
    }
}
```

The thing I like about this is how naturally it fits enums. You enumerate every possible option once, hand it a predicate, and you never have to think about caching, initialization order, or thread safety again.

**Taking it for a spin**

Lazy Constants are still a preview feature in JDK 27, so you'll need to grab the Early Access build and flip the `--enable-preview` flag to play with them.

**Wrapping up**

JEP 531 is not a flashy update — it's a polishing pass. One removal that nudges you toward using the API the way it was designed, and one addition that fills in the last obvious gap. If you were already using Lazy Constants in JDK 26, the migration is pretty much free. If you weren't, the JDK 27 version is the cleanest yet, and a good moment to start.

I'll be watching to see if this is the preview that finally goes final.
