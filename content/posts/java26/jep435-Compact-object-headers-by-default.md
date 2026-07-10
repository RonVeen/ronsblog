---
title: "JEP 534: Compact Object Headers by Default"
description: "JEP 534 makes compact object headers the default in JDK 27, shrinking every object from 12 to 8 bytes and handing you 10–20% heap savings without touching a line of code."
date: 2026-06-29
draft: false
tags: ["java", "java27", "jdk27", "JVM", "jep534", "Project Lilliput", "Project Valhalla"]
cover:
  image: "/images/jep435-compact-object-headers-by-default.png"
  alt: "Compact Object Headers by Default"
series: ["Java 27"]
series_order: 6
categories: ["java"]
---


A few years ago, a colleague of mine was convinced that a memory leak had crept into one of our microservices. The heap was consistently at 70% after startup — before a single request had been processed. We profiled it, combed through the GC logs, and eventually concluded that no, nothing was leaking. It was just object overhead. Lots of small objects, each carrying its overhead around like a briefcase it didn't need.

That particular flavor of pain is about to get a lot less common.

JEP 534, targeting JDK 27, makes Compact Object Headers the default layout in HotSpot. No flag required, no code changes needed. You upgrade your JDK, and every object in your heap quietly shrinks from 12 bytes to 8 bytes. That’s about as close to free money as JVM performance improvements get.
> Every object gets a smaller header. Depending on the object’s layout and alignment, the overall object size may or may not decrease by four bytes, but across an entire heap the savings add up substantially.

But *how* does the JVM squeeze a header that's been 12 bytes for decades down to 8? That's where it gets interesting.

---

## Every Object Carries a Header You Didn't Ask For

Before we get to what changed, let's establish what we're talking about.

Every Java object on the heap is preceded by an *object header* — a block of metadata the JVM uses internally. As a developer you never see it, you can't touch it from Java code, and you're not consulted about it. It's just there, on every object, all the time.

On a 64-bit JVM with compressed class pointers enabled (the default since Java 6), that header is **96 bits — 12 bytes**. For a class with a single `int` field, your object is 12 bytes of header plus 4 bytes of data plus 0 bytes of padding. You asked for 4 bytes of storage. You got 16. Objects are typically aligned to 8-byte boundaries, so the JVM rounds object sizes up where necessary.

Scale that across millions of objects in a typical enterprise application — the kind with hundreds of small DTOs, domain events, wrapper types, and collection entries — and those 12-byte headers add up to a non-trivial slice of your heap.

The header consists of two parts:

- A **64-bit Mark Word**, which stores the identity hash code, GC age bits, and lock state.
- A **32-bit Class Word**, which stores a compressed pointer to the class metadata structure (the thing that tells the JVM what type this object is).

Total: 96 bits. That's been the status quo since the 64-bit JVM became the norm.

---

## The Mark Word, Up Close

Here's the actual layout of the Mark Word before compact headers:

| Bits | Content |
|------|---------|
| 63–38 | *unused* (25 bits) |
| 37–7 | Identity hash code (31 bits) |
| 6 | *unused* (1 bit) |
| 5–2 | GC age (4 bits) |
| 1 | *unused* (1 bit) |
| 0–1 | Tag bits — lock state (2 bits) |

Count the unused bits: 25 + 1 + 1 = **27 unused bits**.

So out of the 96 total bits in the header, only 69 are actually carrying information. The rest is just padding that accumulated over years of incremental JVM changes.

Now here's the key calculation. To go from 96 bits to 64 bits, you need to cut **32 bits** total. But 27 of those bits are already unused — you can eliminate them for free just by packing the layout more tightly. That leaves only **5 bits** you still need to find somewhere else.

Five bits. That's it. The problem is much smaller than "shrink a 96-bit header to 64 bits" suggests.

Those five bits come from the Class Word — by compressing the class pointer from 32 bits down to 22. That saves 10 bits, which is more than enough: 5 go toward closing the gap, and the leftover 5 are put to use for Valhalla (4 bits) and a new internal GC flag (1 bit). Nothing is wasted.

---

## Squeezing the Class Pointer from 32 to 22 Bits

The 32-bit Class Word is where those 10 bits come from.

With a 32-bit pointer, you can address every individual byte within the compressed class space — a 4 GB region of memory where class metadata lives. But do you actually need byte-level precision for class data?

