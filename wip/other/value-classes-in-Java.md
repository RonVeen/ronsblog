---
title: "Value Classes are coming to Java"
date: 2026-06-30
draft: true
tags": ["Java", "JVM", "Data-Oriented Programming", "JEP 401", "Value Classes", "Project Valhalla"]
cover:
  image: "/images/value-classes.png"
  alt: "Value Classes in Java"
description: "Learn what value classes are, why == finally stops lying to you about LocalDate, and where the JVM gets its speed boost from."
categories: ["java"]
---

I once spent a good twenty minutes debugging a failing test before realizing the bug was `d1 == d3` instead of `d1.equals(d3)`. Both were `LocalDate.of(1996, 1, 23)`. Both represented the exact same date. And Java looked me dead in the eye and said "false."

That's not a bug in `LocalDate`. That's just what happens when every object in Java carries an identity, whether it needs one or not. JEP 401 is the JEP that finally lets you opt out.

## The problem with everything having an identity

Java's object model assumes identity by default. That makes sense for a `User`, a database row, a session — things that genuinely *are* something, distinct from other things that happen to look the same.

But most of the objects flying around your codebase aren't that. A `Money` amount, a coordinate, a timestamp, a percentage — these are just data. Two `Money` instances holding `10 EUR` aren't *different euros*. They're the same value, represented twice. Identity adds nothing here except the occasional `==` landmine.

JEP 401 gives you a way to say that out loud to the compiler: "this type has no identity, it's just a value."

## What a value class actually is

The JEP keeps the definition refreshingly short: a value object is a class instance that has only final fields and lacks object identity. That's it. No mutation, no identity, no surprises.

```java
public value class Point {
    private final int x;
    private final int y;

    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }

    public int x() { return x; }
    public int y() { return y; }
}
```

Two `Point` instances built from the same `x` and `y` are, as far as the language is concerned, the same value — not two distinct objects that happen to agree on their contents.

One thing worth being precise about: JEP 401 doesn't make `==` behave like `.equals()`. It's explicitly not a goal to "fix" `==` so it can replace `equals` — the JEP only changes `==` semantics as much as necessary to handle the new kind of object. You'll still want `equals()` for value comparison. The win is more subtle: the *concept* of identity stops being part of the type's contract at all, which is what lets the JVM get clever later.

## A more realistic example

```java
public value class Money {
    private final String currency;
    private final long amountInMinorUnits;

    public Money(String currency, long amountInMinorUnits) {
        this.currency = currency;
        this.amountInMinorUnits = amountInMinorUnits;
    }

    public String currency() { return currency; }
    public long amountInMinorUnits() { return amountInMinorUnits; }

    public Money add(Money other) {
        if (!currency.equals(other.currency)) {
            throw new IllegalArgumentException("Currency mismatch");
        }
        return new Money(currency, amountInMinorUnits + other.amountInMinorUnits);
    }
}
```

Nothing about `Money` needs identity. Two instances with the same currency and amount are interchangeable in every sense that matters to your domain. That's exactly the shape value classes are built for.

## Why the JVM cares

This isn't just a language-level nicety — it's an optimization door the JVM has wanted opened for years. Two techniques are doing the heavy lifting here: scalarization and heap flattening. They attack the same problem from two different angles, so it's worth pulling them apart.

### Scalarization: making the object disappear at compile time

Normally, when you call a method that takes a `Point`, the JVM is passing around a reference — a pointer to wherever that `Point` object lives on the heap. Scalarization lets the JIT compiler skip that step entirely for value objects, by breaking the object down into its raw field values and passing *those* around instead.

Take a method like this:

```java
public Point add(Point other) {
    return new Point(this.x + other.x, this.y + other.y);
}
```

Conceptually, once `Point` is a value class, the JIT can compile this as if it were written:

```java
// effectively, after scalarization:
static int[] add(int this_x, int this_y, int other_x, int other_y) {
    return new int[] { this_x + other_x, this_y + other_y };
}
```

