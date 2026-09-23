---
title: "Embabel Overview and Architecture"
date: 2026-09-23
draft: false
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
The full source code for the article can be found on [git](https://github.com/RonVeen/embabel-blog-agent) account

### 4.1 Setup (Embabel 1.0)

Spring Boot 3.5.x, Java 21 baseline, Spring AI 1.1.7 under the hood. As of 1.0, Embabel resolves from **Maven Central** — no custom repository configuration needed — and ships **dedicated starters per model provider**.

```xml
<properties>
    <java.version>21</java.version>
    <embabel.version>1.0.0</embabel.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>com.embabel.agent</groupId>
        <artifactId>embabel-agent-api</artifactId>
        <version>${embabel.version}</version>
    </dependency>

    <!-- Base agent engine & platform autoconfiguration -->
    <dependency>
        <groupId>com.embabel.agent</groupId>
        <artifactId>embabel-agent-platform-autoconfigure</artifactId>
        <version>${embabel.version}</version>
    </dependency>

    <!-- OpenAI model integration without shell -->
    <dependency>
        <groupId>com.embabel.agent</groupId>
        <artifactId>embabel-agent-openai-autoconfigure</artifactId>
        <version>${embabel.version}</version>
    </dependency>
</dependencies>
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
    default-llm: gpt-5.4-mini
    llms:
      reviewer: gpt-5.4
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
### Taking it for a spin
As I am writing this article while on an airplane on my way to the [WeAreDevelopers conference](https://www.wearedevelopers.com/world-congress-north-america) is San Jose California, it seems only suitable that I ask our setup to write a post about the presentation that I will give there.
Here is my prompt:
```bash
curl -X POST http://localhost:8080/api/v1/blog/write \
  -H "Content-Type: application/json" \
  -d '{"topic": "Write a little story about the presentation of Ron Veen at We Are Developers conference in San Jose California in 2026"}'
  ```

It comes with this response, which was remarkably positive about my talk.
```json
{
  "title": "Improved Blog Post Review: Ron Veen at WeAreDevelopers San Jose 2026",
  "content": "In 2026, WeAreDevelopers San Jose brought together a wide mix of people from across the software world: experienced engineers, curious beginners, startup founders, and students all eager to learn what was next in development. Among the many sessions, Ron Veen’s talk stood out for a simple reason: it felt less like a formal presentation and more like practical advice from someone who understands how real teams build software.\n\nRon opened with a message that immediately resonated: great software development is not about choosing the most complex tools. It is about making clear, reliable decisions that help teams move faster without adding unnecessary confusion. For early-career developers especially, that was a reassuring and valuable point. Rather than relying on buzzwords or abstract theory, Ron used relatable examples from real projects, which made the session easy to follow and genuinely useful.\n\nHe explored several common challenges developers run into as projects grow, including messy code, unclear ownership, and systems that become harder to maintain over time. What made the talk effective was its focus on small, practical habits instead of overwhelming frameworks or rigid rules. He showed how writing cleaner functions, choosing clearer names, and testing important parts of an application early can prevent bigger problems later.\n\nOne of the strongest parts of the session was Ron’s emphasis on teamwork. He pointed out that even highly capable developers can slow progress when communication breaks down. His advice was straightforward: ask questions early, document key decisions, and write code with the next reader in mind. For beginners, this was an important reminder that software development is not just about making something work. It is also about making that work understandable and sustainable for others.\n\nRon also shared a refreshingly practical approach to learning new technologies. His advice was to start small, build something simple, and improve it step by step. Instead of trying to master every tool or trend at once, he encouraged the audience to pick one concept and apply it in a real project. That approach makes learning more concrete, less intimidating, and far more memorable.\n\nThe atmosphere in the room reflected how well the message landed. People were taking notes, nodding along, and clearly responding to the talk’s direct and grounded style. Ron did not try to impress the audience with complexity. He impressed them by being useful.\n\nBy the end of the session, the core takeaway was unmistakable: strong development habits matter more than flashy shortcuts. That is an encouraging message, especially for beginners. It means you do not need to know everything before you begin. You need to understand the fundamentals, practice them consistently, and build with clarity.\n\nFor attendees leaving the hall in San Jose, Ron Veen’s presentation felt like the kind of session that remains valuable long after a conference ends. It offered practical advice, real-world perspective, and a clear path forward for anyone who wants to become a better developer one step at a time.",
  "feedback": "Strong draft overall: clear, accessible, and well-structured for a beginner audience. The tone is friendly, and the main theme—clarity over complexity—comes through consistently. To improve it, I tightened repetition, made the opening more engaging, varied sentence structure, and gave the piece a slightly more polished blog style. I also reduced generic phrasing and strengthened transitions so the post reads more smoothly from start to finish.",
  "seo": {
    "metaDescription": "Ron Veen’s WeAreDevelopers San Jose 2026 talk shared beginner-friendly advice on clean code, teamwork, and practical software habits.",
    "keywords": [
      "Ron Veen",
      "WeAreDevelopers San Jose 2026",
      "beginner-friendly developer talk",
      "software development",
      "clean code",
      "team communication",
      "developer conference"
    ],
    "slug": "ron-veen-wearedevelopers-san-jose-2026-beginner-friendly-story"
  }
}
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


### Conclusion
I think Embabel looks very promising. The Actions that can be loosely defined and then end up being wired together automagically feels very familiar to Spring developers. If you have followed my [Spring AI series](https://ronveen.com/series/spring-ai-in-depth/) then you might feel that you want to incorporate this into it.
