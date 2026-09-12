---
title: "JEP 539: Strict Field Initialization in the JVM (Preview)"
date: 2026-09-10
draft: true
tags: ["Java", "JEP", "JVM", "Project Valhalla", "Java 28"]
cover:
   image: "/images/jep539-strict-initialization-in-jvm.png"
   alt: "Strict Field Initialization in the JVM (Preview)"
series: ["Java 28"]
series_order: 2
description: "How JDK 28's strict field initialization closes the gap between what a final field promises and what the JVM actually enforces."
categories: ["ai", "java"]
---

Every Java field you've ever declared has a secret. Before your constructor or static initializer ever touches it, the JVM has already written something to it, a `0`, a `false`, a `null`. You never see it happen, but it happens, and it's the source of a whole category of bugs that are miserable to track down.

JEP 539 gives the JVM a way to close that gap. It's a preview feature targeted for JDK 28, and it's worth getting comfortable with even before you can use it directly, because a few other Java features are quietly built on top of it.

Case in point: JEP 401, the one about Project Valhalla and value classes, explicitly depends on this JEP. That's exactly why I'm covering strict initialization first and saving value classes for later in the series. And the reach of this one goes even further than value classes, non-nullable types are reportedly on the roadmap too. But let's not get ahead of ourselves.

## The bug that default values enable

Here's the JEP's own example, and it's a good one precisely because the bug is so unremarkable, the kind of circular dependency that sneaks into a codebase without anyone meaning for it to.

```java
class App {

    public static final long appID = Log.currentPID();

    public static void main() {
        IO.println("App[" + appID + "] has started");
        Log.log("Completed 'main'");
    }

}

class Log {

    private static final String prefix = "App[" + App.appID + "]: ";

    public static void log(String msg) {
        IO.println(prefix + msg);
    }

    public static long currentPID() {
        return ProcessHandle.current().pid();
    }

}
```

Run it, and you get something like this:

```
App[96052] has started
App[0]: Completed 'main'
```

Two different values out of one `final` field, in a single run. `App.appID` is `final`, you're supposed to be able to treat it like a constant, and yet `Log.prefix` walked away with a `0` that should never have existed.

What's going on is that calling `Log.currentPID()` from `App` triggers `Log`'s class initialization. At that point `App.appID` hasn't been assigned yet, it's still sitting at its default value, `0`. `Log.prefix` reads that `0` and bakes it into a string, for good. By the time `App.appID` finally gets its real value, `Log.prefix` has already made its mistake.

Reorder the classes and the bug vanishes, which is exactly what makes it nasty: the fix is invisible, and some unrelated change down the line can quietly bring it right back.

## What "strictly initialized" actually means

JEP 539 introduces a new field flag, `ACC_STRICT_INIT`, that a compiler can set on individual fields. Mark a field this way and it simply can't be read before it's been explicitly assigned, full stop. Combine that with `final`, and once it's been read, it can never be written to again.

For static fields, that means a `getstatic` on an unset strictly-initialized field throws rather than quietly handing you a `0` or `null`. For instance fields, the bytecode verifier enforces the same guarantee before the object's constructor even finishes, specifically before the `super()` call returns.

That second part matters more than it sounds. It means these fields have to be set while the object is still what the JEP calls "early larval", before it's handed off to any other code, including a superclass constructor. There's no window where a half-built object with strictly-initialized fields can leak out and get read by something else.

None of this needs a new keyword in the language. `javac` decides which fields qualify based on the language feature being compiled, it's a JVM-level guarantee, not something you opt into with a modifier on the field declaration.

## Why the JIT compiler cares

Here's the part that's easy to skim past but is actually the practical payoff. Once a `final` field is strictly initialized, the JVM knows, not hopes, not assumes, *knows*, that its value can never change after the first read. HotSpot's JIT compiler treats that as a trusted constant.

In practice: once compiled code has read a strictly-initialized final field, it can just reuse that value on every later read instead of going back to memory. Fewer memory accesses in hot paths. Small win per read, but it's the kind of small win that adds up.

## It's a preview feature, and that limits what you can do today

Just to load a class file with `ACC_STRICT_INIT` fields, you need `--enable-preview` at both compile time and run time. Value classes, the other JDK 28 preview feature quietly built on top of this one, rely on it directly, every field of a value class gets marked strict automatically.

Unless you're writing your own bytecode-emitting compiler or already poking at value classes, you won't touch this flag yourself. Still worth knowing it exists though, it explains behavior that would otherwise look confusing.

## The trade-offs you inherit

Strict fields clash with two things enterprise Java leans on constantly: deep reflection and Java serialization.

Deep reflection can't mutate a strictly-initialized final field anymore. `Field.setAccessible` now puts these fields in the non-modifiable bucket, right alongside static final fields and record fields. Try `Field.set` on one and you get an `IllegalAccessException`, and no, the `--enable-final-field-mutation` flag from JEP 500 doesn't get you around it. Want to set one? Go through a constructor. That's the only door left.

Serialization takes a harder hit. `ObjectInputStream` builds objects without ever calling a constructor, it does its own reflective construction instead. That's fundamentally at odds with fields that can only be set during the early-larval state of a real constructor call. So `ObjectOutputStream::writeObject` and `ObjectInputStream::readObject` throw `InvalidClassException` outright for any non-record class with strictly-initialized instance fields, unless you implement `writeReplace` and `readResolve` to swap in a different object along the way.

Honestly? I don't mind this one. Java's default serialization has been a footgun for years, and anything that pushes people toward explicit `writeReplace`/`readResolve`, or away from `ObjectInputStream` altogether, feels like a net win, even where it's inconvenient for existing code.

## What's next

Nothing to migrate here today, this is preview plumbing, not a switch you'll flip on existing classes. But it's exactly the plumbing that value classes, and eventually non-nullable field types, get built on top of. Worth having down now, so the next couple of JEPs in this series don't feel like they're assuming knowledge you haven't got yet.