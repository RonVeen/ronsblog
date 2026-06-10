---
title: "Spring AI Series: Setup Spring AI"
date: 2026-06-09
draft: false
tags": ["Java", "Spring Boot", "AI", "Spring AI"]
cover:
  image: "/images/spring-ai-02-setup.png"
  alt: "Spring AI Series: Introduction to Spring AI"
series: ["Spring AI in Depth"]
series_order: 950
description: "Learn to setup a Spring AI project and understand how it works."
categories: ["ai", "java"]
---

In the previous article, we ran a Spring AI application in about twenty lines of Java. It worked. A `ChatClient` appeared out of nowhere, we called `.prompt().user(...).call().content()`, and Claude answered.

That's the magic of Spring Boot autoconfiguration — and also the danger of it.

When things work without explanation, you're fine right up until you need to debug something, customise something, or explain to a colleague what exactly is happening. Then you're stuck. I've been there more times than I care to admit — staring at a `NoSuchBeanDefinitionException` at 4pm on a Friday, completely unable to explain why a bean that "should just be there" isn't. (It was a missing property. It's always a missing property.)

So before we go any further into Spring AI's features, let's open the hood. This article covers everything you need to set up a proper Spring AI project and understand what Spring is wiring up on your behalf. By the end, you'll be able to look at any Spring AI autoconfiguration and know exactly what it's doing.

We'll also cover something promised in article 1: how to switch from Anthropic to OpenAI, or run a completely free local model via Ollama or LMStudio.

## What You'll Need

Before anything else — the prerequisites:

- **Java 25.** Java 21 works too if you're not on the latest yet.
- **Maven 3.9+ or Gradle 8.8+.** Either works. We'll use Maven in the examples; the equivalent Gradle files are in the [GitHub repository](https://github.com/RonVeen/spring-ai-in-depth).
- **An IDE.** IntelliJ IDEA works best for Java. VS Code with the Java extension pack is fine too.
- **A model provider account.** We'll cover your options — including free ones — in a moment.

That's genuinely it. No Docker, no Kubernetes, no local infrastructure to wrestle with before you've even written a line of code.

## Choosing Your Model Provider

This is the first real decision you'll make in a Spring AI project. Spring AI abstracts over model providers beautifully — but you still need to pick one to start.

Here's the honest picture:

| Provider | Cost | Setup effort | Good for |
|---|---|---|---|
| Anthropic (Claude) | Pay per token (~$5 free credits) | API key only | This series' default |
| OpenAI | Pay per token (some free credits) | API key only | Widely documented |
| Ollama | Free | Install + pull a model | Local dev, zero API costs |
| LMStudio | Free | Install + download a model | Local dev with a GUI |

We'll use **Anthropic** as the default throughout this series — specifically `claude-haiku-4-5-20251001`, the most affordable model in the Claude family. It's perfectly capable for everything we'll build, and it keeps your costs minimal while learning.

If you'd rather pay nothing, **Ollama** is the best free alternative. Models run locally — no API key, no usage limits, no bill at the end of the month. The trade-off is that local models are generally less capable than frontier models and need reasonable hardware (8GB RAM minimum, 16GB recommended). For learning Spring AI, they're more than sufficient.

### Setting Up Anthropic

Head to [console.anthropic.com](https://console.anthropic.com), create an account, and generate an API key under **Settings → API Keys**. New accounts get approximately $5 in free credits — no payment method required.

Store the key as an environment variable. *Never* put it directly in `application.properties` — you will accidentally commit it eventually. We all have. It's fine. Just don't do it again.

**Mac/Linux:**
```bash
export ANTHROPIC_API_KEY=sk-ant-your-key-here
```

**Windows (PowerShell):**
```powershell
$env:ANTHROPIC_API_KEY="sk-ant-your-key-here"
```

### Setting Up OpenAI

Head to [platform.openai.com](https://platform.openai.com), create an account, and generate an API key under **API Keys**. New accounts get a small amount of free credits — enough to follow along with the early articles.

Store the key as an environment variable, same as with Anthropic:

**Mac/Linux:**
```bash
export OPENAI_API_KEY=sk-your-key-here
```

**Windows (PowerShell):**
```powershell
$env:OPENAI_API_KEY="sk-your-key-here"
```
For a more permanent solution, add it to your shell profile (`~/.zshrc`, `~/.bashrc`) or use your IDE's run configuration environment variables. We'll cover production secrets management properly in article 13.

### Setting Up Ollama (Free Alternative)

Install Ollama from [ollama.com](https://ollama.com). Then pull a model:

```bash
ollama pull llama3.2
```

That's it. Ollama runs as a local HTTP server on `http://localhost:11434`. No API key needed, no account, no credit card. Make sure it's running before starting your Spring application — `ollama serve` if it's not already running as a background service.

### Setting Up LMStudio (Free Alternative with a GUI)

If you prefer a graphical interface, [LMStudio](https://lmstudio.ai) is a good option. Download it, browse the built-in model library, download a model, and start the local server from the app. It exposes an OpenAI-compatible API on `http://localhost:1234` by default — which means you can use Spring AI's OpenAI provider to talk to it, with a custom base URL pointing at your machine.

## Creating the Project

The fastest way is [start.spring.io](https://start.spring.io). Select:

- **Project:** Maven (or Gradle)
- **Language:** Java
- **Spring Boot:** 4.0.x
- **Java:** 25
- **Dependencies:** *Anthropic Claude AI* (or whichever provider you're using)

Hit Generate, unzip, open in your IDE.

If you're setting up manually, here's the complete `pom.xml`. This is the same file from article 1.

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
        <version>4.0.0</version>
        <relativePath/>
    </parent>

    <groupId>org.veenx</groupId>
    <artifactId>spring-ai-demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>spring-ai-demo</name>
    <description>Spring AI in Depth — demo project</description>

    <properties>
        <java.version>25</java.version>
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

The Gradle equivalent is in the [GitHub repository](https://github.com/RonVeen/spring-ai-in-depth).

### The Spring AI BOM

Notice the `dependencyManagement` block. Spring AI uses a Bill of Materials — a BOM — to manage the versions of all its internal modules consistently. Declare the BOM version once, and every Spring AI dependency you add inherits it automatically. No `<version>` attributes on individual dependencies, no version mismatch headaches.

The same pattern Spring Boot itself uses. Keep the BOM version in a property — `${spring-ai.version}` — so upgrading the entire Spring AI dependency set is a one-line change.

## The Starter System

Spring AI follows Spring Boot's starter convention. Each model provider has a dedicated starter:

| Provider | Starter artifact |
|---|---|
| Anthropic | `spring-ai-starter-model-anthropic` |
| OpenAI | `spring-ai-starter-model-openai` |
| Ollama | `spring-ai-starter-model-ollama` |
| Google Vertex AI | `spring-ai-starter-model-vertex-ai-gemini` |
| Azure OpenAI | `spring-ai-starter-model-azure-openai` |
| Mistral | `spring-ai-starter-model-mistral-ai` |

Each starter pulls in the necessary dependencies and — critically — an autoconfiguration class that wires up the right Spring beans when it finds the correct properties on the classpath. You only need one starter to get a working chat model. But you can include multiple starters if you want to work with multiple providers simultaneously — and we will.

## What Autoconfiguration Actually Does

Spring Boot's autoconfiguration fires `@ConditionalOn*` checks at startup — is a certain class on the classpath? Is a certain property set? If all conditions pass, the autoconfiguration registers its beans into the application context. You can always override any autoconfigured bean by defining your own. It's "sensible defaults, not forced defaults."

When Spring AI's Anthropic autoconfiguration activates, it creates two key beans:

**`AnthropicChatModel`** — the model abstraction. This implements Spring AI's `ChatModel` interface, the common contract for all chat models regardless of provider. It's built using Spring AI 2.0's new builder — internally backed by the official Anthropic Java SDK rather than a hand-rolled HTTP client. You'll rarely construct this manually, but knowing it exists matters.

**`ChatClient.Builder`** — not Anthropic-specific, but auto-registered by Spring AI's core autoconfiguration once it detects a `ChatModel` bean. This is the builder you inject into your application code.

So when you write this:

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

The `ChatClient.Builder` you're injecting was registered by Spring AI's autoconfiguration, backed by an `AnthropicChatModel`. Two layers of abstraction, wired automatically. That's the chain.

### The Model API Layer

`ChatModel` — the interface `AnthropicChatModel` implements — is Spring AI's lowest-level chat abstraction:

```java
ChatResponse call(Prompt prompt);
```

A `Prompt` in, a `ChatResponse` out. You'll rarely use `ChatModel` directly in application code — `ChatClient` is the higher-level, more ergonomic API. But knowing `ChatModel` exists matters for two reasons: it's what you inject when writing tests (easier to mock), and it's what autoconfiguration actually creates. `ChatClient` is built *on top of* `ChatModel`.

`ChatModel` is the engine. `ChatClient` is the dashboard.

## Configuring Your Application

Spring AI is configured through Spring Boot's standard property system. We use `application.properties` throughout this series rather than YAML — not because YAML is wrong, but because properties files are easier to copy-paste individual lines from, and there's no indentation to misalign. For complex nested configuration YAML is genuinely nicer; for tutorial examples, properties win on clarity.

### Anthropic

```properties
# Required
spring.ai.anthropic.api-key=${ANTHROPIC_API_KEY}

# The model to use — we use Haiku throughout this series (cheapest option)
spring.ai.anthropic.chat.options.model=claude-haiku-4-5-20251001

# Optional tuning — we'll cover what these mean in article 3
spring.ai.anthropic.chat.options.max-tokens=1024
spring.ai.anthropic.chat.options.temperature=0.7
```

> We're deliberately using `claude-haiku-4-5-20251001` — Anthropic's most affordable model — throughout this series. It's more than capable for learning and experimentation. When you move to production, swap in a more powerful model by changing this single property.

### OpenAI

Swap the dependency in `pom.xml` to `spring-ai-starter-model-openai`, then:

```properties
spring.ai.openai.api-key=${OPENAI_API_KEY}
spring.ai.openai.chat.options.model=gpt-4o-mini
spring.ai.openai.chat.options.max-tokens=1024
spring.ai.openai.chat.options.temperature=0.7
```

Your application code doesn't change. At all.

### Ollama (Free, Local)

Swap the dependency to `spring-ai-starter-model-ollama`:

```properties
spring.ai.ollama.base-url=http://localhost:11434
spring.ai.ollama.chat.options.model=llama3.2
```

No API key. Make sure Ollama is running before starting your application.

### LMStudio (Free, Local)

LMStudio exposes an OpenAI-compatible API, so use the OpenAI provider with a custom base URL:

```properties
spring.ai.openai.base-url=http://localhost:1234
spring.ai.openai.api-key=lm-studio
spring.ai.openai.chat.options.model=local-model
```

The `api-key` value is required by the provider but ignored by LMStudio — any non-empty string works.

### Switching Providers Is One Dependency Swap

This bears repeating. Your application code — the `ChatClient` calls, the prompt construction, the response handling — is identical regardless of which provider you're using. The only things that change are the starter dependency and a handful of properties.

That's not marketing copy. It genuinely works that way. We'll see it in practice throughout the series.

## Running Two Providers Simultaneously

Here's where it gets interesting. Sometimes you want both Anthropic and OpenAI active in the same application — perhaps using Claude for creative tasks and GPT-4o-mini for structured extraction, or routing between providers based on cost. Spring AI handles this cleanly.

First, include both starters:

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-anthropic</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-openai</artifactId>
</dependency>
```

Both providers need their properties set:

```properties
spring.ai.anthropic.api-key=${ANTHROPIC_API_KEY}
spring.ai.anthropic.chat.options.model=claude-haiku-4-5-20251001

spring.ai.openai.api-key=${OPENAI_API_KEY}
spring.ai.openai.chat.options.model=gpt-4o-mini
```

Then define a `ChatClient` bean for each, backed by its respective model:

```java
package org.veenx.springai.demo.config;

import org.springframework.ai.anthropic.AnthropicChatModel;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.openai.OpenAiChatModel;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AiConfig {

    @Bean
    @Qualifier("anthropic")
    public ChatClient anthropicChatClient(AnthropicChatModel model) {
        return ChatClient.builder(model)
                .defaultSystem("You are a helpful assistant specialised in Java and Spring Boot.")
                .build();
    }

    @Bean
    @Qualifier("openai")
    public ChatClient openAiChatClient(OpenAiChatModel model) {
        return ChatClient.builder(model)
                .defaultSystem("You are a precise data extraction assistant. Always respond in JSON.")
                .build();
    }
}
```

Inject by qualifier wherever you need them:

```java
package org.veenx.springai.demo.service;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Service;

@Service
public class MultiProviderService {

    private final ChatClient anthropicClient;
    private final ChatClient openAiClient;

    public MultiProviderService(
            @Qualifier("anthropic") ChatClient anthropicClient,
            @Qualifier("openai") ChatClient openAiClient) {
        this.anthropicClient = anthropicClient;
        this.openAiClient = openAiClient;
    }

    public String chat(String message) {
        return anthropicClient.prompt().user(message).call().content();
    }

    public String extract(String message) {
        return openAiClient.prompt().user(message).call().content();
    }
}
```

Standard Spring qualifier injection. No Spring AI-specific magic involved.

One thing to be aware of: when multiple `ChatClient.Builder` beans exist, Spring's autoconfiguration for `ChatClient.Builder` may complain about ambiguity. The solution is to always construct your `ChatClient` beans explicitly from the typed model beans (`AnthropicChatModel`, `OpenAiChatModel`) rather than from the generic `ChatClient.Builder`. That's what the example above does, and it sidesteps the issue entirely.

## Customising the ChatClient Bean

For single-provider applications, the cleaner pattern is a dedicated configuration class rather than constructing `ChatClient` inline in each component:

```java
package org.veenx.springai.demo.config;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AiConfig {

    @Bean
    public ChatClient chatClient(ChatClient.Builder builder) {
        return builder
                .defaultSystem("You are a helpful assistant specialised in Java and Spring Boot.")
                .build();
    }
}
```

Any component in your application can now inject `ChatClient` directly. The `defaultSystem` call sets a system prompt that applies to every conversation through this `ChatClient` — we'll cover system prompts properly in article 3.

## Overriding Autoconfiguration

Autoconfiguration is a default, not a mandate. If you need to go beyond what properties allow — custom retry behaviour, a specific base URL, additional SDK configuration — you can construct the model bean manually using Spring AI 2.0's builder:

```java
package org.veenx.springai.demo.config;

import org.springframework.ai.anthropic.AnthropicChatModel;
import org.springframework.ai.anthropic.AnthropicChatOptions;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

@Configuration
public class AiConfig {

    @Bean
    @Primary
    public AnthropicChatModel customChatModel() {
        return AnthropicChatModel.builder()
                .apiKey(System.getenv("ANTHROPIC_API_KEY"))
                .defaultOptions(AnthropicChatOptions.builder()
                        .model("claude-haiku-4-5-20251001")
                        .maxTokens(2048)
                        .build())
                .maxRetries(3)
                .build();
    }
}
```

The `@Primary` annotation tells Spring to prefer this bean over the autoconfigured one when there's ambiguity. For most applications, property-based configuration is sufficient and the cleaner approach — but it's useful to know you can drop down to the builder when you need to.

## Debugging Autoconfiguration

At some point, something won't wire up. Spring Boot has a built-in tool for exactly this situation.

Add this to `application.properties`:

```properties
logging.level.org.springframework.boot.autoconfigure=DEBUG
```

This produces a condition evaluation report at startup — a full list of every autoconfiguration class, whether it activated, and if not, why not. It's verbose, but when you're trying to work out why your `ChatClient` bean isn't being created, it's invaluable.

For Spring AI specifically, the property prefix to look for is `spring.ai`. If autoconfiguration isn't firing, it's almost always either a missing property or a missing starter. The condition report will tell you which — saving you that Friday afternoon.

## Profiles for Multiple Environments

A common pattern is using Spring profiles to switch providers between environments — Ollama locally, Anthropic in production:

```
src/main/resources/
├── application.properties          # shared config
├── application-local.properties    # Ollama settings
└── application-prod.properties     # Anthropic settings
```

`application.properties`:
```properties
spring.ai.anthropic.chat.options.model=claude-haiku-4-5-20251001
```

`application-local.properties`:
```properties
spring.ai.ollama.base-url=http://localhost:11434
spring.ai.ollama.chat.options.model=llama3.2
```

`application-prod.properties`:
```properties
spring.ai.anthropic.api-key=${ANTHROPIC_API_KEY}
```

Run locally with `-Dspring.profiles.active=local`. Deploy to production with `-Dspring.profiles.active=prod`. The application code never changes.

## Putting It All Together

Here's a complete, properly structured Spring AI project that ties together everything in this article:

`AiConfig.java`:
```java
package org.veenx.springai.demo.config;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AiConfig {

    @Bean
    public ChatClient chatClient(ChatClient.Builder builder) {
        return builder
                .defaultSystem("You are a helpful assistant specialised in Java and Spring Boot.")
                .build();
    }
}
```

`AssistantService.java`:
```java
package org.veenx.springai.demo.service;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.stereotype.Service;

@Service
public class AssistantService {

    private final ChatClient chatClient;

    public AssistantService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    public String ask(String question) {
        return chatClient
                .prompt()
                .user(question)
                .call()
                .content();
    }
}
```

`SpringAiDemoApplication.java`:
```java
package org.veenx.springai.demo;

import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.veenx.springai.demo.service.AssistantService;

@SpringBootApplication
public class SpringAiDemoApplication implements CommandLineRunner {

    private final AssistantService assistantService;

    public SpringAiDemoApplication(AssistantService assistantService) {
        this.assistantService = assistantService;
    }

    @Override
    public void run(String... args) {
        String response = assistantService.ask(
                "What's the difference between @Component, @Service, and @Repository in Spring?"
        );
        System.out.println(response);
    }

    public static void main(String[] args) {
        SpringApplication.run(SpringAiDemoApplication.class, args);
    }
}
```

`application.properties`:
```properties
spring.ai.anthropic.api-key=${ANTHROPIC_API_KEY}
spring.ai.anthropic.chat.options.model=claude-haiku-4-5-20251001
```

Clean separation of concerns. The configuration class owns the `ChatClient` bean. The service owns the business logic. The application class owns the entry point. Each has one job. This structure carries through the rest of the series.

## What's Next

You now have a properly structured Spring AI project, a clear understanding of what autoconfiguration wires up, the ability to switch providers with a dependency swap, and a multi-provider pattern ready to use when you need it.

In the next article, we go deep on `ChatClient` itself — the fluent API, prompt construction, system messages, default options, and how to structure a configuration that scales cleanly as your application grows. If `.prompt().user(...).call().content()` has been a bit of a black box so far, article 3 opens it up completely.

The engine is running. Time to learn the dashboard.

*This is part 2 of a 13-part series.*

[← Previous: Spring AI in Depth - 01 - Introduction to Spring AI]
[Next: Spring AI in Depth - 03 - ChatClient: The Heart of Spring AI →]
