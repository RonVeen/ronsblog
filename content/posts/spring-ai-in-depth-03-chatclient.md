---
title: "Spring AI Series: 3-ChatClient"
date: 2026-06-19
draft: true
tags": ["Java", "Spring Boot", "AI", "Spring AI"]
cover:
  image: "/images/spring-ai-03-chatclient.png"
  alt: "Spring AI Series: Spring AI ChatClient"
series: ["Spring AI in Depth"]
series_order: 3
description: "Learn the ChatClient fluent API inside out — prompts, defaults, model options, response metadata, and streaming — while building the first endpoint of a real support ticket system."
categories: ["ai", "java"]
---

Somewhere in articles 1 and 2, I kept writing this line and asking you to take it on faith:

```java
chatClient.prompt().user(question).call().content();
```

It works. It reads nicely. And if you're anything like me, the fact that it works *without you understanding why* has been quietly bothering you since article 1.

Today we fix that. This article takes `ChatClient` apart — every method in that chain, every configuration option that matters, and the parts of the API we haven't even touched yet. By the end, the fluent API won't be a magic incantation anymore. It'll just be… an API. A well-designed one, but still.

And we're doing it while laying the first brick of something bigger.

## What We're Building

Here's the thing about tutorial code: a `CommandLineRunner` that asks the model about famous pirates teaches you the syntax, but it teaches you nothing about where Spring AI fits in a *real* application. So from this article onward, every feature we cover gets a home in one evolving system.

Meet **BrightCart** — a fictional online retailer that sells, as far as anyone can tell, everything. And like every company that sells everything, BrightCart drowns in support tickets. Where's my order. The package arrived damaged. I want to return this. Why was I charged twice. The classics.

We're building BrightCart's **Support Ticket Intelligence System**: a Spring Boot backend that takes incoming tickets and makes them manageable. Over the course of this series it will learn to:

- Summarize and classify tickets (this article and article 5)
- Look up orders and customers to enrich tickets with context (article 6, tool calling)
- Keep track of ongoing ticket conversations (article 8, chat memory)
- Answer questions grounded in BrightCart's return policies and product manuals (articles 9–10, RAG)
- Process the inevitable photo of a crushed delivery box (article 11, multimodality)
- Do all of this without bankrupting BrightCart on token costs (article 13)

Notice what's *not* on that list: a chatbot. The AI in this system works behind the scenes — it processes, enriches, and assists. The support agents stay in charge. That's a deliberate design choice, and frankly, a more realistic one than bolting a chat window onto everything.

Today's contribution is modest but foundational: an endpoint that accepts a raw customer ticket and returns a clean summary for the support agent. To build it properly, we need to actually understand `ChatClient`.

