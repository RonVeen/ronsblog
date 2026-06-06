---
title: "Spring AI Series: Introduction to Spring AI"
date: 2026-06-06
draft: false
tags": ["Java", "Spring Boot", "AI", "Spring AI"]
cover:
  image: "/images/spring-ai-01-introduction.png"
  alt: "Spring AI Series: Introduction to Spring AI"
series: ["Spring AI in Depth"]
series_order: 1000
description: "Learn what Spring AI is, how it fits into the Spring ecosystem, and get your first chat response in a handful of lines of Java."
---

I still remember the moment it clicked — and the much longer stretch before it did.

I'd been building Spring Boot applications for years. I knew the ecosystem. I knew how to wire beans, configure starters, and navigate the autoconfiguration magic without panicking. Then I started exploring Spring AI, and suddenly I was lost again. *Embeddings*. *Vector stores*. *RAG pipelines*. *Advisors*. *ChatClient vs ChatModel*. A wall of new terminology that felt completely disconnected from everything I already knew.

It wasn't that Spring AI was poorly designed. It wasn't. It's actually quite elegant once you see the shape of it.

The problem was that nobody had drawn the map.

That's what this series is. A map — built for Java developers who know Spring but are new to the AI side of things. Thirteen articles that take you from "what even is this?" to running production-grade AI features in a Spring Boot 4 application. We'll go at depth, not at speed.

But here's the thing. You don't have to wait thirteen articles before something works.

## What Is Spring AI?

Spring AI is the Spring ecosystem's answer to a question Java developers have been asking since ChatGPT made LLMs unavoidable: *how do I integrate this into my existing applications without throwing away everything I know?*

The Python world had LangChain. Java had… nothing official for a while. Spring AI fills that gap.

At its core, Spring AI is an abstraction layer over large language models and the infrastructure around them — model providers, vector databases, embedding models, prompt templates, output parsers. It speaks fluent Spring. You get autoconfiguration, dependency injection, familiar property-based setup, and Spring Boot starters. If you've used Spring Data or Spring Security, the mental model will feel recognisable surprisingly quickly.

It's not a port of LangChain. It's not a thin wrapper around the OpenAI SDK. It's a first-class Spring project with its own opinions, its own abstractions, and — as of Spring AI 2.0 built on Spring Boot 4 — a solid foundation for serious production work.

## Where It Fits in the Spring Ecosystem

Think of Spring AI the same way you think about Spring Data or Spring Security. It's not replacing anything — it's adding a new dimension.

Your existing `@Service` beans, your JPA repositories, your REST controllers — they all stay exactly where they are. Spring AI slots in alongside them. You can have a `ChatClient` bean injected into your service layer just as naturally as a `JdbcTemplate` or a `RestClient`.

That's the key insight. Spring AI doesn't ask you to build a new kind of application. It asks you to add new capabilities to the kind you already build.

## What We're Building Toward

Over the course of this series, we'll cover:

- The core abstractions — ChatClient, the Model API, Prompt templates
- Getting reliable structured output from LLMs
- Tool calling, so your models can interact with your existing services
- Advisors, Spring AI's middleware layer for cross-cutting concerns
- Chat memory and conversation state
- Full RAG pipelines — ingestion, embeddings, vector stores, retrieval
- Multimodality — images and audio
- Observability, evaluation, and testing
- Production concerns — cost control, caching, fallback strategies, secrets

Each article builds on the previous ones. By the end, you'll have a complete picture — not just a collection of isolated examples.

## A Note on Tooling

Throughout this series, we'll use **Claude** (Anthropic) as the default model provider. It's what I use day to day, and it keeps the examples consistent and reproducible.

That said, Spring AI's abstraction layer means switching providers is genuinely trivial — usually a single dependency swap and a handful of changed properties. Like when you need to switch JPA providers. It is easy, and you already know how to do that. In the next article, we'll cover how to switch to OpenAI or run a local model via Ollama or LMStudio. If you're already comfortable with model provider configuration, you can skip that section entirely.

For the agentic coding parts — generating boilerplate, scaffolding configurations, writing test stubs — we'll use **Claude Code**. The prompts will be explicit and reproducible, so any terminal-based coding agent you prefer will work just as well.

