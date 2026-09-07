---
title: "Embabel Overview and Architecture"
date: 2026-09-09
draft: true
tags: ["Java", "AI", "Spring-AI", "Embabel"]
cover:
  image: "/images/embabel.png"
  alt: "Embabel Overview and Architecture"
description: "Learn about Rod Johnson's newest project that might shake  the Java world like Spring did 25 years ago."
categories: ["ai", "java"]
---

# Embabel: an agent framework for the JVM

*A working reference: philosophy, architecture, a worked example, and REST invocation. Updated for Embabel 1.0 (GA, July 20, 2026).*

---

## 1. Philosophy

Embabel is built by no other than Rod Johnson. Every Java (should) know who he is. He is the creator of the Spring Framework, the most used Java framework for more that two decades. 
It is build on the premise that neither raw LLM calls nor MCP tool access are enough to build reliable business applications. MCP is an important step forward and Embabel embraces it, but a higher-level orchestration layer is still needed — especially for explainability (why were choices made) and discoverability, which MCP doesn't address on its own.

Two engineering values run through the design:

- **Decomposition** — break complex tasks into small, focused actions rather than one mega-prompt. Ordinary software engineering discipline doesn't get suspended just because AI is involved.
- **Type-safe domain modeling** — build a rich domain model (Kotlin data classes or Java records) so prompts are typesafe, toolable, and survive refactoring.

The framework targets the enterprise/JVM world rather than being a research toy — the goal is minimizing nondeterminism and integrating with existing business systems, not chasing novelty. It's built on Spring/Spring AI, so it inherits DI, configuration, and testability instead of reinventing them. Rod Johnson's own framing: *Spring AI is like the Servlet API, and Embabel is like Spring MVC.*

## 2. The problem it's trying to fix

Most agent frameworks (LangGraph and similar) make you hand-wire a graph of nodes and edges — a state machine you maintain yourself. Adding a capability means rewiring the graph.

Embabel's answer borrows from game AI: **Goal-Oriented Action Planning (GOAP)**. Rather than wiring together a graph of nodes, the planner works out the path itself. Concretely, Embabel models agentic flows using actions, goals, and conditions, and — unlike simpler frameworks — leverages GOAP for proper planning, so applications can combine steps in novel ways and dynamically replan after each action, forming an OODA loop.

Why this matters practically: treating planning as a distinct capability with a specialized algorithm, rather than relying solely on the LLM, yields:

- **Efficiency** — fewer unnecessary LLM calls
- **Adaptability** — dynamic replanning when state changes
- **Composability** — new actions and goals can be added without touching existing code
- **Parallelism** — independent actions can run concurrently

Crucially, the planner itself is deterministic code, not another LLM call — a distinction that buys explainability and predictability in a world of non-deterministic AI responses.

## 3. Architecture

Three core building blocks:

- **Actions** — discrete steps the agent can take, annotated in code, each with preconditions and effects.
- **Goals** — what the agent is trying to achieve, determined dynamically rather than hard-coded.
- **Plans** — a sequence of actions toward a goal. After *every* action, the system replans, so it adapts mid-flight instead of following a fixed script.

It's a Spring citizen throughout: `@Agent` is a stereotype annotation built on `@Component`, so agents get full DI, configuration binding, and testing support. It sits on top of Spring AI for actual model calls, so it works with any Spring AI–supported provider. There's a **blackboard**-style shared state that actions read and write, a **personas** mechanism for reusable role/goal/backstory prompt fragments, and support for multi-agent systems, agentic RAG, and utility AI for more open-ended tasks. Written in Kotlin, designed to be very Java-friendly.

One trade-off worth naming: teams should think carefully about action pre- and post-conditions — a poorly specified condition can lead the planner somewhere unintended. The framework provides a testing approach that lets you mock blackboard state and assert on which actions were planned.

## 4. Worked example: a blog-writing agent

### 4.1 Setup (Embabel 1.0)

Spring Boot 3.5.x, Java 21 baseline, Spring AI 1.1.7 under the hood. As of 1.0, Embabel resolves from **Maven Central** — no custom repository configuration needed — and ships **dedicated starters per model provider**.

```xml
<properties>
    <java.version>21</java.version>
    <embabel.version>1.0.0</embabel.version>
</properties>

<dependencies>
    <dependency>
        <groupId>com.embabel.agent</groupId>
        <artifactId>embabel-agent-starter-shell</artifactId>
        <version>${embabel.version}</version>
    </dependency>
    <dependency>
        <groupId>com.embabel.agent</groupId>
        <artifactId>embabel-agent-starter-openai</artifactId>
        <version>${embabel.version}</version>
    </dependency>
</dependencies>

<!-- no <repositories> block needed — resolves from Maven Central -->
```

