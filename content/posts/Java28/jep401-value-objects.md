---
title: "JEP 401: Value Objects (Preview)"
date: "2026-09-22"
draft: false
tags: ["Java", "JEP", "JVM"]
series: ["Java 28"]
series_order: 3
cover:
  image: "/images/jep401-value-objects.png"
  alt: "JEP 401: Value Objects (Preview)"
description: "How JDK 28's value objects let you opt out of identity for immutable data, and why that's more than a performance trick."
---

A while back I [wrote](/posts/value-classes-are-coming-to-java) about the first time `==` looked me dead in the eye and said `false` for two `LocalDate` objects that were, by any sane definition, the same date. That post was about a preview feature that was still finding its feet. JEP 401 has since integrated into JDK 28, and it's still preview (you need `--enable-preview` to touch it), but the design is done enough now that it's worth going past the "identity is weird, huh" pitch and into the parts that'll actually bite you.

Because there's more here than a friendlier `==`. Value classes change how objects get constructed, what you're allowed to do while a constructor is still running, which classes can extend which, and what happens the moment you try to synchronize on the wrong thing.

## The two-sentence recap

A value class is immutable and doesn't have identity. Its instances are only distinguished by their field values, `==` compares those fields instead of memory addresses, and the JVM is free to represent them however it wants, on the stack, in registers, packed into an array, wherever is cheapest. You mark one with the `value` modifier:

```java
value record Point(int x, int y) {}
```

If you want a value class that isn't a record, because your internal state doesn't map cleanly onto your constructor arguments, that works too. The JEP's example is a currency type that stores euros and cents packed into a single `long` for compactness:

```java
value class EURCurrency {

    private long cs;  // implicitly final

    private EURCurrency(long cs) { this.cs = cs; }

    public EURCurrency(long e, int c, boolean neg) {
        this(neg ? -e * 100 - c : e * 100 + c);
    }

    public long euros() { return Math.abs(cs) / 100; }
    public int cents() { return (int) Math.abs(cs) % 100; }
    public boolean negative() { return cs < 0; }
}
```

Not transparent enough to be a record, but still immutable and interchangeable. That's the actual bar for "should this be a value class": immutable, and you genuinely don't care whether two instances with the same state are the same object.

## `==` and `equals` aren't always the same story

Here's the part that trips people up. For value objects, `==` recursively compares fields, and `equals` is whatever you defined. Usually those agree. They don't have to.

Take a `Substring` value class that represents a slice of a string without copying characters:

```java
value class Substring {

    private String str;
    private int start, end;

    public Substring(String s, int i, int j) {
        str = s; start = i; end = j;
    }

    public String toString() {
        return str.substring(start, end);
    }

    public boolean equals(Object o) {
        return o instanceof Substring && toString().equals(o.toString());
    }

    public int hashCode() {
        return Objects.hash(Substring.class, toString());
    }
}
```

Two `Substring` instances built from different source strings can represent the exact same character sequence and be `equals`, while their internal fields, and therefore `==`, disagree. Same thing happens with `float`/`double` fields holding different `NaN` bit patterns, which most floating-point code treats as interchangeable but `==` does not. If you're migrating a class to `value` and its state isn't as transparent as a record's components, don't assume the default field-by-field `==` semantics line up with whatever `equals` contract your callers already depend on. Write the tests for that before you flip the switch.

## Value classes can still have a family tree

I'd assumed, wrongly, that `value` meant "final, full stop, no hierarchy." Not quite. A value class is implicitly final unless you declare it `abstract`, in which case it can be extended, by other value classes or by identity classes.

`Number` is the JEP's example of a good candidate for this: no fields, nothing identity-sensitive, and both `Integer` (a value class) and `BigInteger` (an identity class) can extend it.

```java
abstract value class Number implements Serializable {
    public abstract int intValue();
    public abstract long longValue();
    public byte byteValue() { return (byte) intValue(); }
}
```

You can sealed-restrict an abstract value class the same way you always could:

```java
sealed abstract value class UserID
    permits EmailID, PhoneID, UsernameID {}

value class EmailID extends UserID {
    private String name, domain;
}
```

The rule that actually matters: a value class can extend `Object` or an abstract value class, never an identity class. `Object` gets a special exemption, being neither abstract nor a value class itself, despite letting value classes extend it. Worth knowing before you go hunting for a `java.lang.Value` superclass that doesn't exist.

## The construction rules are the real change

