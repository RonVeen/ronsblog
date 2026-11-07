---
title: "Jev and System One Models: The Decision Layer Your Java Code Has Been Missing"
date: 2026-11-07
draft: false
tags: ["Java", "AI", "Jev", "Decision models", "SystemOne"]
cover:
  image: "/images/introducing-jev.png"
  alt: "Jev and System One Models: typed decisions for Java"
description: "Call TypeSafe's Jev model from plain Java to classify, score, and route support tickets with calibrated probabilities, then branch on confidence like a smart if-statement."
categories: ["ai", "java"]
---


Here's a thought that took me embarrassingly long to have. Most of what we ask AI to do in a backend isn't conversation at all. It's decisions. Is this ticket urgent? Which queue does it belong in? Is this input abusive? Should this action need a human to sign off? None of those want a paragraph of prose back. They want a typed answer your code can branch on.

And yet we've been reaching for chat models to do it, because chat models were the only game in town. You prompt Claude or GPT with "classify this ticket and rate its urgency from 1 to 5," parse the response, hope it didn't wrap the number in a friendly sentence, and quietly accept that the "4" it gave you has no reliable meaning. It might be a 4 because the ticket is genuinely urgent, or because the model was feeling generous. You can't tell. There's no calibration behind it.

A new class of model is built specifically for this, and it's worth your attention. It's called a System One model. And one of the first public ones is called Jev, and it genereated a lot of interest recently. It does something no LLM does: it refuses to generate text at all. You send it a state and some typed questions, and it hands back typed decisions with calibrated probabilities. That's it. No prose, no parsing, no hoping.

This is a standalone piece, but it grew out of my [Spring AI in Depth](/series/spring-ai-in-depth/) series, where I'm building a support-ticket assistant called BrightCart with Claude and Spring AI. In [part 5](/posts/spring-ai-in-depth-05-structured-outputs/) I built a ticket classifier the LLM way. This article is what happens when you hand that same job to a model actually designed for it.

## What Is a System One Model?

The name comes from Daniel Kahneman's *Thinking, Fast and Slow*. System 1 is fast, intuitive, snap-judgment thinking. System 2 is slow, deliberate reasoning. LLMs, with their chain-of-thought and their reasoning traces, are System 2 machines. Jev is deliberately the other thing: fast, structured, decisive.

The practical difference comes down to one word: *output*.

An LLM outputs strings. Strings are infinitely flexible, which is exactly their problem for automation. A string can be the answer you wanted, or a polite preamble in front of the answer, or a hallucination, or a refusal, or valid JSON with one field subtly wrong. Your code has to parse it and validate it and handle the day it comes back malformed. There's always a non-zero chance the model goes off the rails.

A System One model outputs typed decisions. You define the shape of the answer in advance, and the model fills it in. It literally cannot return a type error, because the structure is fixed before the call is made. There's nothing to parse and nothing to validate, because the response is already the typed value you asked for.

Jev gives you three kinds of question, and you can ask many of them in a single call:

A **Choice** picks one option from a fixed list. "Which category is this ticket?" with options `billing`, `delivery`, `return`. This is classification.

A **Score** returns a number on a rubric, typically zero to one. "How urgent is this ticket?" This is rating and prioritisation.