> Note: 1.0 does **not** yet support Spring Boot 4 / Spring AI 2.0. That's targeted for Embabel 2.0. A community workaround exists (a manual `Jackson2ObjectMapperBuilder` bean to bridge Jackson 2 → 3) but it's a stopgap, not a recommendation, for anyone starting fresh.

### 4.2 Domain model

Plain Java records — the LLM's output is forced into this exact shape, no manual JSON parsing.

```java
public record BlogDraft(String title, String content) {}
public record ReviewedPost(String title, String content, String feedback) {}
public record SEOTags(String metaDescription, List<String> keywords, String slug) {}
public record PublishedPost(String title, String content, String feedback, SEOTags seo) {}
```

### 4.3 The agent

```java
@Agent(description = "Write and review a blog post about a given topic")
public class BlogWriterAgent {

    @Action(description = "Write a first draft of the blog post")
    public BlogDraft writeDraft(UserInput userInput, AI ai) {
        return ai.withDefaultLlm()
            .withId("blog-post-draft-writer")
            .creating(BlogDraft.class)
            .fromPrompt("""
                Write a blog post about %s.
                Keep it practical and beginner-friendly.
                """.formatted(userInput.getContent()));
    }

    @Action(description = "Review and improve the draft")
    public ReviewedPost reviewDraft(BlogDraft draft, AI ai) {
        return ai.withLlmByRole("reviewer")
            .withId("blog-post-reviewer")
            .creating(ReviewedPost.class)
            .fromPrompt("""
                Review and improve this blog post:
                Title: %s
                Content: %s
                """.formatted(draft.title(), draft.content()));
    }

    @Action(description = "Generate SEO metadata for the blog post")
    public SEOTags generateSEOTags(BlogDraft draft, AI ai) {
        return ai.withDefaultLlm()
            .withId("blog-post-seo-tagger")
            .creating(SEOTags.class)
            .fromPrompt("""
                Generate SEO metadata for this blog post:
                Title: %s
                Content: %s

                Provide a meta description under 160 characters,
                5-8 relevant keywords, and a URL-friendly slug.
                """.formatted(draft.title(), draft.content()));
    }

    @Action(description = "Assemble the final publish-ready post")
    @AchievesGoal(description = "A polished, SEO-tagged blog post ready to publish")
    public PublishedPost publishPost(ReviewedPost reviewed, SEOTags seo) {
        return new PublishedPost(reviewed.title(), reviewed.content(), reviewed.feedback(), seo);
    }
}
```

Model routing per role, configured externally rather than hard-coded:

```yaml
embabel:
  models:
    default-llm: gpt-4.1-mini
    llm:
      reviewer: gpt-4.1
```

### 4.4 How the planner reasons about it

Each `@Action` has **preconditions** (what must exist in shared state before it can run) and **effects** (what becomes true after it runs). The planner is a GOAP solver: given the current state and the declared goal (`@AchievesGoal`), it searches over the graph of actions, matching preconditions against effects, and finds a path to the goal — replanning after every step rather than committing to a fixed script upfront.

**Two-action case** (`writeDraft` → `reviewDraft`): a straight line. `writeDraft` produces the only thing `reviewDraft` needs, so there's exactly one legal path.

```
UserInput
    │
    ▼
writeDraft (default llm, cheap)
    │
    ▼
BlogDraft
    │
    ▼
reviewDraft (achieves goal)
    │
    ▼
ReviewedPost (goal output)
```

**Three-action case, with the branch:** `reviewDraft` and `generateSEOTags` both only need `BlogDraft` and don't depend on each other, so the planner has no ordering constraint between them — they can run in either order or in parallel. `publishPost` has two preconditions and can't fire until both are satisfied.

```
                     UserInput
                         │
                         ▼
                  writeDraft (default llm, cheap)
                         │
                         ▼
                     BlogDraft
                    ╱          ╲
                   ▼            ▼
       reviewDraft          generateSEOTags
     (llm role: reviewer)   (parallel with review)
           │                        │
           ▼                        ▼
      ReviewedPost               SEOTags
                    ╲          ╱
                     ▼        ▼
                  publishPost (achieves goal)
                         │
                         ▼
                   PublishedPost (goal reached)
```

This fan-out/fan-in shape is the payoff of GOAP over a hand-wired graph: adding `generateSEOTags` didn't require rewiring anything. The planner discovered the branch and the merge point on its own, purely from each action's declared inputs and outputs.

## 5. Invoking it over REST

You don't invoke a specific agent by name — you invoke Embabel for a **goal type** and let it find whichever agent can produce it.

