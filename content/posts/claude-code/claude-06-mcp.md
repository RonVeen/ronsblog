---
title: "Claude Code - 06 - MCP"
date: 2026-07-02
draft: false
description: "Learn how to connect Claude Code to external tools and data sources using the Model Context Protocol, with Postgres as the worked example."
cover:
  image: images/Claude-06-mcp.png
  alt: Claude Code slash commands
series: ["Claude Code"]
series_order: 6
tags: ["Claude Code", "MCP", "Developer Tools", "AI"]
categories: ["AI"]
---

A couple of months ago I was debugging a tricky data issue and I had three windows open. Claude Code on the left. A psql terminal on the right. A schema diagram in a browser tab somewhere behind. The loop went like this: ask Claude what it thought the issue was, copy a query Claude suggested into psql, run it, paste the output back into Claude, repeat.

That's not pair programming. That's being a tape recorder between two systems that should be talking to each other.

What I wanted was for Claude Code to *just look at the database itself*. To run its own queries. To check the schema without me middleman-ing the conversation. And that's exactly what MCP is for.

This is Part 6 of the Claude Code series, and it's the one where Claude Code stops being a clever text generator and starts being a tool that actually reaches into your environment. We'll cover what MCP is, how to plug in a ready-made server (Postgres, as the worked example), a tour of other servers worth knowing about, the OAuth flow for remote servers, and the practical concerns — security, tool overload, and what to do when things don't connect.

## What MCP Actually Is

MCP stands for **Model Context Protocol**. It's an open standard, originally from Anthropic, that defines how an AI client (Claude Code, Cursor, ChatGPT desktop, whatever else) talks to external tools.

Before MCP, every integration was custom made. Want Claude to talk to GitHub? Create a custom connector. Slack? Different custom connector. Your internal API? Another custom one. The community started calling this the N×M problem — N AI clients times M tools means N×M connectors that someone has to write and maintain.

MCP turns that N×M into N+M. Build one MCP server that knows how to talk to GitHub, and *every* MCP-compatible AI client can use it. Build one server for your company's internal helpdesk, and you can plug it into Claude Code today and Cursor tomorrow without changing a line.

A useful mental model: MCP is to AI clients what JDBC is to Java applications. A standard protocol with pluggable drivers. Or, if you prefer the comparison that matters for the rest of this series: MCP is what LSP did for editors, applied to AI tools. Same idea, same shape, different problem domain. LSP is Part 7's topic, and the symmetry is no accident — the MCP designers explicitly borrowed the architecture.

### The vocabulary

Three terms keep coming up. It's worth pinning them down early so the rest of the article makes sense.

A **client** is the AI tool doing the work. Claude Code, in our case.

A **server** is the thing exposing capabilities to the client. The Postgres MCP server exposes database query tools. A GitHub MCP server exposes issue and PR tools. The server is a separate process — it might be a Node.js script downloaded on demand, a Python service, or a Spring Boot application running on its own port.

A **transport** is how the client and server talk. Two transports matter in practice. **stdio** runs the server as a subprocess of Claude Code and communicates over stdin/stdout — simple, fast, perfect for local development. **HTTP** (specifically Streamable HTTP) treats the server as a regular web service, which is what you want for anything that needs to be shared, deployed, or run separately from the client. Most local servers use stdio. Hosted and remote servers use HTTP.

What a server actually exposes are **tools**, **resources**, and **prompts**. Tools are functions Claude can call — "list tickets", "create issue", "run query". Resources are read-only data Claude can fetch — files, docs, database exports. Prompts are reusable prompt templates the server provides. In practice, tools are by far the most common; many servers expose only tools, and that's perfectly fine.

## Plugging in Postgres

Let's start with the simplest possible win: give Claude Code direct read-only access to a Postgres database. Five minutes from zero to a working setup.

### The one-line install

Claude Code ships with an `mcp` subcommand for managing connections. The fastest way to add the official Postgres server:

```bash
claude mcp add postgres -s user \
  -- npx -y @modelcontextprotocol/server-postgres \
  "postgresql://readonly_user:secret@localhost:5432/helpdesk_db"
```

A few things to notice. The `-s user` flag means "install this globally for me, on this machine." We'll come back to the alternatives in a moment. The `--` separates Claude Code's own arguments from the command it'll run as the server. Everything after that is the actual server command — in this case `npx` will download and run the official Postgres MCP server on the fly.

Restart Claude Code, run `/mcp` from inside an interactive session, and you'll see your server listed as connected. The `/mcp` panel is worth getting to know. It shows every configured server, its status (connected, disconnected, pending approval, rejected), the number of tools each one exposes, and — for servers that need authentication — a button to kick off the OAuth flow. It's the dashboard for the whole MCP layer.

### What you can actually do with it

You don't call the tools by name. You just ask questions, and Claude figures out which tool to use:

```
Which tickets in the helpdesk_db have been open for more than 30 days?
```

Claude calls a tool to list the schemas, then one to describe the `tickets` table, then runs a `SELECT` with the right `WHERE` clause and shows you the result. No copy-pasting. No tape recorder.

Where it really starts to earn its keep is in the iterative work. A few examples from real sessions:

```
The bookings table has 14M rows and our nightly export is getting slow.
Look at the indexes and tell me what's missing.
```

Claude reads the table definition, queries `pg_indexes`, and either points out a redundant index or suggests one that would cover your most expensive query. You can ask it to write the migration script. You can ask it to explain why the planner is picking a sequential scan over your existing index. None of that requires you to leave the conversation.

Or, when you're new to a codebase:

```
Walk me through the data model. Start from the most-referenced table.
```

Claude queries the foreign keys, sorts by reference count, and gives you a tour of the schema in order of importance — the kind of thing that would take you an hour with a diagram tool.

### A worked session: hunting an index, drafting a migration

The two-liners above are fine for getting the idea across, but you don't really feel what MCP is doing until you've watched a real session play out. Here's one — close to verbatim, with the noise trimmed.

The setup: the `tickets` table in our `helpdesk_db` has grown to about 8 million rows, and the support dashboard's "open tickets by priority" query is starting to drag. While I'm in there, I also want to add a `resolved_at` column so we can finally calculate resolution times properly. I open Claude Code, confirm the Postgres server is connected via `/mcp`, and ask:

```
The query "SELECT * FROM tickets WHERE status = 'OPEN' AND priority = 'HIGH'
ORDER BY created_at DESC LIMIT 50" is slow. Look at the table, check the
existing indexes, and tell me what's missing.
```

Claude does three things in sequence. First, it asks Postgres for the column layout:

```
[Tool: query("SELECT column_name, data_type FROM information_schema.columns
              WHERE table_name = 'tickets'")]
```

Then it queries `pg_indexes` to see what's already there:

```
[Tool: query("SELECT indexname, indexdef FROM pg_indexes
              WHERE tablename = 'tickets'")]
```

Two indexes come back: the primary key on `id`, and a single-column index on `status`. Finally, it runs `EXPLAIN` on the actual query — without `ANALYZE`, because it doesn't want to actually execute a slow query just to measure it — to see what the planner is doing. The result confirms what it suspected: the planner is using the `status` index, but then sorting every matching row to satisfy the `ORDER BY`, which is where the time goes.

Its recommendation:

> The existing `idx_tickets_status` is being used, but the planner is sorting roughly 1.2M matched rows to get the most recent 50. A composite index on `(status, priority, created_at DESC)` would let the planner walk the index in order and stop after 50 rows. Here's the index:
>
> ```sql
> CREATE INDEX CONCURRENTLY idx_tickets_status_priority_created_at
>     ON tickets (status, priority, created_at DESC);
> ```
>
> I'd recommend `CONCURRENTLY` to avoid locking out writes during the build, since this table is 8M rows. The trade-off is that the migration can't run inside a transaction.

Good. Now the second half of what I wanted:

