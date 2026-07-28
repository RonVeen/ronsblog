---
title: "Claude Code - 10 - Putting It All Together: A Production Setup for a Spring Boot Service"
date: 2026-07-29
draft: true
cover:
  image: "/images/Claude-10-putting-it-all-together.png"
  alt: "Claude 10 - Putting It All Together"
tags: ["Claude Code", "Hooks","AI"]
categories: ["AI"]
series: ["Claude Code"]
series_order: 10
description: "Nine articles in, here's how the pieces fit together. A concrete, opinionated Claude Code setup for a Spring Boot microservice — organised by the concerns it addresses, not the files it uses."
---


There's a specific moment when a Claude Code setup crosses over from *"collection of tricks"* to *"environment I actually trust."* For me it was a morning at work a few weeks after I'd finished the last big round of `.claude/` cleanup. I opened a session on a service I'd been fighting a bug in the day before, typed one sentence about the bug, and Claude Code — before I did anything else — knew what branch I was on, what other work was in progress, what JIRA ticket the bug was tracked in, what the team's transactional conventions were, and which of the four review subagents I'd want to run before committing. I didn't have to explain any of that. The setup did.

The fix took under half an hour. Most of it was actually thinking about the bug. The boring parts — context, style, safety checks, review, audit trail — happened in the background, because the setup handles them without me. That morning wasn't the day the setup started working. It was the day I noticed how much of what I used to do at the start of a session I no longer had to.

That's what this article is about. Nine articles of accumulated setup, and how the pieces fit together to produce a session that starts already ready.

I'm going to walk through the setup by concern rather than by file. There's a reason. A collection of files is a reference; a set of concerns is a design. What matters when you're building your own is understanding *what problem each layer solves* — and, more importantly, *what problem it solves in combination with the other layers*. Almost every payoff in a mature Claude Code setup comes from tools working together, not from tools working alone.

Six concerns cover everything my setup does. Grounding, extension, convention, safety, review, and audit. Each is a section below, with concrete file content and reasoning. The example project is fictional but the setup is close to what I run in practice: **BrightCart's support-desk service**, a Spring Boot 4 microservice handling incoming support tickets, payments for chargeable incidents, and outbound customer notifications. Java-heavy, Spring-heavy, JMS-heavy. If your stack is different, the concerns still apply; only the specific tools change.

At the end there's a *"files at a glance"* section for readers who want the reference view. Everything above it is the design.

## Concern 1: Grounding Claude in the Project

The point of grounding is to eliminate the *"let me tell you about our project first"* phase of every session. When you type your first prompt, Claude Code should already know what the project is, what conventions it follows, what state it's currently in, and what you're actively working on. Nothing you should have to type twice.

Three tools carry this weight in my setup: `CLAUDE.md` for durable context, a `SessionStart` hook for volatile context, and a small set of skills for pattern-recognised context.

**CLAUDE.md: the durable brief.**

Every session opens against this file. It's the closest thing Claude Code has to a project's onboarding document, and it should be treated with the same care. Below is a trimmed version of the BrightCart CLAUDE.md — the real one is around 90 lines, this shows the shape.

