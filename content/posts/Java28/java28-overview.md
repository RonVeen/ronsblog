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

Java 28 is due be launched in Mart 2027
At this moment, these are the JEPs that are going in this release.


**JEP 401 Value Objects**
The very release of Project Valhalla deliverables, focussing on value class, i.e. classes without identify.


**JEP535 Shenandoha GC: Generation by Default**
The Shenandoha Garbage Collector now uses the generational approach already know from other GCs like G1. 


**JEP539 Strict Field initialization in the JVM**
Strictly-initialized fields must be initialized before they are read, offering stronger integrity guarantees.


**JEP504 Simple JSON API**
Exactly what the title says, a simple JSON API.


**JEP541 Deprecate the macOS/x64 Port for Remocal**
Depricate the macOS Port so it can be removed in a future release.

**JEP 542 PEM Encodings of Cryptographic Objects**
Another round of reviews for the PEM Encodings of Cryptographic Objects API. Nothing has changed from [the version in Java 27](/posts/java-27-jep-538-pem-encodings-of-cryptographic-objects-third-preview/). 