A **Noul** returns a yes/no with a probability. "Is this customer angry?" This is a boolean judgment. (The name is TypeSafe's own; think of it as a calibrated boolean.)

And here's the part that actually matters, the thing that separates this from asking an LLM for structured output: every answer comes with a calibrated probability. Calibrated means the confidence number is honest. When Jev says it's 90% sure, it's right about 90% of the time. An LLM, even prompted for a confidence score, tends to be overconfident and inconsistent; if a model can do a task 95% of the time but can't tell you *which* 5% it's wrong on, you can't safely automate it. Calibration is what makes the automation safe, and it's the whole pitch.

TypeSafe makes some bold performance claims alongside this: on their own workflow benchmarks, Jev runs end-to-end in 70 to 500 milliseconds against 3 to 329 seconds for frontier LLMs, at a fraction of the cost, with input priced at $0.042 per million tokens and output free. I'd treat the headline "193x faster, 444x cheaper" numbers with the usual caution you apply to any vendor's own benchmark, they built the evals, and they say as much. But the architecture genuinely is different, and the *shape* of the advantage, parallel typed decisions instead of sequential token generation, is real whether or not the exact multiplier holds for your workload.

The one claim that isn't a benchmark, and that I find genuinely compelling, is zero hallucination. Not "fewer hallucinations." Zero, by construction. Since the output structure is fixed and the model selects within it rather than generating free text, there's no room for it to invent a field or a value that doesn't exist. That's not marketing, it's a property of the design.

But SystemOne is not the only provider of these kind of models. Cloudflare is offering [Clef](https://huggingface.co/Cloudflare/clef), [Clef Omni](https://huggingface.co/Cloudflare/clef-omni),  and [Clef-flash](https://huggingface.co/Cloudflare/clef-flash). These open weights models have an API that is fully compatible with Jev and SystemOne.

## Meet BrightCart

A quick introduction for anyone arriving cold, since this grew out of the series. BrightCart is a fictional online retailer that sells, as far as anyone can tell, everything. And like every company that sells everything, it drowns in support tickets: where's my order, this arrived damaged, I want a refund, why was I charged twice.

The job we care about today is triage. Every incoming ticket needs three things decided before a human ever touches it. What's it about (category). How urgent is it (priority). And is the customer angry (so we can route the hot ones to someone senior). That's a Choice, a Score, and a Noul. It's as if BrightCart's problem was designed to show off exactly what Jev does.

In the series, I solve this with Claude through Spring AI. Here, we solve it with Jev, and then I'll be honest at the end about which one you actually want for which part of the job.

## Try It Before You Write Any Code

Before touching Java, go play. TypeSafe has a browser playground that needs no key and no signup, you just type a state and some questions and watch the typed answers come back. It's the fastest way to build intuition for what Choice, Score, and Noul actually return, and I'd genuinely do this first. Paste in a furious all-caps ticket and watch the `is_angry` Noul light up. It clicks in about thirty seconds.

Playground's at [typesafe.ai](https://typesafe.ai/). Go, poke it, come back.

## Getting a Key

When you're ready to call it from code, you need an API key. The good news since launch: the waitlist is gone. You sign in at `console.typesafe.ai` with Google or an email code, open the API Keys section, and create one. Store it in an environment variable called `TYPESAFE_API_KEY`, the same way you'd handle any API credential.

On cost, let me be straight, because "free" gets thrown around loosely with this. The *playground* is free. The *API* is not free, but it's close enough that it doesn't matter for learning: input tokens are $0.042 per million, and output tokens are free. A support ticket is a few hundred tokens. You could run every example in this article hundreds of times over and not clear a single cent. There's no self-serve free tier with a monthly allowance, but at these prices you won't notice.

Quick sanity check that your key works, straight from the shell:

```bash
curl https://api.typesafe.ai/v1/models \
  -H "Authorization: Bearer $TYPESAFE_API_KEY"
```

That hits `GET /v1/models` and lists what your account can call. If it returns JSON instead of a 401, you're in.

## Calling Jev From Java

There's no official Java SDK, only Python and JavaScript, with the raw HTTP API as the documented path for everyone else. That's fine, and for a standalone program you can actually run, it keeps things simple: a single POST against `POST /v1/systemone`, handled by Java's built-in `HttpClient`. No framework, no Spring context, just a small Maven project with Jackson for the JSON. Here's the complete `pom.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.veenx</groupId>
    <artifactId>jev-triage</artifactId>
    <version>0.0.1-SNAPSHOT</version>

    <properties>
        <maven.compiler.release>25</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.18.2</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>exec-maven-plugin</artifactId>
                <version>3.6.2</version>
                <configuration>
                    <mainClass>org.veenx.jev.TicketTriage</mainClass>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

One dependency, Jackson, with `HttpClient` coming from the JDK. The `exec-maven-plugin` is build tooling, not a runtime dependency; it's there so `mvn compile exec:java` knows which class to launch without you passing it on the command line.

First, the request shape. Jev wants a `state` (the thing being judged, our ticket text) and a `questions` object. Each question is keyed by a name you choose, and its value describes the question: a `type`, an `instructions` line, and, for choice and score questions, a `criteria` block that spells out what each option or level means. Here's the JSON for BrightCart's triage, all three questions in one call:

```json
{
  "model": "jev-latest",
  "state": "Hi so I ordered the deluxe espresso machine TWO WEEKS ago (order BC-48291) and tracking said delivered but there is NOTHING. I paid 379 euros!! I want a refund or replacement ASAP.",
  "questions": {
    "category": {
      "type": "choice",
      "instructions": "Which category best fits this support ticket",
      "criteria": {
        "delivery": "Shipping, tracking, or non-delivery problems",
        "billing": "Charges, refunds, or payment issues",
        "return": "Returning or exchanging an item",
        "product_defect": "A faulty or damaged product",
        "account": "Login, profile, or account access",
        "other": "Anything that fits none of the above"
      }
    },
    "urgency": {
      "type": "score",
      "instructions": "How urgently this ticket needs attention",
      "criteria": [
        "Routine, no time pressure",
        "Should be handled soon",
        "Drop everything, highly time-sensitive"
      ]
    },
    "is_angry": {
      "type": "noul",
      "instructions": "The customer is expressing anger or frustration"
    }
  }
}
```

A few things to notice about that shape, because they're where a first attempt tends to go wrong. The `questions` value is an object keyed by name, not an array; send an array and Jev rejects it with a 422. A choice's `criteria` is a map from each option to a short description, so the model knows what "return" versus "product_defect" actually means. A score's `criteria` is an ordered list of rubric levels, lowest first, and the answer comes back as the *index* of the level that fits, not a 0-to-1 fraction. A noul just needs `instructions`.

One state, three questions, one round trip. Jev evaluates all three against the ticket in parallel, which is the whole reason it's fast.

Now the Java. We build the request with Jackson so we're not hand-escaping anything, send it with `HttpClient`, and read the response back as a tree. Here's a small, self-contained client:

```java
package org.veenx.jev;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;

public class JevClient {

    private static final String ENDPOINT = "https://api.typesafe.ai/v1/systemone";

    private final HttpClient http = HttpClient.newHttpClient();
    private final ObjectMapper mapper = new ObjectMapper();
    private final String apiKey;

    public JevClient(String apiKey) {
        this.apiKey = apiKey;
    }

    public JsonNode evaluate(JsonNode requestBody) throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(ENDPOINT))
                .header("Authorization", "Bearer " + apiKey)
                .header("Content-Type", "application/json")
                .POST(HttpRequest.BodyPublishers.ofString(mapper.writeValueAsString(requestBody)))
                .build();

        HttpResponse<String> response = http.send(request, HttpResponse.BodyHandlers.ofString());

        if (response.statusCode() != 200) {
            throw new IllegalStateException("Jev returned " + response.statusCode() + ": " + response.body());
        }

        return mapper.readTree(response.body());
    }
}
```

Nothing exotic. A POST with a Bearer header, and Jackson handling the JSON in both directions so there's no string-escaping to get wrong. If you've called any REST API from Java in the last five years, this is muscle memory.

Now the entry point, a `TicketTriage` class whose `main` builds the request and fires it. Small helpers construct each question type, keyed into the `questions` object:

```java
package org.veenx.jev;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ArrayNode;
import com.fasterxml.jackson.databind.node.ObjectNode;

