---
title: "JEP 540: Simple JSON API"
date: "2026-09-24"
draft: true
tags: ["Java", "Java 28", "JEP"]
series: ["Java 28"]
series_order: 5
cover:
   image: "/images/jep540-simple-json-api.png"
   alt: "JEP 540: Simple JSON API"
description: "How to parse and generate JSON in plain Java, without pulling in Jackson, once JEP 540 lands."
---

<!-- TODO(Ron): swap in your own opening anecdote here. Something concrete from a
Belastingdienst integration where you reached for Jackson (or worse, manual string
parsing) just to pull one field out of a response. The more specific, the better,
a real endpoint name, a real "why is this pom.xml 40MB" moment. Draft below is a
placeholder so the structure reads, replace it, don't just tweak it. -->

I still remember the first time I had to grab a single value out of a JSON response and reached for Jackson to do it. Add the dependency, register the `ObjectMapper`, write a DTO for a payload I was going to look at exactly once, all to read one field. It felt like renting a moving truck to carry a suitcase.

That's been the story of JSON in Java for two decades now. Jackson, Gson, JSON-P, all excellent libraries, all doing far more than you need for the quick stuff. Meanwhile in Python you write `json.loads(response.text)["forecast"][0]["temp"]` and you're done. Java never had an answer for that. Until now, maybe.

## What JEP 540 actually proposes

JEP 540, currently sitting at Candidate status for JDK 28, is called *Simple JSON API (Incubator)*. It's an incubating module, so nothing here is locked in stone yet, but the shape of it is worth understanding early.

The idea is small on purpose. A `Json` class with a `parse` method that turns a JSON document into a tree of `JsonValue` instances. `JsonValue` is a sealed interface, and it has exactly six subtypes, matching the six things a JSON document can contain: `JsonObject`, `JsonArray`, `JsonString`, `JsonNumber`, `JsonBoolean`, and `JsonNull`.

Sealed means the compiler knows the full set of subtypes in advance. If you've read my piece on primitive patterns in the JDK 27 JEP series, this should feel familiar, sealed hierarchies are what make exhaustive switch expressions possible without a `default` branch you never wanted to write in the first place.

```java
switch (value) {
    case JsonObject o -> handleObject(o);
    case JsonArray a  -> handleArray(a);
    case JsonString s -> handleString(s);
    case JsonNumber n -> handleNumber(n);
    case JsonBoolean b -> handleBoolean(b);
    case JsonNull n   -> handleNull(n);
}
```

No default clause, no "this should never happen" comment above a `RuntimeException`. The compiler already knows there's nowhere else to go.

## Reading values, the honest way

Each subtype exposes conversions appropriate to what it actually is. `JsonObject` gives you `asMap()`, `JsonArray` gives you `asList()`, and the primitive types give you the usual `asInt()`, `asString()`, and friends.

Here's the part I like: if you call `asInt()` on something that turns out to be a `JsonString`, it doesn't try to be clever and parse the string into a number for you. It just throws a `JsonValueException`. No silent coercion, no "helpful" behavior that quietly hides a bug three services downstream. You get exactly the type you asked for, or you get told you were wrong to expect it.

<!-- TODO(Ron): this is a great spot for one of your "I've been burned by silent
coercion before" war stories. Even a single sentence about a Jackson or Gson
gotcha you've hit in production would sell this point way better than my
explaining why it's good in the abstract. -->

## A walkthrough, tax-office style

The JEP's own example pulls average forecast temperatures out of a National Weather Service response. Fine example, but let's make it something closer to home.

Say you're calling out to a downstream service that returns a tax bracket lookup, and all you want is the applicable rate for a given income band.

```java
String body = """
    {
      "brackets": [
        { "band": "0-38000", "rate": 0.3697 },
        { "band": "38000-76000", "rate": 0.3748 },
        { "band": "76000+", "rate": 0.4950 }
      ]
    }
    """;

JsonValue root = Json.parse(body);
JsonArray brackets = root.asObject().get("brackets").asArray();

double topRate = brackets.values().stream()
    .map(JsonValue::asObject)
    .mapToDouble(b -> b.get("rate").asNumber().asDouble())
    .max()
    .orElseThrow();

System.out.println(topRate);
```

No Jackson dependency, no DTO class, no `@JsonProperty` annotations for a payload you're reading once. Just the JDK doing the one thing you actually needed.

<!-- TODO(Ron): if you've got a real (anonymized) payload shape from a
Belastingdienst service, swap it in here instead. It'll read as something you
actually ran, not something I made up to sound plausible. -->

## What it deliberately leaves out

This is not a Jackson replacement, and it was never trying to be one. There's no data binding, so no mapping straight into your own Java objects. There's no JSON5 support, no comments, no trailing commas. There's no streaming parser for gigabyte-sized documents. If you need any of that, you're still reaching for Jackson, Gson, or JSON-P, and that's fine. JEP 540 exists for the other 80% of cases, the ones where you just need to look at a response and grab a couple of fields.

## The pushback

Not everyone's thrilled with the ergonomics. Over on Hacker News, one of the more common complaints is that building a `JsonArray` out of plain Java values takes more ceremony than people expected:

```java
JsonArray.of(List.of(JsonString.of("SUN"), JsonString.of("SunRsaSign")));
```

Every value has to be explicitly wrapped before it goes in. No automatic conversion from a `List<String>` to a `JsonArray` of `JsonString`. It's a fair criticism, and it's the kind of thing that tends to get sanded down during the incubation period, so don't assume the final API looks exactly like this.

<!-- TODO(Ron): your own take on this API ergonomics debate goes here. You've got
strong opinions from the Spring AI ChatClient work on when a bit of ceremony is
worth it versus when it's just friction. This is the spot to use them. -->

## Why not records and sealed interfaces all the way down

If you've followed my data-oriented programming writing, you'll have the same reaction I did, why isn't this just a sealed interface with record implementations? That's the textbook DOP shape for exactly this kind of problem.

Turns out the JDK team considered it and passed. Nicolai Parlog covered this well on Inside Java Newscast, records would expose their internals as part of the public API contract, and for something as foundational as a JDK-shipped JSON type, the team wants room to evolve the implementation later without breaking anyone. So instead you get interfaces with hidden implementations, `JsonValue` at the top, with accessor methods instead of public record components. Less pure from a DOP standpoint, more defensible for a class that's going to outlive several of your projects.

## Where things stand

JEP 540 is a Candidate JEP for JDK 28 as of late July 2026. Candidate is an earlier stage than Proposed to Target, so treat everything above as "this is the current shape," not "this is what ships." Incubator modules can and do change between now and general availability, sometimes significantly, based on exactly the kind of feedback showing up on Hacker News and the mailing lists right now.

<!-- TODO(Ron): your closer. Worth landing somewhere on "about time" versus "too
little too late" given how deeply Jackson is embedded in enterprise Java
codebases, including your own at Belastingdienst. That opinion is the whole
reason someone reads to the end of this one. -->