```
While you're in there, also add a resolved_at TIMESTAMPTZ column to the
tickets table. It should be nullable.
```

This is where the read-only setup pays off. Claude doesn't just blindly run the `ALTER TABLE`. It responds:

> I can't run DDL against this database — the `readonly_user` role we're connected as only has `SELECT` permissions, which is the correct setup for an exploratory MCP connection. What I can do is generate the full migration file for you to review and apply through your normal migration tool.

And then it writes the file:

```sql
-- V20260618__add_resolved_at_and_index_tickets.sql

-- Add resolution-time tracking
ALTER TABLE tickets
    ADD COLUMN resolved_at TIMESTAMPTZ;

-- Composite index for the dashboard's "open by priority, newest first" query
CREATE INDEX CONCURRENTLY idx_tickets_status_priority_created_at
    ON tickets (status, priority, created_at DESC);
```

Notice what just happened. The MCP server gave Claude enough access to *understand* the database — schema, indexes, query plans, row counts — without giving it the access to *change* it. Claude reasoned about the problem with full context, then produced an artifact (the migration file) that goes through your normal review and deployment process. The dangerous step — actually running the DDL against the database — stays in human hands, where it belongs.

This is the pattern that scales. A read-only MCP role gives Claude the visibility it needs to be genuinely useful, while keeping the side effects on the rails. The same pattern translates to other systems: read-only access to your monitoring stack, your CI logs, your ticket system. Claude does the analysis; you (or your normal automation) execute the changes.

### The three scopes

You'll see references to three configuration scopes in the docs, and the difference matters more than it looks.

**User scope** (`-s user`) stores the server in your global config at `~/.claude.json`. Available in every project, only on your machine. This is for personal tools — GitHub, your local Postgres for prototyping, anything you reach for across projects.

**Project scope** (`-s project`) writes the config into `.mcp.json` at the root of the current project. This file is meant to be checked into git. Anyone who clones the repo and runs Claude Code gets the same MCP servers available. This is the right scope for team-shared tools — the project's database, its CI integration, its issue tracker. The first time someone opens a project with a `.mcp.json` they haven't seen before, Claude Code shows the servers as "pending approval" and asks them to confirm — sensible, given the security implications.

**Local scope** (`-s local`) is per-project but *not* checked in, written to `.claude/settings.json`. Useful for the awkward middle case: a project-specific server, but with credentials or paths that shouldn't be shared.

A simple rule that's served me well: if you'd put it in a team wiki, use project scope. If you'd put it in your personal notes, user scope. If you'd put it in a `.env` file, local scope.

The `.mcp.json` file is plain JSON and worth knowing by hand, because that's what your team will read in code review:

```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-postgres",
        "postgresql://readonly_user:secret@localhost:5432/helpdesk_db"
      ]
    }
  }
}
```

## A Tour of Other Servers Worth Knowing

Postgres is a great first example because the value is immediate, but it's hardly the only server worth installing. A short tour of the ones I keep reaching for, and when each one earns its place.

**GitHub** is the most-installed MCP server by a wide margin, and for good reason. It exposes tools for issues, pull requests, branches, commits, file contents, and search. Worth installing if your work touches a public or accessible GitHub repo at all. Claude can read your PR's diff, summarise the discussion, draft a response, or open a new issue — without you ever leaving the terminal.

**Filesystem** sounds redundant — doesn't Claude Code already have file access? — but it's actually for *other* parts of the filesystem. The built-in tools are scoped to your current project; the Filesystem MCP server lets you safely give Claude access to specific other paths. Useful when you want it to read a shared reference doc, a sibling project, or your `~/Documents/architecture-notes` folder, without granting blanket access.

**Brave Search** (or any web search server) gives Claude live web access for a single conversation, independently of whether the built-in search is enabled. Worth knowing about for the times you specifically want results from your own search provider, or want to use a different API quota than the built-in.

