---
title: "Claude Code - 07 - LSP: Semantic Search for the Whole Codebase"
date: 2026-07-08
draft: true
cover:
  image: images/Claude-07-mcp.png
  alt: "LSP: Semantic Search for the Whole Codebase"
tags: ["Claude Code", "LSP", "Java", "Developer Tools", "AI"]
categories: ["AI"]
series: ["Claude Code"]
series_order: 7
description: "How Claude Code uses the Language Server Protocol to understand your code the way your IDE does — with Java and Eclipse JDT LS as the worked example."
---
I spent forty-five minutes last week undoing a refactor that should have taken ten.

The change looked simple. Rename `Ticket.getStatus()` to `Ticket.getStatusText()` — we'd added a proper enum-typed `getStatusEnum()` a while back, and the string version needed to be less ambiguous. I asked Claude Code to do the rename across the codebase. Standard stuff.

Claude confidently reported back: *"Renamed 47 references to getStatus across 23 files."* It felt fast. It felt done.

Then Maven exploded. Compilation errors in a class I hadn't thought about. Test failures in a service that didn't even use tickets. And, most confusingly, a `User.getStatusText()` method that had never existed and now did — for reasons I had to reverse-engineer.

The problem was that half the domain classes in the project have a `getStatus()` method. `User`, `Order`, `Payment`, `Notification`, all of them. Grep doesn't know Java. When Claude searched for `getStatus`, it got hundreds of hits across dozens of classes and had to *guess* which ones were `Ticket.getStatus()` and which weren't. It guessed wrong on a handful — enough to introduce compilation errors, enough to accidentally rename methods on classes that had nothing to do with the change.

This is the problem that LSP solves. Grep operates on text. LSP operates on symbols. If Claude Code could ask a Java-aware server *"find me all references to `Ticket.getStatus()`"*, the answer would be exact — not one match too many, not one too few. No guessing.

That's what this article is about.

## What LSP Is, Quickly

LSP stands for **Language Server Protocol**. It's an open standard, originally from Microsoft for VS Code, that describes how a language-aware tool exposes its knowledge to a client. Every major editor speaks it now. Every IDE feature you take for granted uses LSP under the hood: go-to-definition, find-references, hover-for-type-info, the red squigglies that appear as you type, rename-with-safety. All of it is LSP.

The architecture will look familiar if you read Part 6. A client (the editor, or Claude Code) talks to a server (the language server) over a defined protocol. The transport is stdio or a socket. The client sends requests — *"what's the type of this symbol at line 47, column 12?"* — and the server responds with structured answers. MCP borrowed heavily from LSP's design; the two are architectural cousins.

The key difference: MCP servers are usually thin adapters to external systems. LSP servers are heavy, stateful things that maintain a full semantic index of your codebase. Eclipse JDT LS holds an entire Eclipse-style workspace in memory. `gopls` parses and type-checks every file in your Go module. That's why they take real time to start — and why, once running, they can answer questions that grep can only guess at.

## Why Text Search Isn't Enough

Grep is a wonderful tool. It's fast, universal, makes zero assumptions about what it's reading. For finding a comment, a log message, a config value — grep is unbeatable.

But grep doesn't know that `getStatus()` in `Ticket.java` and `getStatus()` in `User.java` are different methods on different classes. It doesn't know that the `save` in `repository.save(ticket)` is `TicketRepository.save`, not `UserRepository.save`. It can't tell that a method call `service.process(input)` is resolving to a specific overload based on the type of `input`. It has no concept of `import` statements renaming a class into scope, so `Config` in one file and `Config` in another are — to grep — the same string.

An AI tool relying on grep for code navigation ends up doing what a new developer does on their first day: reading files, guessing at meanings, building a mental model from context. It works. It's slow. It's error-prone in exactly the ways my rename story showed.

LSP fixes this by giving Claude Code access to the same semantic understanding your IDE has. When Claude asks *"find all references to `Ticket.getStatus()`"*, the language server returns a precise list — every actual usage, no false positives, no false negatives. When Claude asks *"what type does this method return?"*, it gets a real answer, not a text-matched guess.