import java.util.List;
import java.util.Map;

public class TicketTriage {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    public static void main(String[] args) throws Exception {
        String ticket = """
                Hi so I ordered the deluxe espresso machine TWO WEEKS ago (order BC-48291) \
                and tracking said delivered but there is NOTHING. I paid 379 euros!! \
                I want a refund or replacement ASAP.""";

        ObjectNode body = MAPPER.createObjectNode();
        body.put("model", "jev-latest");
        body.put("state", ticket);

        ObjectNode questions = body.putObject("questions");

        questions.set("category", choice(
                "Which category best fits this support ticket",
                Map.of(
                        "delivery", "Shipping, tracking, or non-delivery problems",
                        "billing", "Charges, refunds, or payment issues",
                        "return", "Returning or exchanging an item",
                        "product_defect", "A faulty or damaged product",
                        "account", "Login, profile, or account access",
                        "other", "Anything that fits none of the above")));

        questions.set("urgency", score(
                "How urgently this ticket needs attention",
                List.of(
                        "Routine, no time pressure",
                        "Should be handled soon",
                        "Drop everything, highly time-sensitive")));

        questions.set("is_angry", noul(
                "The customer is expressing anger or frustration"));

        JevClient jev = new JevClient(System.getenv("TYPESAFE_API_KEY"));
        var result = jev.evaluate(body);

