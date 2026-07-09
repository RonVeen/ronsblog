---
title: "Claude Code - part 5: Custom Commands, Automate Your Own Workflow"
date: 2026-06-24
draft: false
categories: ["AI", "Developer Tools"]
tags: ["AI", "Claude Code", "Developer Tools", "Spring Boot", "Productivity"]
description: "Stop retyping the same instructions every session. Custom commands let you encode your team's conventions once and invoke them with a single slash."
cover:
  image: images/claude-code-05-custom-commands.png
  alt: Claude Code custom commands
series: ["Claude Code"]
series_order: 5
---

Every team has its boilerplate. At clients, every new REST controller we write follows the same shape: a `@RestController` with a `@RequestMapping`, constructor injection (never field injection), a `@Slf4j` logger, `ResponseEntity` return types, proper HTTP status codes, and a matching service interface already wired in. Oh, and Lombok. Always Lombok.

The first few times I asked Claude Code to generate a controller, I typed all of that out. Every time. A paragraph of instructions just to get the scaffolding right before the interesting work could begin.

After the fourth time, I thought: this is exactly the kind of thing a computer is supposed to remember.

That's custom commands. You write the instructions once, save them as a Markdown file, and from that point on you invoke them with `/your-command-name`. Claude Code reads the file, follows the instructions, and you never type that paragraph again.

## Where Commands Live

Custom commands are just Markdown files in a specific directory. There are two places you can put them, and the distinction matters.

### Project commands: `.claude/commands/`

A `commands` folder inside the `.claude` directory at the root of your project. Commands here are **project-scoped** — they're checked into version control alongside your code and shared with everyone on the team.

This is the right place for commands that encode your team's conventions: your naming patterns, your architectural decisions, the libraries you've standardised on. When a new developer joins the team and opens Claude Code, your project commands are already there.

```
your-project/
├── .claude/
│   └── commands/
│       ├── spring-controller.md
│       ├── test.md
│       └── openapi.md
├── src/
└── pom.xml
```

### Personal commands: `~/.claude/commands/`

A `commands` folder in your home directory's `.claude` folder. Commands here are **global** — available in every project you open, but only on your machine.