```markdown
# BrightCart Support-Desk Service

Spring Boot 4 microservice handling incoming support tickets, chargeable
incident payments, and outbound customer notifications. Deployed to
OpenShift. Owned by the customer-experience platform team.

## Build and test

- `mvn clean verify` — full build with tests
- `mvn compile -o` — offline compile only (used by the compile-on-edit hook)
- `mvn spotless:apply` — format Java sources (run automatically by hook)

## Architecture

- REST controllers under `nl.brightcart.support.web`
- Services under `nl.brightcart.support.service`
- JMS handlers under `nl.brightcart.support.messaging`
- JPA entities under `nl.brightcart.support.domain`
- All outbound HTTP through the `client` package;
  each client has an interface + implementation
- MapStruct for entity ↔ DTO mapping (never hand-written mappers)

## Conventions (non-negotiable)

- Constructor injection only. No `@Autowired` on fields.
- Lombok `@Slf4j` for loggers; `log.info(...)` at method entry.
- `ResponseEntity<T>` return types from every endpoint.
- `@Transactional` only on `public` methods of Spring stereotypes
  (`@Service`, `@Repository`). The transactional-guard hook enforces this;
  do not bypass.
- Java records for DTOs; Lombok `@Value` only where a record cannot be used.
- Every JMS handler must acknowledge only after the DB commit.
  Read `docs/adr-004-message-ack.md` before touching handlers.

## Code intelligence

Prefer LSP over Grep and Glob for code navigation:
- `goToDefinition` / `goToImplementation` for source jumps
- `findReferences` before any rename or signature change
- `workspaceSymbol` for class/method lookup
- `hover` for type info

Use Grep only for text patterns in comments, logs, and config values.
After every edit, check LSP diagnostics before moving on.

## Active work

Current sprint focus: the payment-refund path (JIRA project SUP-4XX).
See sprint board.
```

Two things worth naming.

First, this file is durable. It captures what doesn't change day-to-day — the architecture, the conventions, the code-navigation coaching. It doesn't capture what changes hourly. That's the SessionStart hook's job.

Second, the *"conventions"* section is doing more than it looks like. Every line here is enforced somewhere else in the setup: the constructor-injection rule shows up in the `/spring-controller` command; the `@Transactional` rule shows up in the guard hook; the JMS-ack rule shows up in a subagent's system prompt. The document exists twice — once as prose for Claude to read, once as machinery that enforces the rule. That redundancy is intentional. Prose explains; machinery enforces.

**SessionStart: injecting fresh state.**

Volatile facts — current branch, uncommitted changes, active JIRA ticket — go through a `SessionStart` hook rather than CLAUDE.md. Anything that would be stale within an hour doesn't belong in a checked-in file.

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash $CLAUDE_PROJECT_DIR/.claude/hooks/session-context.sh"
          }
        ]
      }
    ]
  }
}
```

The script emits a short markdown block on stdout, which Claude Code prepends to the initial system context:

```bash
#!/usr/bin/env bash
set -euo pipefail

branch=$(git rev-parse --abbrev-ref HEAD 2>/dev/null || printf 'no branch')
dirty=$(git status --porcelain 2>/dev/null | wc -l)
ticket=$(printf '%s' "$branch" | grep -oE 'SUP-[0-9]+' || printf 'none')

cat <<EOF
## Current session context

- Branch: \`$branch\`
- Uncommitted files: $dirty
- JIRA ticket (from branch name): $ticket
- Started at: $(date -u +%Y-%m-%dT%H:%M:%SZ)
EOF
```

Small, cheap, entirely deterministic. Every session opens with this fresh block visible to Claude. I no longer type *"we're on the payment-refund branch"* — Claude knows.

**Skills: pattern-recognised context.**

The three or four skills I have installed activate automatically when Claude sees the right context. The two that pull the most weight:

- `spring-controller-conventions/` — activates when Claude opens or writes a `*Controller.java` file. Injects the exact controller shape (constructor injection, `ResponseEntity` return types, `@Operation` on each method).
- `jms-message-handler/` — activates on files under `messaging/`. Injects the ack-after-commit rule with a code sketch.

Skills carry the shape of things you write repeatedly. They aren't tutorials Claude reads once; they're context that arrives exactly when it's needed. In practice they mean I stop having to remind Claude of the same three lines in every controller I ask it to write.

**If you had to pick one.** CLAUDE.md. Nothing else in this article works without it. The SessionStart hook and skills are multipliers on CLAUDE.md's value, not substitutes for it.

## Concern 2: Extending What Claude Can See and Do

Grounding gives Claude what your project looks like. Extension gives Claude what your world looks like: the database, the source repository, the internal systems Claude wouldn't otherwise reach. Two mechanisms cover this: MCP for external data and actions, LSP for semantic code intelligence.

**MCP servers: three that pay for themselves.**