        var ticketDecision = TicketDecision.from(result);
        println(ticketDecision);

        var router = new TicketRouter();
        var routing = router.route(ticketDecision);
        println(routing);
    }

    private static ObjectNode choice(String instructions, Map<String, String> criteria) {
        ObjectNode q = MAPPER.createObjectNode();
        q.put("type", "choice");
        q.put("instructions", instructions);
        ObjectNode c = q.putObject("criteria");
        criteria.forEach(c::put);
        return q;
    }

    private static ObjectNode score(String instructions, List<String> levels) {
        ObjectNode q = MAPPER.createObjectNode();
        q.put("type", "score");
        q.put("instructions", instructions);
        ArrayNode c = q.putArray("criteria");
        levels.forEach(c::add);
        return q;
    }

    private static ObjectNode noul(String instructions) {
        ObjectNode q = MAPPER.createObjectNode();
        q.put("type", "noul");
        q.put("instructions", instructions);
        return q;
    }
}
```

Building the request with Jackson's node API instead of a formatted string means the ticket text, quotes and newlines and all, gets escaped correctly for free. No hand-rolled escaper, nothing to ship that you'd be embarrassed by. Run it the usual way, `mvn compile exec:java` or your IDE's run button.

## Reading the Decision

Run it, and Jev answers all three questions at once:

```json
{
  "model": "jev-1.13.0",
  "answers": {
    "category": {
      "type": "choice",
      "choice": "delivery",
      "confidence": 0.91,
      "probabilities": {
        "delivery": 0.94,
        "return": 0.04,
        "billing": 0.01,
        "product_defect": 0.0,
        "account": 0.0,
        "other": 0.0
      }
    },
    "urgency": {
      "type": "score",
      "score": 2.0,
      "confidence": 0.86,
      "legend": {
        "0": "Routine, no time pressure",
        "1": "Should be handled soon",
        "2": "Drop everything, highly time-sensitive"
      },
      "probabilities": { "0": 0.02, "1": 0.12, "2": 0.86 }
    },
    "is_angry": {
      "type": "noul",
      "noul": 0.88
    }
  },
  "usage": { "input_tokens": 71, "output_tokens": 58 }
}
```

Look at what you got back, and notice each answer carries its own `confidence` separate from the value. The category is `delivery`, and `confidence` of 0.91 says Jev is genuinely sure; the probabilities confirm it, with `return` a distant second at 0.04. Urgency came back as `score` 2.0, which is the *index* into our criteria list, so level 2, "drop everything." And `is_angry` is 0.88, which, given the all-caps and the double exclamation marks, tracks.

That `confidence` field is the one to pay attention to, because it's the thing this whole article is about. It's the calibrated number you threshold on. And `usage` at the bottom shows the cost: a few dozen input tokens, which at $0.042 per million rounds to nothing.

Every value here is something your code acts on directly. No parsing a sentence, no regex. The shape was fixed before the call, so the response arrived as the typed thing you wanted.

Let me pull the raw response into a small typed record so the routing code that follows stays clean. Note that for the score we keep the integer level, not a fraction:

```java
package org.veenx.jev;

import com.fasterxml.jackson.databind.JsonNode;

import java.util.HashMap;
import java.util.Map;

public record TicketDecision(
        String category,
        double categoryConfidence,
        int urgencyLevel,
        double angerProbability
) {

    public static TicketDecision from(JsonNode response) {
        JsonNode answers = response.get("answers");

        JsonNode category = answers.get("category");
        String chosen = category.get("choice").asText();
        double categoryConfidence = category.get("confidence").asDouble();

        int urgencyLevel = answers.get("urgency").get("score").asInt();
        double anger = answers.get("is_angry").get("noul").asDouble();

        return new TicketDecision(chosen, categoryConfidence, urgencyLevel, anger);
    }
}
```

Now we have a typed `TicketDecision`. Here's where it gets genuinely useful.

## The Smart If-Statement

This is the part that made me sit up, and it's TypeSafe's own framing: a System One call is a *smart if-statement*. Because every decision carries honest confidence, you can write ordinary branching logic that knows when to trust itself and when to call for help.

Think about what that enables for BrightCart. A ticket that's clearly a delivery problem, clearly at the top urgency level, and clearly from an angry customer can be auto-escalated to a senior agent's queue with no human triage at all. But a ticket where the category confidence is weak should go to a human to sort out, precisely because the model is telling you it isn't sure. You act autonomously when confidence is high, and you escalate when it isn't.

That's not a vague aspiration. It's an `if` statement, and the thresholds are yours to set:

```java
package org.veenx.jev;

