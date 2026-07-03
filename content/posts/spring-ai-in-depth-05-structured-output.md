---
title: "Spring AI Series: 5-Structured Outputs"
date: 2026-07-03
draft: false
tags: ["Java", "Spring Boot", "AI", "Spring AI"]
cover:
  image: "/images/spring-ai-05-structured-outputs.png"
  alt: "Spring AI Series: Structured Outputs"
series: ["Spring AI in Depth"]
series_order: 5
description: "Turn free-text LLM responses into typed Java objects — enums, records, lists, and maps — and give BrightCart a ticket classifier you can actually route on."
categories: ["ai", "java"]
---

Let me show you the exact moment a Spring AI demo turns into a Spring AI *application*.

In the last two articles, BrightCart's summarizer happily returned prose, and prose was the right answer — a human support agent reads it. But the moment I tried to make the system *do* something with a model's answer, I hit a wall. I wanted to classify tickets so they could be routed automatically: delivery problems to the logistics team, billing issues to finance, and so on. So I asked the model to classify a ticket, and it cheerfully replied:

> "This ticket appears to be primarily about a delivery issue, though there's also a billing component since the customer mentions being charged."

Lovely. Now route on *that*. In code.

```java
if (response.contains("delivery")) {
    // but it also contains "billing"...
}
```

That's the cliff every LLM application walks up to. Free text is perfect for humans and useless for `switch` statements. What I needed wasn't a sentence about the category — I needed *the category*. A real Java enum I could branch on without parsing English.

That's what this article is about: getting the model to hand back typed Java objects instead of prose. By the end, BrightCart will have a classifier that returns a real `TicketAnalysis` record — and the strings, as promised at the end of article 4, finally end.