The full source code lives in the [GitHub repository](https://github.com/RonVeen/spring-ai-in-depth/tree/03-chat-client).

## Creating a ChatClient

You've seen the autoconfigured route in article 2: Spring AI detects a `ChatModel` on the classpath and registers a `ChatClient.Builder` bean for you.

```java
package org.veenx.springai.demo.config;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AiConfig {

    @Bean
    public ChatClient chatClient(ChatClient.Builder builder) {
        return builder.build();
    }
}
```

But there are actually three ways to get a `ChatClient`, and knowing all three tells you a lot about how the pieces fit:

**1. From the autoconfigured builder** — what we just did. Simplest, and right for single-provider applications.

**2. Directly from a model** — `ChatClient.create(chatModel)`. One line, no builder, no defaults. Useful in tests and quick experiments.

**3. From a manually created builder** — `ChatClient.builder(chatModel)`. Full control, and the way to go when you're running multiple providers like we did at the end of article 2.

```java
ChatClient quickClient = ChatClient.create(chatModel);

ChatClient configuredClient = ChatClient.builder(chatModel)
        .defaultSystemPrompt("You are a support ticket assistant.")
        .build();
```

One Spring AI 2.0 detail worth knowing: if you create your `ChatClient` beans manually from typed models, disable the autoconfigured builder by setting `spring.ai.chat.client.enabled=false`. Otherwise Spring's autoconfiguration and your manual beans may step on each other's toes. For the single-provider setup we use in this article, the autoconfigured builder is exactly what we want, so leave it on.

## The Fluent API, Taken Apart

Let's dissect the chain. Here it is again, annotated:

```java
String response = chatClient
        .prompt()       // 1. start building a prompt
        .user(text)     // 2. set the user message
        .call()         // 3. send it to the model (blocking)
        .content();     // 4. extract the response text
```

**`prompt()`** opens the fluent chain. It comes in three flavors: `prompt()` with no arguments for full control, `prompt(String)` as a shortcut that sets the user message directly, and `prompt(Prompt)` for passing a pre-built `Prompt` object — useful when prompts are constructed elsewhere in your application.

**`user(...)`** sets the user message — the actual input you want the model to act on. There's a lambda variant we'll meet in a moment that unlocks templating.

**`call()`** executes the request. This is the blocking, synchronous variant — your thread waits until the model has produced its complete answer. Its sibling `stream()` doesn't wait, and gets its own section later in this article.

**`content()`** extracts the model's answer as a plain `String`. It's one of several response extractors — `chatResponse()` gives you the full structured response with metadata, and `entity()` maps the output to a Java type (that one is article 5's star, so we'll leave it alone for now).

That's the whole trick. Build a prompt, send it, extract what you need. Everything else in this article is variations and configuration on top of this chain.

## System Prompts vs User Prompts

So far we've only set user messages. But there's a second message type that's arguably more important: the system prompt.

> **Concept Primer — System and User Prompts**
> LLMs distinguish between two main kinds of input. The *system prompt* defines the model's role, behavior, and constraints — "you are a support assistant, be concise, never invent order numbers." The *user prompt* is the actual input to process — the customer's ticket text. Think of the system prompt as the job description and the user prompt as the task that lands on the desk. The model treats system instructions with higher authority, which is exactly why behavioral rules belong there and not in the user message. If you already knew this — told you it was skippable.

In the fluent API, the system prompt gets its own method:

```java
String summary = chatClient
        .prompt()
        .system("""
                You are a support ticket assistant for BrightCart, an online retailer.
                Summarize customer tickets in 2-3 sentences.
                Be factual. Do not speculate about causes or promise solutions.
                """)
        .user(ticketText)
        .call()
        .content();
```

The separation matters more than it looks. The system prompt is *yours* — stable, version-controlled, carefully worded. The user message is *theirs* — unpredictable, occasionally furious, sometimes written entirely in capital letters. Keeping them apart is the first step toward prompts you can actually maintain, and it's also a security boundary: instructions in the system prompt are much harder for a mischievous user message to override.

We'll go much deeper into prompt design in article 4. For now, the rule of thumb: behavior in the system prompt, data in the user prompt.

## Defaults: Configure Once, Use Everywhere

Writing the same system prompt on every call gets old fast. The builder lets you bake in defaults:

```java
package org.veenx.springai.demo.config;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.prompt.ChatOptions;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AiConfig {
    @Bean
    public ChatClient ticketSummaryClient(ChatClient.Builder builder) {
        return builder
                .defaultSystem("""
                        You are a support ticket assistant for BrightCart, an online retailer.
                        Summarize customer tickets in 2-3 sentences for support agents.
                        Mention any order numbers verbatim.
                        Be factual. Do not speculate about causes or promise solutions.
                        """)
                .defaultOptions(
                        (ChatOptions.Builder) ChatOptions.builder()
                                .temperature(0.2)
                                .maxTokens(512))
                .build();
    }
}
```
> Note that there are API changes in Spring AI 2.0, so `defaultSystemPrompt` is now `defaultSystem`. The same goes for `defaultOptions` — it now expects a `ChatOptions.Builder` instead of a fully built `ChatOptions` object. This change allows you to set defaults without having to construct the options upfront, giving you more flexibility in how you configure your `ChatClient`.

Now every call through this `ChatClient` carries that system prompt automatically. Your call sites shrink back down to:

```java
String summary = chatClient.prompt().user(ticketText).call().content();
```

Defaults exist for most of the fluent API: `defaultSystemPrompt`, `defaultUser`, `defaultOptions`, and `defaultAdvisors` (those last ones are article 7 material — patience). The override rule is what you'd hope for: per-call values win over defaults. Set a default system prompt in the bean, override it in one specific call when needed, and everything else keeps using the default.

This is the pattern that makes multiple `ChatClient` beans worthwhile, by the way. A `ChatClient` isn't just a connection to a model — it's a connection *plus a personality*. BrightCart will eventually have one client configured for summarization and another for classification, each with its own defaults, both backed by the same model.

## The Big Three Model Options

Article 2 promised an explanation of those mysterious properties — `temperature`, `max-tokens`, `top-p` — and here it is. There are a dozen-plus options you *can* tune, but three do most of the work in practice.

> **Concept Primer — Why Models Have Knobs**
> An LLM generates text one token at a time, and at each step it has a probability distribution over possible next tokens. The options below control how the model samples from that distribution and when it stops. They're request-level settings — no retraining, no fine-tuning, just "behave differently for this call."

### Temperature

Temperature controls randomness. At `0.0`, the model almost always picks the most likely next token — same input, (nearly) same output, every time. At `1.0`, it samples more freely and gets creative.

For BrightCart's ticket summaries, creativity is a bug, not a feature. A customer reporting a damaged package does not need poetry. We want low temperature:

```properties
spring.ai.anthropic.chat.options.temperature=0.2
```

Rule of thumb: factual tasks (summarization, extraction, classification) live at `0.0–0.3`. Conversational tasks feel natural around `0.5–0.7`. Anything above `0.8` is for brainstorming and creative writing — and for generating bug reports you'll struggle to reproduce.

### Max Tokens

The hard ceiling on response length, measured in tokens — roughly three-quarters of a word each. Two reasons to care: cost, since providers bill per token, and protection, because without a ceiling a chatty model can produce far more output than you need.

One sharp edge: max tokens is a *cutoff*, not a target. If the model hits the limit mid-thought, the response just stops. The model doesn't summarize faster to fit — it gets cut off like a conference speaker at the end of their slot. We'll see how to detect that in the metadata section below.

```properties
spring.ai.anthropic.chat.options.max-tokens=512
```

For 2–3 sentence summaries, 512 is generous. Better slightly too generous than truncated.

### Top-p

Top-p (also called nucleus sampling) is temperature's subtler sibling. Instead of scaling randomness, it restricts the model to the smallest set of tokens whose combined probability reaches the threshold. At `0.9`, the model samples only from the most plausible 90% of probability mass and discards the weird tail entirely.

In practice: adjust temperature *or* top-p, not both. Tuning both at once is how you end up with behavior nobody can explain in the retro. For most applications, setting temperature and leaving top-p at its default is the right call — which is exactly what we do for BrightCart.

### Setting Options in Code

Properties set application-wide defaults. For per-client or per-call control, use `ChatOptions`:

```java
package org.veenx.springai.demo.config;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.prompt.ChatOptions;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AiConfig {

    @Bean
    public ChatClient chatClient(ChatClient.Builder builder) {
        return builder
                .defaultSystemPrompt("""
                        You are a support ticket assistant for BrightCart, an online retailer.
                        Summarize customer tickets in 2-3 sentences.
                        Be factual. Do not speculate about causes or promise solutions.
                        """)
                .defaultOptions(ChatOptions.builder()
                        .temperature(0.2)
                        .maxTokens(512)
                        .build())
                .build();
    }
}
```

`ChatOptions` is the portable, provider-neutral options type — it covers temperature, max tokens, top-p, and the other common settings. Provider-specific types like `AnthropicChatOptions` exist for options unique to one provider, but reaching for the portable type first keeps your configuration provider-agnostic. The full list of every available option is in the [Spring AI reference documentation](https://docs.spring.io/spring-ai/reference/2.0/).

## Beyond content(): ChatResponse and Metadata

`content()` is convenient, but it throws away information you've already paid for. The `chatResponse()` extractor gives you the full picture:

```java
package org.veenx.springai.demo.service;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.model.ChatResponse;
import org.springframework.stereotype.Service;

@Service
public class TicketSummaryService {

    private final ChatClient chatClient;

    public TicketSummaryService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    public String summarize(String ticketText) {
        ChatResponse response = chatClient
                .prompt()
                .user(ticketText)
                .call()
                .chatResponse();

        var usage = response.getMetadata().getUsage();
        System.out.printf("Tokens — prompt: %d, completion: %d, total: %d%n",
                usage.getPromptTokens(),
                usage.getCompletionTokens(),
                usage.getTotalTokens());

        var finishReason = response.getResult().getMetadata().getFinishReason();
        System.out.println("Finish reason: " + finishReason);

        return response.getResult().getOutput().getText();
    }
}
```

Two pieces of metadata earn their keep immediately:

**Token usage** tells you what each request actually cost. Prompt tokens are your input (system prompt included — yes, you pay for your own instructions on every call), completion tokens are the model's output. When BrightCart processes ten thousand tickets a month, these numbers stop being a curiosity and start being a line item. Article 13 builds real cost controls on top of this; for now, just know the data is there.

**Finish reason** tells you *why* the model stopped. The value you want says the model finished naturally (for Anthropic, `end_turn`). The value that should trigger alarm bells indicates the max tokens ceiling cut the response off mid-thought (`max_tokens`). A truncated summary silently stored in your database is exactly the kind of bug that surfaces three weeks later in a customer escalation — checking the finish reason is how you catch it on day one.

For everyday calls where you don't need any of this, `content()` remains the right choice. But production code paths — and BrightCart's summarizer is one — should look at the metadata.

## Streaming: Don't Make Them Wait

Here's an uncomfortable truth about LLMs: they're slow. A multi-paragraph response can take several seconds to generate, and `call()` makes your caller wait for every last token before showing anything.

For a background batch job, who cares. But for anything with a human staring at it, those seconds feel eternal. You've experienced the fix yourself — every AI chat interface you've ever used shows the answer appearing word by word. That's not a visual gimmick. The model genuinely produces tokens one at a time, and streaming forwards each one to the client the moment it exists. Perceived latency drops from "is this thing broken?" to near-instant, because the first words appear within a few hundred milliseconds.

In Spring AI, you swap `call()` for `stream()`:

```java
Flux<String> stream = chatClient
        .prompt()
        .user(ticketText)
        .stream()
        .content();
```

The return type is a Reactor `Flux<String>` — an asynchronous stream of text chunks. If you've never touched reactive programming: don't panic, and more importantly, don't feel obliged to start now. You don't need reactive expertise to *use* streaming, because Spring handles the plumbing. Return the `Flux` from a controller method and Spring MVC streams it to the HTTP client as server-sent events:

```java
package org.veenx.springai.demo.web;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import reactor.core.publisher.Flux;

@RestController
@RequestMapping("/api/tickets")
public class TicketStreamController {

    private final ChatClient chatClient;

    public TicketStreamController(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    @PostMapping(value = "/summarize/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> summarizeStreaming(@RequestBody String ticketText) {
        return chatClient
                .prompt()
                .user(ticketText)
                .stream()
                .content();
    }
}
```

Try it with curl and watch the summary arrive in pieces:

```bash
curl -N -X POST http://localhost:8080/api/tickets/summarize/stream \
     -H "Content-Type: text/plain" \
     -d "I ordered a coffee machine two weeks ago, order BC-48291. The tracking says delivered but nothing arrived. My neighbor also did not receive it. Please help."
```

The `-N` flag disables curl's buffering so you actually see the streaming effect.

When should BrightCart stream? Anywhere a support agent is watching the screen — a summary appearing live in the agent dashboard beats a spinner every time. When shouldn't it? The automated pipeline that processes tickets overnight gains nothing from streaming; `call()` is simpler and the metadata handling is more straightforward. Pick per use case, not per application.

## Putting It Together: BrightCart's First Endpoint

Time to assemble the pieces into the system's first real feature. One question worth answering first: why a REST controller instead of the `CommandLineRunner` from articles 1 and 2?

Two reasons. It's the realistic shape — BrightCart's ticket system will be called by other systems, and HTTP endpoints are how that happens. And it's *easier to experiment with* — you tweak a prompt, fire another curl request, and see the result. No application restart for every attempt. The CommandLineRunner was great for "does this work at all"; we've outgrown it.

One new dependency in the `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

The configuration — everything this article covered, in one bean:

```java
package org.veenx.springai.demo.config;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.prompt.ChatOptions;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AiConfig {
    @Bean
    public ChatClient ticketSummaryClient(ChatClient.Builder builder) {
        return builder
                .defaultSystem("""
                        You are a support ticket assistant for BrightCart, an online retailer.
                        Summarize customer tickets in 2-3 sentences for support agents.
                        Mention any order numbers verbatim.
                        Be factual. Do not speculate about causes or promise solutions.
                        """)
                .defaultOptions(
                        (ChatOptions.Builder) ChatOptions.builder()
                                .temperature(0.2)
                                .maxTokens(512))
                .build();
    }
}
```

The service — with the metadata check that separates production code from demo code:

```java
package org.veenx.springai.demo.service;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.model.ChatResponse;
import org.springframework.stereotype.Service;

@Service
public class TicketSummaryService {

    private static final Logger log = LoggerFactory.getLogger(TicketSummaryService.class);

    private final ChatClient ticketSummaryClient;

    public TicketSummaryService(ChatClient ticketSummaryClient) {
        this.ticketSummaryClient = ticketSummaryClient;
    }

    public String summarize(String ticketText) {
        ChatResponse response = ticketSummaryClient
                .prompt()
                .user(ticketText)
                .call()
                .chatResponse();

        var usage = response.getMetadata().getUsage();
        log.info("Ticket summarized — tokens used: {} prompt, {} completion",
                usage.getPromptTokens(), usage.getCompletionTokens());

        var finishReason = response.getResult().getMetadata().getFinishReason();
        if (!"end_turn".equalsIgnoreCase(String.valueOf(finishReason))) {
            log.warn("Summary may be truncated — finish reason: {}", finishReason);
        }

        return response.getResult().getOutput().getText();
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
import org.veenx.springai.demo.service.TicketSummaryService;

@RestController
@RequestMapping("/api/tickets")
public class TicketController {

    private final TicketSummaryService summaryService;

    public TicketController(TicketSummaryService summaryService) {
        this.summaryService = summaryService;
    }

    @PostMapping("/summarize")
    public String summarize(@RequestBody String ticketText) {
        return summaryService.summarize(ticketText);
    }
}
```

Start the application and feed it a properly chaotic customer ticket:

```bash
curl -X POST http://localhost:8080/api/tickets/summarize \
     -H "Content-Type: text/plain" \
     -d "Hi so I ordered the deluxe espresso machine like TWO WEEKS ago (order BC-48291) and the tracking said delivered Tuesday but there is NOTHING. I checked with my neighbors, nothing. I paid 379 euros for this!! I want a refund or a new machine ASAP. Also your hold music is terrible."
```

The response:

```
Customer reports that order BC-48291 (deluxe espresso machine, €379) shows as delivered on Tuesday but has not been received.    
They have checked with neighbors and are requesting either a refund or replacement.   
Customer also commented negatively about hold music
```

Calm, factual, and the hold music complaint made it in as well. The support agent gets exactly what they need in three seconds of reading instead of untangling the original.

That's BrightCart's first feature shipped. Small, but the structure underneath — configured client, service with metadata checks, clean controller — is the foundation everything else builds on.

## What's Next

You now know `ChatClient` properly: three ways to create it, the full fluent chain, system versus user prompts, defaults and overrides, the big three options, response metadata, and streaming. The black box is open.

But you may have noticed our system prompt is doing a lot of heavy lifting with hand-written instructions, and our user messages are raw strings glued together. That doesn't scale. In the next article we cover prompt engineering the Java way — `PromptTemplate`, variables, reusable prompt structures, and how to keep prompts maintainable when BrightCart's collection inevitably grows from one to twenty.

The dashboard is mastered. Time to learn what to say into the microphone.

*This is part 3 of a 13-part series.*