It turns out, no. Classes are chunky. Most class data structures occupy between 0.5 KB and 1 KB of memory. So instead of addressing every byte, the JVM can divide the 4 GB class space into **1,024-byte (1 KB) blocks**, and address those blocks instead.

The math:

```
4 GB / 1 KB = 4 × 1024 × 1024 × 1024 / 1024 = 4,194,304 blocks
log₂(4,194,304) = 22 bits
```

22 bits address all 4 million blocks. The full 32-bit pointer can be recovered at runtime by shifting the 22-bit block index left by 10 bits — no information is actually lost, just stored more compactly.

That saves 10 bits on the class pointer. Combined with eliminating the 27 unused bits from the Mark Word, the JVM now has room to flatten everything into a single **64-bit compact header**:

| Bits | Content |
|------|---------|
| 63–42 | Compressed class pointer (22 bits) |
| 41–11 | Identity hash code (31 bits) |
| 10–7 | Reserved for Project Valhalla (4 bits) |
| 6–3 | GC age (4 bits) |
| 2 | Self Forwarded Tag (1 bit) |
| 1–0 | Tag bits — lock state (2 bits) |

Everything that was there before is still there. Plus four bonus bits reserved for Valhalla, and one new bit you haven't seen before.

---

## What's the Self Forwarded Tag?

During garbage collection, when the GC copies an object to a new memory address, it needs to leave a forwarding pointer at the old address so any remaining references can be updated. The old layout did this by overwriting the upper 62 bits of the Mark Word with the new address — which was fine, because the Class Word was separate and untouched.

With a compact header, there's no separate Class Word anymore. If the GC overwrote 62 bits of the compact header with a forwarding pointer, it would destroy the class pointer in the process — and without knowing what type an object is, the JVM can't do anything useful with it.

The solution is the **Self Forwarded Tag** bit. During evacuation there are situations where an object ends up forwarding to itself, the JVM simply sets this one bit. The class pointer stays intact. The JVM knows the object is self-forwarded, and can handle it accordingly.

One bit. Elegant.

---

## What This Means in Practice

The header shrinks from 12 bytes to 8 bytes. That 4-byte saving per object sounds modest, but it compounds.

For applications with large numbers of small objects — which is most Java applications — the published numbers are:

- **10–20% reduction in heap usage** (OpenJDK JEP data)
- **22% less heap** in Amazon's production measurements across hundreds of services
- **8–11% more throughput** (depending on workload), thanks to better CPU cache utilization
- **22% less heap + 8% less CPU** in SPECjbb2015 benchmarks

The throughput improvement is the part that surprises people. Smaller objects mean more of them fit in a CPU cache line. Fewer cache misses means less time waiting for memory. It's a free performance win that has nothing to do with your code.

For Spring Boot microservices processing HTTP traffic, the gains will vary depending on your object graph. Applications that allocate lots of small, short-lived objects (request-scoped beans, DTOs, Jackson deserialization intermediates) will see the most benefit. A service that mostly allocates large byte arrays for streaming responses will see less.
> Applications dominated by large arrays or large objects won’t see nearly the same gains, because the header represents only a tiny fraction of each allocation.

---

## How to Use It (Or Not)

In JDK 27, compact object headers are **on by default**. You don't need to do anything. Upgrade, redeploy, profit.

If you're on JDK 25 or 26 and want to try it ahead of time:

```bash
# JDK 25 or 26 — enable explicitly
java -XX:+UseCompactObjectHeaders -jar your-service.jar
```

If you're on JDK 24 (experimental phase, not recommended for production):

```bash
# JDK 24 — unlock experimental first
java -XX:+UnlockExperimentalVMOptions -XX:+UseCompactObjectHeaders -jar your-service.jar
```

And if for some reason you need to disable it in JDK 27 — perhaps you have a native agent or JVM extension that makes assumptions about header layout — there's an escape hatch:

```bash
# JDK 27 — disable if needed
java -XX:-UseCompactObjectHeaders -jar your-service.jar
```

Note that `UseCompressedClassPointers` — the option to disable compressed class pointers — has been removed in JDK 27 entirely. It was deprecated in JDK 25 and couldn't be combined with compact headers anyway.

---

## A Note on Compatibility