This is the right place for commands that reflect your personal workflow rather than team conventions. Things like `/standup` (summarise what I worked on today for the daily standup), `/explain` (give me a detailed breakdown of this code as if I'm new to it), or `/diff-review` (review the current git diff before I commit). These are personal productivity tools that have nothing to do with any specific project's conventions.

A practical rule of thumb: if you'd put it in a team wiki, it goes in the project. If you'd put it in your personal notes, it goes in `~/.claude/commands/`.

## The Anatomy of a Command File

A command file is a Markdown file. The filename becomes the command name — `spring-controller.md` becomes `/spring-controller`. The content is the instruction Claude receives when you invoke it.

At its simplest:

```markdown
Generate a Spring Boot REST controller following our team conventions.
```

But you'll almost always want a frontmatter block at the top:

```markdown
---
description: Generate a Spring Boot REST controller with constructor injection, Lombok, and ResponseEntity
allowed-tools: Read, Write, Bash
---

Generate a Spring Boot REST controller following our team conventions.
```

The `description` shows up in the `/help` menu and in autocomplete — it's how you and your team will remember what the command does six months from now. The `allowed-tools` key pre-approves the tools Claude can use while running this command, so you're not interrupted by permission prompts mid-execution.

## A Closer Look at allowed-tools

The `allowed-tools` key isn't a single on/off switch — it's a list, and for `Bash` specifically, it supports scoping down to individual commands rather than granting blanket access.

**The tool names** map to Claude Code's built-in tools: `Read`, `Write`, `Edit`, `Bash`, `Glob`, `Grep`, and `WebFetch` cover most command use cases. Listing a tool name on its own grants full access to that tool for the duration of the command:

```yaml
allowed-tools: Read, Write, Bash
```

This means Claude can read any file, write any file, and run *any* shell command without prompting you — convenient, but broad.

**Scoping Bash to specific commands.** This is where it gets more useful. Instead of a bare `Bash`, you can restrict it to a pattern:

```yaml
allowed-tools: Bash(mvn test:*), Bash(mvn verify:*)
```

Now Claude can run `mvn test` and `mvn verify` (and anything matching those prefixes) without a permission prompt, but anything else — `rm`, `git push`, `curl` — still asks for your explicit approval. For a command like `/test`, this is the difference between "Claude can run my test suite unattended" and "Claude can run anything it wants unattended."

**Applying this to the three commands in this article:**

```yaml
# spring-controller.md — only touches files, no shell access needed
allowed-tools: Read, Write

# test.md — needs to read the controller and write the test, plus optionally run it
allowed-tools: Read, Write, Bash(mvn test:*)

# openapi.md — only edits an existing file, no new files, no shell
allowed-tools: Read, Edit
```

Notice `openapi.md` uses `Edit` rather than `Write` — it's modifying an existing file in place, not creating a new one. Being specific here isn't pedantry; it's documentation. Anyone on your team reading the frontmatter can tell at a glance exactly what a command is permitted to do, without reading the full instruction body.

**A word of caution.** `allowed-tools` is a convenience, not a sandbox. It tells Claude Code "don't bother asking me about this," not "verify this action is safe." For a personal command in `~/.claude/commands/`, granting broad access is your own call to make. For a project command checked into `.claude/commands/` and shared with the team, treat `allowed-tools` with the same scrutiny you'd give a CI pipeline permission — scope it to exactly what the command needs, and nothing more. A command that only ever generates controllers has no business being granted unrestricted `Bash`.

> ## A Side Note: Other Frontmatter Options

Next to `allowed-tools`, there are a number of other options that can be specified in a command's frontmatter. None of them are required, but a few are worth knowing about.

`argument-hint` shows placeholder text in the autocomplete when someone starts typing your command — purely cosmetic, but useful on a team command nobody but you wrote:

```yaml
argument-hint: controller_name=... entity_name=... dependencies=... base_path=...
```

`model` pins a specific model to the command, overriding whatever's active in the session. Use it both ways: force `opus` on something like `/spring-controller` where careful reasoning pays off, or force `haiku` on something trivial like a commit-message generator where speed matters more than depth.

```yaml
model: claude-opus-4-6
```

`disable-model-invocation` locks a command to manual use only — Claude can never decide to run it on its own. Worth adding to anything with side effects you want full control over, like a hypothetical `/deploy` command. None of the three commands in this article need it, since you're always the one typing them.

```yaml
disable-model-invocation: true
```

And one that isn't frontmatter at all: alongside named parameters, you can use **positional arguments** — `$1`, `$2`, and so on — when the order of inputs is fixed and unambiguous:

```markdown
Review PR #$1 with priority $2.
```

It sits between `$ARGUMENTS` and named parameters in terms of structure: more organised than one undivided blob, but without the self-documenting clarity of `entity_name=Conference`. For a command with two or three genuinely positional, never-confused values, it's a reasonable middle ground.

## Parameters: Making Commands Flexible

A command that always generates the same thing isn't very useful. Parameters let you pass values in at invocation time.

There are two syntaxes. For named parameters, use `{parameter_name}` inside the command body:

```markdown
Generate a Spring Boot REST controller named {controller_name} that manages {entity_name} entities,
with {dependencies} injected via constructor injection.
```

You invoke it like this:

```
/spring-controller controller_name=ConferenceController entity_name=Conference dependencies=ConferenceService,ConferenceRepository
```

For simpler commands that take a single value, `$ARGUMENTS` captures everything you type after the command name:

```markdown
Add OpenAPI/Swagger annotations to the class at $ARGUMENTS.
```

Invoked as:

```
/openapi src/main/java/nl/belastingdienst/controller/ConferenceController.java
```

## $ARGUMENTS vs Named Parameters: Choosing the Right One

Both syntaxes get values into your command, but they solve different problems, and picking the wrong one makes a command either annoying to use or fragile to maintain.

**`$ARGUMENTS` is positional and raw.** Whatever you type after the command name gets dropped in as a single block of text, unparsed. Claude Code doesn't know or care what's inside it — it's your command body's job to make sense of it. This makes it perfect for the simple case: one value, one slot.

```
/openapi src/main/java/nl/belastingdienst/controller/ConferenceController.java
```

Here, `$ARGUMENTS` becomes that whole file path. There's nothing to disambiguate, so there's nothing to name.

But `$ARGUMENTS` falls apart the moment you need more than one piece of information. Suppose you tried to use it for the controller command:

```
/spring-controller ConferenceController Conference ConferenceService /api/conferences
```

Now `$ARGUMENTS` is the string `ConferenceController Conference ConferenceService /api/conferences`, and your command body has to guess which word means what, based on position. Swap two arguments by accident, or have a teammate forget the order, and Claude either misinterprets the command or asks you to clarify — which defeats the purpose of having a one-line shortcut in the first place.

**Named parameters trade brevity for clarity.** Each value is explicitly labelled at the call site:

```
/spring-controller controller_name=ConferenceController entity_name=Conference dependencies=ConferenceService base_path=/api/conferences
```

It's more typing, but it's self-documenting — anyone reading that line (including future-you, three months from now) knows exactly what each value means without opening the command file to check the order. It's also order-independent: `entity_name=Conference controller_name=ConferenceController` works exactly the same.

There's a practical side benefit too: in the command body, `{entity_name}` can be referenced multiple times in different places (the endpoint path, the DTO name, the docstring), while `$ARGUMENTS` only ever gives you the one undivided blob — if you need the same input used three different ways, named parameters are really the only sane option.

**A simple rule that's served me well:** if your command takes one piece of input and there's no ambiguity about what it is — a file path, a class name, a git ref — use `$ARGUMENTS`. The moment you have two or more distinct values that could conceivably be confused with each other, switch to named parameters. The controller command has four; that's an easy call. The `/test` and `/openapi` commands earlier only need a single file path; `$ARGUMENTS` is the obviously better fit there, which is exactly why I used it.

## The Spring Boot Controller Command

Here's the full command I use for generating controllers. This is the one that replaced that paragraph I used to type four times.

**`.claude/commands/spring-controller.md`**

```markdown
---
description: Generate a Spring Boot REST controller with full CRUD, constructor injection, and Lombok
allowed-tools: Read, Write
---

Generate a Spring Boot 3 REST controller with the following specification:

## Controller details
- Controller name: {controller_name}
- Entity/resource managed: {entity_name}
- Dependencies to inject: {dependencies} (comma-separated list of service/repository interfaces)
- Base request mapping: {base_path}

## Required conventions (non-negotiable)
- Annotate with @RestController and @RequestMapping("{base_path}")
- Use @Slf4j from Lombok for logging — add log.info() at the start of each method
- Use constructor injection only — no @Autowired on fields, no field injection
- All constructor parameters must be final
- Return ResponseEntity<T> from every endpoint
- Use proper HTTP status codes: 200 OK, 201 Created (with Location header), 204 No Content, 404 Not Found

## Endpoints to generate
- GET /{base_path} — return all entities (ResponseEntity<List<{entity_name}Dto>>)
- GET /{base_path}/{id} — return one entity or 404 (ResponseEntity<{entity_name}Dto>)
- POST /{base_path} — create entity, return 201 with Location header (ResponseEntity<{entity_name}Dto>)
- PUT /{base_path}/{id} — update entity or 404 (ResponseEntity<{entity_name}Dto>)
- DELETE /{base_path}/{id} — delete entity, return 204 (ResponseEntity<Void>)

## Additional requirements
- Import jakarta.validation.Valid and annotate POST/PUT request bodies with @Valid
- Add @Operation(summary = "...") from SpringDoc OpenAPI on each method (brief, one-line summaries)
- Place the controller in the same package as existing controllers in this project
- After generating the controller, also generate the {entity_name}Dto record if it does not already exist

## What NOT to do
- Do not use @Autowired
- Do not use Optional.get() without a check — use orElseThrow with a meaningful message
- Do not catch generic Exception — use specific exception types
```

Now invoke it:

```
/spring-controller controller_name=ConferenceController entity_name=Conference dependencies=ConferenceService base_path=/api/conferences
```

And here's what Claude Code generates:

```java
package nl.belastingdienst.conference.controller;

import jakarta.validation.Valid;
import lombok.extern.slf4j.Slf4j;
import nl.belastingdienst.conference.dto.ConferenceDto;
import nl.belastingdienst.conference.service.ConferenceService;
import io.swagger.v3.oas.annotations.Operation;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.support.ServletUriComponentsBuilder;

import java.net.URI;
import java.util.List;

@Slf4j
@RestController
@RequestMapping("/api/conferences")
public class ConferenceController {

    private final ConferenceService conferenceService;

    public ConferenceController(final ConferenceService conferenceService) {
        this.conferenceService = conferenceService;
    }

    @Operation(summary = "Retrieve all conferences")
    @GetMapping
    public ResponseEntity<List<ConferenceDto>> getAllConferences() {
        log.info("Retrieving all conferences");
        return ResponseEntity.ok(conferenceService.findAll());
    }

    @Operation(summary = "Retrieve a conference by ID")
    @GetMapping("/{id}")
    public ResponseEntity<ConferenceDto> getConferenceById(@PathVariable Long id) {
        log.info("Retrieving conference with id {}", id);
        return ResponseEntity.ok(conferenceService.findById(id)
                .orElseThrow(() -> new ConferenceNotFoundException("Conference not found with id: " + id)));
    }

    @Operation(summary = "Create a new conference")
    @PostMapping
    public ResponseEntity<ConferenceDto> createConference(@Valid @RequestBody ConferenceDto conferenceDto) {
        log.info("Creating new conference: {}", conferenceDto.name());
        ConferenceDto created = conferenceService.create(conferenceDto);
        URI location = ServletUriComponentsBuilder.fromCurrentRequest()
                .path("/{id}")
                .buildAndExpand(created.id())
                .toUri();
        return ResponseEntity.created(location).body(created);
    }

    @Operation(summary = "Update an existing conference")
    @PutMapping("/{id}")
    public ResponseEntity<ConferenceDto> updateConference(@PathVariable Long id,
                                                          @Valid @RequestBody ConferenceDto conferenceDto) {
        log.info("Updating conference with id {}", id);
        return ResponseEntity.ok(conferenceService.update(id, conferenceDto)
                .orElseThrow(() -> new ConferenceNotFoundException("Conference not found with id: " + id)));
    }

    @Operation(summary = "Delete a conference")
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteConference(@PathVariable Long id) {
        log.info("Deleting conference with id {}", id);
        conferenceService.deleteById(id);
        return ResponseEntity.noContent().build();
    }
}
```

And the DTO record, generated alongside it:

```java
package nl.belastingdienst.conference.dto;

public record ConferenceDto(
        Long id,
        String name,
        String location,
        String startDate,
        String endDate
) {}
```

That's a production-ready controller — matching our team's conventions exactly — from a single command invocation. No boilerplate typing, no forgetting the Location header on the POST, no accidentally using field injection.

## The WebMvcTest Command

Testing is the other place where boilerplate accumulates fast. Here's a command that generates a `@WebMvcTest` for an existing controller.

**`.claude/commands/test.md`**

```markdown
---
description: Generate a @WebMvcTest class for an existing Spring Boot controller
allowed-tools: Read, Write
---

Generate a @WebMvcTest test class for the controller at $ARGUMENTS.

## Steps
1. Read the target controller file to understand its endpoints and dependencies
2. Generate a test class in the corresponding test package (mirror the main package structure)

## Required conventions
- Use @WebMvcTest({ControllerClass}.class)
- Mock all service dependencies with @MockBean
- Use MockMvc with @Autowired
- Write one test method per endpoint: test the happy path and at least one error case (404, 400)
- Use static imports for MockMvcResultMatchers and MockMvcRequestBuilders
- Name test methods descriptively: shouldReturn200WhenConferenceFound(), shouldReturn404WhenConferenceNotFound()
- Use @DisplayName on the class with the controller name
- Assert both the HTTP status code and the response body content using jsonPath()
```

Invoked as:

```
/test src/main/java/nl/belastingdienst/conference/controller/ConferenceController.java
```

Claude Code reads the controller, understands its endpoints, and generates a matching test class — including the right mocks, the right assertions, and named test methods that actually describe what they test.

## The OpenAPI Command

The third command in our project toolkit adds OpenAPI documentation to a class that doesn't have it yet — useful when you're retrofitting documentation onto existing code.

**`.claude/commands/openapi.md`**

```markdown
---
description: Add SpringDoc OpenAPI annotations to an existing REST controller
allowed-tools: Read, Write
---

Add SpringDoc OpenAPI 2.x annotations to the REST controller at $ARGUMENTS.

## What to add
- @Tag(name = "...", description = "...") on the class — derive a sensible name from the controller name
- @Operation(summary = "...", description = "...") on every public endpoint method
- @ApiResponse annotations for each realistic HTTP response code the endpoint can return
  (200, 201, 204, 400, 404, 500 as applicable per endpoint)
- @Parameter(description = "...") on @PathVariable and @RequestParam parameters

## What NOT to change
- Do not modify any existing logic, only add annotations
- Do not reformat code that is not being annotated
- Do not add imports that are already present
```

This one is deliberately narrow in scope. The `What NOT to change` block is just as important as the `What to add` block — it keeps Claude from "helpfully" reformatting code that doesn't need touching.

## Tips for Writing Good Commands

A few things I've learned after building a handful of these:

**Be explicit about what not to do.** Claude Code is eager to help, which sometimes means it does more than you asked. A "do not" section is not defensive — it's precise.

**Mirror your existing code.** The instruction "place the controller in the same package as existing controllers in this project" is more powerful than specifying a hardcoded package name. Claude Code will scan what's there and match it, which means the command works across different projects.

**Don't build the command speculatively.** Pick the single most-typed paragraph from your last week of Claude Code sessions. That's your first command. Build it, use it three times, refine it. A command you've actually needed is worth ten you thought you might need.

**Use `allowed-tools` to avoid permission interruptions.** If your command reads and writes files, add `allowed-tools: Read, Write` to the frontmatter. Without it, Claude Code will prompt for permission on every file operation, which defeats the purpose of automation.

## Where Custom Commands End and Skills Begin

You may have noticed something: the commands above are detailed instructions that Claude reasons through. They're not deterministic — Claude interprets them and applies judgement. That's what makes them powerful.

But there's a related feature called **Skills** that takes this idea further. Where a command is a saved prompt you invoke explicitly, a skill is a set of instructions Claude Code can *choose to invoke automatically* based on what it's working on. Skills also support a richer structure: they can include examples, decision trees, and metadata that helps Claude know when the skill applies.

Part 3 of this series covers skills in detail — including how to build the `SKILL.md` format, when Claude Code triggers a skill without being asked, and how skills and custom commands can complement each other in the same project.

For now, the rule of thumb is simple: if you find yourself invoking a command so often that you wish Claude Code would just *know* to use it, that's a candidate for a skill.

---

*Next up: part 6 covers MCP — the Model Context Protocol — and how to connect Claude Code to external tools like databases, GitHub, and your own internal APIs.*
