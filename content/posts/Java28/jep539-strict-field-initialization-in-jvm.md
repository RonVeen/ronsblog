---
title: "JEP 539: Strict Field Initialization in the JVM (Preview)"
date: 2026-09-10
draft: true
tags: ["Java", "JEP", "JVM", "Project Valhalla", "Java 28"]
cover:
   image: "/images/jep539-strict-initialization-in-jvm.png"
   alt: "Spring AI Series: RAG End to End"
series: ["Java 28"]
series_order: 2
description: "How JDK 28's strict field initialization closes the gap between what a final field promises and what the JVM actually enforces."
categories: ["ai", "java"]
---

Every Java field you've ever declared has a secret. Before your constructor or static initializer ever touches it, the JVM has already written something to it. A `0`, a `false`, a `null`. You never see this happen, but it happens, and it's the source of a whole category of bugs that are miserable to track down.

JEP 539 gives the JVM a way to close that gap. It's a preview feature targeted for JDK 28, and it's worth understanding even before you can use it directly, because it's the foundation a few other Java features are quietly standing on.   
In fact, in JEP401, yes, you're right, that is the one about Project Valhalla and Value classes, it is stated that it depends on this JEP 539.   
And that is the reason why I am describing this JEP first, and the Value classes JEP at a later time.
But the outreach of this JEP goes even beyond Value classes towards non-nullable classes in the future.
But let's not get ahead of ourselves.

## The bug that default values enable

Here's the JEP's own example, and it's a good one because the bug in it is completely unremarkable. This is the kind of circular dependency that sneaks into a codebase without anyone intending it.

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

Two different values for a `final` field, in the same run. `App.appID` is `final`, it's supposed to be safe to treat as a constant, and yet `Log.prefix` captured a `0` that was never supposed to exist.

What happened is that calling `Log.currentPID()` from `App` triggers `Log`'s class initialization. At that point, `App.appID` hasn't been assigned yet, so it's sitting at its default value, `0`. `Log.prefix` reads that `0` and bakes it into a string, permanently. By the time `App.appID` actually gets its real value, `Log.prefix` has already made its mistake.

Reorder the classes, and the bug disappears. That's the worst kind of bug, one where the fix is invisible and unrelated code changes can silently reintroduce it.

## What "strictly initialized" actually means

JEP 539 introduces a new field flag, `ACC_STRICT_INIT`, that compilers can set on individual fields. A field marked this way can't be read before it's been explicitly assigned, full stop. If it's also `final`, once it's been read, it can never be written to again.

For static fields, that means a `getstatic` on an unset strictly-initialized field throws an exception rather than quietly handing back a `0` or `null`. For instance fields, the same guarantee is enforced by the bytecode verifier before the object's constructor even finishes, specifically, before the `super()` call returns.

That second part matters more than it sounds like. It means these fields have to be set while the object is still what the JEP calls "early larval", before it's been handed off to any other code, including a superclass constructor. There's no window where a half-built object with strictly-initialized fields can leak out and get read by something else.

None of this requires a new keyword in the language. `javac` decides which fields qualify based on the language feature being compiled, this is a JVM-level guarantee, not something you opt into with a modifier on the field declaration.

## Why the JIT compiler cares

Here's the part that's easy to skim past but is actually the practical payoff. Once a `final` field is strictly initialized, the JVM knows, not hopes, not assumes, *knows* that its value can never change after the first read. HotSpot's JIT compiler treats that as a trusted constant.

Practically, that means once compiled code has read a strictly-initialized final field once, it can reuse that value on every subsequent read without going back to memory. Fewer memory accesses in hot code paths. It's a small thing per read, but it's exactly the kind of small thing that compounds.

## It's a preview feature, and that limits what you can do today

To even load a class file with `ACC_STRICT_INIT` fields, you need `--enable-preview` at both compile time and run time. Value classes, the other JDK 28 preview feature this JEP quietly underpins, rely on it directly, every field of a value class gets marked strict automatically.

If you're not writing your own bytecode-emitting compiler and you're not experimenting with value classes yet, you won't touch this flag directly. But it's worth knowing it's there, because it explains behavior you'll otherwise find confusing.

## The trade-offs you inherit

Strict fields don't play nicely with two things enterprise Java leans on constantly, deep reflection and Java serialization.

Deep reflection can no longer mutate a strictly-initialized final field. `Field.setAccessible` now categorizes these fields as non-modifiable, the same bucket that static final fields and record fields already live in. Attempting `Field.set` on one throws `IllegalAccessException`, and the `--enable-final-field-mutation` flag from JEP 500 won't override it. If you need to set one, you go through a constructor. That's the only door.

Serialization takes a harder hit. `ObjectInputStream` builds objects without calling a constructor at all, it does its own reflective construction. That's fundamentally incompatible with fields that can only be set through the early-larval state of a real constructor call. So `ObjectOutputStream::writeObject` and `ObjectInputStream::readObject` throw `InvalidClassException` outright for any non-record class with strictly-initialized instance fields, unless you implement `writeReplace` and `readResolve` to substitute a different object during the process.

Honestly, I don't mind this one. Java's default serialization has been a footgun for years, anything that pushes people toward explicit `writeReplace`/`readResolve` or away from `ObjectInputStream` entirely feels like a net win, even if it's an inconvenient one for existing code.

## What's next

There's nothing to migrate here today, this is preview infrastructure, not a feature you'll be flipping on for existing classes. But it's the plumbing that value classes and, eventually, non-nullable field types are built on top of. Worth understanding now, so the next two or three JEPs in this series don't feel like they're assuming knowledge you don't have yet.
