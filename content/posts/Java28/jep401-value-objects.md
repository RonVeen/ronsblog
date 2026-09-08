---
title: "JEP 401: Value Objects (Preview)"
date: "2026-09-18"
draft: true
tags: ["Java", "JEP", "JVM"]
series: ["Java 28"]
series_order: 3
cover:
  image: "/images/jep401-value-objects.png"
  alt: "JEP 401: Value Objects (Preview)"
description: "How JDK 28's value objects let you opt out of identity for immutable data, and why that's more than a performance trick."
---

Take a `LocalDate`. Two instances representing the same year, month, and day are `equals()`, but they're not `==`. That's always felt slightly wrong, and JEP 401 is the JDK finally admitting it.

## Identity was never the point for immutable data

Every object in Java has identity, whether it needs it or not. For mutable state, identity earns its keep, it's how you tell apart two things that happen to look the same right now but might diverge later. A text editor's `Line` objects are a good example, two lines might contain the same characters today, but you need to know which one the user is about to edit.

Immutable data doesn't have that problem. A `LocalDate` representing `1996-01-23` is never going to become a different date. There's nothing to distinguish two instances of it except an identity that exists purely because the language requires one.

```jshell
jshell> LocalDate d1 = LocalDate.of(1996, 1, 23)
d1 ==> 1996-01-23

jshell> LocalDate d3 = d1.plusYears(30).minusYears(30)
d3 ==> 1996-01-23

jshell> d1.equals(d3)
$4 ==> true

jshell> d1 == d3
$6 ==> false
```

`equals()` says yes, `==` says no. Every Java developer has been bitten by this at least once, usually with `Integer`, which makes the inconsistency even worse because it only sometimes holds:

```jshell
jshell> Integer i = 1, j = 1;
jshell> i == j
$3 ==> true

jshell> Integer x = 1996, y = 1996;
jshell> x == y
$6 ==> false
```

`Integer` caches small values, so `1 == 1` happens to work by accident, and `1996 == 1996` doesn't. That's not a language guarantee, it's an implementation detail leaking through the `==` operator, and it's exactly the kind of thing that makes new developers distrust their own mental model of the language.

There's also a cost angle, not just a correctness one. Requiring identity for something as simple as a date means the JVM has to allocate heap memory for every instance and chase a pointer every time you read one, even though the entire useful state of a `LocalDate` would fit in 48 bits. Arrays of `LocalDate` end up as arrays of pointers to scattered heap objects, instead of a tight block of primitive data, and that hurts cache locality on top of the allocation cost.

## What a value class actually is

JEP 401 introduces the `value` modifier. Apply it to a class, and instances of that class become value objects, immutable and interchangeable, with `==` comparing field values instead of memory addresses.

```java
value class EURCurrency {

    private long cs;  // implicitly final

    private EURCurrency(long cs) { this.cs = cs; }

    public EURCurrency(long e, int c, boolean neg) {
        this(neg ? -e * 100 - c : e * 100 + c);
    }

    public long euros() { return Math.abs(cs) / 100; }
    public int cents() { return (int) Math.abs(cs) % 100; }
}
```

A `value` class is implicitly `final`, both the class itself and every one of its fields. That's not a stylistic choice, it's a requirement, mutability and identity-free semantics don't mix. Fields can hold anything, including references to ordinary identity objects like `String`, there's no restriction there.

Records that are already effectively immutable are natural candidates:

```jshell
jshell> value record Point(int x, int y) {}
jshell> Point p = new Point(17, 3)
jshell> new Point(17, 3) == p
$4 ==> true
```

But you're not limited to records. `EURCurrency` above stores euros and cents packed into a single `long`, which means its fields don't map one-to-one to its constructor arguments. That rules out a record, but it's still a perfectly good value class.

## Thirty classes in the JDK just became value classes

This is the part that actually affects you even if you never write the `value` keyword yourself. With preview features enabled, `Integer`, `Long`, `Double`, `Boolean`, `Optional`, `LocalDate`, `LocalTime`, `LocalDateTime`, `Duration`, and about twenty others become value classes. `Integer x = 1996, y = 1996; x == y` finally returns `true`, consistently, because the inconsistency was never a language guarantee to begin with.

`String` is a notable holdout. Its API and implementation lean on identity in a few places, so it stays an identity class regardless of preview flags.

## == and equals can still disagree, on purpose

Value semantics for `==` doesn't mean `==` and `equals()` become the same thing. They're allowed to diverge when a class's *internal* representation differs from its *observable* value. The JEP's own example is a `Substring` class that avoids allocating a new `char[]` by storing a source string and two coordinates:

```java
value class Substring {
    private String str;
    private int start, end;

    public String toString() { return str.substring(start, end); }

    public boolean equals(Object o) {
        return o instanceof Substring && toString().equals(o.toString());
    }
}
```

```jshell
jshell> Substring sub1 = new Substring("ringing", 1, 4);
jshell> Substring sub2 = new Substring("ringing", 4, 7);
jshell> sub1.equals(sub2)
$3 ==> true

jshell> sub1 == sub2
$4 ==> false
```

Both instances render as `"ing"`, so `equals()` agrees they're the same value. But `==` compares field values recursively, and the underlying `str`, `start`, and `end` fields are genuinely different between the two instances, so `==` says no. That's correct behavior, not a bug, `==` is telling you the internal state differs even though the observable value doesn't. If you want value equality, use `equals()`. That advice hasn't changed, JEP 401 just makes `==` slightly more useful without pretending it replaces `equals()`.

## What you lose by opting in

A few identity-sensitive operations simply don't work on value objects. `synchronized` is the big one:

```jshell
jshell> synchronized (d1) { d1.notify(); }
|  Error:
|  unexpected type
|    required: a type with identity
|    found:    java.time.LocalDate
```

That's a reasonable trade. If you're synchronizing on a `LocalDate`, something's already gone sideways in your design, monitors exist to protect mutable shared state, and value objects have none.

## It's preview, and it depends on JEP 539

Like most of what's landing in JDK 28's preview set, this needs `--enable-preview` at compile and run time, `javac --release 28 --enable-preview` and `java --enable-preview`. Compile against `LocalDate` with preview enabled, and you get the value-object version. Compile without it, and you get the identity-object version from JDK 27. You can't mix the two in one run.

Underneath, this leans directly on JEP 539's strict field initialization. A value class needs the JVM to guarantee that every field is set exactly once before the object escapes, that's the enforcement mechanism that makes it safe to freely copy or eliminate a value object's identity in the first place. If you read that piece already, this is the payoff.

I like this feature more than I expected to going in. It's rare to see the JDK correct something this fundamental to the object model, `==` on immutable data has been quietly wrong since day one, and this is the JDK admitting it rather than working around it forever with more `equals()` conventions and static analysis warnings.

>   
> I have already written about Value classes and Project Valhalla previously, see [this](/posts/value-classes-are-coming-to-java/) article for an even more detailled explanantion.   
>    

## What's next

Preview features mean nothing here is locked yet, and the interaction with generics, `Object[]` arrays, and boxing still has rough edges the JEP itself flags as future work. Next up in this series, a look at how these value semantics interact with the rest of the JDK 28 preview stack.
