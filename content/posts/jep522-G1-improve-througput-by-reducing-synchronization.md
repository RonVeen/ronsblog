---
title: "Java 26 - part 5: JEP-522 Turbocharging G1 with 'Double Buffering'"
date: 2026-02-02
draft: false
tags: ["Java", "JEP", "Java 26", "G1", "Garbage Collection"]
---

![Final](/images/jep522-g1-gc-improved-througput.png)

If you run Java in production, you are likely using the G1 ("Garbage First") Garbage Collector (it's the default, after all). 
In Java 26, G1 is getting a significant "free" performance boost—estimated at **5–15% better throughput**—without you changing a single line of code.

This is one of those JEPs that does its work silently in the background (literally), but it's implications are so significant that it deserves a blog post.

**The Problem:**
To clean up memory efficiently, G1 needs to track which objects point to which other objects. It does this using a "Card Table" (think of it as a cheat sheet of modified memory areas). 
Previously, your application threads and the GC's background threads were fighting over the *same* Card Table. 
To prevent corruption, they had to use synchronization (locks), which created a bottleneck.
This is not very different from how you would a synchronized method, except that it's happening in the background.

**The Fix (JEP 522):**
Java 26 introduces a **second Card Table**.
It works like "double buffering" in graphics. Your application threads write to **Card Table A** (no locks required!), while the GC background threads process **Card Table B**. When the GC needs to catch up, they simply **swap** the tables atomically.

**The "Code" Impact:**
Because the application threads no longer need complex locking logic when updating memory, the "write barrier" code injected into your app is drastically smaller.
*   **Before:** ~50 CPU instructions per reference update.
*   **After:** ~12 CPU instructions.

#### Source Code Example
You cannot "call" JEP 522 directly because it changes the machine code the JVM generates under the hood. However, it affects every single object assignment you write.

Here is what is happening conceptually:

```java
public class UserSession {
    private User user;

    public void updateUser(User newUser) {
        // JEP 522 OPTIMIZES THIS LINE:
        this.user = newUser; 
    }
}
```

**What happens under the hood?**
When you write `this.user = newUser`, the JVM injects a "Write Barrier"—a snippet of code that updates the Card Table so the GC knows you changed a reference.

**Before Java 26 (Pseudo-Assembly):**
```text
STORE newUser INTO this.user
LOCK CardTable_Mutex   <-- Expensive synchronization!
MARK CardTable Entry
UNLOCK CardTable_Mutex
```

**In Java 26 (Pseudo-Assembly):**
```text
STORE newUser INTO this.user
MARK Local_CardTable Entry  <-- Fast, no locks, 4x fewer instructions
```

***

### Deep Dive: Understanding the G1 Garbage Collector

To understand why this change matters, we need to look at what G1 actually is and why it's the "Goldilocks" of Java garbage collectors.

#### 1. What does G1 do?
The **Garbage-First (G1) Garbage Collector** is designed to balance **throughput** (how much work your app does) with **latency** (how long your app pauses to clean memory). 
It tries to meet a user-defined pause time target (e.g., "don't stop for more than 200ms") while still processing a high volume of transactions.

#### 2. How does it do it?
G1 operates differently than older collectors like Serial or Parallel:

*   **Regions:** Instead of one giant block of memory, G1 divides the heap into thousands of small regions (e.g., 2,048 chunks).
*   **Evacuation:** It reclaims memory by copying live objects from "full" regions into fresh, empty regions. The old regions are then discarded.
*   **Garbage-First:** It tracks which regions contain the most "garbage" (dead objects) and prioritizes cleaning those first—hence the name "Garbage-First".
*   **Card Tables & Write Barriers:**
    *   Since G1 collects regions individually, it needs to know if an object in *Region A* references an object in *Region B*.
    *   Scanning the whole heap to find these references would be too slow.
    *   Instead, it uses a **Card Table** to track "dirty" areas where references were updated.
    *   Every time you assign a value (`a.field = b`), a **Write Barrier** updates this table. 
    * JEP 522 makes this specific mechanism much faster by removing the need for threads to lock when updating the table.

#### 3. Why choose G1 over other GCs?
Choosing a GC is about choosing which trade-off you can live with:

| Garbage Collector | Best For... | The Trade-off |
| :--- | :--- | :--- |
| **Parallel GC** | **Raw Throughput.** Great for batch processing or data crunching where you don't care if the app freezes for 2 seconds occasionally. | **High Latency.** It stops the world completely to clean up. |
| **ZGC / Shenandoah** | **Low Latency.** Great for trading platforms or UI backends where pauses must be under 1ms. | **High Overhead.** To achieve such low latency, they use more CPU and memory, often resulting in lower total throughput than G1. |
| **G1 GC (Default)** | **The Balance.** The best choice for most general-purpose web servers and microservices. | **Jack of all trades.** It isn't as fast as Parallel or as smooth as ZGC, but it avoids the extremes of both. |

**Summary:** You choose G1 when you want a stable, predictable system that doesn't freeze for long periods but still handles a heavy workload efficiently. 
With the JEP 522 updates in Java 26, G1 is now significantly closer to Parallel GC in terms of raw speed while keeping its latency benefits.