```java
@RestController
@RequestMapping("/api/v1/blog")
public class BlogController {

    private final AgentPlatform agentPlatform;

    public BlogController(AgentPlatform agentPlatform) {
        this.agentPlatform = agentPlatform;
    }

    @PostMapping("/write")
    public PublishedPost write(@RequestBody BlogRequest request) {
        var invocation = AgentInvocation
            .builder(agentPlatform)
            .options(ProcessOptions.builder().verbosity(v -> {
                v.showPlanning(true);
                v.showPrompts(true);
            }).build())
            .build(PublishedPost.class);

        return invocation.invoke(new UserInput(request.topic()));
    }

    public record BlogRequest(String topic) {}
}
```

```java
@SpringBootApplication
@EnableAgents
public class BlogAgentApplication {
    public static void main(String[] args) {
        SpringApplication.run(BlogAgentApplication.class, args);
    }
}
```

```bash
curl -X POST http://localhost:8080/api/v1/blog/write \
  -H "Content-Type: application/json" \
  -d '{"topic": "How to get started with Spring Boot"}'
```

### What's happening

- **`AgentPlatform`** — injected like any Spring bean; the runtime that knows about every `@Agent` registered in the context.
- **`.build(PublishedPost.class)`** — the key line. Not "run `BlogWriterAgent`," but "give me an object of type `PublishedPost`." Embabel scans registered agents, finds the `@AchievesGoal` method whose return type matches, and selects that agent.
- **`.invoke(new UserInput(...))`** — places the object on the blackboard. The planner works backward from the goal: needs `PublishedPost` → needs `ReviewedPost` + `SEOTags` → both need `BlogDraft` → needs `UserInput`, already present. Runs the full branching graph synchronously and returns the result as the HTTP response body.

### Lifecycle, end to end

```
HTTP POST /write
      │
      ▼
BlogController.write()
      │
      ▼
AgentInvocation.build(PublishedPost.class)
   (goal-based agent lookup)
      │
      ▼
UserInput placed on blackboard
      │
      ▼
GOAP planning & execution
   (writeDraft → reviewDraft ∥ generateSEOTags → publishPost)
      │
      ▼
PublishedPost returned as JSON
```

### Before production

1. **Synchronous by default.** Multiple LLM calls can take a while — long enough to risk an HTTP timeout. `AgentInvocation` has `invokeAsync()`, returning a `CompletableFuture<PublishedPost>`, so you can return a job id immediately and poll separately.
2. **No auth or rate limiting here.** Add a `@ControllerAdvice` for agent-execution failures, and something to stop repeated calls from running up the model bill.
3. **Version drift.** The API surface moved fast pre-1.0 (`0.2.0` through `0.4.0-SNAPSHOT`). Double-check package paths against whatever version you pull in.

## 6. Implications of the 1.0 GA release (July 20, 2026)

**Doesn't affect the code above.** `@Agent`, `@Action`, `@AchievesGoal`, `AgentPlatform`, `AgentInvocation` are unchanged. The theme of 1.0 is maturity: experimental APIs — including the RAG APIs — were promoted to production status, and deprecated methods were removed before the stability guarantee kicked in.

**Does affect the build setup:**

- Now on **Maven Central** on stable coordinates — no custom repository blocks needed.
- **Dedicated starters per model provider** (`embabel-agent-starter-openai`, `-anthropic`, `-ollama`, `-bedrock`, etc.) instead of a generic Spring AI starter alongside the Embabel one.
- Built on **Spring AI 1.1.7 and Spring Boot 3.5.x**, with a **Java 21 baseline** — not Spring Boot 4 or Spring AI 2.0. That jump is targeted for Embabel 2.0; there's already a 2.0 development line in the repo.

**Other notable 1.0 additions** (not used in this example, but worth knowing about): generic media and document support (multimodal inputs), MCP server health exposed via Spring Boot Actuator, chat message events, configurable planner behavior when multiple goals share a return type, and new model integrations (Z.ai GLM, updated DeepSeek names) alongside the existing OpenAI, Anthropic, Bedrock, Google GenAI, Ollama, and LM Studio options.

---

*Sources: [Embabel GitHub](https://github.com/embabel/embabel-agent), [Embabel 1.0 release notes](https://github.com/embabel/embabel-agent/releases/tag/v1.0.0), Dan Vega's [Embabel First Look](https://www.danvega.dev/blog/embabel-first-look) and [Embabel 1.0 Is Here](https://www.danvega.dev/blog/embabel-1-0-ga), [BootcampToProd's REST API walkthrough](https://bootcamptoprod.com/embabel-framework-rest-api-agent/), [InfoQ coverage](https://www.infoq.com/news/2026/08/embabel-1/).*