---
title: "Java 27: What Actually Made the Cut"
date: 2026-09-15
draft: false
tags: ["Java", "JDK 27", "JVM"]
series: ["Java 27"]
cover:
  image: "/images/java27-all-in-one.png"
  alt: "Pattern Matching for primitives"
description: "A walkthrough of everything that landed in Java 27, and which ones are actually worth your attention."
categories: ["java"]
---

Nine JEPs. Nine separate articles. Somewhere around March I sat down to write a quick overview post for Java 27, and here we are in the middle of September still talking about it. Java 27 is officially released.

So let's zoom out. If you've been following along JEP by JEP, some of this will be a refresher. If you haven't, and you just want to know what's actually changing when you type `java -version` from now on, this is the post for you.

One thing worth saying up front: Java 27 is not an LTS release. JDK 29 is, a year from now. That colors how I'd recommend you treat some of what's below, especially the previews. But a handful of these changes will matter to you the moment you upgrade, LTS or not.

Here's the full list, roughly in the order I'll walk through them:

| 523, G1 becomes the default GC on all hardware, Final
| 534 | Compact object headers, 12 bytes to 8, on by default | Final |
| 527 | Post-quantum key exchange for TLS 1.3 | Final |
| 536 | JFR redacts secrets before writing to disk | Final |
| 538 | PEM API for keys and certificates | Preview (3rd) |
| 533 | Structured Concurrency, sharper exception typing | Preview (7th) |
| 532 | Primitive types in `switch`, `instanceof`, patterns | Preview (5th) |
| 531 | Lazy Constants, plus `Set.ofLazy` | Preview (3rd) |
| 537 | Vector API, SIMD math in portable Java | Incubator (12th) |

Four of these ship as finished, default-on behavior. Five are still previews or incubators in various stages of "almost there." I'll take them roughly in that order too, starting with the two that change your production systems without you lifting a finger.

And as a picture says more than a 1000 words (or lines of code), here is a graphic overview.
![All JEPs in Java 29](/images/java27-infographic-grid-1.png)