## What Claude Code Actually Uses LSP For

Once you have LSP wired up, Claude Code gains access to a set of new tools. The ones that matter in practice:

**Go-to-definition** — given a symbol at a specific position, jump to where it's defined. This is the workhorse. Instead of grepping for a method name and reading files to figure out which match is the definition, Claude asks the language server and gets one exact answer.

**Find references** — the inverse: given a symbol, find every place that uses it. This is what would have saved my rename-refactor. Instead of grepping `getStatus` and guessing which matches belong to `Ticket`, Claude asks *"where is `Ticket.getStatus` referenced?"* and gets a precise list.

**Workspace symbol search** — search for a symbol by name across the entire project. Faster and more accurate than glob-plus-grep, especially in codebases with strong naming conventions.

**Hover** — get the type, signature, and documentation for a symbol without opening the file it's defined in. A single tool call replaces "grep, open, read, close, back to task."

**Diagnostics** — errors, warnings, and hints published by the language server, in real time as files change. Every time Claude edits a file, the language server re-analyzes it and pushes back the current diagnostic list. Claude sees the compile errors immediately, not after you notice them. This is the multiplier: it turns Claude's editing loop into a compiler-aware feedback loop.

**Call hierarchy** — incoming and outgoing calls for a method. *"Who calls this?"* and *"what does this call?"*, expanded across the whole project.

**Rename** — semantically-safe rename that updates all references correctly, in one operation. This is the specific tool that would have saved me forty-five minutes.

The pattern: LSP tools are *precision* tools. They give exact answers to questions where grep gives fuzzy ones. But they only work when Claude has something specific to ask about — a symbol name, a file position, a class reference. Grep is still the discovery tool ("is there a class that looks like this?"); LSP is the precision tool for what you've found.

## Getting LSP Working for Java

For Java, the language server is **Eclipse JDT LS** — the same one VS Code, several IntelliJ setups, and most other Java-capable editors use. It's mature, well-maintained, and understands everything you throw at it: Maven, Gradle, multi-module projects, Java 8 through 25.

Setup is three steps: install the language server, install the Claude Code plugin, enable it.

### Step 1: install jdtls

Homebrew on macOS or Linux is the easiest path:

```bash
brew install jdtls
```

On Windows or if you'd rather manage it manually, download the latest release from the Eclipse JDT LS project and extract it somewhere sensible (`~/tools/jdtls` is a decent default). Add the `bin` directory to your PATH.

One requirement worth flagging: modern jdtls versions need **Java 21 or higher** to run themselves — separate from whatever Java version your project targets. If your default `java` is still 17 or 11, you'll need to point jdtls at a newer JDK explicitly. Verify jdtls starts at all:

```bash
jdtls --help
```

