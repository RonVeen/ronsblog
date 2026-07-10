---
title: "Java 27 - part 1: JEP 523 - G1 Becomes the Default Everywhere"
date: 2026-05-22
draft: false
tags: ["java", "java27", "jdk27", "garbage-collection", "g1", "jep523", "performance"]
cover:
  image: "/images/jep523-g1-default-everywhere.png"
  alt: "Structured Concurrency"
series: ["Java 27"]
series_order: 2
categories: ["java"]
---
**G1 Garbge Collector, Always.**   
If you've ever wondered which garbage collector the JVM actually picked for you when you *didn't* tell it which one to use, the answer was always a little annoying: "well, it depends."

That "it depends" is exactly what JEP 523, targeting JDK 27, gets rid of. From now on, when you don't specify a collector on the command line, the JVM always picks **G1**. No exceptions, no hardware-dependent surprises. Let me walk you through what's changing and — more importantly — whether you should care.

> This is part 2 of my Java 27 series. If you want the bigger picture of where Java 27 is heading, start with the [series overview](https://ronveen.com/posts/java27-overview/).

## First, a quick recap: who picks the collector?

The HotSpot JVM ships with several garbage collectors, each tuned for a different priority. ZGC chases ultra-low latency, Parallel maximizes raw throughput, and G1 ("Garbage First") tries to give you a sensible balance of both. When you don't name one explicitly, the JVM picks a default for you.

And that's where it got interesting. Since JDK 9 — via [JEP 248](https://openjdk.org/jeps/248) — G1 has been the default. But only on what the JVM considers "server-class" hardware. In practice that meant a machine with **2 or more CPUs and roughly 1792 MB (≈ 2 GB) of physical memory or more**. On anything smaller — a single CPU *and* less than that memory threshold — the JVM quietly fell back to the **Serial** collector.

Why Serial down there? Because back in the JDK 9 days, Serial genuinely had the edge in those cramped conditions. Lower throughput overhead, smaller memory footprint. On a tiny box, that mattered.

Note the past tense.

## So why change it now?

Because that advantage has largely evaporated. Over the last several releases, G1 has been quietly improving across *every* metric, to the point where it's now competitive with Serial even at small heap sizes. The JEP makes the case on three fronts:

- **Throughput** — This is the big one, and it's the direct payoff of the work I covered in [part 5 of the Java 26 series](https://ronveen.com/posts/jep522-g1-improve-througput-by-reducing-synchronization/). [JEP 522](https://openjdk.org/jeps/522) reduced G1's synchronization overhead with its "double buffering" / second card table trick, and that pushed G1's maximum throughput right up close to Serial's — even in the constrained scenarios where Serial used to win.
- **Latency** — G1 was *always* better here. It reclaims memory in the old generation through incremental collections instead of leaning on big stop-the-world full GCs, so its worst-case pauses have consistently beaten Serial's. This was never the problem.
- **Memory footprint** — Recent releases have trimmed G1's native memory usage down to roughly Serial's level. This was historically Serial's home turf, and G1 has now caught up.

Put those three together and the conclusion writes itself: G1 is now good enough to take Serial's place in the one corner where Serial still held on.

## What this actually means for you

Honestly? For most readers, **nothing visible** — and that's kind of the point.

If you're deploying Spring Boot services to a cloud platform, an OpenShift cluster, or any container with a couple of cores and a gig or two of RAM, you were already getting G1. Nothing changes for you except that the JVM's decision-making got simpler to reason about. No more "wait, which collector is running in *this* environment?" Whatever box it lands on, it's G1.

Where you'll *actually* notice this is on the small, constrained machines that used to get Serial:

- Tiny single-core containers and tightly-capped CI runners
- Function-style / serverless workloads on minimal resource allocations
- Raspberry Pi-class hardware and other small SBCs (something I've got a soft spot for)

On those, you'll now get G1 by default — and in many cases better worst-case latencies as a result, since that was always G1's strong suit.

One honest caveat, because I don't like overselling things: the JEP's stated goal is that throughput, footprint, and startup time "should not degrade *significantly*" in those constrained cases. That's deliberately not a promise of *zero* regression. On a genuinely minimal single-core box, Serial can still occasionally edge G1 out on footprint or cold-start time. So if you've measured your specific workload and Serial wins — keep using it. Which brings me to the reassuring part.

## What is NOT changing

This JEP is purely about the *default selection algorithm*. That's it. To be crystal clear:

- Serial is **not** deprecated and **not** being removed. It's still there.
- G1's behavior and functionality are **not** changing — only how often it gets auto-selected.
- Your explicit choice always wins. If you know your app runs best on Serial (or Parallel, or ZGC), name it on the command line and the JVM will respect that, every time:

```bash
java -XX:+UseSerialGC -jar my-app.jar
```

The default is just a starting point — a sensible one, chosen *for* you rather than *by* you. JEP 523 doesn't take the steering wheel away; it just stops making you guess which collector showed up for work today.

## The takeaway

Since JDK 9 the rule was "G1, *unless* your hardware is small, in which case Serial." JEP 523 collapses that into a single, boring, wonderful rule: **G1, always.** It's a small change on paper, but it removes a genuine source of "wait, why is it behaving differently here?" — and those are exactly the kind of changes I appreciate.

If you want to go one level deeper on *why* this became possible, go read about [JEP 522 and G1's double buffering](https://ronveen.com/posts/jep522-g1-improve-througput-by-reducing-synchronization/) — JEP 523 is basically the victory lap for that work.
