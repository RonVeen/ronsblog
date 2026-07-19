---
title: "Java 27 - JEP 537 - Vector API, Twelfth Incubator"
description: "A look at what the Vector API is actually for, and why it's still incubating after twelve rounds."
date: 2026-07-27
draft: true
tags: ["Java", "Security", "Project Panama", "Vector API", "JEP537"]
cover:
  image: "/images/jep537-vector-api.png"
  alt: "JEP 537 - Vector API, Twelfth Incubator"
series: ["Java 27"]
series_order: 10
categories: ["java"]
---

I checked in on the Vector API the way you check in on a home renovation that's been "almost done" for years. You poke your head in, nothing's changed, and someone tells you it's waiting on one more delivery. Except this delivery has been "arriving soon" since 2014.

That's JEP 537. Twelfth incubation round. Let's talk about what it actually does, and why it's still sitting in the waiting room.

## What the Vector API Is For

If you've never touched it: the Vector API lets you write code that expresses vector computations — operations applied across multiple values at once — and have that reliably compile down to the SIMD instructions your CPU actually supports. AVX on x64, SVE on ARM, whatever the hardware offers.

Normally that kind of optimization is either invisible (the JIT auto-vectorizes a tight loop *if* you're lucky and the pattern is simple enough) or it means dropping into native code with JNI and losing every safety guarantee Java gives you. The Vector API sits in between: you write portable Java, using an API that's explicit about lanes, shapes, and species, and the JVM turns it into real vector instructions where the hardware allows.

Where this earns its keep is anywhere you're doing the same arithmetic across large arrays, over and over: numerical computing, image and audio processing, codecs, similarity search, the inner loops of machine learning workloads. Anywhere "loop over a float array and do the same three operations sixty million times" describes your hot path, this is the API you want to reach for instead of praying the JIT vectorizes it for you.

```java
var species = FloatVector.SPECIES_PREFERRED;
int upperBound = species.loopBound(data.length);

for (int i = 0; i < upperBound; i += species.length()) {
    var v = FloatVector.fromArray(species, data, i);
    v.mul(2.0f).intoArray(result, i);
}
```

That loop processes a whole vector's worth of floats per iteration instead of one at a time — and it does it without you writing a single line of C.

## What Changed in JDK 27: Nothing

I want to be precise here, because "nothing changed" is doing real work as a description, not just a punchline. JEP 537 re-incubates the Vector API for JDK 27 with no substantial implementation changes since JDK 25. The one item worth mentioning is a version bump — the bundled SLEEF library, used for ARM and RISC-V vector math intrinsics, moved from 3.6.1 to 3.9.0.

That's it. That's the whole changelog. Twelve rounds of incubation — JDK 16 through JDK 27 — and this lap is a dependency bump.

## Still Waiting on Valhalla

Here's the actual reason nothing moves: the Vector API is explicitly staying in incubation until the necessary features of Project Valhalla become available as preview features. Once that happens, the plan is to promote it from incubation to preview — the next rung up, not even the finish line.

Why does a vector math API care about value classes? Because right now, `Vector<E>` instances get special-cased treatment in the JIT compiler just to avoid the object identity and boxing overhead a full-blown heap-allocated object would carry. It works, but it's a workaround the compiler team built specifically to hold this API together until value classes exist to do the job properly. Once Valhalla lands, vectors are expected to become actual value classes, and all that special-casing becomes unnecessary machinery instead of a permanent crutch.

And here's where it gets almost funny: Valhalla is, for the first time in this project's very long life, not vaporware. JEP 401, Value Classes and Objects, was confirmed for integration into OpenJDK in June 2026, targeting JDK 28 — as a preview feature. Twelve years after the project was announced, and the pull request alone is over 197,000 lines across more than 1,800 files, which tells you something about how deep this change actually runs.

So: after a decade-plus of "any year now," there's an actual preview with an actual JDK number attached to it. Forgive me if I'm not popping champagne. JDK 28 is a preview, not the finished thing, and it's not even the LTS release — that's JDK 29, a year later. Brian Goetz himself has already pre-emptively told the "they'll never ship it" crowd that they'll simply pivot to "but they didn't ship the important part." He's not wrong. He's basically written the sequel to his own movie before this one's finished.

## What This Means for the Vector API

Practically: nothing changes for you today. Keep using the Vector API in `jdk.incubator.vector` exactly as you have been, behind whatever module flags you're already passing. Nothing in JEP 537 asks you to touch your code.

But for the first time in a long while, the "waiting on Valhalla" line at the bottom of this JEP has an actual date behind it instead of just vibes. If JDK 28's preview holds up, the Vector API's thirteenth incubation round might finally have something to report other than a library version number.

I've been writing "it's waiting on Valhalla" in Java release notes for years now. I'm not holding my breath — but for the first time, I'm at least glancing at the calendar.