---
title: "Claude Code - part 4: Slash Commands"
date: 2026-06-17
draft: false
tags: ["AI", "Claude Code"]
description: "Most Claude Code users only type plain prompts. The ones who get the most out of it know their slash commands. Here's the ones that matter."
cover:
  image: images/Claude-04-slash-commands.png
  alt: Claude Code slash commands
series: ["Claude Code"]
series_order: 4
categories: ["AI"]
---

It was a Friday afternoon and I was in the middle of a long debugging session. I had been going back and forth with Claude Code for maybe forty minutes — adding context, correcting assumptions, exploring three different theories about why a JMS listener was silently dropping messages. The session was getting noisy. Old context from theories I'd already ruled out was still sitting there, cluttering the conversation.

I didn't know about `/compact` yet.

So I did what any reasonable developer does: I closed the terminal, opened a fresh session, and spent the next ten minutes re-explaining everything from scratch. The problem, the stack, the IBM MQ setup, the cipher suite quirks, all of it. Again.

When I later discovered `/compact` — a single command that summarises old context while preserving what matters — I felt the particular flavour of frustration that comes with realising you've been doing something the hard way for weeks.

That's what this post is about. The slash commands that change how you work, and why they exist.

## What Slash Commands Actually Are

When you're inside a Claude Code session, type `/` and an autocomplete menu appears. These aren't prompts — they're controls. Where a regular prompt tells Claude *what to do*, a slash command tells Claude *how to behave*, or performs an action on the session itself.

There are three flavours. **Built-in commands** are hardcoded into the CLI — fast, deterministic, and always available. **Bundled skills** (see the  [previous](/posts/claude-03-skills/) article) also ship with Claude Code but are prompt-based under the hood: they hand Claude a detailed playbook and let it reason its way through the work. And **custom commands** are ones you write yourself — but that's the subject of part 5.

For now, let's focus on the built-ins that actually matter day-to-day.

## Managing Your Context Window

This is where I wish I'd started.

### /clear

Wipes the conversation history and starts fresh. Simple, obvious, and vastly underused. Every time you switch tasks — and I mean *every* time — run `/clear` first. Yesterday's debugging session has no business being in today's feature work. The context you carry forward costs tokens and, more expensively, it can subtly steer Claude toward patterns and assumptions that no longer apply.

Think of `/clear` as a clean desk. You wouldn't start a new project with someone else's papers everywhere.

### /compact

This is the one I wish I'd known on that Friday afternoon. `/compact` summarises the older parts of your conversation while keeping the important bits. You stay in the session — no re-explaining required — but the token count drops significantly.

You can even tell it what to prioritise:

```
/compact retain the error handling patterns and the database schema
```

The general rule of thumb from experienced Claude Code users is: reach for `/compact` when you're at around 80% context usage, and `/clear` when you're switching to a genuinely new task. The `/context` command shows you where you currently stand.

### /context

Shows your current context window usage. It's the equivalent of checking your fuel gauge. Run it when you're in a long session and things start feeling slow or slightly off — Claude's quality tends to degrade as the context fills up, and catching it early is easier than dealing with it later.

## Setting Up a New Project

### /init

This is the command you should run the moment you open Claude Code in a project for the first time. It scans your codebase and auto-generates a `CLAUDE.md` file — the project briefing document that Claude reads at the start of every session.

It picks up your directory structure, detects your build tool, finds your test runner, and drafts conventions based on what it sees. The output isn't perfect, but it's a solid starting point that you then refine over time.

I've talked more about `CLAUDE.md` in [part 2 of this series](/posts/claude-02-configuration/), but the short version is: the more context Claude has about your project upfront, the less time you spend correcting it mid-session. `/init` gets you there in seconds instead of minutes.

### /memory

Opens an editor for your `CLAUDE.md` files directly. Use this when you want to add something that `/init` missed — a naming convention, an architectural decision, a library preference. Whatever you add here persists across sessions.

There's also a shorthand: prefix any message with `#` and it gets added to memory on the spot. Type `# always use MapStruct for DTO mapping` in the middle of a session, and Claude will remember it going forward. Very handy for capturing decisions as you make them.

## Understanding What's Happening

### /cost

Shows token usage and estimated cost for the current session. If you're on API billing rather than a subscription, this is essential. But even on a subscription, it's interesting to see just how token-hungry a long debugging session can get. A typical feature session might cost a few cents. A complex refactor with lots of back-and-forth can surprise you.

### /status

A quick snapshot of your current session: which model is active, what mode you're in, your current configuration. Useful when you've been context-switching between projects and can't remember if you switched models earlier.

## Code Quality Commands

### /review

Runs a code review on your changes. Claude reads the diff and gives you structured feedback — logic errors, edge cases, readability, style. I use this before committing anything significant. It's not a replacement for a human reviewer, but it catches the embarrassing stuff before it reaches one.

### /security-review

Same idea, but focused entirely on security: injection risks, exposed credentials, authentication issues, insecure configurations. If you're working on anything that handles user data, payment flows, or external APIs, make this part of your pre-merge checklist.

## Navigating Time: Rewind and Fork

This is where Claude Code starts to feel genuinely different from a chat interface.

### /rewind

Your undo button. When Claude goes down the wrong path — and it will, occasionally — `/rewind` rolls back the conversation *and* the file changes to an earlier checkpoint. Claude creates implicit checkpoints as you work, so you can step back through them.

The keyboard shortcut `Esc + Esc` opens the same menu and lets you choose whether to rewind code only, conversation only, or both. In my experience, reaching for `Esc + Esc` the moment something feels wrong — rather than waiting for Claude to finish and then arguing it back to the right answer — saves significant time. Rewind and re-prompt. It's almost always faster.

### /branch (or /fork)

