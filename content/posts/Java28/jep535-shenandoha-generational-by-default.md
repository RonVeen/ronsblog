---
title: "JEP 535: Shenandoah GC Goes Generational by Default"
date: "2026-11-11"
draft: false
tags: ["Java", "Java 28", "Shenandoah", "GC", "JVM"]
series: ["Java 28"]
series_order: 4
cover:
  image: "/images/jep535-shenandoha-generational-by-default.png"
  alt: "Java 28"
description: "JEP 535, *Shenandoah GC: Generational Mode by Default*, is targeted for JDK 28. It switches Shenandoah's default collection mode from non-generational to generational, and starts deprecating the non-generational mode with the intent to remove it in a future release."
---
# JEP 535: Generational Shenandoah Takes the Wheel

Imagine vacuuming your entire house, attic, guest bedrooms, and basement included, every single time you drop a crumb on the kitchen floor. It is thorough, sure. You will never have dust bunnies in the attic. Brought into a busy kitchen, though, you will spend half your life pushing a vacuum around rooms nobody has stepped foot in for months, and eventually, you won't be able to keep up with the crumbs in the kitchen.

For years, that is exactly how single-generation concurrent garbage collectors operated. JEP 535, *Shenandoah GC: Generational Mode by Default*, is targeted for JDK 28. It switches Shenandoah's default collection mode from non-generational to generational, and starts deprecating the non-generational mode with the intent to remove it in a future release.

## What is Shenandoah

The **Shenandoah Garbage Collector** is an ultra-low-pause GC designed for the JVM that performs object compaction concurrently alongside running application threads. Unlike traditional collectors whose Stop-The-World (STW) pause times grow with the size of the heap, Shenandoah maintains consistently low pause times (typically single-digit milliseconds) regardless of whether your heap is 1 GB or 200 GB. This makes it a primary choice when application responsiveness, predictable performance, and meeting strict tail-latency SLAs ($p99$/$p99.9$) take priority over raw batch processing speed.

Typical use cases include low-latency web APIs and microservices, algorithmic financial trading engines, real-time game or media streaming backends, and large in-memory data stores such as Hazelcast or Cassandra. In these scenarios, traditional GC pauses could cause noticeable UI stutters, broken cluster heartbeats, or missed trades. The main trade-off is a slight increase in overall CPU utilization and a minor reduction in peak throughput compared to collectors like `ParallelGC`, making Shenandoah ideal for environments with sufficient CPU resources dedicated to eliminating latency spikes.

> In case you're wondering where the name Shenandoah comes from: Shenandoah comes from the Shenandoah Valley and Shenandoah River in Virginia, USA. It was named by Christine Flood and the engineering team at Red Hat, who originally created and open-sourced the garbage collector. Naming OpenJDK projects and JVM components after geographical landmarks, national parks, and rivers is a longstanding tradition in the Java community.

## The Problem with Treating Every Object the Same

Shenandoah is a low-pause, mostly concurrent garbage collector. In its original, non-generational form (`satb`, short for Snapshot-At-The-Beginning), it tracks liveness across the entire heap in a single pass, regardless of how long any given object has actually been around.

That is wasteful in a specific, predictable way. The weak generational hypothesis never misses: most objects in a typical application die young. If you run a Spring Boot service handling BrightCart checkout requests, each incoming HTTP payload spawns a small blizzard of request-scoped data, temporary Jackson parser buffers, intermediate collections, and JDBC result set wrappers. Ninety-five percent of those objects are stone-cold dead before the HTTP 200 response even hits the client's network card. Meanwhile, a small fraction of objects, like caches, connection pools, and Spring singleton beans, live for the entire lifetime of the application.

A collector that re-scans the whole heap on every cycle ends up repeatedly re-verifying that those same long-lived objects are still alive, doing work that was already true the last several times it checked.

Worse yet, because concurrent collection runs *while* your application threads are still allocating like crazy, classic Shenandoah needed a generous headroom buffer of free heap space. When the heap fills up faster than a concurrent cycle can keep pace, non-generational Shenandoah falls back to a degenerated or full Stop-The-World collection across the entire heap. On heaps that mix long-lived and short-lived data, that fallback pause can blow right past the strict latency SLAs you chose Shenandoah for in the first place.

## What Generational Mode Changes

Generational Shenandoah separates the heap into a **Young** generation and an **Old** generation:

1. **Frequent, cheap Young collections:** Short-lived objects are collected quickly within the young generation while its regions are still warm in CPU caches, without touching the old generation at all.
2. **Rare Old collections:** Long-lived objects get promoted to the old generation and are swept far less frequently, since its contents change far less often.
3. **Smaller footprint, steadier pauses:** Production workloads running generational Shenandoah show memory footprint reductions of roughly 15 to 25 percent, along with much more predictable pause behavior under allocation-heavy workloads like HTTP servers and streaming pipelines.

This isn't a brand-new engine being bolted on in JDK 28. Generational mode debuted as an experimental feature in JDK 24 (JEP 404) and graduated to production status as an opt-in flag in JDK 25 (JEP 521). JEP 535's change is narrower but decisive: it makes that battle-tested generational mode the default, so applications get it out of the box without extra configuration.

Shenandoah is also in good company here. While collectors like `G1`, `Parallel GC`, `Serial GC`, and the older `CMS` were generational from the start, `ZGC` made the exact same jump to generational mode by default in JDK 23 (JEP 474) before removing its non-generational mode in JDK 24 (JEP 490).

## What This Requires from You

Nothing, in terms of application code. This is a JVM default change, not an API change. Applications already running on Shenandoah get the generational behavior automatically after upgrading to JDK 28, with no annotations, code changes, or tuning flags required.

Up through JDK 27, opting into Shenandoah gave you the classic single-generation `satb` collector unless you explicitly asked for generational mode:

```bash
# JDK 25 - 27: You had to ask nicely for generational mode
java -XX:+UseShenandoahGC -XX:ShenandoahGCMode=generational -jar brightcart-order-service.jar
```

Starting in **JDK 28**, `-XX:ShenandoahGCMode` defaults to `generational`. You just turn Shenandoah on:

```bash
# JDK 28: Generational mode out of the box
java -XX:+UseShenandoahGC -jar brightcart-order-service.jar
```

The non-generational mode remains available for now via an explicit flag, though it is officially **deprecated** as of JEP 535 and slated for removal in a future release. Any workload currently relying on non-generational tuning should plan to migrate now:

```bash
# JDK 28: Falls back to classic single-gen, but prints a deprecation warning
java -XX:+UseShenandoahGC -XX:ShenandoahGCMode=satb -jar brightcart-order-service.jar
```
```text
OpenJDK 64-Bit Server VM warning: Non-generational mode of Shenandoah GC is deprecated and will be removed in a future release.
```

## Observing the Difference in Code

Even though your business logic doesn't change, your JMX metrics, Micrometer exporters, and GC logs will reflect the new heap topology immediately.

Want to see how the two modes expose themselves at runtime and stress-test young-generation churn against a tenured cache? Drop this self-contained benchmark into a file and run it under both modes on a JDK 28 build:

```java
import java.lang.management.GarbageCollectorMXBean;
import java.lang.management.ManagementFactory;
import java.lang.management.MemoryPoolMXBean;
import java.math.BigDecimal;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;
import java.util.concurrent.ThreadLocalRandom;

/**
 * Run with Default (Generational) Mode in JDK 28:
 *   java -XX:+UseShenandoahGC -Xmx512m  ShenandoahGenDemo.java
 *
 * Compare with Deprecated Classic (SATB) Mode:
 *   java -XX:+UseShenandoahGC -XX:ShenandoahGCMode=satb -Xmx512m ShenandoahGenDemo.java
 */
public class ShenandoahGenDemo {

    record LineItem(String sku, int quantity, BigDecimal unitPrice) {}
    record OrderSnapshot(UUID orderId, Instant createdAt, List<LineItem> items, byte[] payloadBuffer) {}

    // Simulates Old-Gen tenured state (e.g., an in-memory product catalog or session cache)
    private static final List<OrderSnapshot> TENURED_CACHE = new ArrayList<>();

    public static void main(String[] args) {
        printCollectorTopology();

        // Step 1: Populate ~180MB of long-lived objects that will promote to Old Gen
        System.out.println("\nWarming up tenured catalog cache...");
        for (int i = 0; i < 3_000; i++) {
            TENURED_CACHE.add(createOrder(60 * 1024)); // ~60KB each
        }

        // Step 2: Simulate heavy HTTP request churn (short-lived allocations that die immediately)
        System.out.println("Simulating high-rate ephemeral order validation churn...");
        long start = System.nanoTime();
        long checksum = 0;

        for (int request = 0; request < 250_000; request++) {
            OrderSnapshot ephemeralOrder = createOrder(4 * 1024); // 4KB ephemeral payload
            checksum += ephemeralOrder.items().size() + ephemeralOrder.payloadBuffer()[0];
        }

        long elapsedMs = (System.nanoTime() - start) / 1_000_000;
        System.out.printf("Completed 250,000 ephemeral allocations in %d ms (checksum: %d)%n%n", elapsedMs, checksum);

        printGcSummary();
    }

    private static OrderSnapshot createOrder(int bufferSize) {
        var rng = ThreadLocalRandom.current();
        var items = List.of(
                new LineItem("SKU-" + rng.nextInt(1000), rng.nextInt(1, 5), BigDecimal.valueOf(29.99)),
                new LineItem("SKU-" + rng.nextInt(1000), rng.nextInt(1, 3), BigDecimal.valueOf(149.50))
        );
        byte[] buffer = new byte[bufferSize];
        buffer[0] = (byte) rng.nextInt(127);
        return new OrderSnapshot(UUID.randomUUID(), Instant.now(), items, buffer);
    }

    private static void printCollectorTopology() {
        System.out.println("=== Active Memory Pools ===");
        for (MemoryPoolMXBean pool : ManagementFactory.getMemoryPoolMXBeans()) {
            if (pool.getName().contains("Shenandoah")) {
                System.out.printf(" - Pool: %-28s | Type: %s%n", pool.getName(), pool.getType());
            }
        }
    }

    private static void printGcSummary() {
        System.out.println("=== GC Collection Summary ===");
        for (GarbageCollectorMXBean gc : ManagementFactory.getGarbageCollectorMXBeans()) {
            System.out.printf(" - Collector: %-25s | Count: %4d | Time: %4d ms%n",
                    gc.getName(), gc.getCollectionCount(), gc.getCollectionTime());
        }
    }
}
```

