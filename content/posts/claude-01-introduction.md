---
title: "Claude Code - 01 - Introduction"
date: 2026-05-20
draft: false
tags: ["Claude", "Claude Code", "Anthropic", "AI", "Agent Coding"]
cover:
  image: "/images/Claude-part1-introduction.png"
  alt: "Introduction to Claude Code"
series: ["Claude Code"]
series_order: 1
---
I remember the first time I tried a GitHub Copilot suggestion in IntelliJ IDEA. It felt like magic. You start typing a method name, and suddenly the entire implementation materialises in grey ghost text. You press Tab, and — bam — it's there.

That was impressive. But it was still *you* doing the driving.

In 2026, the landscape has shifted. Claude Code feels different. It's less "autocomplete on steroids" and more "a colleague who sits next to you, holds the entire 1M token context of your codebase in their head, and then just... gets to work." Let me show you what I mean.

Let me show you what I mean.

## What Is Claude Code?

Claude Code is an agentic AI coding assistant that runs directly in your terminal (though a VS Code extension is now available for the IDE-loyalists). Built by Anthropic, it’s powered by the latest **Claude 5 Sonnet** and **Opus 4.7** models. It doesn’t just suggest text; it navigates project structures, understands complex architectural patterns, writes files, executes bash commands, fixes build errors, and iterates until the task is complete.

You interact with it via a simple CLI:

```bash
claude
```

That's it. You're now in an interactive session with an AI that has read-access to your project and is ready to do real work.

## From Autocomplete to Agent

Here's the mental model shift that took me a moment to internalise.

Traditional AI coding tools (Copilot, Tabnine, etc.) work at the **cursor level**. You write, they suggest. It's a tight feedback loop with you firmly in control.

Claude Code works at the **task level**. You describe what you want to achieve, and it figures out the steps. It might read three different files to understand your architecture, write a new service class, update your `pom.xml`, and run your tests — all as part of answering one prompt.

That's what people mean when they say "agentic." It's not just generating text; it's taking actions.

## A Taste of What It Can Do

Let me give you a concrete example. Say you have a Spring Boot application and you want to add a new REST endpoint to expose a list of conferences. Instead of Googling the boilerplate, writing the controller, wiring the service, and defining the DTOs yourself, you just tell Claude Code:

```
Add a REST endpoint GET /conferences that returns a list of ConferenceDto objects. Use our existing MapStruct and IBM MQ patterns.
```

Claude Code will:
1. **Analyze:** Scan your project to locate existing MapStruct mappers and MQ configurations.
2. **Plan:** Present a step-by-step list of the files it intends to create or modify.
3. **Execute:** Create the ConferenceDto, the ConferenceService, and the ConferenceController.
4. **Verify:** It will attempt to run the build. If a test fails, it reads the stack trace and fixes the code automatically before you even see the error.



## How It Actually Works

Claude Code is powered by Anthropic's Claude models (specifically Claude Sonnet for the heavy lifting) and has access to a set of built-in tools:

- **File reading & writing** — it can open, edit, and create any file in your project
- **Bash execution** — it can run shell commands, Maven/Gradle builds, tests, and more
- **Web search** — it can look things up when it's uncertain
- **MCP integration** — it can connect to external tools via the Model Context Protocol
- **Plan Mode** — it created a markdown-based execution plan *before* touching your code
- **Checkpointing** — if Claude messes up a 10-file refactor, your can now roll back the session

This is all opt-in and transparent. Claude Code asks for your confirmation before doing anything destructive, and it shows you exactly what it's doing. You're never flying blind (well, almost never … )

## The CLAUDE.md File: Your Project Briefing

One of my favourite features is the `CLAUDE.md` file. Drop this in the root of your project and Claude Code will read it at the start of every session. Think of it as a briefing document:

```markdown
# My Spring Boot Project

## Build
mvn clean install

## Test
mvn test

## Conventions
- Use MapStruct for DTO mapping
- All controllers go in the `controller` package
- Follow existing naming: XxxController, XxxService, XxxRepository
- IBM MQ is used for messaging — see MessageConfig.java for setup
```

The more context you give it, the better its output. It's one of those tools that rewards the effort you put into setting it up.

