---
title: "Java 26"
date: 2026-01-22
draft: false
tags: ["Java", "Java 26"]
---

![Final](/images/java26.png)

We've only just seen the release of Java 25, the latest LTS version of the popular language.
But on March, 21st, the next iteration of the Java language, Java 26, will be launched.

## What is in there?
Java 26 will be made up of 10 Java Enhancement Proposals.
Here is a short summary. In the coming weeks, I will talk about each of these in more detail on this site.

[JEP 500: Prepare to Make Final Mean Final](/posts/jep500-finals/)

This feature issues warnings when deep reflection is used to modify `final` fields, aiming to restore their integrity and enable JVM optimizations like constant folding,. It prepares developers for a future release where such modifications will be blocked by default to prevent undefined behavior.
<br>


[JEP 504: Remove the Applet API](/posts/jep504-Remove-the-applet-API/)

This JEP permanently removes the `java.applet` package and related classes, which were previously deprecated for removal in JDK 17. Since web browsers no longer support applets and the Security Manager was disabled in JDK 24, the API is now obsolete. 
<br>


[JEP 516: Ahead-of-Time Object Caching with Any GC](/posts/jep516-Aot-object-caching-with-any-gc/)

This enhancement allows the Project Leyden AOT cache to support all garbage collectors, including ZGC, by introducing a "GC-agnostic" format using logical indices. Previously, caches were tied to specific GCs due to memory layout differences, forcing a tradeoff between fast startup and low latency. The JVM now streams objects into memory and materializes them according to the active collector's requirements.
<br>

[JEP 517: HTTP/3 for the HTTP Client API](/posts/jep517-http3-for-the-http-client-api/)
This update enables the `java.net.http.HttpClient` to support HTTP/3, utilizing QUIC over UDP for improved performance in packet-loss environments.

[JEP 522: G1 GC: Improve Throughput by Reducing Synchronization](/posts/jep522-G1-improve-througput-by-reducing-synchronization/)
This performance improvement for the G1 garbage collector introduces a second "card table" to track heap modifications. This allows application threads and GC optimizer threads to update separate tables, eliminating synchronization locks and simplifying write barriers. The result is a 5–15% throughput increase for applications with heavy reference updates.

**JEP 524: PEM Encodings of Cryptographic Objects (Second Preview)**
This preview feature provides a standardized API (`PEMEncoder` and `PEMDecoder`) to convert cryptographic keys and certificates to and from the PEM textual format. This second preview adds support for encrypting and decrypting `KeyPair` and `PKCS8EncodedKeySpec` objects, simplifying code that previously required manual Base64 parsing.

**JEP 525: Structured Concurrency (Sixth Preview)**
Structured concurrency treats groups of related tasks running in different threads as a single unit of work, ensuring clearer error handling and preventing thread leaks. This sixth preview updates the `Joiner` interface to return lists of results rather than streams and refines the `onTimeout` handling. 

**JEP 526: Lazy Constants (Second Preview)**
Renamed from "Stable Values," this feature allows variables to be initialized lazily via a computing function while being treated as true constants by the JVM for optimization purposes. Changes in this preview include removing low-level methods, disallowing `null` values, and adding `List.ofLazy` and `Map.ofLazy` factories.

**JEP 529: Vector API (Eleventh Incubator)**
This API allows developers to express complex vector computations that compile to efficient SIMD hardware instructions at runtime. It remains in incubation without substantial changes, waiting for Project Valhalla's value classes to become available for future integration.

**JEP 530: Primitive Types in Patterns, instanceof, and switch (Fourth Preview)**
This feature removes restrictions on using primitive types in pattern matching, `switch`, and `instanceof`, enabling checks like `i instanceof byte` for safe casting. The fourth preview enhances dominance checks to catch more unreachable code errors and refines the definition of unconditional exactness.



