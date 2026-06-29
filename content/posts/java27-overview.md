---
title: "Java 27 - Overview"
date: 2026-03-17
draft: false
tags: ["Java", "Java 27"]
series: ["Java 27"]
series_order: 1
cover:
  image: "/images/java27-series.png"
  alt: "Java 27"

---

Java 27 is currently in [rampdown phase one](https://openjdk.org/jeps/3#rdp-1) and will reach general availability on September, 15th, 2026.
The release will consist of 9 [JEPs](https://openjdk.org/jeps/1).

## What is in there?
Java 27 will consist of these JEPs.

[**JEP 523 G1 becomes the default GC everywhere**](/posts/jep523-g1-default-everywhere/)   
Simplifies Java's garbage collection behavior by making the G1 Garbage Collector the default choice across all hardware, removing the old rule where small or single-core machines would automatically fall back to the Serial collector.
   
[**JEP 527 Post-quantum Hybrid Key Exchange for TLS 1.3**](/posts/jep527-post-quantum-tlsjep527-post-quantum-tls/)   
Introduces a hybrid key exchange for Java 27 that layers new quantum-resistant math (ML-KEM) on top of proven traditional encryption to ensure data remains secure.
   
[**JEP 531 Lazy Constants**](/posts/jep531-lazy-constants/)   
Delivers a third preview pass for Java 27's Lazy Constants API, focusing on polishing the deferred immutability feature rather than completely reinventing it.
   
[**JEP 532 Primitive types in patterns**](/posts/jep532-primitive-types-in-patterns/)   
Another round of review for primitives in patterns. Nothing new in this JEP, it is just asking the community for feedback.
   
[**JEP 533 Structured Concurrency, 7th preview**](/posts/jep533-structured-concurrency)
The seventh review of the new structured concurrency API.

[**JEP 534 Compact Object Headers by Default**](/posts/jep435-compact-object-headers-by-default)   
After proving successful in production for the two releases, Compact Object Headers will now become the default instead of an opt-in.

**JEP 536 JFR In-process Data Redaction**   

**JEP 537 Vector API, twelfth Incubator**      
Yet another round of review for the new Vector API, without any new features.

**JEP 538 PEM Encodings for Cryptographic Objects**      
Third preview for this API withou any new features.