> If want the details on all of these features than take a look at my Java 27 series. You can find all nine JEP deep-dives in the [series archive](https://ronveen.com/series/java-27/).

## The two you'll feel without doing anything

Two JEPs in this release are the kind that improve your production systems while you're asleep. No code changes, no flags, no migration guide. You upgrade the JDK and things just get a little better.

**JEP 523 makes G1 the default garbage collector, everywhere.** Since JDK 9, G1 was the default only on "server-class" hardware, roughly 2 or more CPUs and 2GB of RAM. Anything smaller quietly fell back to Serial. That distinction made sense in 2017, when Serial genuinely had the edge on tiny boxes: lower throughput overhead, smaller footprint. It doesn't make sense anymore, because G1 has spent the last several releases closing that gap. Throughput improved thanks to [JEP 522's synchronization work](https://ronveen.com/posts/jep522-g1-improve-througput-by-reducing-synchronization/) back in the Java 26 cycle, memory footprint has been trimmed down toward Serial's territory, and latency was never a contest, G1's incremental old-generation collection has always beaten Serial's stop-the-world approach there. Put those three together and Serial simply doesn't have a corner left to defend. JEP 523 collapses the old "it depends" into one rule: G1, always.

If you're running Spring Boot on OpenShift or in a container with a couple of cores, this changes nothing for you, you were already on G1. Where you'll notice it is on the small stuff. Tiny CI runners, serverless functions on minimal resource allocations, a Raspberry Pi cluster if you happen to have one sitting in a home office collecting dust (no comment). Worth knowing: nothing about Serial itself is going away, and your explicit choice still wins every time.

```bash
# still works exactly as before, if you know Serial genuinely suits your workload
java -XX:+UseSerialGC -jar my-app.jar
```

The JEP's own language is honest about this, it says throughput and footprint "should not degrade *significantly*" on constrained hardware, which is deliberately not a promise of zero regression. If you've actually measured your workload on a genuinely minimal box and Serial wins, keep using it. Just don't assume it wins by default anymore.

**JEP 534 shrinks every object header from 12 bytes to 8.** This is the one I'd genuinely call free money. Every Java object carries a header the JVM uses for locking, hashing, and type information, and it's been 96 bits since the 64-bit JVM became the norm. The breakdown is a 64-bit Mark Word (identity hash, GC age, lock bits) plus a 32-bit Class Word pointing at the type metadata. Turns out 27 of those 96 bits were just sitting there unused, dead weight accumulated over years of incremental changes nobody went back to clean up. Trim those for free, then squeeze the class pointer from 32 bits down to 22 by addressing class metadata in 1KB blocks instead of individual bytes, and you land on a tidy single 64-bit header. Four of the bits saved are already earmarked for Project Valhalla, one goes to a new "self forwarded" GC tag, and the rest close the gap.

No API changes. No bytecode changes. If your code compiles and runs on JDK 25 or 26, it runs on 27 with compact headers on, full stop. Amazon reported 22% less heap and 8% less CPU running this across hundreds of production services, and SAP's SapMachine distribution has shipped it as default for a while already, so the stability story is solid going in. For an app dominated by small, short-lived objects, request-scoped beans, DTOs, the intermediate objects Jackson creates while deserializing, that's a real number, not a rounding error. If your service mostly streams large byte arrays, you won't notice much, the header barely registers next to a big allocation.

Curious what it actually looks like on your own classes? Grab JOL and compare JDK 26 against JDK 27:

```java
import org.openjdk.jol.info.ClassLayout;

public class JolExample {
    static class Person {
        int age;
    }

    public static void main(String[] args) {
        System.out.println(ClassLayout.parseClass(Person.class).toPrintable());
    }
}
```

Run it on 26 with `-XX:+UseCompactObjectHeaders` and again on 27 with nothing at all, and you'll see the header itself drop from 12 bytes to 8. The overall object size might not shrink by the full four bytes, alignment still rounds things up, but across a heap with millions of objects, it adds up fast.

The neat part is these two JEPs are quietly related. Both come out of years of incremental JVM engineering that nobody puts on a conference slide, and both hand you a better default for doing precisely nothing.

## Security, three different ways

Security in this release isn't one big headline feature. It's three JEPs chipping away at three completely different problems, and honestly that's a more useful shape than a single flashy addition would have been.

**JEP 527 adds post-quantum key exchange to TLS 1.3**, on by default. The threat model here is "harvest now, decrypt later": someone records your encrypted traffic today, and waits for a quantum computer capable of cracking it, at which point today's traffic becomes yesterday's readable secret. JEP 527 layers a quantum-resistant algorithm called ML-KEM on top of the classical elliptic curve math we already trust, so your connection stays safe as long as *either* lock holds, not both. If you're using standard `javax.net.ssl`, you get this automatically, no code changes required, and the primary scheme (`X25519MLKEM768`) is designed to be fast enough that you shouldn't notice a performance hit.

It's also the payoff of work that's been building for a couple of releases, and the chain is worth knowing if you want the full picture. JEP 452 and 478 gave the JDK the basic vocabulary for this kind of cryptography, the Key Encapsulation Mechanism API and Key Derivation Function API. JEP 496 and 497 added the actual quantum-resistant math itself, ML-KEM for encryption, ML-DSA for signatures, but left developers to wire it up manually. JEP 527 is the one that finally plugs that math directly into TLS, so nobody has to write that wiring themselves. Java's arguably ahead of most major platforms here, turning this on by default rather than leaving it opt-in.

**JEP 536 redacts secrets from JFR recordings before they ever hit disk.** I've told this story before but it's worth repeating because it's such an easy trap to fall into: someone on my old team attached a `.jfr` file to a vendor support ticket, not realizing it also contained a database password we'd passed in as a `-D` system property at startup. Nobody caught it until the ticket had already left the building. JFR captures every environment variable, system property, and command-line argument present at startup, which is fantastic for debugging and terrible for anything you plan to share outside your team.

```
$ jfr print --events InitialSystemProperty,InitialEnvironmentVariable dump.jfr

jdk.InitialSystemProperty {
  key   = "javax.net.ssl.keyStorePassword"
  value = "[REDACTED]"
}
jdk.InitialEnvironmentVariable {
  key   = "ACCESS_TOKEN"
  value = "[REDACTED]"
}
```

As of JDK 27, values matching patterns like `*password*`, `*token*`, `*secret*`, or `*credential*` get swapped for `[REDACTED]` right as the event is captured, not scrubbed afterward as an extra step somebody has to remember to run. There's a `redact-argument` list too, for command-line arguments, deliberately slightly narrower than the property list so it doesn't flag something innocent like `--author` as a secret. You can extend either list for your own naming conventions with a `+` prefix, and honestly you probably should. If your team calls its secrets something like `envelope` or `sealed`, JFR has no way of knowing that's sensitive.

**JEP 538 is the PEM API's third preview**, and it's the least dramatic of the three, which is exactly why it's useful. Parsing `-----BEGIN PRIVATE KEY-----` blocks by hand has been a rite of passage for Java developers for decades, strip the header, Base64-decode the middle, hand the bytes to a `KeyFactory`, hope you picked the right algorithm. I've written that exact helper method myself more than once. `PEMEncoder` and `PEMDecoder` do the whole thing in a few lines instead of a dozen:

```java
PEMDecoder decoder = PEMDecoder.of();
switch (decoder.decode(pem)) {
    case PublicKey publicKey -> handle(publicKey);
    case PrivateKey privateKey -> handle(privateKey);
    default -> throw new IllegalArgumentException("Unexpected PEM content");
}
```

This round doesn't add much new capability so much as sand down the rough edges from round two: `DEREncodable` becomes the more honest `BinaryEncodable`, `getKey`/`getKeyPair` drop a `Provider` parameter nobody wanted to pass, and there's a new unchecked `CryptoException` for when a checked `GeneralSecurityException` is more ceremony than the situation calls for. If you were already on JEP 524 in JDK 26, migrating is mechanical, a rename here, a simplified signature there.

## Language features, and the long tail of "still cooking"

This is where Java 27 gets a bit less exciting on paper, and I think that's fine. Not every release needs a headline language feature.

**JEP 533 sharpens Structured Concurrency for its seventh preview.** This one actually earned its "worth reading" badge this round, unlike some of the others on this list. If you haven't touched it, the core idea is that subtasks spawned inside a scope live and die within that scope's lexical lifetime, no rogue threads outliving the method that started them, no exceptions swallowed silently in the background.

```java
try (var scope = StructuredTaskScope.open(
        Joiner.<String, ExecutionException>allSuccessfulOrThrow())) {

    Subtask<String> task1 = scope.fork(() -> callServiceA());
    Subtask<String> task2 = scope.fork(() -> callServiceB());

    try {
        scope.join();
    } catch (ExecutionException e) {
        throw new RuntimeException("A subtask failed", e.getCause());
    }

    return task1.get() + task2.get();
}
```

`StructuredTaskScope` and `Joiner` now carry a third type parameter, `R_X`, that captures exactly what exception `join()` throws, so the compiler can tell you instead of leaving you to dig through Javadoc. The old `Joiner.onTimeout()`, which cancelled a scope but left you guessing whether you got a usable partial result, is gone, replaced by an explicit `timeout()` that throws a proper `CancelledByTimeoutException` you can catch by name. And `awaitAll()`, which waited for every subtask but silently swallowed failures until you called `.get()` and got blindsided, has been removed outright. That's exactly the kind of API that looks fine in a demo and bites you the first time a downstream service actually falls over in production. If you're migrating from an earlier preview, the one thing to actually budget time for is finding every `catch (FailedException e)` in your codebase and changing it to `ExecutionException`. Mechanical fix, but the compiler won't do it for you until you recompile against 27.

**JEP 532, primitive types in patterns, is the fifth preview and changes literally nothing from JEP 530.** I mean that as a compliment. The JDK team looked at it, decided the design was solid, and just let it bake for another cycle. If you missed it earlier, the short version is that `switch`, `instanceof`, and pattern matching finally work on primitives the way they already work on objects:

```java
long tooBig = 3_000_000_000L;
System.out.println(tooBig instanceof int); // false, exceeds Integer.MAX_VALUE

float inexact = 42.5f;
System.out.println(inexact instanceof int); // false, the .5 would be lost
```

That's a lossless-conversion check baked into the language, no more hand-rolled range checks and hoping you got the boundaries right. It works inside record patterns too, so you can destructure straight down to a primitive type. Nice to have. Not urgent, and after five previews with zero behavioral changes, whatever you write against JDK 27 should look identical when this eventually goes final.

**JEP 531 gives Lazy Constants their third preview**, removing `isInitialized()` and `orElse()` because developers kept using them to reintroduce the exact null-checking dance Lazy Constants were designed to eliminate in the first place, and adding `Set.ofLazy(...)` to complete the trio alongside `List.ofLazy` and `Map.ofLazy`. If you've got a fixed universe of feature flags or config options where checking membership is expensive, an `EnumSet` combined with a predicate fits this naturally:

```java
static final Set<Option> OPTIONS =
        Set.ofLazy(EnumSet.allOf(Option.class), Application::isEnabled);
```

First call to `contains()` for a given option evaluates the predicate. Every call after that returns the cached answer. You never have to think about initialization order or thread safety, which is really the whole pitch of Lazy Constants in one sentence.

**JEP 537 re-incubates the Vector API for the twelfth time**, and the entire changelog is a version bump on a bundled math library. I'm not going to pretend that's exciting. Checking in on it these days feels like checking in on a home renovation that's been "almost done" since 2014, which is genuinely how long ago this project got announced.

If you've never touched it, the pitch is worth understanding even while it's stuck in limbo. Normally, getting your code to use your CPU's SIMD instructions, the kind that apply one operation across multiple values in a single instruction, means either hoping the JIT auto-vectorizes a tight loop for you, or dropping into native code with JNI and giving up every safety guarantee Java gives you. The Vector API sits in between: you write portable Java against an API that's explicit about lanes and shapes, and the JVM compiles it down to real vector instructions on whatever hardware you're running.

```java
var species = FloatVector.SPECIES_PREFERRED;
int upperBound = species.loopBound(data.length);

for (int i = 0; i < upperBound; i += species.length()) {
    var v = FloatVector.fromArray(species, data, i);
    v.mul(2.0f).intoArray(result, i);
}
```

That loop processes a whole vector's worth of floats per iteration, not one at a time, and there's not a single line of C involved. Anywhere your hot path looks like "loop over a large array doing the same arithmetic millions of times", numerical computing, audio and image codecs, similarity search, this is the tool, if it ever ships.

What is interesting this round is *why* it's stuck. The Vector API is deliberately staying in incubation until Project Valhalla's value classes exist to replace some JIT special-casing that's currently holding vectors together as a workaround. Right now `Vector<E>` gets compiler-level special treatment just to dodge the object identity and boxing overhead a regular heap-allocated object would carry. It works, but it's scaffolding, not a real foundation. Once value classes exist, vectors become actual value classes and the workaround becomes unnecessary.

Here's the actual news: value classes, the centerpiece of Valhalla, are now targeted for a JDK 28 preview via JEP 401, which is the first time in years that sentence has had an actual JDK number attached instead of just vibes. The pull request behind it is reportedly over 197,000 lines across 1,800-plus files, which tells you something about how deep this change runs into the JVM. I'm not popping champagne, JDK 28 isn't even the next LTS, that's JDK 29 a year later, and it's still only a preview. But after over a decade of "any year now" in release notes, having an actual date to glance at is new.

## So, should you upgrade?

If you're already comfortable moving off LTS between major releases, yes, there's real value here even ignoring the previews entirely. Compact object headers alone are worth the version bump for most Spring Boot services, and it costs you nothing to get it. If you're on a strict LTS-to-LTS cadence, you'll get all nine of these bundled into JDK 29 anyway, with a few more preview rounds of polish stacked on top by then.

Either way, the previews are the part I'd actually keep an eye on between now and then. Structured Concurrency feels like it's converging rather than churning, seven previews in and this round was the first where I'd call the changes meaningful rather than cosmetic, so I'd bet on it going final around JDK 29. Primitive types in patterns is arguably done in every sense but name, five previews with zero design changes is about as clear a signal as the JDK team ever gives. The Vector API, less so, that one's still waiting on a dependency outside its own control. But for the first time in years, that's a calendar problem rather than an open design question.

None of these nine JEPs are individually the kind of thing that gets a standing ovation at a conference. No new syntax that changes how you structure a class, no headline feature you'll see on a slide. What you get instead is a release that quietly makes the JVM itself smaller, safer, and a bit more honest about what it's doing under the hood, while three long-running previews inch closer to being finished. I'll take that trade most years.

Development never stops, and Java 28 is already well on its say. They first JEPs have been added. Of course, you can follow all the developments in my [Java 28](/series/java-28/) series.
Given I've apparently been saying "waiting on Valhalla" in release notes for well over a decade now, that it's going to be fun to write about it.

