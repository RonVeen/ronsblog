---
title: "JEP 535: Shenandoah GC Goes Generational by Default"
date: "2026-09-17"
draft: true
tags: ["Java", "Java 28"]
series: ["Java 28"]
series_order: 4
cover:
  image: "/images/jep535-shenandoha-generational-by-default.png"
  alt: "Java 28"
description: "What changes when Shenandoah's generational mode becomes the default in JDK 28."
---

JEP 535, *Shenandoah GC: Generational Mode by Default*, is targeted for JDK 28. It switches Shenandoah's default collection mode from non-generational to generational, and starts deprecating the non-generational mode with the intent to remove it in a future release.


## What is Shenandoah
The **Shenandoah Garbage Collector** is an ultra-low-pause GC designed for the JVM that performs object compaction concurrently alongside running application threads. Unlike traditional collectors whose Stop-The-World (STW) pause times grow with the size of the heap, Shenandoah maintains consistently low pause times—typically single-digit milliseconds—regardless of whether your heap is 1 GB or 200 GB. This makes it the primary choice when application responsiveness, predictable performance, and meeting strict tail-latency SLAs ($p99$/$p99.9$) take priority over raw batch processing speed.

Typical use cases include low-latency web APIs and microservices, algorithmic financial trading engines, real-time game or media streaming backends, and large in-memory data stores (such as Hazelcast or Cassandra). In these scenarios, traditional GC pauses could cause noticeable UI stutters, broken cluster heartbeats, or missed trades. The main trade-off is a slight increase in overall CPU utilization and a minor reduction in peak throughput compared to collectors like `ParallelGC`, making Shenandoah ideal for environments with sufficient CPU resources dedicated to eliminating latency spikes.

> In case you're wondering where then name Shenendoa comes from: Shenandoah comes from the Shenandoah Valley and Shenandoah River in Virginia, USA.
> It was named by Christine Flood and the engineering team at Red Hat, who originally created and open-sourced the garbage collector. Naming OpenJDK projects and JVM components after geographical landmarks, national parks, and rivers is a longstanding tradition in the Java community.

## The problem with treating every object the same

Shenandoah is a low-pause, mostly concurrent garbage collector. In its original, non-generational form, it tracks liveness across the entire heap in a single pass, regardless of how long any given object has actually been around.

That's wasteful in a specific, predictable way. Most objects in a typical application die young, request-scoped data, temporary buffers, intermediate collections. A small fraction, caches, connection pools, configuration objects, live for the lifetime of the application. A collector that re-scans the whole heap every cycle ends up repeatedly re-verifying that the same long-lived objects are still alive, work that was already true the last several times it checked.

When the heap fills up faster than a concurrent cycle can keep pace, non-generational Shenandoah falls back to a full stop-the-world collection across the entire heap. On heaps that mix long-lived and short-lived data, that full collection can produce pause times far outside what the collector is otherwise known for.

## What generational mode changes

Generational Shenandoah separates the heap into a young generation and an old generation. Short-lived objects are collected quickly within the young generation, without touching the old generation at all. The old generation is swept far less frequently, since its contents change far less often.

This isn't a new capability being introduced by JEP 535, generational mode has existed as an opt-in flag for several releases and has matured through that period. JEP 535's change is narrower: it makes that existing mode the default, so applications get it without any configuration.

Reported results from production workloads running the generational mode show memory footprint reductions of roughly 15 to 25 percent, along with more predictable pause behavior under allocation-heavy workloads such as HTTP servers and streaming pipelines.

Shenandoa is not the first GC to get this generational feature. While GC's like 'G1', 'Parallel GC', 'Serial GC', and 'CMS' already have this feature by default, it was recently also added to 'ZGC'

## What this requires from you

Nothing, in terms of code. This is a JVM default change, not an API. Applications already running on Shenandoah get the generational behavior automatically after upgrading to JDK 28, with no annotations, flags, or tuning required to get the improved behavior.

The non-generational mode remains available for now via explicit flag, but it's deprecated as of this JEP, with removal intended in a later release. Anything currently relying on non-generational-specific tuning should plan to migrate before that happens.

## A footnote about GC's
Java has a number of GC's that each serve a different purpose. As the demands on applications became different, total available memory increased, and more processors became available, the need for different GC's arose.   
Here's a list of the current GC's, their main purpose and when they went to production
* **Serial GC** – Minimal memory, single-threaded – December 1998 (JDK 1.2)
* **Parallel GC** – Maximizes throughput over pause times – February 2002 (JDK 1.4)
* **CMS GC** – Early low-pause concurrent collector – June 2003 (JDK 1.4.2)
* **G1 GC** – Balanced performance for large heaps – April 2012 (JDK 7u4)
* **Epsilon GC** – No-op allocation with zero reclamation – September 2018 (JDK 11)
* **ZGC** – Scalable sub-millisecond pause times – September 2020 (JDK 15)
* **Shenandoah GC** – Ultra-low pauses via concurrent compaction – September 2020 (JDK 15)

