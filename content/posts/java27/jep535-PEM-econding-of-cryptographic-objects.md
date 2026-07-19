---
title: "Java 27 - JEP 538 - PEM Encodings of Cryptographic Objects (Third Preview)"
description: "The PEM API's third preview lands in JDK 27 with real API polish and no more excuses to hand-roll PEM parsing yourself."
date: 2026-07-20
draft: true
tags: ["Java", "Security", "Cryptography"]
cover:
  image: "/images/jep535-pem-encodings.png"
  alt: "JEP 538 - PEM Encodings of Cryptographic Objects (Third Preview)"
series: ["Java 27"]
series_order: 9
categories: ["java"]
---
Somewhere in one of my customers repo there's a private method that parses `-----BEGIN PRIVATE KEY-----` blocks by hand: strip the header, strip the footer, Base64-decode what's left, hand the bytes to a `KeyFactory`, hope you picked the right algorithm. I wrote something almost exactly like it years ago. It wasn't hard. It was just the kind of tedious that makes you wonder why the JDK never gave you this for free.

JEP 538 is that "for free" version, and as of JDK 27 it's done cooking through preview status for the third time. Third preview, and this round reads less like new ideas and more like the API team sanding down every edge someone tripped over in round two.

## What PEM Encoding Actually Is

PEM stands for Privacy-Enhanced Mail, a name that oversells what the format actually is by several decades — it was built for e-mail, and now it's used for almost anything except. It's defined in RFC 7468, and the shape of it couldn't be simpler: Base64-encoded binary data, wrapped in a header and footer that name what's inside.

```
-----BEGIN PUBLIC KEY-----
MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEi/kRGOL7wCPTN4KJ2ppeSt5UYB6u
cPjjuKDtFTXbguOIFDdZ65O/8HTUqS/sVzRF+dg7H3/tkQ/36KdtuADbwQ==
-----END PUBLIC KEY-----
```

Strip the `BEGIN`/`END` markers, Base64-decode the middle, and you get the object's standard binary encoding — X.509 for public keys, certificates, and certificate revocation lists; PKCS#8 for private keys. That's genuinely the whole format: a label plus a Base64 blob.

You've touched this more than you'd think even if the acronym never came up by name. TLS certificate chains from your CA arrive as PEM. OpenSSL generates and converts PEM. OpenSSH stores your keys in a PEM-flavored format. Hardware tokens like YubiKeys read and write PEM. It's one of those formats that quietly became load-bearing infrastructure for the entire security ecosystem without a formal decision ever being made to let that happen.

## Why the JDK Needed to Catch Up

The strange part is how long the Java Platform went without a first-class API for any of this. Every key and certificate object already knows how to produce its own binary encoding, and `java.util.Base64` handles the text conversion — but wiring those two pieces together was left entirely to whoever needed it.

Encoding was tedious but survivable. Decoding was worse: parse the header text to identify the object type, pick the matching factory, get the algorithm right, and hope the input wasn't malformed. Encrypting a private key on the way out took over a dozen lines by itself. A 2022 Java Cryptographic Extensions survey confirmed this was a real, widely-felt pain point — not just a personal grievance from developers who'd been burned by it.

## Third Preview, Same Shape, Sharper Edges

The API's shape hasn't moved since JEP 470 introduced it in JDK 25: `PEMEncoder` and `PEMDecoder` as your two entry points, immutable and reusable. What JEP 538 changes is a set of refinements that all point the same direction — fewer surprises at the edges:

**`PEM` becomes an ordinary class instead of a record.** Small on paper, but it unlocks new constructors that accept Base64 content directly as a `byte[]`, which a record's fixed-shape components made awkward to add.

**`DEREncodable` is renamed `BinaryEncodable`.** The old name borrowed from DER, a specific binary encoding, for an interface that's really just "this has a binary representation." The new name says what it means instead of what it used to be called internally.

**`EncryptedPrivateKeyInfo` gains `getKeyPair` methods.** You could already decrypt down to a `PrivateKey`; now, when the encoded data carries both halves, you can decrypt straight to a full `KeyPair`.

**`getKey`/`getKeyPair` drop the `Provider` parameter.** What used to take a password plus a `Provider` now just takes a `Key`. One argument instead of two.

**`PEMDecoder.withFactory` becomes `withFactoriesOf`.** Purely a naming fix — the method picks which `Provider` supplies key and certificate factories, and the new name says so without making you guess.

**A new `CryptoException` class.** An unchecked exception for cryptographic failures at runtime, for the cases where a checked `GeneralSecurityException` is more ceremony than the situation warrants.

None of this breaks code written against JEP 524 in any dramatic way — it's a rename here, a simplified signature there. It's the kind of third round that makes you think the team spent round two actually using the thing and writing down what annoyed them.

## The API, Briefly

If you haven't used any of this yet, here's the shape of it. Encoding a key pair:

```java
PEMEncoder encoder = PEMEncoder.of();
String pem = encoder.encodeToString(new KeyPair(publicKey, privateKey));
```

Encrypting a private key on the way out:

```java
String pem = encoder.withEncryption(password).encodeToString(privateKey);
```

Decoding, letting pattern matching sort out what came back:

```java
PEMDecoder decoder = PEMDecoder.of();
switch (decoder.decode(pem)) {
    case PublicKey publicKey -> handle(publicKey);
    case PrivateKey privateKey -> handle(privateKey);
    default -> throw new IllegalArgumentException("Unexpected PEM content");
}
```

Or skip the pattern match if you already know what you're expecting:

```java
ECPublicKey key = decoder.decode(pem, ECPublicKey.class);
```

That's a real API doing in three lines what used to take a dozen. Distinct, explicit paths for encryption and decryption, a `BinaryEncodable` sealed interface tying together everything from keys to certificates to CRLs, and a fallback `PEM` object for the odd case — a PKCS#10 certificate request, say — where the JDK doesn't already have a matching type.

It's still a preview API in JDK 27, so `--enable-preview` is required on both `javac` and `java` before any of this compiles or runs.

## Wrapping Up

Third preview in, and this one reads like convergence rather than churn — refinement, not redesign. To be clear about what "closed and delivered" actually means here: that's the JEP's own process status, confirming this round shipped in JDK 27 — it says nothing about the API itself graduating out of preview. The PEM API is still exactly that, a preview feature, same as it was in JEP 470 and JEP 524. `--enable-preview` isn't going away until a future JEP finalizes the design for real. If you're migrating from JEP 524, expect a rename here and a simpler method signature there, and not much else to relearn.

Three rounds of "almost final" is a lot of rounds. But unlike some other previews I could name in this series, this one's problem was never the design — it was just deciding when to stop tweaking it.