If you get help output (be warned, it's enormous), you're set.

### Step 2: install the Java LSP plugin

Claude Code's LSP support works through its plugin system, added in v2.0.74. The official plugin marketplace ships preinstalled and includes an LSP plugin for each supported language, Java among them:

```bash
claude plugin install java-lsp@claude-plugins-official
```

Verify:

```bash
claude plugin list
```

You should see `java-lsp` listed. If it shows as disabled, enable it:

```bash
claude plugin enable java-lsp
```

If you can't find a Java plugin on the official marketplace, the community [Piebald-AI marketplace](https://github.com/Piebald-AI/claude-code-lsps) covers Java as well as Kotlin, Scala, and a long tail of other languages. Add it with `claude plugin marketplace add Piebald-AI/claude-code-lsps`, then install `java-lsp@piebald-ai`. The mechanics are identical.

### Step 3: enable and verify

Open `~/.claude/settings.json` (create it if it doesn't exist) and add — or add to — the `enabledPlugins` block:

```json
{
  "enabledPlugins": {
    "java-lsp@claude-plugins-official": true
  }
}
```

Restart Claude Code. On startup, watch for these lines in the debug log (`claude --debug` if you're on the CLI):

```
LSP server plugin:java-lsp:java initialized
LSP server instance started: plugin:java-lsp:java
```

That's the signal that jdtls has spun up. On a fresh workspace, first-start takes fifteen to sixty seconds because jdtls has to build its initial semantic index. Subsequent starts are much faster.

The most reliable end-to-end test is to ask Claude Code a question that only LSP can answer accurately:

```
In this project, find all references to Ticket.getStatus()
```

If LSP is working, you'll see a `findReferences` tool call in the log with a precise list of results. If you see Claude reaching for `Grep` instead, LSP isn't wired up — or, more likely, it is, but the model isn't being told to prefer it. Which brings us to the most important part.

### Coaching Claude to actually use LSP

Here's the frustrating truth I wish someone had told me earlier: Claude Code's built-in system prompt biases the model toward `Grep` and `Glob` for code navigation. Even with LSP enabled and running perfectly, the model will reach for grep out of habit. Community measurements suggest LSP usage sits at around one to two percent of code-navigation calls in the default configuration. That's not because LSP is broken — it's because the model's defaults haven't been overridden.

The fix is to explicitly override the default via your `CLAUDE.md`. Add this block:

```markdown
## Code Intelligence

Prefer LSP over Grep, Glob, and Read for code navigation:

- Use `goToDefinition` / `goToImplementation` to jump to source,
  not Grep-then-Read.
- Use `findReferences` to see all usages across the codebase.
- Use `workspaceSymbol` to find where a class or method is defined.
- Use `documentSymbol` to list all symbols in a file.
- Use `hover` for type info without opening the file.
- Use `incomingCalls` / `outgoingCalls` for call hierarchy.

Before renaming or changing a method signature, use `findReferences`
to identify all call sites first.

Use Grep and Glob only for text pattern searches (comments, strings,
config values) where LSP does not help.

After writing or editing code, check LSP diagnostics before moving on.
Fix any type errors or missing imports immediately.
```

Drop that in the `CLAUDE.md` at your project root. From here on, Claude has explicit permission — and instruction — to prefer LSP for the tasks it's actually good at. Usage rates jump dramatically once this coaching is in place. This is not optional. Without it, you have LSP installed and idle. With it, you have LSP doing the work.

### Common gotchas

**jdtls not on the PATH.** If you installed manually, the plugin might not find the binary. The plugin's `.lsp.json` config specifies the command it expects; if the actual binary is somewhere else, adjust it or add a symlink into a PATH directory.

**Workspace not detected.** jdtls looks for `pom.xml`, `build.gradle`, or an Eclipse `.project` file to identify the project root. In a monorepo without a build file at the root, it may fail to build the workspace. Launch Claude Code from the specific subproject directory that does have a build file.

**First-start silence.** Because jdtls takes a while to index, Claude Code can appear to fall back to Grep during the first minute of a session even when LSP is enabled. Give it a minute. Ask a symbol-lookup question later in the session and you should see LSP tools being used.

**Old Java version.** If you have Java 17 as your default, jdtls will refuse to start. Either upgrade your default JDK to 21+ or configure jdtls to use a specific JDK via its `-vm` argument.

## Beyond Java: a Quick Aside

The same three-step pattern works for every LSP-supported language: install the language server binary, install the corresponding Claude Code plugin, enable it. The names change; the mechanics don't.

**TypeScript / JavaScript.** The language server is `vtsls` (a fork of `typescript-language-server` with better multi-project support). Install with `npm install -g @vtsls/language-server typescript`. Then `claude plugin install typescript-lsp@claude-plugins-official`.

**Python.** The traditional choice is `pylsp`; the newer `ty` server (from the Astral team, same folks who ship Ruff and uv) is significantly faster and worth trying if you're already on Ruff-based tooling. Either works.

**Go.** `gopls` is the official server and ships with the Go toolchain. If your Go installation is in your PATH, you already have gopls. Install the plugin.

**Rust, C#, Kotlin, Scala, PHP, Ruby, and others** — all covered by the official or Piebald-AI marketplaces. The CLAUDE.md coaching block from the Java section applies identically to every language. Same instructions, same effect.

## Three Small Sessions Worth Trying

The best way to feel the difference LSP makes is to run a few sessions with it on. Three that reveal the pattern nicely.

### 1. A rename-refactor across a Spring Boot project

Pick a domain method with a common name — `getStatus`, `getName`, `save`, `find`. Something you have on multiple classes. Ask Claude Code:

```
Rename Ticket.getStatus to Ticket.getStatusText across the project.
Use LSP findReferences to identify all call sites first, then apply
the rename.
```

Watch the tool-call log. Claude should call `findReferences` on `Ticket.getStatus`, get back a precise list of usages, and apply the rename only at those locations. Compare that to what would have happened with grep — hundreds of matches across similar methods on unrelated classes, half of which would end up wrongly modified.

The proof: run `mvn compile` after. If it builds cleanly, LSP earned its keep. If it doesn't — well, you learned something too.

### 2. "Why is this method being called?"

Pick a method whose call graph you don't fully understand — a utility, a callback, something inherited from an older part of the codebase. Ask:

```
Find all incoming calls to NotificationService.enqueue and give me a
one-line summary of what each caller is doing. Use LSP call hierarchy.
```

Claude uses `incomingCalls` to build the tree, then reads only the calling methods (not entire files) to summarise each caller's intent. In a large project, this is the difference between spending twenty minutes tracing method calls manually and getting a structured summary in a single turn.

### 3. Diagnostic-driven fixing

Deliberately break something in a scratch branch. Change a method signature and don't update its callers, delete an import, rename a class without renaming its usages. Then ask:

```
Check LSP diagnostics across the project and fix the errors that
have appeared. Explain each fix.
```

With LSP running, Claude sees the compiler's view of the codebase — every unresolved reference, every type mismatch, every missing import. It works through the diagnostic list and fixes them in a targeted way. Without LSP, this is the kind of task where Claude spends most of its time re-reading files trying to figure out what's wrong.

This is the pattern that scales into agentic loops. Diagnostics from LSP give Claude a reliable stopping condition: keep editing until the diagnostics are clean. Without LSP, there's no stopping condition — Claude has to guess when it's done.

## The Catch: LSP Isn't Free

Two things worth being honest about.

**LSP servers are hungry.** jdtls in particular is a Java process holding an entire workspace index in memory. For a large Spring Boot project, that's easily 1-2 GB of RAM. If you're already running IntelliJ, VS Code, a browser with fifty tabs, and a Docker daemon, adding jdtls in the background will not be free. On modern developer machines it's fine; on constrained hardware, plan for it.

**Not everything benefits.** For pure text search — finding a config value, a log message, a specific string — grep is genuinely faster and more appropriate than LSP. The two are complementary, not competing. LSP is a precision tool for questions about *code structure*; grep is a discovery tool for questions about *text content*. Getting the CLAUDE.md coaching right means guiding Claude toward LSP for the former and grep for the latter, not blanket-preferring one over the other. Watch the wording of the coaching block above — it explicitly reserves grep for text patterns, comments, and config values.

**The model has to be coached, repeatedly.** I said this above but it's worth saying twice, because it's the thing most people get wrong. Without the CLAUDE.md coaching, LSP is installed and idle. Community reports of "LSP doesn't seem to help" almost always come down to a missing or too-brief coaching block. If your usage numbers look low, don't blame the tool — check the coaching.

Once you have the setup dialled in — jdtls running, plugin enabled, CLAUDE.md coaching in place — the difference is real. My rename-refactor mishap wouldn't have happened with LSP on. Neither would half the *"Claude confidently claimed it was done, and then I found the loose ends"* sessions I've had over the last year. That's the sales pitch: fewer loose ends.

---

That's LSP. In Part 8 we shift gears from Claude Code's environment (MCP, LSP) to Claude Code's *strategy*: **agents and subagents**. How Claude Code decides to spin up a specialised worker for part of a task, when doing so pays off, when it doesn't, and how to design tasks that benefit from the pattern.

*This is part 7 of a 10-part series on Claude Code.*