Forks your current conversation into a new parallel session. The original is preserved exactly as it is; you switch into the branch to try a different approach. If the branch works out, great. If it doesn't, you close it and you're back where you started.

I use this whenever I'm unsure which of two approaches is better. Rather than committing to one and hoping, I fork, try it, and compare. It feels like having a save point in a video game — you can take risks you otherwise wouldn't.

*(Note: the command was renamed from `/fork` to `/branch` in v2.1.77, but `/fork` still works as an alias.)*

## Taking Control of the Session

### /model

Lists the available models and lets you switch mid-session. In practice, most work happens on **Claude Sonnet** — it's the fast, capable daily driver. But for genuinely hard problems (complex refactors, subtle architectural decisions, tricky debugging), switching to **Claude Opus** is worth the extra tokens. For quick, simple tasks, **Claude Haiku** is fast and cheap.

The ability to switch models without leaving the session is more useful than it sounds. I'll often start a session on Sonnet, hit a hard problem, switch to Opus for that specific question, then switch back. You're paying for reasoning where it counts, not everywhere.

### /effort

Controls how deeply Claude reasons before responding. The levels run from `low` through `medium`, `high`, `xhigh`, and `max`. In practice, `xhigh` is the current default for Opus on coding tasks, and `max` is available for the hardest problems.

The useful pattern here is switching *down* as much as switching up. If you're doing a long agentic task with lots of subagents doing simple work (formatting, renaming, extraction), dropping them to `low` effort can cut token costs significantly without any noticeable quality loss. Save the deep reasoning for the turns that actually need it.

Running `/effort` without an argument opens an interactive slider, which is a nice touch.

### /goal

Sets a session-level completion condition that Claude keeps working toward across multiple turns. Instead of giving Claude one instruction at a time, you describe what "done" looks like and let it drive:

```
/goal All failing tests in the payment module should pass, with no new test skipped
```

While a goal is active, a live overlay shows elapsed time, turn count, and tokens used. You can clear it with `/goal clear` when you want to take back manual control.

This is a command that rewards trust. Give Claude a well-defined goal on a well-defined problem, and you can walk away from the terminal and come back to a finished result.

### /plan

Toggles plan mode. When plan mode is on, Claude proposes what it intends to do *before* touching any files. For big refactors or anything destructive, this is invaluable. You get to read the plan, push back on it, refine it, and only then let Claude execute.

You can also toggle plan mode with `Shift+Tab` without breaking your flow.

### /todos

Shows the current task list Claude is tracking for the session. When you give Claude a multi-step task, it internally breaks it down into steps. `/todos` makes that list visible. It's a useful sanity check — sometimes what Claude thinks it's doing and what you think it's doing are subtly different, and it's better to find that out at step one than step seven.

## Quick Lookups and Feedback

### /btw

One of my favourite additions. `/btw` lets you ask a side question without it becoming part of your main conversation thread:

```
/btw what's the difference between @Transactional(readOnly=true) and a regular @Transactional?
```

Claude answers, and then your context is exactly as it was before you asked. No noise, no drift. It's the equivalent of tapping a colleague on the shoulder for a quick answer without derailing either of your trains of thought.

Use `/btw` liberally. Side questions are free, context-wise. Don't pollute your working session with "wait, how does this work again?" tangents.

### /feedback

Submits feedback or bug reports directly to Anthropic from inside the session. Since v2.1.141, you can attach recent session history (last 24 hours or 7 days) so reports that span multiple sessions include enough context to actually be useful. The alias `/bug` works too.

It's a small thing, but having feedback built directly into the tool rather than requiring a browser tab means you're more likely to actually report the things that frustrate you.

### /insights

Shows analytical insights about your current session — patterns in your prompts, tool usage breakdown, cost distribution, and suggestions for more efficient workflows. Think of it as Claude Code's version of a performance profile for your own working style.

I find it most useful at the end of a long session to understand where the tokens actually went and whether there are habits worth changing.

## One More Thing: /powerup

This one sits slightly apart from the rest, so I'll present it as the bonus it is.

`/powerup` runs you through a series of short interactive lessons — animated demos of Claude Code features you might not have discovered yet. It's clearly aimed at newer users, and the content is a bit basic if you're already deep in the tool. But there's a reason I mention it.

I ran it after about a month of daily Claude Code use, expecting to click through dismissively. I found two commands I hadn't heard of. One of them saved me real time the same week.

Run it once. You might be surprised what's in there.

## A Note on /help

Everything above can be discovered from inside Claude Code itself. Type `/help` and you get the live list of every command available in your current session — built-ins, bundled skills, your custom commands, and any commands that have come in via MCP server connections.

Any cheat sheet, including this post, can go out of date as Claude Code ships updates. `/help` is always current. Make it your first stop when something feels like it should exist but you're not sure of the name.

---

## In Practice

The commands I reach for most, roughly in order:

1. `/clear` — every time I switch tasks
2. `/init` — every time I open a new project
3. `/compact` — when a long session starts to feel bloated
4. `/rewind` — the moment something goes wrong, before it gets worse
5. `/branch` — whenever I'm unsure which approach to take
6. `/review` — before committing anything significant
7. `/plan` — before any large-scale change
8. `/btw` — for every "quick question" that doesn't belong in the main thread
9. `/goal` — for well-defined, hands-off tasks I want to leave running
10. `/model` + `/effort` — together, when I hit something genuinely hard

Start with the first five and you'll already be using Claude Code differently than most people. The rest you'll discover as the need arises — which is, frankly, the best way to learn any tool.

---

*Next up: part 5 covers custom slash commands — how to build your own, the `{parameter_name}` syntax, and the Spring Boot workflows I've automated for daily use at Belastingdienst.*