Read-only Postgres against the support-desk schema. GitHub for PR and issue work. And BrightCart's internal helpdesk MCP server — a Spring Boot service my team built that exposes ticket search, ticket detail, and (a very restricted set of) ticket-update tools.

`.mcp.json` at the project root:

```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-postgres",
        "postgresql://claude_readonly:${SUPPORT_DB_PASSWORD}@localhost:5432/support_desk"
      ]
    },
    "helpdesk": {
      "type": "http",
      "url": "http://localhost:8090/mcp"
    }
  }
}
```

The Postgres role has `SELECT`-only access on the `support` schema and nothing else. This is a hard constraint enforced at the database, not a soft one enforced by Claude Code being polite. The correct MCP scope pattern is *don't grant what you can't afford to lose*, and read-only access to the schema is what you want for exploration anyway.

GitHub lives in `~/.claude.json` (my personal config, not the project's) because it's authenticated with a PAT that only I have. From Part 6:

```bash
claude mcp add --transport http github \
  https://api.githubcopilot.com/mcp/ \
  --header 'Authorization: Bearer ${GITHUB_MCP_TOKEN}'
```

**What these unlock.** Not *"Claude can now do things"*; that framing is misleading. What these unlock is *Claude can now do them without me leaving the session*. Before MCP, checking a schema meant opening `psql`; checking a PR meant opening a browser; checking a helpdesk ticket meant opening a JIRA-adjacent tool. Now every one of those is a Claude tool call. The interruption cost of context-switching between windows was the actual cost being paid. MCP eliminates that cost.

**LSP: making Claude think in symbols.**

The `jdtls-lsp` plugin from Part 7 is enabled globally. Combined with the *"Code Intelligence"* block in CLAUDE.md, this is what makes rename-refactors and cross-file reasoning actually work.

I won't recap the install here — Part 7 covers it. The important pattern for the capstone is the CLAUDE.md coaching block that makes Claude actually reach for LSP. Without the coaching, LSP is idle. With it, Claude uses `findReferences` before rename-refactors, `hover` for type checks, and diagnostics after every edit. Same installation, order-of-magnitude better output.

**The interaction that emerges.** MCP and LSP together give you *one integrated environment* rather than two. MCP reaches outward (database, GitHub, helpdesk); LSP reaches inward (into the semantic structure of your code). The reader doesn't distinguish between them in practice — they experience a session where Claude *"just knows"* the schema, the code, and the outside world.

**If you had to pick one.** LSP, provided you're on Java or a language with a decent language server. The productivity gain from semantic rename and cross-file reference lookup is larger than any single MCP integration. MCP is a productivity multiplier; LSP is a quality multiplier.

## Concern 3: Encoding Team Conventions

Grounding tells Claude what conventions exist. Encoding is where those conventions become *cheap to invoke* — where *"write a controller our way"* becomes a single command rather than a paragraph. Three mechanisms: skills for automatic application, slash commands for explicit invocation, and subagent definitions for specialised work.

**Skills.** Already covered under grounding. The overlap is intentional — skills serve both the grounding concern (they inject relevant context) and the encoding concern (they encode team conventions). Same tool, two purposes.

**Slash commands.** Three I use daily:

- `/spring-controller` — the full command from Part 5, generating a controller with the team's exact conventions from named parameters.
- `/jms-listener` — a JMS handler with dead-letter-queue support, retry semantics, and the ack-after-commit pattern.
- `/release-notes` — reads the git log since the last tag, groups PRs by area, produces structured markdown.

Slash commands earn their place when they encode work you do repeatedly at a coarse-enough grain that a full paragraph of instructions is worth saving. Anything smaller than that becomes a skill (automatic) or nothing at all (just prompt for it).

The `/spring-controller` command's frontmatter, as a reminder:

```markdown
---
description: Generate a Spring Boot REST controller with constructor injection, Lombok, and ResponseEntity
allowed-tools: Read, Write
argument-hint: controller_name=... entity_name=... dependencies=... base_path=...
model: sonnet
---
```

The `argument-hint` field is doing serious work in a team context — new colleagues see the shape of the command immediately from the autocomplete, without having to open the source file.

**Subagent definitions: the four reviewers.**

From Part 8, the multi-perspective review is the shape of subagent work that has genuinely paid off in my setup. `.claude/agents/` holds four review subagents (security, transactional, error-handling, test-coverage) plus one investigation subagent (`legacy-explorer`) I use when opening an unfamiliar part of the codebase.

The reviewers are triggered via a slash command that spawns all four in parallel:

`.claude/commands/review-service.md`:

```markdown
---
description: Run a full multi-perspective review of a Spring service.
allowed-tools: Task
argument-hint: file_path=/path/to/service.java
---

Run a full multi-perspective review of the service at {file_path}. Spawn
the four review subagents (security-reviewer, transactional-reviewer,
error-handling-reviewer, test-coverage-reviewer) in parallel. Wait for all
to complete, then produce a synthesised report grouped by severity, with
findings deduplicated where two reviewers flagged the same line.
```

Invocation: `/review-service file_path=src/main/java/nl/brightcart/support/service/PaymentService.java`. Two minutes later, the report is on my screen.

**The interaction that emerges.** Skills, commands, and subagents form a spectrum: automatic → invoked → dispatched. Simple context injection happens through skills without me thinking. Small deliberate scaffolding happens through commands with a one-liner. Large multi-perspective work happens through subagent orchestration with a paragraph. Which mechanism you use depends on the work's grain — not on some abstract preference for one over the others.

**If you had to pick one.** Slash commands. Skills are nice but hard to get right; subagents are heavy machinery. A well-written slash command for your team's most repeated task pays for itself in a week.

## Concern 4: Safety and Guardrails

Now the deterministic layer. Grounding, extension, and encoding are all *Claude-decides* mechanisms — the model reads them and chooses to apply them. Safety needs to be *Claude-can't-avoid*: rules that fire whether Claude remembers or not, whether the prompt is careful or careless, whether it's your first session on this project or your five-hundredth.

Four layers cover this in the BrightCart setup.

**Layer 1: MCP scope restrictions.**

Not obvious as a safety layer, but the strongest one. The Postgres role is `SELECT`-only. The helpdesk MCP server's ticket-update tool requires a specific header that Claude Code doesn't send (only humans get it, from a separate interface). The GitHub PAT has read-only public-repo scope. Every capability Claude has via MCP is bounded by the credential Claude was given, not by what Claude decides to do.

This is the pattern to internalise: **the safest capability is the one the model can't call, not the one the model politely declines to call.** If Claude's Postgres role can't `UPDATE`, no CLAUDE.md convention or hook is needed to prevent Claude from `UPDATE`-ing. The database enforces the rule.

**Layer 2: Subagent tool allowlists.**

From Part 8. Every reviewer subagent has `tools: Read, Grep, Glob` and nothing else. They cannot write, cannot shell out, cannot fetch. Their work is bounded by construction.

The one exception is the `test-coverage-reviewer`, which additionally has `Bash(mvn test:*)` — scoped to running the test suite specifically. Same pattern: only the capability the role needs, nothing more.

**Layer 3: PreToolUse hooks.**

Three of the ones from Part 9 are in my setup. Each fires deterministically before Claude can proceed.

- `protect-main-branch.sh` — blocks commits directly to `main`, `master`, or `develop`.
- `transactional-guard.sh` — blocks `@Transactional` applied to private methods or non-Spring classes.
- `appconfig-secret-guard.sh` — blocks writes to `application*.yml` that include inline passwords instead of `${...}` substitution.

Each is a shell script under thirty lines. Together they cover three of the most common footguns in Java Spring work: committing to the wrong branch, misapplying a Spring annotation, leaking credentials into config. Every session gets this protection, regardless of what Claude decides.

**Layer 4: Permission modes on sensitive subagents.**

The reviewers don't need to touch files, so their `permissionMode` is `default` — they'll ask before doing anything unusual. The `test-coverage-reviewer` runs `mvn test`, which is safe enough for `acceptEdits`. Nothing in my setup uses `bypassPermissions`; the ergonomic cost of a few permission prompts is much lower than the risk of a subagent doing something unexpected.

**The interaction that emerges.** Safety isn't one control; it's four independent layers, each of which would be insufficient alone but which together create a robust perimeter. MCP restricts what Claude can do outward. Subagent allowlists restrict what specialised workers can do. Hooks restrict what the main thread can do. Permission modes restrict what happens automatically without a human check. Any single layer failing still leaves three others.

**If you had to pick one.** MCP scope restrictions. The safest capability is the one the model can't call.

## Concern 5: Review and Quality

Grounding, extension, encoding, safety — all of that is about what Claude *can and should do*. Review is about what Claude *did*. It's the last mile before code lands in git.

Three mechanisms carry this weight, all covered in earlier parts.

**Multi-perspective subagent review.** Already covered in Concern 3 above (the `/review-service` command). Two minutes of parallel work by four specialists. Not a substitute for human review, but a first pass that catches the layer of issues a human reviewer would probably catch — on a fast enough loop that it happens before I commit, rather than after I open a PR.

**Post-edit format hook.** `PostToolUse` on `Write|Edit`, scoped to `.java` files, runs `mvn spotless:apply` on the changed file. This is the hook I installed first, the one from Part 9's opener. Nothing exciting; it just runs. My CI has never failed a spotless check since.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [{
          "type": "command",
          "command": "cd $CLAUDE_PROJECT_DIR && bash .claude/hooks/format-java.sh",
          "async": true
        }]
      }
    ]
  }
}
```

Note the `async: true`. The format hook doesn't need to block Claude's next turn — its work is fire-and-forget. Making it async means the session stays responsive even during a burst of edits.

**LSP diagnostics as a stopping condition.** Covered in Part 7. The CLAUDE.md coaching block tells Claude to check LSP diagnostics after every edit. When Claude writes broken code, LSP surfaces the errors *within the same turn*, and Claude fixes them without me noticing. The alternative — Claude confidently reporting *"done"* while three files don't compile — is exactly the failure mode LSP diagnostics prevent.

**The interaction that emerges.** These three work at different distances from the code. LSP diagnostics fire per-edit, catching type errors before the session moves on. The format hook fires per-write, keeping style consistent. The review subagents fire on demand, catching design-level issues before commit. Together they cover three grain sizes of quality control, each too expensive to do manually at that frequency and each cheap when automated.

**If you had to pick one.** LSP diagnostics. Formatting is easy to catch in CI; design review is a slow-loop human activity; but the *"Claude claimed it worked and it doesn't compile"* failure mode is what LSP specifically fixes, and it happens in almost every session without LSP coaching.

## Concern 6: Observability and Audit

The last concern. In a regulated environment — Belastingdienst, a bank, a hospital, anywhere with a real audit obligation — this concern isn't optional. Even outside those environments, being able to answer *"what did Claude actually do?"* is useful for reviewing your own work.

Three mechanisms cover this.

**Per-tool-call audit log.** A `PostToolUse` hook on every tool call appends a structured JSON line to `.claude/audit.log`. Timestamp, session ID, tool name, and a summary of the input. It's not a full transcript — that's what `~/.claude/sessions/<session_id>/transcript.jsonl` is for — but it's a compact record that answers *what tools ran, when, and why*.

```bash
#!/usr/bin/env bash
set -euo pipefail