When you run this with `-XX:+UseShenandoahGC` on JDK 28, look at the `MemoryPoolMXBean` output first: instead of a single monolithic `Shenandoah` heap pool, you see distinct **`Shenandoah Young Gen`** and **`Shenandoah Old Gen`** pools.

## Production Gotchas Before You Upgrade

Flipping a default flag sounds harmless until your observability pipeline or startup scripts trip over it on deployment day. Keep three things on your radar:

* **Grafana dashboards and alerting rules:** Because the memory pool names split into `Shenandoah Young Gen` and `Shenandoah Old Gen`, any Prometheus or Micrometer query hardcoded to alert on `jvm_memory_used_bytes{id="Shenandoah"}` will suddenly stop reporting data. Check your dashboard filters before bumping the base image to JDK 28.
* **GC log parsers and stderr checks:** If you have strict CI/CD health checks or wrapper scripts that fail a deployment when unexpected `WARNING` lines appear on `stderr`, and your ops team had `-XX:ShenandoahGCMode=satb` explicitly baked into `JAVA_TOOL_OPTIONS`, that new deprecation warning will trip them.
* **Remembered-set write barriers:** Generational collectors have to track references that point from Old-generation objects into Young-generation objects (the "remembered set"). That adds a tiny bit of card-marking work to reference writes compared to non-generational `satb`. For 99% of microservices, the massive reduction in marking work dwarfs that barrier cost... but if you run an unconventional workload that mutates millions of references inside long-lived Old arrays every second with almost zero young allocation, benchmark both modes first.

## A Footnote About Java's Garbage Collectors

Java provides a whole lineup of garbage collectors, each engineered for a distinct operational profile. As application demands evolved, total available RAM grew from megabytes to terabytes, and multi-core processors became the norm, new collectors stepped in to solve new bottlenecks.

Here is the roll call of Java's garbage collectors, their primary design goal, and when they first went into production:

* **Serial GC** (December 1998, JDK 1.2): Minimal memory footprint, single-threaded collection for small heaps and client apps.
* **Parallel GC** (February 2002, JDK 1.4): Maximizes raw application throughput over pause times using multi-threaded Stop-The-World phases.
* **CMS GC** (June 2003, JDK 1.4.2): The early pioneer of low-pause concurrent collection (later deprecated in JDK 9 and removed in JDK 14).
* **G1 GC** (April 2012, JDK 7u4): Balanced throughput and bounded pause times for large heaps; the longtime default collector.
* **Epsilon GC** (September 2018, JDK 11): A no-op allocator with zero memory reclamation, built for ultra-short-lived jobs and microbenchmarking.
* **ZGC** (September 2020, JDK 15): Scalable sub-millisecond pause times across multi-terabyte heaps.
* **Shenandoah GC** (September 2020, JDK 15): Ultra-low pause times via concurrent compaction, independent of heap size.