**Slack and Linear** belong in the same bucket: team-communication and ticket-tracking integrations. The pattern is the same — Claude can search, summarise, draft, and create entries in either system. Whether they're worth installing depends on whether you actually do that work from inside Claude Code, or whether you'd rather keep that workflow in the dedicated apps. For me, Linear is a clear yes, Slack a hesitant maybe. Your mileage will depend on how you work.

**Playwright** is the one I underestimated initially. It lets Claude drive a real browser — navigate to a URL, click things, fill forms, take screenshots, run assertions. For UI verification during agentic work ("make this change and verify the homepage still renders correctly"), it's the difference between trusting Claude's diff blindly and having it actually check.

None of these are mandatory. The community guidance — which matches my experience — is that five or six well-chosen servers cover most workflows, and a long tail of twenty servers actively hurts. Every connected server is a set of tools Claude has to consider on every turn. More choices means slower decisions and more chance of picking the wrong tool. Prune ruthlessly.

## Remote Servers and OAuth

Everything so far has been local — servers running on your own machine, talking over stdio or to localhost over HTTP. The other half of the MCP ecosystem is **remote servers**: hosted services that you connect to over the public internet.

The configuration looks similar — JSON in `.mcp.json` — but uses a URL instead of a command:

```json
{
  "mcpServers": {
    "linear": {
      "type": "http",
      "url": "https://mcp.linear.app/mcp"
    }
  }
}
```

The difference is authentication. A local Postgres server might use the credentials baked into its connection string. A hosted Linear server uses OAuth, the same way the Linear web app does.

Claude Code handles this gracefully. The first time you try to use a tool from a server that needs auth, it tells you the server requires authentication and points you at `/mcp`. From the `/mcp` panel you trigger the OAuth flow — Claude Code opens a browser tab, you sign in to the provider as you normally would, and the resulting token is stored locally and used for subsequent calls.

The flow is standard OAuth 2.0 with PKCE. The token lives in your local Claude Code config. If it expires, the next call triggers a re-auth automatically. No special configuration on your side beyond the URL.

This matters because the OAuth pattern is what makes managed and hosted MCP servers actually viable. The pattern of "every developer pastes a production credential into a JSON file on their laptop" doesn't scale beyond about five people; OAuth gives you proper per-user identity and revocable access. If you're at a company looking at MCP for the first time, the hosted-server pattern is almost always the better starting point.

## Managing Tool Overload

A practical concern that shows up as soon as you have more than two or three servers connected: tool overload.

Every connected MCP server exposes its tools to Claude on every turn. A Postgres server might expose five tools. A GitHub server might expose twenty. A Kubernetes server might expose thirty. Connect all three and you have fifty-five tools competing for space in the context window — and competing for Claude's attention when it picks which one to call.

Claude Code handles this with an automatic **tool search** mechanism. When the total number of available tools exceeds roughly ten percent of the context window, Claude Code activates tool search. Instead of loading every tool description into the context on every turn, it loads a search index. When Claude needs a tool, it searches the index first, then loads just the relevant tool descriptions on demand.

This is transparent — you don't configure anything. But it's worth knowing about, because it affects how you should think about tool naming.

Clear, descriptive tool names make the search work better. A tool called `list_tickets` with the description "List all helpdesk tickets, optionally filtered by status" gets matched against "show me the open support tickets" far more reliably than a tool called `getData` with the description "Gets data." If you're connecting servers you don't control, you can't do much about this. But if you're building or choosing between alternatives, this is a real quality signal worth weighing.

The other lever is being deliberate about which servers you connect at which scope. The Kubernetes server doesn't need to be in user scope, polluting every project's tool list. Put it in project scope on the projects where you actually do Kubernetes work, and out of everyone else's way.

## Security and Trust

A few words on this, because it doesn't get said often enough.

An MCP server is a process running on your machine, with whatever permissions you give it. The official Postgres server has read-only access by default — that's a property of the server's design, not a guarantee MCP makes for you. Some servers can write. Some can delete. Some can make outbound network calls to anywhere on the internet. "It's in the official registry" is not a substitute for understanding what a server actually does.