payload=$(cat)
session_id=$(printf '%s' "$payload" | jq -r '.session_id // ""')
tool_name=$(printf '%s' "$payload" | jq -r '.tool_name // ""')
timestamp=$(date -u +%Y-%m-%dT%H:%M:%SZ)

# One-line JSON per tool call
jq -n \
  --arg ts "$timestamp" \
  --arg sid "$session_id" \
  --arg tn "$tool_name" \
  '{timestamp: $ts, session_id: $sid, tool_name: $tn}' \
  >> "$CLAUDE_PROJECT_DIR/.claude/audit.log"
```

Rotated weekly. Queryable with `jq`. Sufficient for the *"which commits used Claude, which tools, when"* question.

**Git commit attribution.** From Part 9's Example 1. Every commit made via Claude Code gets `Claude-Session-Id` and `Claude-Session-Started` trailers. Combined with the audit log, this means I can answer *"which commits came out of which session, and what tools did that session use?"* — the smallest useful unit of AI-assistance provenance.

**Session-end summary.** A `SessionEnd` hook writes a compact summary line to `~/.claude/session-summary.log`: session ID, duration, number of tool calls, number of `Write`/`Edit` calls, list of files touched. Not for anyone else to read — for me, months later, when I want to answer *"when did I work on the refund flow?"* or *"how much of last quarter's ticket work was AI-assisted?"*

**The interaction that emerges.** These three work at three time horizons. The audit log answers real-time and near-term questions (*"what just happened?"*). Commit trailers answer historical questions on git-log time scale (*"which commits touched the refund flow via Claude?"*). Session summaries answer strategic questions (*"how has our AI use evolved over the quarter?"*). Different tools, different horizons, one continuous record.

**If you had to pick one.** Git commit trailers. They live in a data store you already back up, they're grep-able from any git host's UI, and they're the smallest possible commitment that answers the *"was this AI-assisted?"* question.

## Files at a Glance

For the reader who reached this section and wants the reference view:

```
brightcart-support/
├── CLAUDE.md                                   # team conventions, architecture, LSP coaching
├── .mcp.json                                   # postgres (read-only), helpdesk MCP
├── .claude/
│   ├── settings.json                           # hook config, plugin enable
│   ├── agents/
│   │   ├── security-reviewer.md
│   │   ├── transactional-reviewer.md
│   │   ├── error-handling-reviewer.md
│   │   ├── test-coverage-reviewer.md
│   │   └── legacy-explorer.md
│   ├── commands/
│   │   ├── spring-controller.md
│   │   ├── jms-listener.md
│   │   ├── release-notes.md
│   │   └── review-service.md
│   ├── hooks/
│   │   ├── session-context.sh                  # SessionStart: current branch, ticket
│   │   ├── format-java.sh                      # PostToolUse: spotless:apply
│   │   ├── protect-main-branch.sh              # PreToolUse: block commits to main
│   │   ├── transactional-guard.sh              # PreToolUse: enforce Spring AOP rules
│   │   ├── appconfig-secret-guard.sh           # PreToolUse: no inline passwords
│   │   ├── audit-log.sh                        # PostToolUse: per-tool-call log
│   │   ├── git-attribution.sh                  # PreToolUse: commit trailers
│   │   └── session-summary.sh                  # SessionEnd: personal summary
│   └── skills/
│       ├── spring-controller-conventions/
│       ├── jms-message-handler/
│       └── dto-mapping/
└── ...