The full source is in the [GitHub repository](https://github.com/RonVeen/spring-ai-in-depth).

## The Problem With Parsing Prose

Before the fix, let's sit in the pain for a second, because it's worse than it first looks.

If you go the naive route — ask for free text and parse it — you're signing up to handle every way the model might phrase the same answer. "Delivery issue." "This is about delivery." "Shipping problem." "The package didn't arrive." All correct, all different strings. Your parsing logic becomes a growing pile of `contains()` checks and lowercase comparisons that breaks the first time the model rephrases something, which is to say constantly.

You could tighten the prompt — "respond with only one word" — and that helps, until the day the model adds a polite "Sure! " in front and your exact-match comparison silently fails. You're now debugging a routing bug caused by a chatbot's good manners.

The real fix isn't a better prompt. It's telling Spring AI what *type* you want back and letting it handle the rest.

## entity(): From String to Type

Here's the method that changes everything. Instead of `.content()` at the end of the chain, you call `.entity()` and hand it a Java type:

```java
SomeType result = chatClient
        .prompt()
        .user(input)
        .call()
        .entity(SomeType.class);
```

Spring AI does two things on your behalf. Before the call, it appends formatting instructions to your prompt describing exactly the structure it wants back — a JSON schema derived from your Java type. After the call, it parses the model's response and deserializes it into an instance of your type. You get a typed object; the string-wrangling happens out of sight.

That's the whole idea. Now let's make it concrete, starting as simply as possible.

## Starting Simple: An Enum

BrightCart needs to sort tickets into categories. The cleanest possible representation of "one of a fixed set of options" in Java is an enum:

```java
package org.veenx.springai.demo.model;

public enum TicketCategory {
    DELIVERY,
    BILLING,
    RETURN,
    PRODUCT_DEFECT,
    ACCOUNT,
    OTHER
}
```

Six categories, nothing exotic. Now — can Spring AI hand us back one of these directly? Almost. Enums work best when wrapped in a record rather than requested bare; it gives the converter a JSON object to target and avoids ambiguity about how a lone enum value should be serialized. So we wrap it:

```java
package org.veenx.springai.demo.model;

public record TicketClassification(TicketCategory category) {}
```

And the call:

```java
TicketClassification classification = chatClient
        .prompt()
        .system("""
                You are a support ticket classifier for BrightCart, an online retailer.
                Classify each ticket into exactly one category.
                """)
        .user(ticketText)
        .call()
        .entity(TicketClassification.class);

TicketCategory category = classification.category();
```

That `category` is a real `TicketCategory`. Not a string that says "DELIVERY" — the actual enum constant. You can `switch` on it, pass it to a router, store it in a typed column, all without a single `contains()` check:

```java
switch (category) {
    case DELIVERY -> routeToLogistics(ticket);
    case BILLING -> routeToFinance(ticket);
    case RETURN -> routeToReturns(ticket);
    default -> routeToGeneralQueue(ticket);
}
```

This is the moment the cliff disappears. The model still does the hard part — understanding a furious, rambling, all-caps ticket and deciding it's fundamentally a delivery problem — but it gives you the answer in a form your code can actually use.

## How entity() Actually Works

It's worth a one-minute peek behind the curtain, because "the model just returns Java objects" sounds like magic and magic is hard to debug.

There's no magic. When you call `.entity(TicketClassification.class)`, Spring AI uses a `BeanOutputConverter` under the hood. Before sending your prompt, the converter generates a JSON schema from your record and *appends it to the prompt* as formatting instructions — essentially adding "respond with JSON matching this exact shape" to whatever you wrote. The model, now told precisely what structure to produce, returns JSON. The converter then deserializes that JSON into your record using Jackson, the same library you've used a hundred times.

So `.entity()` is really "inject format instructions, then parse the result." Knowing that demystifies the failure modes too: if the model returns something that isn't valid JSON, or JSON that doesn't match your type, the parsing step is where it'll break. More on that shortly.

One practical consequence: your record's field names matter. They become the JSON keys the model is asked to produce, so name them clearly. `category` is a better instruction to the model than `c`.

## Building Up: A Richer Record

A bare category is useful, but BrightCart's support leads want more from a single classification pass. While the model is already reading the ticket, why not have it extract everything relevant in one shot? Priority, the order number if one's mentioned, and a one-line reason for the classification:

```java
package org.veenx.springai.demo.model;

public record TicketAnalysis(
        TicketCategory category,
        Priority priority,
        String orderNumber,
        String reason
) {}
```

With a second enum for priority:

```java
package org.veenx.springai.demo.model;

public enum Priority {
    LOW,
    MEDIUM,
    HIGH,
    URGENT
}
```

The call barely changes — we just describe the richer shape in the prompt and ask for the bigger type:

```java
TicketAnalysis analysis = chatClient
        .prompt()
        .system("""
                You are a support ticket analyst for BrightCart, an online retailer.
                Analyze each ticket and extract:
                - category: the single best-fitting category
                - priority: how urgently this needs attention
                - orderNumber: the order number if one is mentioned, otherwise null
                - reason: a one-sentence justification for the category and priority
                Be factual. Base priority on customer impact and urgency cues.
                """)
        .user(ticketText)
        .call()
        .entity(TicketAnalysis.class);
```

One model call, four extracted fields, all typed. The `orderNumber` comes back as a `String` (or `null` when absent — the model honors that instruction surprisingly well), `category` and `priority` as real enums, `reason` as the human-readable explanation your audit log will thank you for.

Notice what we *didn't* do: make four separate calls for four pieces of information. The model read the ticket once and gave us everything. That's not just convenient — at BrightCart's volume, it's four times fewer API calls and four times less token spend than the naive approach. Article 13 will care about that a lot.

## Collections: When One Answer Isn't Enough

Sometimes the structured thing you want back is a *list*. A single ticket might mention several distinct issues, or reference multiple order numbers, or contain a handful of action items for the support agent. Spring AI handles collections, with one small syntactic wrinkle.

For a list of a custom type, you can't write `.entity(List<ActionItem>.class)` — Java erases the generic, so `List.class` wouldn't tell Spring AI what's *in* the list. The fix is `ParameterizedTypeReference`, Spring's standard tool for capturing generic type information:

```java
package org.veenx.springai.demo.model;

public record ActionItem(String description, Priority priority) {}
```

```java
import org.springframework.core.ParameterizedTypeReference;

List<ActionItem> actionItems = chatClient
        .prompt()
        .system("""
                You are a support assistant for BrightCart.
                Extract a list of concrete action items a support agent should take
                to resolve this ticket. Each item has a description and a priority.
                """)
        .user(ticketText)
        .call()
        .entity(new ParameterizedTypeReference<List<ActionItem>>() {});
```

The `new ParameterizedTypeReference<List<ActionItem>>() {}` looks a little odd if you haven't met it before — note the trailing `{}`, which creates an anonymous subclass so the generic type survives erasure. It's a standard Spring idiom, not a Spring AI invention; you may have seen it in `RestClient` and `RestTemplate` calls. Spring AI reuses it here so the converter knows it's building a `List<ActionItem>`, not a list of something unknown.

For a plain list of strings, there's an even simpler converter:

```java
import org.springframework.ai.converter.ListOutputConverter;
import org.springframework.core.convert.support.DefaultConversionService;

List<String> tags = chatClient
        .prompt()
        .user(u -> u
                .text("Suggest 3-5 short tags for this support ticket:\n\n{ticket}")
                .param("ticket", ticketText))
        .call()
        .entity(new ListOutputConverter(new DefaultConversionService()));
```

### Maps for Dynamic Shapes

Records are the right choice when you know the fields ahead of time, which is almost always. But occasionally you want a flexible bag of key-value pairs whose keys you don't know in advance — say, a set of extracted attributes that varies by product type. For that, ask for a `Map`:

```java
import org.springframework.core.ParameterizedTypeReference;

Map<String, Object> extracted = chatClient
        .prompt()
        .user(u -> u
                .text("Extract any product attributes mentioned in this ticket as key-value pairs:\n\n{ticket}")
                .param("ticket", ticketText))
        .call()
        .entity(new ParameterizedTypeReference<Map<String, Object>>() {});
```

A word of advice from experience: reach for `Map` rarely. A `Map<String, Object>` throws away most of the type safety we came here for — you're back to casting values and hoping. When you know the shape, use a record. `Map` is the escape hatch for the genuinely dynamic case, not the default.

## A Note on When It Goes Wrong

I've shown you the happy path, and most of the time the happy path is what you'll get. But honesty compels a caveat: the model is *not guaranteed* to return parseable output. It might wrap its JSON in commentary, hit the max-tokens ceiling mid-object, or occasionally just produce something malformed. When that happens, the parsing step throws.

For BrightCart's classifier we're going to accept that risk for now and deal with it properly later — a thrown exception on a bad parse is at least loud and visible, not a silent wrong answer. But "wrap it in a retry, validate the result, and fail gracefully" is a real production concern, and it has a real home: article 12, where we cover testing, evaluation, and resilience. For now, just know the sharp edge exists and we haven't forgotten it.

## BrightCart Gets a Classifier

Time to ship the feature. We'll add a dedicated `/classify` endpoint alongside the existing `/summarize` one, following the same clean structure: a configured client, a service, a controller.

First, the externalized system prompt — applying the lesson from article 4, the classifier's instructions live in a file, not a Java string:

`src/main/resources/prompts/ticket-classify-system.st`:
```
You are a support ticket analyst for BrightCart, an online retailer.
Analyze each ticket and extract:
- category: the single best-fitting category for the ticket
- priority: how urgently this needs attention, based on customer impact and urgency cues
- orderNumber: the order number if one is mentioned, otherwise null
- reason: a one-sentence justification for the chosen category and priority

Be factual. Do not invent order numbers. If the ticket is ambiguous, choose
the closest category and reflect the uncertainty in the reason.
```

The model classes:

```java
package org.veenx.springai.demo.model;

public enum TicketCategory {
    DELIVERY,
    BILLING,
    RETURN,
    PRODUCT_DEFECT,
    ACCOUNT,
    OTHER
}
```

```java
package org.veenx.springai.demo.model;

public enum Priority {
    LOW,
    MEDIUM,
    HIGH,
    URGENT
}
```

```java
package org.veenx.springai.demo.model;

public record TicketAnalysis(
        TicketCategory category,
        Priority priority,
        String orderNumber,
        String reason
) {}
```

The configuration — a second `ChatClient` bean dedicated to classification, with a low temperature because classification wants consistency, not creativity:

```java
package org.veenx.springai.demo.config;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.prompt.ChatOptions;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.Resource;

import java.io.IOException;
import java.nio.charset.StandardCharsets;

@Configuration
public class ClassifierConfig {

    @Value("classpath:prompts/ticket-classify-system.st")
    private Resource classifySystemPrompt;

    @Bean
    public ChatClient ticketClassifierClient(ChatClient.Builder builder) throws IOException {
        return builder
                .defaultSystemPrompt(classifySystemPrompt.getContentAsString(StandardCharsets.UTF_8))
                .defaultOptions(ChatOptions.builder()
                        .temperature(0.0)
                        .maxTokens(512)
                        .build())
                .build();
    }
}
```

> We set `temperature` to `0.0` here. Classification is a task where you want the *same* ticket to produce the *same* category every time — determinism over creativity. This is the low end of the range we discussed in article 3, and it's exactly where classification belongs.

Since BrightCart now has two `ChatClient` beans — the summarizer from earlier articles and this classifier — remember the note from article 2: with multiple beans you inject by name. Spring matches the bean name `ticketClassifierClient` to the constructor parameter, so name your parameters to match.

The service:

```java
package org.veenx.springai.demo.service;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.stereotype.Service;
import org.veenx.springai.demo.model.TicketAnalysis;

@Service
public class TicketClassifierService {

    private static final Logger log = LoggerFactory.getLogger(TicketClassifierService.class);

    private final ChatClient ticketClassifierClient;

    public TicketClassifierService(ChatClient ticketClassifierClient) {
        this.ticketClassifierClient = ticketClassifierClient;
    }

    public TicketAnalysis classify(String ticketText) {
        TicketAnalysis analysis = ticketClassifierClient
                .prompt()
                .user(ticketText)
                .call()
                .entity(TicketAnalysis.class);

        log.info("Ticket classified as {} ({}), order={}",
                analysis.category(), analysis.priority(), analysis.orderNumber());

        return analysis;
    }
}
```

And the controller:

```java
package org.veenx.springai.demo.web;

import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.veenx.springai.demo.model.TicketAnalysis;
import org.veenx.springai.demo.service.TicketClassifierService;

@RestController
@RequestMapping("/api/tickets")
public class ClassifierController {

    private final TicketClassifierService classifierService;

    public ClassifierController(TicketClassifierService classifierService) {
        this.classifierService = classifierService;
    }

    @PostMapping("/classify")
    public TicketAnalysis classify(@RequestBody String ticketText) {
        return classifierService.classify(ticketText);
    }
}
```

Because the endpoint returns a `TicketAnalysis` record, Spring serializes it straight to JSON for the caller. Let's throw our chaotic espresso machine ticket at it:

```bash
curl -X POST http://localhost:8080/api/tickets/classify \
     -H "Content-Type: text/plain" \
     -d "Hi so I ordered the deluxe espresso machine like TWO WEEKS ago (order BC-48291) and the tracking said delivered Tuesday but there is NOTHING. I paid 379 euros for this!! I want a refund or a new machine ASAP."
```

And the response:

```json
{
  "category": "DELIVERY",
  "priority": "HIGH",
  "orderNumber": "BC-48291",
  "reason": "Customer reports a paid order marked delivered but not received, and is requesting a refund or replacement urgently."
}
```

Now *that* you can route on. `category` drives the queue, `priority` drives the SLA, `orderNumber` is already extracted for the next step, and `reason` gives the agent context. Four useful, typed fields from one model call and one chaotic customer.

## What's Next

BrightCart can now read a ticket and tell you, in clean typed Java, what it's about and how urgent it is. The strings are gone — replaced by enums, records, and lists your code can actually work with.

But notice what the classifier *can't* do. It extracted order number `BC-48291`, but it has no idea whether that order exists, what's in it, or where it actually is. It's working purely from the customer's words. To resolve "where's my order," BrightCart needs to reach out of the model and into its own systems — the order database, the shipment tracker, the customer record.

That's tool calling, and it's where things get genuinely interesting. In article 6, we'll give the model hands: the ability to call BrightCart's own Java methods to look up real data mid-conversation. The classifier guesses from text; the next version *checks*.

*This is part 5 of a 13-part series.*

[← Previous: Spring AI Series: 4-Prompt Engineering in Java]
[Next: Spring AI Series: 6-Tool Calling in Spring AI →]