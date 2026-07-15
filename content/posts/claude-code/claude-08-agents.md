---
title: "Claude Code - 08 - Agents & Subagents"
date: 2026-07-15
draft: false
cover:
  image: images/Claude-08-agents.png
  alt: "Agents and subagents: the mental model for when to use them, and when not."
tags: ["Claude Code", "Agents", "Java", "Spring Boot", "AI"]
categories: ["AI"]
series: ["Claude Code"]
series_order: 8
description: "The mental model for subagents, when to use them, plus a section on when you shouldn't."
---

The moment subagents clicked for me was during a code review I'd already once let through.

A colleague had sent through a PR touching one of our JMS message handlers — a service that processes incoming payment-status webhooks from a downstream provider, updates the corresponding ticket, and acknowledges the message back to the broker. Two hundred lines, four external touch-points, and one atomic-looking flow that was actually anything but.

The initial review from a single Claude Code thread came back the way these things often do: *"The handler looks well-structured. Consider extracting the parsing into a helper class. Log messages could be more descriptive."* Fair points. Also, not the kind of review that catches anything you'd be embarrassed by in production.

Something bothered me. This was a handler that touched message authentication, database transactions, external client calls, error handling, and JMS acknowledgement semantics all in one place. Any of those individually would have taken a careful reviewer twenty minutes. The idea that one prompt could hold all five concerns in mind, at real depth, was optimistic — the same optimism that has a single human reviewer skimming the same class in fifteen minutes and rubber-stamping it.

So I tried again, but differently. Four subagents. One focused only on security. One on transactional correctness. One on error handling. One on test coverage. Each with its own narrow system prompt and its own tool restrictions. Running in parallel, each looking at the same handler with a completely different lens.

The result was a review board, not a reviewer. Four independent findings, each specific and defensible. The security agent flagged that message payloads were being trusted without verification of the provider's signature header. The transactional agent flagged a JMS `acknowledge()` that could fire before the database commit, meaning a crash between the two would silently drop the message. Neither of those would have surfaced from a single-thread review, and both are the kind of thing that ruins a Sunday afternoon in production.

The insight wasn't that subagents are magic. It's that four cheap perspectives running in isolation beat one deeper perspective trying to juggle everything at once. That's the shape of work where parallelism actually pays — and the shape where it doesn't is worth naming just as clearly.

This article is about both.

## Agents, and Then Subagents

One bit of vocabulary before we go further. In modern AI, an **agent** is any system where the LLM decides what to do next in a loop — reading state, choosing an action, running it, observing the result, and going again until the job is done or it needs your input. That loop is what separates *"prompt the model, get a response"* from *"give the model a task, let it drive."*

By that definition, Claude Code itself is an agent. Every session you open is one long agent loop: you give it a task, it reads files, edits them, runs commands, observes what happened, and keeps going. When you type into that terminal, you're not chatting. You're delegating.

A **subagent** is what happens when that main agent spawns another one to handle a specialised piece of work. Same underlying LLM, but a fresh instance dispatched internally, with its own context window, its own system prompt, its own tool restrictions, and — optionally — a different model. It does the narrow job you brief it on, reports back to the main agent, and disappears.

The metaphor that has stuck for me: the main agent is the project lead, and subagents are contractors brought in for one narrow job with a written brief and a deliverable. Not colleagues you chat with — contractors you dispatch. The rest of this article is about when to hire them, and when to just do the work yourself.

## The Two Things That Make Subagents Worth the Cost

Subagents are Markdown files in `.claude/agents/` (project-scoped, shared with the team via git) or `~/.claude/agents/` (personal). Each file defines a specialised worker with YAML frontmatter — name, description, tools, model — followed by a system prompt in the body.

But the mechanics matter less than understanding what makes them worth the overhead. Two things:

**Context isolation.** A subagent starts with an empty context window. It doesn't inherit the parent conversation's history. That's the entire point. When you ask a security-focused reviewer to look at a service class, you don't want its judgment coloured by the last three hours of debugging discussion. It sees only the file (or the task brief) and its own instructions. And, symmetrically, whatever the subagent reads and reasons about doesn't pollute the parent thread's context.

**Parallel execution.** The parent thread can spawn several subagents at once, each working independently, each with its own context. When the work genuinely fans out — different files, different concerns, different modules — you get a real speedup and, more importantly, a real quality improvement, because none of the subagents is compromising its attention across topics.

Everything else about subagents — tool restrictions, model choice per role, permission scopes — is in service of those two properties. If a task doesn't benefit from either isolation or parallelism, subagents are the wrong tool. This is important enough to say twice, so I will.



## When Subagents Are the Wrong Tool

Given the article's mission — being deliberate about when subagents earn their cost — this section belongs before the examples, not after them. Subagents are the most misused feature in Claude Code, and the misuse follows a small number of predictable patterns.