public class TicketRouter {

    // Tune these to your tolerance for mistakes.
    private static final double CATEGORY_CONFIDENCE = 0.85;
    private static final int URGENCY_ESCALATE = 2;      // top level of our 0,1,2 rubric
    private static final double ANGER_ESCALATE = 0.70;

    public Routing route(TicketDecision d) {

        // Not confident about the category? A human decides.
        if (d.categoryConfidence() < CATEGORY_CONFIDENCE) {
            return Routing.humanReview("Low category confidence: " + d.categoryConfidence());
        }

        // Confident, top-urgency, and the customer is angry: fast-track to a senior agent.
        if (d.urgencyLevel() >= URGENCY_ESCALATE && d.angerProbability() >= ANGER_ESCALATE) {
            return Routing.escalate(d.category());
        }

        // Confident and routine: straight into the normal queue for its category.
        return Routing.queue(d.category());
    }
}
```

With a small supporting type:

```java
package org.veenx.jev;

public record Routing(String destination, String reason) {

    public static Routing escalate(String category) {
        return new Routing("senior-queue:" + category, "urgent and angry");
    }

    public static Routing queue(String category) {
        return new Routing("queue:" + category, "routine");
    }

    public static Routing humanReview(String reason) {
        return new Routing("human-triage", reason);
    }
}
```

Our espresso-machine ticket, delivery at 0.91 confidence, urgency level 2, anger 0.88, sails straight into `senior-queue:delivery` with the reason "urgent and angry." No human had to read it to get it there. But a mumbled, ambiguous ticket where the category confidence only reaches 0.6 lands in `human-triage` instead, because the model was honest enough to admit it wasn't sure, and your code was smart enough to listen.

That last sentence is the entire value proposition. An LLM would have given you a category with the same breezy confidence whether it was 95% sure or 55% sure. Jev tells you the difference, and that difference is what lets you automate the easy 80% safely while routing the hard 20% to a person.

## Where This Fits, and Where It Doesn't

I'll be honest about the boundaries, because the hype around any new model skips this part and it's the most useful thing I can tell you.

Jev is not a replacement for your LLM. It's a replacement for the *decisions* you were awkwardly forcing your LLM to make. It will classify the ticket, score its urgency, and judge the customer's mood, fast, cheap, and with confidence you can trust. What it will not do is write the empathetic reply back to the customer, because that's string generation, and Jev doesn't do strings. That's Claude's job, and Claude is excellent at it.

So the real architecture isn't Jev *or* an LLM. It's both, each doing what it's actually good at. Jev decides; the LLM writes. In BrightCart terms: Jev triages the incoming ticket and routes it, and when it's time to draft a response, the LLM takes over with full context. The decision layer and the language layer are different tools, and the mistake we've all been making is using the language layer for both.

If that split resonates, the language half is exactly what my [Spring AI in Depth](/series/spring-ai-in-depth/) series builds out: a full support assistant on Spring AI and Claude, with tool calling, memory, RAG, the works. [Part 5](/posts/spring-ai-in-depth-05-structured-outputs/) is where I build the LLM version of the very classifier we just handed to Jev, worth a look if you want to see the two approaches back to back and judge for yourself.

My honest take after playing with it? For the decision layer, this is the right shape of tool, and going back to coaxing a chat model into calibrated-sounding numbers feels wrong once you've seen the alternative. Whether Jev specifically wins long-term, closed model, early days, pricing the company admits it can't yet prove is sustainable, I genuinely don't know. But System One as a category is pointing at something real. Try the playground. You'll see it in about a minute.

*Found this useful? It's a spin-off from my [Spring AI in Depth](/series/spring-ai-in-depth/) series, where I build the whole BrightCart assistant with Spring AI and Claude, start to finish.*