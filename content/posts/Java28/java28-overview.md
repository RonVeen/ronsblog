---
title: "Java 28 - Overview"
date: 2026-09-13
draft: false
tags: ["Java", "Java 28"]
series: ["Java 28"]
series_order: 1
cover:
  image: "/images/java28-series.png"
  alt: "Java 28"
---

## Java 28 is due to be launched in March 2027
### At this moment, these are the JEPs that are going in this release.


- **JEP 401 Value Objects**
The very release of Project Valhalla deliverables, focussing on value class, i.e. classes without identify.


- **JEP535 Shenandoha GC: Generation by Default**
The Shenandoha Garbage Collector now uses the generational approach already know from other GCs like G1. 


- **[JEP539 Strict Field initialization in the JVM](/posts/jep-539-strict-field-initialization-in-the-jvm-preview/)**
Strictly-initialized fields must be initialized before they are read, offering stronger integrity guarantees.

- **JEP504 Simple JSON API**
Exactly what the title says, a simple JSON API.

- **JEP541 Deprecate the macOS/x64 Port for Remocal**
Depricate the macOS Port so it can be removed in a future release.

- **JEP 542 PEM Encodings of Cryptographic Objects**
Another round of reviews for the PEM Encodings of Cryptographic Objects API. Nothing has changed from [the version in Java 27](/posts/java-27-jep-538-pem-encodings-of-cryptographic-objects-third-preview/). 