**Trivial tasks that touch shared assumptions.** Adding a field to a domain object. Small mechanical changes across a handful of files. Anything a mid-career developer would do in ten minutes with an IDE and the right muscle memory. The overhead of spawning a subagent, briefing it, waiting for it, reading its report, and reconciling its output with other subagents' output is only worth paying when the work is either genuinely large or genuinely independent. Neither is true for *"add a field."*

**Sequential dependencies.** If step B needs the output of step A, and step C needs the output of step B, you don't have parallel work. You have a pipeline. Trying to parallelise it means paying the coordination overhead of subagents while getting none of the benefits — because the parent thread is going to end up serialising them anyway.

**Small codebases where isolation buys nothing.** If your entire project fits comfortably in one Claude Code session's context, context isolation isn't solving a problem. A single well-prompted main thread does the same work with less overhead and less coordination risk.

**"Just to be thorough" subagents that duplicate the main thread's work.** Spawning a generic *"reviewer"* subagent that reads the same files the main thread just wrote doesn't give you a second perspective — it gives you the same model looking at the same input, priced twice. If you want a different perspective, the subagent must have a different lens: different tools, different system prompt, different constraints. Same model with a different attitude counts. Same model with the same attitude does not.

**Any task a skill or a slash command would handle.** Skills (Part 3) apply automatically based on context. Slash commands (Parts 4 and 5) are cheap and deterministic. Subagents are heavy. If the same result can come from a well-written skill or a one-line command, use those instead. Save subagents for the tasks where the isolation and parallelism actually pay for themselves.

That's the "when not." Now the "when."

## Example 1: Multi-Perspective Code Review

This is the shape where subagents genuinely earn their keep, and the one I reach for most often.

The setup: a colleague has written a `TicketPaymentService` — a 400-line Spring `@Service` that handles the "customer pays for a support-desk incident" flow at BrightCart. Multiple concerns are entangled in there. Security: who is allowed to trigger a payment? Transactional correctness: what happens if the external payment call succeeds but the ticket-status update fails? Error handling: network errors, insufficient funds, downstream timeouts. Test coverage: are the failure paths actually exercised, or only the happy path? Any single reviewer looking at this under time pressure catches one or two of those concerns and misses others. Ask them to look at all four at once and quality drops across the board.

The pattern is to define four specialist reviewers, each with a narrow focus, run them in parallel, and synthesise their findings.

Here is one of them in full. This lives at `.claude/agents/security-reviewer.md` so the whole team can use it:

```yaml
---
name: security-reviewer
description: Reviews Spring services for security concerns. Use for services handling authorisation, external calls, or user input.
tools: Read, Grep, Glob
model: sonnet
---

You are a senior application security engineer specialising in Spring Boot.

Your job is to review the target class for security issues, and only for
security issues. Do not comment on style, structure, error handling, or
testing. Other reviewers handle those.

Focus on:
- Missing or incorrect authorisation checks (@PreAuthorize, @Secured,
  method-level security).
- Injection risks in JPQL, native SQL, and shell or process calls.
- Sensitive data leaking into logs (payment details, PII, tokens).
- Trust of caller-supplied data without validation.
- Outbound calls to external services without authentication, or with
  credentials in code.

Output format:
- One-paragraph summary of what the class does.
- A list of findings, each labelled CRITICAL, HIGH, or INFO.
- Each finding includes: file:line, one-sentence description, one-sentence
  recommendation.

If you find no issues at a given severity, say so explicitly. Do not pad.
```

Notice the specific choices. `tools: Read, Grep, Glob` — no `Write`, no `Bash`, no MCP servers. This subagent cannot modify anything, only read and report. `model: sonnet` — cost-effective and capable enough for structured review; no point paying for Opus when the output shape is bounded. The system prompt is deliberately narrow, and the *"do not comment on"* clause is doing critical work: it enforces the specialisation. Without that line, a Sonnet-driven reviewer will drift into style comments within three findings.

The other three subagents follow the same shape with different lenses:

- **transactional-reviewer** — focuses on `@Transactional` placement, propagation, rollback rules, JPA session management, and cross-service transactional boundaries. Same tools, same model, different system prompt.
- **error-handling-reviewer** — focuses on exception hierarchies, retry semantics, circuit-breaker gaps, silent failures, and error responses returned to callers.
- **test-coverage-reviewer** — focuses on coverage of edge cases, error paths, and boundary conditions in the corresponding `*Test.java` files. This one also gets `Bash` in its tool list so it can run `mvn test -Dtest=TicketPaymentServiceTest` and read the report.

Now, from the main thread:

```
Run a full multi-perspective review of TicketPaymentService.java. Use the
four review subagents in parallel and give me a synthesised report grouped
by severity, with duplicates merged where two reviewers flagged the same
line.
```