This is the part I think gets underplayed. Value objects can't have identity leak out mid-construction, which means the constructor itself now runs under stricter rules than you're used to.

An object being built is "larval," a term I like more than I expected to. If a larval object escapes the constructor before it's fully initialized, other code can observe fields that haven't been set yet, or watch a supposedly-final field mutate later. Flexible constructor bodies (from Java 25) split construction into an early phase, before `super(...)`, and a late phase, after. In the early phase you can set fields, but you can't call instance methods or reference `this`, because there's no guarantee the object is safe to use yet.

For value classes, the compiler enforces this by default: constructor code runs in the early phase, and `super()` gets inserted at the *end* of the constructor rather than the start. This fails to compile:

```java
value class Name {
    String name;
    int length;

    private int strLength() {
        return name.length();
    }

    Name(String n) {
        name = n;
        length = strLength();  // invokes this.strLength() — not allowed here
    }
}
```

Make `strLength` static and it's fine, because now you're not touching `this`:

```java
value class Name {
    String name;
    int length;

    private static int strLength(String n) {
        return n.length();
    }

    Name(String n) {
        name = n;
        length = strLength(name);  // OK
        super();                   // fields are all set by now
    }
}
```

The knock-on effect I didn't expect: as of JDK 28, with preview enabled, *every* record, value or identity, adopts these same construction rules. That's a real source-incompatibility risk if you've got record constructors that call instance methods or pass `this` around before validation. The JEP's own example is a `Node` record whose canonical constructor calls `this.toString()` for an error message and now fails to compile. Their survey of real-world code says this pattern is rare. I'd still grep for it before touching the preview flag on anything with records in it.

## Migrating an existing class isn't free

The JEP is upfront that flipping an identity class to `value` is a compatibility decision, not a formatting one. A few things worth actually checking before you do it to something with existing callers:

Public constructors are a problem if any caller relies on them producing distinguishable instances, think object-identity-based caching or dedup logic built on `==`. The JDK's own answer to this, going back to Java 9 deprecating `Integer`'s constructor in favor of `Integer.valueOf`, is the playbook: push people toward factory methods first, deprecate the constructor, migrate later.

Synchronization on instances breaks outright, either a compile error or an `IdentityException` at runtime, and callers with public constructors are exactly the ones likely to have been locking on their own created instances.

And if the class exposes sensitive data, `==` and `System.identityHashCode` become a side channel for probing field values. Value objects were never designed to resist that, so don't reach for `value` on anything where that matters.

Serialization is its own headache. Value records serialize automatically, but a non-record value class implementing `Serializable` needs `writeReplace` and `readResolve`, because deserialization can't safely populate strictly-initialized fields the way a constructor can. Skip that and you get an `InvalidClassException` at runtime, not compile time. Honestly, for a feature being sold as "immutability with less ceremony," needing two extra methods just to keep serialization working is the kind of thing that's going to catch people off guard the first time it happens in production rather than in a test.

## Flattening, briefly, and one detail worth knowing

I covered scalarization and flattening in the last post, so I won't redo the diagrams. The one thing worth adding: flattened references need to be read and written atomically, which on today's hardware caps them at around 64 bits for *mutable* fields. A `LocalDateTime` field on a regular class can't be flattened for that reason.

But that atomicity constraint only applies because the field can change. A value object's fields never mutate after construction, full stop, so a field *of a value class* can hold a flattened reference even past that 64-bit ceiling in ways a mutable field never could. It's a small distinction, but it's the reason value classes get more of this optimization than you might expect just from reading about the size limit.

## Why isn't `String` a value class?

It's immutable. It's interchangeable. It is, by every definition the JEP itself gives, the textbook case for this feature. And it's explicitly excluded, because its API and implementation have decades of dependencies on object identity baked in that nobody's untangling for JDK 28. That's the one that bugs me most about this whole JEP: the poster child for "immutable data that doesn't need identity" is the one type that can't have it, at least not yet.

The thirty classes that *did* make the cut, `Integer` and friends, the `Optional` family, most of `java.time`, were already specified as value-based for years, precisely so this migration wouldn't be a surprise. If you want to see what "was designed for this all along" actually buys you, `Objects.hasIdentity()` will tell you which side of the line any given object falls on.

If you're planning to touch `--enable-preview` on a real codebase, start with your test suite and go looking for anything that synchronizes on a boxed type or a `LocalDate`. That's where this either quietly works or loudly doesn't.