This is a JVM-internal change. No Java API is affected. No bytecode changes. No source incompatibilities. If your application compiles and runs on JDK 25 or 26, it will run on JDK 27 with compact headers on.

The only code you might need to revisit is anything that uses `sun.misc.Unsafe` to directly access object headers by memory offset — JVMTI agents, some profilers, some serialization frameworks. If you're using a standard Spring Boot stack, none of this applies to you.

The OpenJDK team ran the full JDK test suite against it. Amazon has been running it in production across hundreds of services, including backports to JDK 17 and JDK 21. SAP's SapMachine JDK distribution already ships with compact headers on by default. The stability track record is solid.

---

## What Comes Next

Project Lilliput — the research project behind all of this — isn't done. There's already a JEP draft targeting **4-byte object headers**. That would halve the header size again, from 8 bytes to 4 bytes. The draft acknowledges this may come with up to 5% throughput regression in some workloads, but the memory savings could be substantial enough to justify it for memory-constrained environments.

JEP 534 is also deliberate preparation for Project Valhalla. Those four reserved bits in the compact header layout are earmarked for value class support — the feature that will let you declare `value class Point { int x; int y; }` and have the JVM flatten instances of it directly into arrays and other objects, allowing many instances to be flattened directly into arrays and enclosing objects instead of always existing as separate heap objects.

The compact header gives Valhalla the bit budget it needs to tag objects as value instances, null-restricted references, and other concepts the type system will eventually need to express. JEP 534 isn't just an optimization — it's infrastructure.

---

JEP 534 is the kind of JEP that makes upgrading JDK versions feel worthwhile without needing a compelling language feature to justify it. Your application gets a free memory reduction and a potential throughput boost, in exchange for updating a version number.

That microservice that was sitting at 70% heap before a single request? I'd genuinely like to know where it lands on JDK 27.

---
## Try it yourself
Curious how much your own objects benefit? Run the same program with JOL (Java Object Layout) on JDK 26 and JDK 27. The output won’t always show objects shrinking by four bytes—alignment still matters—but you’ll see the header itself shrink from 12 bytes to 8 bytes, and across millions of objects those savings add up.
### 1. Add the dependency

For Maven:
```<dependency>
<groupId>org.openjdk.jol</groupId>
<artifactId>jol-core</artifactId>
<version>0.17</version>
</dependency>
```

### 2. Create a tiny example
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
Running this prints something like:   
   
Person object internals:
|OFF|SZ|TYPE DESCRIPTION|
|---|--|----------------|
| 0 |    8   |     (mark)|
| 8  |   4  |      (class)|
| 12  |  4 |   int age|

Instance size: 16 bytes


Notice something interesting.   
Although the header is 12 bytes, the object is 16 bytes, because of object alignment.

### 3. Compare JDK 26 and JDK 27

Run exactly the same program twice.

JDK 26
```java
java \
        -XX:+UseCompactObjectHeaders \
        -cp target/classes:$HOME/.m2/repository/org/openjdk/jol/jol-core/0.17/jol-core-0.17.jar \
com.example.JolExample
```

JDK 27
```java
java \
        -cp target/classes:$HOME/.m2/repository/org/openjdk/jol/jol-core/0.17/jol-core-0.17.jar \
com.example.JolExample
```

You should see something closer to   
   
Person object internals:
|OFF|SZ|TYPE DESCRIPTION|
|---|--|----------------|
|0 | 8 | (header)|
|8 | 4 | int age |

Instance size: 16 bytes

Notice that the header shrank from 12 to **8 bytes**.

The overall object size may or may not shrink depending on alignment in such a small example, but it's a good indication that the JVM is squeezing out the header.

### 4. Print VM information

JOL can also print JVM details:
```java
import org.openjdk.jol.vm.VM;

public class Main {
   void main(String... args) {
     System.out.println(VM.current().details());
   }
}
```

This shows things like:

* object alignment
* compressed oops
* compressed class pointers
* reference size
* object header size

### 5. Graph the entire object graph

One of JOL’s nicest features is:
```java
System.out.println(
    GraphLayout.parseInstance(myObject).toFootprint()
);
```
This will print a graph of the entire object graph, including all of its references.
|COUNT |    AVG |   SUM|
|------|--------|------|
|10000 |     24 |  240000   java.lang.String |
|10000 |     16 |  160000   byte[] |