A few patterns that hold up:

For databases, create a dedicated role with only the permissions Claude actually needs. For most exploratory use cases, that's `SELECT` on specific schemas, nothing else. The "give Claude full DBA access just in case" approach will end the way you imagine it will.

For credentials, prefer environment variable substitution over inline strings in `.mcp.json`. The same file you'd happily check into git changes character entirely when it contains a production password.

For project-scoped servers checked into git, treat the file with the same scrutiny you'd give a CI pipeline definition. Anyone who clones the repo and runs Claude Code is, by default, going to be prompted to approve those servers — but their first reaction will be to click yes. If your `.mcp.json` says "this server gets your production database connection string from `$PROD_DB_URL`," every developer on the team needs to understand that before approving.

For remote/hosted servers using OAuth, the situation is generally better — the auth boundary is enforced by the provider, not by what's on your laptop. This is one of the reasons the hosted pattern is preferable at company scale, beyond just the credential-sprawl benefit.

## When Things Don't Connect

The first time you set up an MCP server, there's a reasonable chance it won't connect. The three most common reasons, in order:

**JSON syntax errors.** A trailing comma in `.mcp.json`, a missing quote, a stray character. Run the file through `jq` before debugging anything else. This catches at least half of all "MCP server won't connect" reports.

**The command doesn't resolve.** Claude Code spawns the server command as a subprocess. If `npx` isn't on the PATH that Claude Code sees, or if the command requires a tool that isn't installed, the server will silently fail to start. Run the command manually in a terminal, the exact one from your config, and watch the error. Use absolute paths for binaries if you're not sure.

**The server itself crashes on startup.** Run the server manually with the same arguments and watch stderr. This is by far the fastest way to debug — `/mcp` will tell you the server is disconnected, but it won't tell you *why*.

The `claude mcp list` command from outside an interactive session is also useful. It shows every configured server and its current state, including the ones still pending your approval.

## Your Turn: Hands-On with GitHub

Everything up to here you've read along with. Time to actually do it.

GitHub is the best second server to install for a reader who's followed this far: it's the most-used MCP server in the world, the official hosted version handles authentication cleanly via the OAuth flow we covered earlier, and the data you'll work with — public repositories — is the same for everyone. Both exercises below target the [`spring-projects/spring-boot`](https://github.com/spring-projects/spring-boot) repository, so every reader sees the same results and can compare notes.

### Setup

A quick note before we install this one. Earlier in the article I walked through the OAuth flow for hosted MCP servers as the pattern that scales — and in general, that's still true. GitHub's Copilot MCP endpoint, however, doesn't currently implement RFC 7591 Dynamic Client Registration, which is the standard Claude Code uses to negotiate an OAuth client on the fly. Some hosted servers do (Linear, Sentry), some don't (GitHub, at the time of writing). The good news is that the endpoint still accepts a plain bearer token in the `Authorization` header — it doesn't care where the token came from. So for GitHub specifically, we side-step OAuth and use a **Personal Access Token**.

