---
title: "Java 27 - part 4: JEP 532 - Primitive Types in Patterns"
date: 2026-06-15
draft: true
tags: ["java", "java27", "jdk27", "pattern matching", "jep532"]
cover:
  image: "/images/jep532-primitive-types-in-patterns.png"
  alt: "Primitive Types in Patterns"
series: ["Java 27"]
series_order: 5
categories: ["java"]
---
## Same Feature, Fifth Time Around

If you read my [JEP 530 post](https://ronveen.com/posts/jep530-pattern-matching-for-primitives/) at the tail end of the Java 26 series, this one is going to feel familiar. Very familiar.

JEP 532 is the *fifth* preview of "Primitive Types in Patterns, `instanceof`, and `switch`", now targeting JDK 27. And here's the honest headline: **nothing has changed.** No new syntax, no removed methods, no behavioural tweaks. The JDK team looked at JEP 530, decided it was in good shape, and re-proposed it as-is to bake for another release cycle.

So why write about it at all? Because a re-preview is a good excuse to revisit *why* this feature is worth caring about — and if you missed it the first time, this is your refresher.

## What this feature actually does

For years Java has felt like two languages stitched together: one for objects, one for primitives. Pattern matching — the gift that Project Amber gave us through records, sealed classes, and `switch` — only ever worked on the object side. Want to use these tricks with `int`, `double`, or `boolean`? Tough luck.

This feature lifts that restriction in three places:

- **Primitives in `switch`** — every primitive type is now fair game, including `boolean`, `long`, `float`, and `double`. Previously you were stuck with a subset of the integral types.
- **Primitives in `instanceof`** — you can now ask `instanceof` whether converting one primitive to another would lose information. No more silent truncation bugs.
- **Primitive type patterns** — primitives work in pattern contexts everywhere, including nested inside record patterns.

The recurring word, just like in JEP 530, is **uniformity**: primitives finally play by the same rules as reference types.

It's still a preview feature in JDK 27, so it's off by default — you'll need the `--enable-preview` flag to take it for a spin.

Let's look at all three in action.

## 1. Primitives in `switch`

Switching on a `boolean` — something you genuinely could not do before:

```java
boolean isActive = getStatus();
String label = switch (isActive) {
    case true  -> "Active";
    case false -> "Inactive";
};
```

Switching on a `long` without first casting down to `int`:

```java
long statusCode = getHttpStatus();
String description = switch (statusCode) {
    case 200L -> "OK";
    case 404L -> "Not Found";
    case 500L -> "Internal Server Error";
    default   -> "Unknown (" + statusCode + ")";
};
```

And `double` with guards, which makes range logic read like a clean list of rules:

```java
double score = calculateScore();
String grade = switch (score) {
    case double d when d >= 90.0 -> "A";
    case double d when d >= 75.0 -> "B";
    case double d when d >= 55.0 -> "C";
    default                      -> "F";
};
```

## 2. Primitives in `instanceof`

This is the one I find most useful in day-to-day code. The whole point here is the *lossless conversion check* — `instanceof` only matches if the value survives the conversion intact.

```java
long smallNumber = 42L;
long bigNumber   = 1_000_000L;
long tooBig      = 3_000_000_000L;

System.out.println(smallNumber instanceof int); // true  — 42 fits in an int
System.out.println(bigNumber   instanceof int); // true  — 1,000,000 fits fine
System.out.println(tooBig      instanceof int); // false — exceeds Integer.MAX_VALUE
```

It's even more satisfying with `float` → `int`, where the question is whether you'd lose a fractional part:

```java
float exact   = 42.0f;
float inexact = 42.5f;

System.out.println(exact   instanceof int); // true  — no fractional part to lose
System.out.println(inexact instanceof int); // false — the .5 would be lost
```

No more writing your own range checks and praying you got the boundaries right. The compiler now understands "exactness" and won't let a match happen if data would be lost.

## 3. Primitive type patterns

In a plain `instanceof` pattern, binding the matched value straight to a primitive:

```java
Object value = getValue(); // might be Integer, Long, Double…
if (value instanceof int i) {
    System.out.println("Small int: " + i);
} else if (value instanceof long l) {
    System.out.println("Long value: " + l);
}
```

And the part I like best — primitives *inside* record patterns, so you can destructure straight to the type you want:

```java
record Measurement(double value, String unit) {}

Object obj = new Measurement(98.6, "Fahrenheit");
if (obj instanceof Measurement(double temp, String unit)) {
    System.out.println("Temperature: " + temp + " " + unit);
}
```

Combine that with a `switch` and a guard, and you get genuinely clean data extraction:

```java
record Sensor(String name, float reading) {}

Object event = getEvent();
String summary = switch (event) {
    case Sensor(String name, float f) when f instanceof int i
        -> name + " reads exactly " + i + " (whole number)";
    case Sensor(String name, float f)
        -> name + " reads " + f;
    default
        -> "Unknown event";
};
```

## Wrapping up

JEP 532 is the rare article where "what changed" is genuinely "nothing" — and that's fine. A fifth preview with zero changes is the JDK team's way of saying the design has settled and they're just letting it soak. For us, that's good news: whatever you write against it in JDK 27 should look identical when this finally goes final.

If you skipped the primitive-patterns story the first time around, JDK 27 is as good a moment as any to start playing with it. It's the cleanup crew that makes modern Java feel less like ceremony and more like a tool that actually has your back.