Claude spawns all four in parallel. Each reads only what it needs. Each returns a structured report. The main thread merges them into a single ranked list, deduplicates the overlapping findings (the security-reviewer and error-handling-reviewer will often both flag missing exception handling on external calls, for different reasons), and hands you back one document.

This is the shape where the two properties from the previous section pay off explicitly. Each subagent's context contains only the target class and its own system prompt — no distraction, no cross-pollination. Four different perspectives are applied genuinely independently. Wall-clock time is the time of the slowest reviewer, not the sum of all four. And you get a report that reads like it came from a small review board, not from a single reviewer trying to hold four concerns in mind at once and losing detail with each new concern loaded.

Imagine doing this same review with a single main-thread session. You'd prompt through each concern serially. Each new prompt has to carry the context of the previous concerns. By the fourth concern, the model is trying to hold all four in mind while looking at the code, and quality drops. That specific failure mode is what subagents fix.

## Anatomy of a Subagent File

Here is a quick reference for what actually goes in that YAML frontmatter. Only two fields are required (`name` and `description`); the rest are optional but each one earns its place.

| Option | Values | Description |
|--------|--------|-------------|
| `name` | kebab-case string | Required. The subagent's identifier, and how the main agent refers to it. Must match the filename (minus `.md`). Used in explicit invocations like *"use the security-reviewer subagent"*. |
| `description` | short prose sentence | Required, and more consequential than it looks. This is the field the main agent reads to decide when to delegate automatically. Write it in the form *"Use for X..."* to steer that decision. A vague description means the subagent sits unused. |
| `tools` | comma-separated list, or `"*"` | Allowlist of tools this subagent can use. Typical values: `Read`, `Grep`, `Glob`, `Write`, `Edit`, `Bash`, `WebFetch`. Omit or use `"*"` to inherit the parent's full tool set. Restricting reviewers to `Read, Grep, Glob` makes them read-only by construction — a strong safety pattern. |
| `disallowedTools` | comma-separated list | Denylist counterpart to `tools`. Useful when you want to inherit most tools but block one dangerous one, e.g. `disallowedTools: Bash` for a subagent that should never shell out. |
| `model` | `sonnet`, `opus`, `haiku`, or specific model ID | Which Claude model runs this subagent. Pick per role: `haiku` for bulk work over many small files, `sonnet` for structured review, `opus` for hard reasoning. Defaults to the session's active model. |
| `mcpServers` | list of MCP server names | Restrict which MCP servers this subagent can talk to. Useful when one subagent should query Postgres but not GitHub, for example. Omit to allow all connected servers. |
| `permissionMode` | `default`, `acceptEdits`, `plan`, `bypassPermissions` | How the subagent handles permission prompts during its own loop. `acceptEdits` is common for automated fix-up subagents; `plan` forces plan-only mode; `bypassPermissions` skips the safety layer entirely — use with care and only for subagents whose blast radius you're certain of. |
| `maxTurns` | integer | Caps the number of turns in the subagent's own loop before it must return. A guard against runaway investigations. `20` is a reasonable default for open-ended investigation subagents; leave it off for bounded tasks. |
| `color` | color name or hex | Cosmetic — shows up in the terminal UI while this subagent is running. Useful for scanability when three or four subagents are active in parallel. |

The two required fields do the heavy lifting; everything else is about narrowing what the subagent can do and choosing how it does it. The pattern that scales: keep `tools` as tight as possible, pick `model` deliberately per role, and treat `description` as the marketing copy that decides whether the main agent ever hires this subagent at all.

## Example 2: Parallel Investigation of a Legacy Codebase

The setup: you've been asked to add SSO support to a Spring Boot service you didn't write. Fifteen years old. Four generations of authentication mechanisms, at least two of them partially deprecated. A mix of `@PreAuthorize` annotations, servlet filters, and one custom `HandlerInterceptor` that nobody wants to talk about. Before you touch anything, you need to understand how authentication currently works — and *"just read it"* is a week of your life.

This is context-isolation's ideal case. You want fresh eyes on four different aspects of the codebase, none of them distracting the others, and none of them dumping their findings into your main thread's context before you're ready to synthesise.

You don't need custom subagent files for this. Ad-hoc subagents spawned from the main thread work well when the investigation is one-off. The main thread dispatches four narrow briefs:

```
I need to understand how authentication currently works in this Spring
Boot service before I add SSO support. Spawn four investigation subagents
in parallel, each with a narrow brief.

Subagent 1 — Security configuration:
Read the SecurityConfig class and everything it directly imports.
Summarise the authentication chain: the filter order, any custom filters,
and how the AuthenticationManager is composed. Do not read anything
outside the security configuration.

Subagent 2 — Controller protection:
Read every @Controller and @RestController class in this project. For
each protected method, report the mechanism (annotation, filter, none).
Explicitly list any endpoints that appear to have no protection.

Subagent 3 — Configuration files:
Read all application*.yml and application*.properties files. Report
every authentication-related setting: OAuth clients, JWT settings,
session settings, CORS rules, and any active profiles that change them.

Subagent 4 — Outbound clients:
Read the outbound service clients in the `client` package. Report which
downstream services require authentication and how credentials are
supplied — inline, via config, via a shared credential provider.

Wait for all four to complete, then synthesise their findings into a
single "How authentication works in this service" document. Explicitly
highlight any discrepancies between the reports.
```

Four subagents, four narrow briefs, running in parallel. Each returns a structured summary. The main thread synthesises them into a single *"here is how authentication currently works"* document, which is the artifact you actually need to plan the SSO integration.

The critical property here isn't parallel speed. It's that each subagent's context is uncontaminated. The controller subagent isn't influenced by what the security-config subagent read; each forms its own picture. When you synthesise, you get four independent views, and the discrepancies between them — a controller that thinks it's protected while the security config says otherwise — are often the single most valuable finding from the whole investigation.

## Example 3: Framework Upgrade Impact Analysis

The setup: a multi-module Maven project — seven modules, mostly independent responsibilities — is being upgraded from Spring Boot 3.x to Spring Boot 4.0. The breaking changes are documented in Anthropic-sized release notes, but the impact on your specific code depends entirely on which APIs each module actually uses.

This is textbook parallelism. Seven modules, seven investigations, each independent of the others.

The pattern:

```
Spawn one investigation subagent per module in the /modules directory.
Each subagent should:

1. Read the module's pom.xml, main source, and test source.
2. Cross-reference against the Spring Boot 4.0 migration guide (which is
   already open in the WebFetch cache).
3. Produce a report of what will break, what needs manual review, and what
   should just work — grouped by risk level.

Wait for all subagents to complete, then produce a unified upgrade plan
grouped by module and risk level.
```

Seven parallel investigations. Each subagent has full context on its own module and no context on any other — which is exactly right, because in a well-designed multi-module project, modules don't depend on each other's internals. When they all complete, the main thread has seven structured reports to weave into a single plan.

The plan itself isn't the point. The point is that producing this plan in a single-thread session would take hours of context-switching, with high risk of missing something because module 3's findings crowded out module 6's from the main context by the time module 7 was being investigated. Parallel isolation solves both problems simultaneously.

One small refinement worth mentioning. If some of the modules are trivial (a small config module, a shared-types module) and others are complex (the main business-logic module), it's worth varying the model per subagent. Route the trivial modules to Haiku and the complex ones to Sonnet or Opus. Match the reasoning budget to the actual difficulty of each subtask, rather than paying Opus rates for a module that has three classes.

## The Real Cost

Two numbers to internalise.

**Roughly seven times the tokens.** A subagent-heavy workflow can use around seven times the tokens of a single-thread session for the same work, because each subagent maintains its own context and produces its own summary. This is the community consensus figure across 2026; treat it as ballpark, not gospel, and check `/cost` after a subagent-heavy session to calibrate against your own workflows.

**Session length.** Spawning three subagents and waiting for them to complete takes as long as the slowest one. If one of them decides to investigate a rabbit hole, the parent thread is idle. This is fine when the work being done is worth the wait. It's not fine when you're waiting three minutes to get a summary that you could have produced yourself in one.

There's also a soft cost worth naming. Every subagent definition committed to `.claude/agents/` becomes part of your team's shared surface area. When a new colleague opens the project, those subagents auto-load. If your project accumulates fifteen subagents that were useful once and never rediscovered, they cost startup latency and description-matching noise on every session. Prune them the same way you'd prune dead code. `claude` treats them as loaded state, but they're really documentation with a runtime cost.

The takeaway is simple. Subagents are heavy machinery. They pay off spectacularly when the shape of the work matches their strengths, and they waste real time and money when it doesn't. That's why the "when not" section is longer than usual for this series. The wrong answer to *"should I use a subagent for this?"* is almost always *"yes, why not."* The right answer starts with understanding what makes them worth their overhead — context isolation, and parallel execution — and asking honestly whether the task in front of you actually benefits from either one.

If the answer is no, don't reach for the heavy tool.

---

That's agents and subagents. In Part 9 we shift to **hooks** — the smaller, quieter, deterministic cousin of the agent pattern. Where subagents let Claude reason its way through a specialised task, hooks let you attach guaranteed pre- and post-tool actions: run a linter after every edit, block a commit if secrets are detected, log every bash command to a file. Different tool, different job, and in some cases the right answer where a subagent looks tempting.