# The Power of CLAUDE.md and MCP
One of the most useful features remains the CLAUDE.md file. This acts as a persistent briefing document. But in 2026, this is augmented by **Model Context Protocol (MCP)**.
Claude Code can now connect to your local database schema, your Jira tickets, or your company’s internal documentation via MCP servers. It’s no longer just looking at your code; it’s looking at your entire ecosystem.

# Safeguards: Plan Mode and Checkpoints
Handing over the keys to your terminal can be scary. Anthropic addressed this with two critical features:
* **Plan Mode:** Before any code is changed, Claude presents its intent. You can hit Ctrl+G to enter "steering mode" and correct its logic before it starts typing.
* **Checkpointing & Rewind:** If a refactor goes sideways, you can use the /rewind command to roll back the entire session to a previous state. It’s essentially a multi-file "undo" for AI actions.


## What It's Good At
* **Scaffolding & Plumbing:** Handling the "boring" parts of adding new features.
* **Complex Refactoring:** Changing a pattern across 20 files consistently.
* **Automated Debugging:** Paste a production log, and let it trace the issue through your source.
* **Senior Review:** Commands like /ultrareview trigger a high-reasoning pass (usually via Opus 4.7) to catch logic flaws or "code smells" before you commit.

## What to Watch Out For
* **Confidence isn't always Correctness.** Even with Sonnet 5, Claude can be confidently wrong about a library's latest API. Always have a test suite ready.
* **Cost Management.** While the $20 Pro plan exists, 2026-level agentic workflows can burn through usage limits. Many professional teams now opt for the **Claude Max** or API-based billing to ensure they don't hit "rate-limited" walls mid-sprint.


## Getting Started

Installation is straightforward. You need Node.js 18+, and then:

```bash
npm install -g @anthropic-ai/claude-code
claude
```

On first run, it will walk you through authentication. A Pro subscription to Claude.ai gives you access. For heavy users, there are also Max plans available. Or you can use an Anthropic API key directly, and pay-as-you-go.

Once you're in, run `/help` to see the full list of agentic commands.

## My Honest Take

I’ve been using Claude Code for over a year now, and it has fundamentally changed how I start my workday. It removes the friction of "plumbing"—the boilerplate, the configuration, and the "where is that file?" moments.

It’s not magic, and it’s not a replacement for a developer's judgment. It’s a force multiplier. It lets me spend my "brain cycles" on the hard problems while the agent handles the implementation details.

Give it a try. Start with a refactor you’ve been dreading. And keep that /rewind command in your back pocket.


## This is part 1 of a 10 parts series.

Watch out for the next parts:

**[Part 2 — Configuration](/posts/claude-02-configuration/): Taming Claude Code with CLAUDE.md** How to write an effective CLAUDE.md, token-efficient tips, project vs global scope, and how Claude Code uses it at session start.

**Part 3 — Skills: Teaching Claude Code Your Standards** The SKILL.md architecture, auto-triggered vs explicit skills, writing your own, and best practices for skill design.

**Part 4 — Slash Commands: Your Personal Shortcut Library** Built-in commands, project-scoped vs global commands in ~/.claude/commands/, the {parameter_name} syntax, and practical examples for Java/Spring Boot workflows.

**Part 5 — Custom Commands: Automating Your Workflow** Going deeper — building reusable command templates for things like generating a full REST slice, running a code review, or scaffolding a microservice. Real before/after examples.

**Part 6 — MCP: Connecting Claude Code to the World** What the Model Context Protocol is, how to configure MCP servers, practical integrations (databases, APIs, internal tooling), and security considerations.

**Part 7 — LSP: Claude Code Meets Your IDE's Brain** How Language Server Protocol integration gives Claude Code semantic understanding of your code — go-to-definition, find references, type awareness — and what that means in practice.

**Part 8 — Agents & Subagents: Divide and Conquer** The orchestrator/worker model, when Claude Code spins up subagents, how to design prompts that leverage parallel agents, and real-world use cases for complex multi-step tasks.

**Part 9 — Hooks: Automating the Edges** What hooks are, the available hook points (pre/post tool use, session start/end), writing hook scripts, and practical recipes — auto-formatting, git commits, logging, notifications.

**Part 10 — Putting It All Together: A Production-Ready Claude Code Setup** A capstone article combining CLAUDE.md, skills, custom commands, MCP, and hooks into a complete, opinionated setup for a Java/Spring Boot microservices project.