No `Point` object ever gets allocated. The `x` and `y` values just travel through the method as if they were `int` parameters — because that's effectively all they are now. This only kicks in within JIT-compiled code, and it unravels back into a normal heap object the moment the value crosses into code the JIT can't see through (a generic method typed as `Object`, for instance). But inside a hot loop doing `Point` arithmetic, this is the difference between allocating millions of objects and allocating none.

### Heap flattening: making the object disappear in memory

Scalarization helps while code is running. Heap flattening helps once a value object actually needs to be *stored* — in a field, or in an array.

Instead of an array of `Point` storing 100 pointers to 100 separate objects scattered across the heap, a flattened `Point[]` can store the raw `x`/`y` bytes directly, back-to-back, inline in the array itself. No pointer chasing, no separate allocations, and — critically for performance — much better cache locality, since the CPU can pull a whole run of points into cache in one go instead of following 100 scattered references.

There's a real constraint here worth knowing about: the flattened representation has to be small enough to read and write atomically, which on most platforms tops out around 64 bits including a null-tracking flag. So a tiny class like `Point` (two `int`s) is a great flattening candidate, while something with two `int` fields *and* a `double` might not flatten today. That ceiling is expected to rise — 128-bit flattening is on the table for platforms that support atomic access at that size, and a separate Null-Restricted Value Types JEP aims to flatten even larger value classes for code willing to give up some of those atomicity guarantees.

### Why this matters for you

Put together, the advantages are concrete, not theoretical:

- **Less garbage collector pressure.** Objects that never get allocated never need to be collected. If your code creates `Money` or coordinate objects in a hot loop today, a chunk of that allocation simply stops happening.
- **Better cache behavior.** Flattened arrays of value objects behave like arrays of primitives from the CPU's perspective — sequential, predictable, cache-friendly — instead of a scattered web of pointers.
- **No abstraction tax.** This is the part I find genuinely satisfying: you don't have to give up the `Money` or `Point` class and fall back to raw `int[]` arrays to get this performance, which used to be the only way to "program for speed" in Java. You keep the named fields, the validation in the constructor, the convenience methods — and the JVM still gets to treat the data like primitives under the hood.
- **It scales with how many you create.** The benefit is proportional to volume. A handful of `Money` objects won't move the needle; millions of them in a pricing engine or a billing pipeline absolutely will.

That last point is really the headline: this feature pays off most exactly where Java has traditionally been weakest — code that creates enormous numbers of small, short-lived, immutable objects.

## Records vs. value classes

Records already nudged Java toward "this is just data." They generate `equals`, `hashCode`, `toString` for you and signal intent clearly. But a record is still, underneath, an ordinary object with identity — it just hides that fact well.

A value class goes one step further: it doesn't hide identity, it *removes* it. A record says "this is data." A value class says "this is data, and don't even think about asking what its identity is" — and that distinction is exactly what the JVM needs to flatten and scalarize it.

## Who's actually getting migrated

This isn't a theoretical feature aimed at app developers writing their own `Point` classes for fun. The JDK itself plans to migrate suitable existing classes — `Integer` and `LocalDate` among them — to be value classes. Which means that `==` landmine I stepped on earlier? Once `LocalDate` becomes a value class with proper identity-free semantics, that whole category of bug gets a lot less sharp.

## When to reach for one (and when not to)

Good candidates: immutable types, defined entirely by their contents, created in bulk, and conceptually interchangeable — coordinates, money, durations, timestamps.

Bad candidates: anything that needs stable identity, synchronization, mutable lifecycle state, or has to play nice with APIs that depend on reference equality. If the type is really an entity wearing a data-class costume, leave it as a regular object.

## Trying it yourself

This is a preview feature, so you'll need a recent early-access build and the `--enable-preview` flag — don't expect `public value class` to compile on whatever JDK you've got installed today. The integration into OpenJDK mainline is targeting JDK 28, and it's apparently a 197,000-line change, so "still settling" is an understatement.

If you've been burned by an unexpected `false` from `==` on something that was obviously the same value, this is the JEP that's finally coming for that bug. Go grab the Valhalla early-access build and run your own `Point` and `Money` examples through it — seeing the JVM treat them as values instead of objects is the kind of thing that's more convincing in person than on a blog.