**Create the token.** Go to [github.com/settings/personal-access-tokens](https://github.com/settings/personal-access-tokens) and create a fine-grained PAT with:

- **Resource owner:** your personal account
- **Repository access:** Public Repositories (read-only) — nothing else
- **Expiration:** whatever short window suits you

You're only reading public repos, so this is the smallest possible scope. Copy the generated token.

**Store it in an environment variable, not inline in the config.** Add this to your shell rc file (`~/.zshrc`, `~/.bashrc`, whichever you use):

```bash
export GITHUB_MCP_TOKEN="github_pat_..."
```

Open a new terminal to pick up the change. If you'd rather not have the token sit in a plain-text dotfile, macOS Keychain works nicely as a step up:

```bash
# store once
security add-generic-password -a "$USER" -s github-mcp -w "github_pat_..."

# fetch in shell rc
export GITHUB_MCP_TOKEN="$(security find-generic-password -a "$USER" -s github-mcp -w)"
```

Or use `op read 'op://Private/GitHub MCP/token'` if you're on the 1Password CLI. The exact mechanism matters less than the principle: the raw token doesn't end up sitting in `~/.claude.json`.

**Install the MCP server** with the token passed as a header — and note the *single* quotes on the header value:

```bash
claude mcp add --transport http github \
  https://api.githubcopilot.com/mcp/ \
  --header 'Authorization: Bearer ${GITHUB_MCP_TOKEN}'
```

The single quotes are important. Double quotes would let your shell expand `${GITHUB_MCP_TOKEN}` before Claude Code sees the command, which would write your raw token straight into the config file — exactly what you're trying to avoid. Single quotes keep the placeholder literal, and Claude Code expands it fresh from the environment every time it starts the server.

Verify by inspecting the config:

```bash
grep -A2 '"github"' ~/.claude.json
```

You should see the literal string `${GITHUB_MCP_TOKEN}` in the header value, not your actual token. If the raw token is there, remove the server and re-add it with single quotes.

Restart Claude Code, run `/mcp`, and `github` should show as connected — no OAuth prompt, no browser tab, no authentication step to trigger. Quick sanity check before the real exercises:

```
What's the current open issue count in spring-projects/spring-boot?
```

If you get a sensible number back, you're wired up. If not, jump back to "When Things Don't Connect" — the three failure modes there cover almost everything that goes wrong on first install.

**Before you dive in, two things to expect.**

The first is time. These sessions are long. My run of the virtual-threads exercise below took about seven minutes end to end — and the release-notes exercise can run longer still. That's because each of the twenty-plus tool calls Claude ends up making has to hit GitHub's API over HTTP, and the search-heavy ones aren't quick. It's not a stall, it's just genuinely slow work. Kick the prompt off, watch the tool-call log stream for a moment to confirm progress is happening, then go do something else and come back. Trying to sit and stare at it will only make it feel longer.

The second is permission prompts. You just saw one for the sanity check — Claude Code asks for approval before every tool call by default. Twenty-plus tool calls means twenty-plus prompts, which gets tedious fast — especially if you've stepped away and come back to a session waiting on you for an approval. Each prompt offers options along the lines of "just this once," "always for this tool," and "always for this server." Since every tool exposed by the GitHub MCP server is read-only on public repositories — nothing you approve here can actually change anything on GitHub — "always for this server" is the practical choice for the duration of these exercises. Approve it once, walk away, come back to the finished result. Just remember to reconsider that setting if you ever swap this server out for one that can write.

### Hands-on 1: Feature archaeology

A pattern I find myself using more than expected: tracing a specific feature back through the project that shipped it. *Why does this annotation exist? Which issue did it come from? What was the design discussion? Which release did it land in?* Answering those questions used to mean an hour of clicking through GitHub. With an MCP server, it's a single prompt.

Try this one:

```
In the spring-projects/spring-boot repository, find the original issue
that proposed first-class support for virtual threads. Then find the pull
request that closed it. Summarise what shipped, which Spring Boot version
it landed in, and any notable follow-up issues that came out of the rollout.
```

Watch what Claude does. You'll see it search issues for "virtual threads," read the top-ranking result's description, follow the "Closes #..." or "Resolves #..." link to the merging pull request, fetch that PR's metadata to see which release branch it merged into, and cross-reference recent issues that mention virtual threads to find follow-ups. Twenty-plus tool calls in total — a repo the size of Spring Boot returns a lot of noise that Claude has to sift through, which is where the session length comes from. But it all happens in one terminal: no copy-paste, no browser tabs, no context-switching. The trade-off is time for attention. Minutes on the clock, but almost none of them spent actually driving.

What you should end up with is a short feature-history summary you could genuinely keep — issue link, PR link, release version, follow-up issue summaries. Worth saving as a markdown file. This is the pattern, generalised: any non-trivial framework feature has a paper trail, and an MCP server turns that trail into a five-second query.

If you want to push it further: ask Claude to identify the original issue's author, then to find what else they've contributed to the project. Or ask for a similar trace on a feature you actually care about — Docker Compose support, `@ServiceConnection`, AOT runtime hints. The mechanics are identical; only the search term changes.

### Hands-on 2: Draft release notes from merged PRs

The second exercise mirrors the Postgres session more directly: read-only exploration over a real dataset, producing a structured artifact the reader can save and compare against the source of truth.

Pick a stable Spring Boot release — Spring Boot 4.0.0 is a good target, large enough to have substantial content but stable enough that the PR list won't shift around. Try this prompt:

```
Using only the GitHub MCP server (not the gh CLI or any bash tools), look
at the spring-projects/spring-boot 4.0.0 release. Find the merged pull
requests that shipped in that version, group them by area (web, security,
observability, data, build, testing, dependency-management), and draft me
a structured release-notes section in markdown. One heading per area,
bullet points for the changes, each bullet linking back to its PR.
```

The "using only the GitHub MCP server" clause is deliberate, and worth pausing on. If you have the `gh` command-line tool installed and authenticated on your machine, Claude will often reach for it in preference to the MCP server for bulk GitHub work — a single `gh search prs --limit 100` is faster than the several paginated calls the MCP server needs to do the same job. In real work that's a perfectly reasonable choice. For this exercise it's not: we want to see the MCP server actually doing the work, not have Claude quietly route around it. The explicit constraint keeps Claude on the intended path. More generally: Claude picks between available tools based on efficiency, familiarity, and task shape — its instincts are usually reasonable, but when you want it to use one specific path, saying so plainly is the simplest way to steer it. Applies well beyond `gh` versus MCP.

Claude lists the release, fetches its associated PRs, reads the labels and titles on each, groups them, and writes a markdown document. Save the output. Now compare it against [the actual Spring Boot 4.0 release notes](https://github.com/spring-projects/spring-boot/wiki). Where do they line up? Where does Claude's grouping highlight something the official notes group differently? Where did it miss something?

This is the comparison that teaches the pattern. Claude's output is good — usually quite good — but not authoritative. The reviewable artifact is the deliverable, not the final answer. Same lesson as the Postgres migration file, in a different setting.

If you want to extend the exercise: re-run it for two adjacent point releases (say 4.0.4 and 4.0.5) and ask Claude to produce a diff — what changed in the *kind* of work the project did between those releases. That's a question that's genuinely hard to answer without an MCP server, and trivial with one.

### What both exercises have in common

Notice the shape. In both cases, the MCP server gave Claude *visibility* into a system — read-only access to GitHub's data. Claude did the analysis, sometimes across many tool calls, and produced an artifact — a feature-history note, a draft release-notes document — that you then review. The same pattern as the Postgres session, applied to a system that lives entirely outside your machine.

That symmetry is the point of this article. The transport changes (HTTP instead of stdio), the auth changes (OAuth instead of a connection string), the data shape changes (a REST API instead of a SQL database). But the *pattern* — read-only access, multi-step exploration, reviewable artifact — is identical. Once you internalise that pattern, every new MCP server you install fits the same mould.

## Building Your Own

I deliberately haven't covered building your own MCP server in this article, even though it's the most interesting half of the ecosystem. The right place to learn that — for Java and Spring Boot developers especially — is the Spring AI series I'm developing in parallel. Spring AI 2.0 makes building an MCP server genuinely small: annotate a method with `@McpTool`, expose your tools as a Spring bean, let auto-configuration do the rest. But it's a topic that deserves its own depth, with proper coverage of resources, async progress reporting, transport choice, and authentication — not a section tacked onto the end of an MCP overview.

When that article is up, this one will link to it.

---

That covers MCP from the consumer side. In Part 7 we'll look at LSP — the Language Server Protocol — and how Claude Code uses it to gain deep semantic understanding of your code: go-to-definition, find references, type awareness across files. It's the other half of the "Claude reaches into your environment" story, and the contrast with MCP is illuminating.