~/.claude/
├── settings.json                               # personal: LSP plugin enable, model prefs
└── .claude.json                                # personal: github PAT-based MCP
```

Twenty-seven files across five directories. Two years ago I had none of them. Six months into using Claude Code seriously I had four. The setup grew as specific pains appeared and specific patterns paid off. Not every setup needs to be this size — most start with `CLAUDE.md` and a format hook and add layers as need arises.

---

## Closing

That's ten articles. If I could go back and tell my past self one thing about building this setup, it would be: **start small, add layers when a specific pain shows up, and don't optimise for completeness.** A `CLAUDE.md` and a format hook cover the majority of the value most solo developers will ever see. Everything else is worth adding only when a specific problem justifies it. My setup has grown as it has because I work on a specific kind of service, in a specific team, with a specific set of conventions and constraints. Yours will grow differently.

The ten articles in this series each covered a specific tool. The point of this article was to show what happens when the tools interact. The reason to build a mature Claude Code setup isn't to have a lot of files under `.claude/`. It's to reach the point where sitting down at a session feels less like configuring a tool and more like sitting down at your desk.

Thanks for reading the series. If you build a setup you're proud of — or if there's a piece I got wrong or an interaction I missed — I'd genuinely like to hear about it.

*This is part 10 of a 10-part series on Claude Code.*