## Getting Your API Key

Before we can run anything, you need an Anthropic API key. Head to [console.anthropic.com](https://console.anthropic.com), sign up with your email or Google account, and generate a key under Settings → API Keys.

New accounts get approximately $5 in starter credits. No payment method required to claim them — just sign up and go. That's enough to comfortably follow along with this article and the next few.

For sustained experimentation across the full series, those credits will eventually run out. Don't worry — article 2 covers free alternatives in full, including how to run a local model via Ollama or LMStudio with zero API costs and zero usage limits. If you'd rather start there, that's completely valid.

One important note: *never* hardcode your API key in source code or commit it to version control. We'll use an environment variable throughout this series — `${ANTHROPIC_API_KEY}` — and article 2 covers proper secrets management in more detail.

## Let's See Something Work

Enough framing. Here's Spring AI in its simplest form — a Spring Boot 4 application that sends a message to a model and prints the response.

Here's the complete `pom.xml` for the project we'll use throughout this article:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
                             https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.0-M8</version>
        <relativePath/>
    </parent>

    <groupId>org.veenx</groupId>
    <artifactId>spring-ai-demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>spring-ai-demo</name>
    <description>Spring AI in Depth — demo project</description>

    <properties>
        <java.version>21</java.version>
        <spring-ai.version>2.0.0</spring-ai.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.ai</groupId>
                <artifactId>spring-ai-bom</artifactId>
                <version>${spring-ai.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-starter-model-anthropic</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```

> Java 21 is the minimum for this series. If you're on Java 25 — which I'd encourage for cutting-edge features like virtual threads and structured concurrency — just change `<java.version>21</java.version>` to `<java.version>25</java.version>`. Everything in this series works on both.
> Note that we are using Spring Boot 4.0.0-M8. This is the latest available version at the time of writing. Once it becomes final, we will use that version.
Prefer Gradle? The complete `build.gradle` is available in the [GitHub repository](https://github.com/RonVeen/spring-ai-in-depth) along with the full source code for every article in this series.

Set your API key in `application.properties`:

```properties
spring.ai.anthropic.api-key=${ANTHROPIC_API_KEY}
spring.ai.anthropic.chat.options.model=claude-haiku-4-5-20251001
```

> We're deliberately using Claude Haiku — Anthropic's most affordable model — throughout this series. For learning and experimentation it's more than capable, and it keeps your API costs minimal while following along. When you move to production you can swap in a more powerful model with a single property change.

Now the code:

```java
package org.veenx.springai.demo;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SpringAiDemoApplication implements CommandLineRunner {

    private final ChatClient chatClient;

    public SpringAiDemoApplication(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    @Override
    public void run(String... args) {
        String response = chatClient
                .prompt()
                .user("What makes Spring AI different from using an LLM SDK directly?")
                .call()
                .content();

        System.out.println(response);
    }

    public static void main(String[] args) {
        SpringApplication.run(SpringAiDemoApplication.class, args);
    }
}
```

Run it. You'll get a real response from Claude, formatted as plain text, in your console.

That's it.

No HTTP client setup. No JSON serialisation boilerplate. No manual API key injection. Spring AI's autoconfiguration handled the `ChatClient` construction the moment it found your API key and the Anthropic starter on the classpath.

It's a small example — deliberately so. But notice what's already in place: a fully injectable `ChatClient`, a fluent prompt API, and a clean separation between how you build a prompt and how you call the model. These aren't accidents. They're the shape of Spring AI, and we'll be exploring every corner of it across this series.

## What's Next

In the next article, we'll do a proper project setup — starters, dependencies, autoconfiguration deep dive, and how to switch between Anthropic, OpenAI, and Ollama. We'll also look at the Model API layer that sits beneath `ChatClient`, so you understand what Spring AI is actually wiring up on your behalf.

If the example above already has you curious about what `ChatClient.Builder` is doing under the hood — good. That's exactly where we're headed.

*This is part 1 of a 13-part series.*

[Next: Spring AI in Depth - 02 - Project Setup and Auto-Configuration Deep Dive →]