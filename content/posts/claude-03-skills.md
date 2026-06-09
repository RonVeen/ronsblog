---
title: "Claude Code - 03 - Skills"
date: 2026-05-22
draft: false
tags: ["Claude", "Claude Code", "Anthropic", "AI", "Agent Coding", "Skills"]
cover:
  image: "images/Claude-03-skills.png"
  alt: "Using skills with Claude Code"
series: ["Claude Code"]
series_order: 3
categories: ["ai"]
---
# Claude Code — Teaching It Your Standards with Skills

In [Part 2](https://ronveen.com/posts/claude-02-claude-md/), we turned `CLAUDE.md` into a precision instrument. We wrote down our conventions, our architecture, our rules — and watched Claude Code go from producing generic boilerplate to code that could survive a code review.

But here's the thing I noticed after a few weeks of working this way: my `CLAUDE.md` was getting fat.

It started small. Build commands, naming conventions, a few architecture rules. Then I added a section on how to generate a complete REST slice. Then testing patterns. Then a procedure for running database migrations with Flyway. Before I knew it, the file was 300 lines long, and every single line was loaded into every single session — whether Claude needed it or not.

That's when I discovered Skills.

## What Are Skills?

A skill is a standalone set of instructions that Claude Code loads *on demand*. You create a directory with a `SKILL.md` file in it, and Claude Code adds it to its toolkit. When a task matches the skill's description, Claude loads it automatically. Or you can invoke it explicitly by typing `/skill-name`.

If `CLAUDE.md` is the employee handbook you hand to the new hire on day one, a skill is a laminated procedure card you pull out of the drawer when you need it. The handbook stays on the desk; the procedure card goes back in the drawer when you're done.

Here's the key difference: **CLAUDE.md is "always-on." Skills are "on-demand."**

## Skills vs CLAUDE.md: When to Use Which

This is the question I get most often, so let me draw a clear line.

**Use CLAUDE.md for facts** — things Claude needs to know in every session, regardless of what task you give it. Build commands. Package structure. Naming conventions. The kind of stuff a new team member needs on day one, before they've been assigned any work.

**Use skills for procedures** — multi-step workflows that Claude only needs when performing a specific task. Generating a REST slice. Running a code review. Scaffolding a new microservice. Setting up Testcontainers.

If you've worked with Spring, the analogy is straightforward. `CLAUDE.md` is like `application.yml` — loaded at startup, always in memory. A skill is like a `@Component` with `@Lazy` — it exists, Spring knows about it, but it's only instantiated when something actually needs it.

Here's a quick decision table:

| If it's… | Put it in… | Why |
| --- | --- | --- |
| A build command | CLAUDE.md | Needed every session |
| A naming convention | CLAUDE.md | Applies to all code |
| A 10-step procedure for generating a REST endpoint | Skill | Only needed for that task |
| A testing checklist with Testcontainers setup | Skill | Only needed when writing tests |
| "Use constructor injection" | CLAUDE.md | One-line rule, always relevant |
| "Here's how to set up a Flyway migration with rollback" | Skill | Multi-step procedure, rarely needed |

The rule of thumb: if it's a fact or a one-liner, it belongs in `CLAUDE.md`. If it's a procedure or a checklist, it belongs in a skill.

## The Token Budget Trick

Here's the part that made me move half of my `CLAUDE.md` into skills.

When Claude Code starts a session, it loads the **name and description** of every installed skill into its context. Descriptions are capped at 1,536 characters each — so maybe 300–400 tokens per skill. Not nothing, but a fraction of a full procedure. The full body of the `SKILL.md`? It stays on disk until Claude actually invokes the skill.

Contrast that with `CLAUDE.md`, where *everything* loads at startup. Every line. Every turn. Every session. Remember the "token tax" from Part 2?

Let me put numbers on this. Say you have a 50-line procedure for generating a REST slice in your `CLAUDE.md`. That's maybe 500 tokens. Over a 40-turn session, you're paying 20,000 tokens for a procedure you might use once.

Move it to a skill, and you pay a few hundred tokens for the description in every session, plus the full 500 tokens only in the session where you actually use it. In the sessions where you don't invoke the skill — which is most of them — you save the full procedure cost.

This is why the official guidance from Anthropic says: if a section of your `CLAUDE.md` has grown into a procedure rather than a fact, move it to a skill.

## Anatomy of a SKILL.md

Every skill lives in its own directory and has a `SKILL.md` file as its entrypoint. The file has two parts: YAML frontmatter between `---` markers, and markdown content with the actual instructions.

Here's the simplest possible skill:

```
.claude/skills/hello/
└── SKILL.md
```

```yaml
---
description: Say hello in a project-appropriate way.
---

Greet the user and list the top 3 most recently modified files in the project.
```

That's it. A description (so Claude knows when to use it) and instructions (so Claude knows what to do). The directory name — `hello` — becomes the command: `/hello`.

Let's look at a more realistic example. Here's a skill for generating a REST endpoint in our conference service:

```yaml
---
name: rest-slice
description: >
  Generate a complete REST endpoint slice for a Spring Boot entity.
  Use when asked to create a new endpoint, add CRUD operations,
  or scaffold a REST API for an entity. Includes controller, service,
  repository, DTOs, mapper, and tests.
allowed-tools: Bash(mvn *) Read Write Edit
---

# REST Slice Generator

## Procedure

1. **Identify the entity**: Determine the entity name from the user's request
2. **Check existing patterns**: Read an existing controller to match the project style
3. **Generate the slice**:
   - Entity class with JPA annotations in `entity` package
   - `XxxRequest` record (with Bean Validation) in `dto` package
   - `XxxResponse` record in `dto` package
   - `XxxMapper` interface (MapStruct, `componentModel = "spring"`) in `mapper` package
   - `XxxRepository` (extends `JpaRepository<Xxx, UUID>`) in `repository` package
   - `XxxService` with constructor injection in `service` package
   - `XxxController` with `@Timed` annotations in `controller` package
4. **Security**: GET endpoints are public. POST/PUT/DELETE require `@PreAuthorize("isAuthenticated()")`
5. **Tests**:
   - `XxxServiceTest` with JUnit 5 + Mockito
   - `XxxControllerTest` with @WebMvcTest, including 401 and 400 test cases
6. **Verify**: Run `mvn test` and fix any failures before reporting done
```

Let's break down the frontmatter fields.

### The Frontmatter

**`name`** — Display name. Optional; defaults to the directory name. Use lowercase, hyphens, and numbers only.

**`description`** — This is the most important field. It's not documentation — it's a *trigger*. Claude does fuzzy matching against this string when deciding whether to load the skill. A vague description means the skill silently never fires. Put the key use case first, be specific, and include the verbs a user would naturally say.

```yaml
# Bad — too vague
description: Help with REST APIs

# Good — specific, includes trigger phrases
description: >
  Generate a complete REST endpoint slice for a Spring Boot entity.
  Use when asked to create a new endpoint, add CRUD operations,
  or scaffold a REST API for an entity.
```

**`allowed-tools`** — Tools Claude can use without asking you for permission while this skill is active. Think of it like a mini permission grant. For a REST generation skill, you want Claude to be able to read files, write files, and run Maven without pestering you for approval on each step.

**`disable-model-invocation`** — Set to `true` for skills you want to invoke manually with `/skill-name`. Use this for anything with side effects: deploying, committing, sending messages. You don't want Claude deciding to deploy because your code "looks ready."

**`context`** — Set to `fork` to run the skill in an isolated subagent. The skill gets its own context window, doesn't see your conversation history, and reports back with a summary. Great for expensive, focused tasks like a deep code review.

**`model`** — Override which model runs the skill. You might want your `code-review` skill to use Opus for deep reasoning, while a `format-imports` skill can get by with Haiku.

**`effort`** — Set the reasoning effort level. Options: `low`, `medium`, `high`, `xhigh`, `max`. Pair this with `model` to get cheap-and-fast for trivial skills or deep-and-thorough for critical ones.

**`paths`** — Glob patterns that limit when the skill activates. Like path-scoped rules in `.claude/rules/`, but for skills. A skill with `paths: "src/test/**"` only triggers when Claude is working with test files.

Here's the full set in one glance:

| Field | Purpose | Example |
| --- | --- | --- |
| `name` | Display name (max 64 chars) | `rest-slice` |
| `description` | Trigger text (critical!) | "Generate a complete REST endpoint…" |
| `when_to_use` | Extra trigger context (appended to description) | "Trigger phrases: scaffold, new endpoint" |
| `allowed-tools` | Auto-approved tools | `Bash(mvn *) Read Write Edit` |
| `disable-model-invocation` | Manual-only | `true` |
| `context` | Run in subagent | `fork` |
| `agent` | Which subagent | `Explore`, `Plan`, or custom |
| `model` | Model override | `opus`, `sonnet`, `haiku` |
| `effort` | Reasoning depth | `high`, `max` |
| `paths` | File pattern trigger | `src/main/java/**/controller/**` |
| `user-invocable` | Hide from `/` menu | `false` |
| `hooks` | Lifecycle hooks | See Part 9 |

### The Body

The body is just markdown. Write it like a procedure for a colleague — what to do, in what order, and what to check.

A few guidelines:

**Be procedural, not encyclopaedic.** The body should say "do X, then Y, then Z." If you find yourself writing paragraphs of background knowledge, that belongs in a reference file (we'll get to that).

**Keep it under 500 lines.** Once a skill loads, its content stays in context for the rest of the session. Every line is a recurring token cost — the same tax problem as `CLAUDE.md`, just localised to sessions where the skill is active.

**Include verification steps.** Every good skill ends with "run the build" or "run the tests." Claude Code can iterate on its own mistakes if you give it a way to check its work.

## Where Skills Live

Just like `CLAUDE.md`, skills have a scope hierarchy:

| Scope | Location | Applies to |
| --- | --- | --- |
| **Enterprise** | Managed settings | All users in your org |
| **Personal** | `~/.claude/skills/<skill-name>/SKILL.md` | All your projects |
| **Project** | `.claude/skills/<skill-name>/SKILL.md` | This project only |
| **Plugin** | `<plugin>/skills/<skill-name>/SKILL.md` | Where the plugin is enabled |

When skills share the same name across levels, enterprise overrides personal, and personal overrides project. Plugin skills use a `plugin-name:skill-name` namespace, so they can't conflict.

For Java developers, here's a practical split:

**Personal skills** (`~/.claude/skills/`): things that follow you across projects. A `code-review` skill with your personal checklist. A `git-commit` skill that formats messages the way you like them.

**Project skills** (`.claude/skills/`): things your team agrees on. The `rest-slice` generator that follows *this* project's patterns. A `flyway-migration` skill that knows your database setup. These get committed to version control.

## Supporting Files: Keeping SKILL.md Lean

A skill doesn't have to be a single file. The directory can contain reference docs, templates, example outputs, even scripts — and Claude only loads them when it needs them.

```
.claude/skills/rest-slice/
├── SKILL.md                  # Main instructions (required)
├── references/
│   ├── testing-patterns.md   # Detailed testing recipes
│   └── security-config.md    # Security setup reference
├── templates/
│   └── controller.md         # Template for a new controller
└── scripts/
    └── validate-slice.sh     # Script to verify the slice is complete
```

You reference these from your `SKILL.md`:

```markdown
## Additional Resources
- For testing patterns including Testcontainers, see [testing-patterns.md](references/testing-patterns.md)
- For security configuration details, see [security-config.md](references/security-config.md)
```

The beauty is that `SKILL.md` stays focused on the *procedure*, while the reference files provide the *depth*. Claude loads the references only when it actually needs them — for instance, when it's about to write a test and wants to check the Testcontainers pattern.

This is the equivalent of having a short runbook on the wall and a thick operations manual on the shelf. You follow the runbook for the happy path. You pull the manual when something's tricky.

## Dynamic Context Injection: The !`command` Trick

This is one of my favourite features, and it's wildly underused.

Inside a `SKILL.md`, you can write `` !`command` `` at the start of a line. Claude Code runs the command *before* the skill content is sent to Claude, and replaces the placeholder with the command's output. Claude never sees the command — only the result.

Why is this useful? Because it lets your skill inject *live project state* into Claude's context.

Here's an example. Say you're building a skill that generates a new REST endpoint, and you want Claude to know exactly what dependencies are available in your project:

```yaml
---
name: rest-slice
description: Generate a complete REST endpoint slice for a Spring Boot entity.
---

## Current Project Dependencies

!`mvn dependency:tree -DoutputType=text 2>/dev/null | head -60`

## Current Entity Examples

!`find src/main/java -name "*Entity.java" -exec basename {} \; 2>/dev/null`

## Procedure

1. Review the dependencies above to determine what's available
2. Check the existing entities for naming and annotation patterns
3. Generate the slice following existing conventions
...
```

When you invoke `/rest-slice`, Claude Code runs `mvn dependency:tree` and the `find` command first. By the time Claude sees the skill, the placeholders have been replaced with actual output. Claude knows you have MapStruct 1.5.5 and Spring Security 6.4, not because you wrote it in a static file, but because it was read from your live `pom.xml` moments ago.

For multi-line commands, use a fenced code block opened with `` ```! ``:

````markdown
## Current Test Configuration

```!
cat src/test/resources/application-test.yml 2>/dev/null || echo "No test config found"
```
````

One caveat: the `!` must be at the start of a line or immediately after whitespace. If it follows another character, it's treated as literal text. And substitution runs once — command output isn't re-scanned for further placeholders.

## Progressive Example: From Ad-Hoc to Production-Grade

Time for the fun part. Let's continue our Conference Service from Part 2 and add a Speaker entity. We'll see how skills progressively improve the output, just like `CLAUDE.md` did in the previous article.

### Round 1: No Skill — Just a Prompt

You're in a session. Your `CLAUDE.md` from Part 2 is loaded (conventions, naming, security, observability rules). You type:

```
Generate a complete REST slice for a Speaker entity. 
A speaker has a name, bio, company, and a link to their photo.
Include full CRUD and tests.
```

Claude Code reads your `CLAUDE.md`, picks up the conventions, and produces a reasonable slice. You get a `SpeakerController`, `SpeakerService`, `SpeakerRepository`, DTOs as records, MapStruct mapper, `@Timed` annotations, `@PreAuthorize` on mutating endpoints — the works.

But here's what's missing:

- **No Testcontainers.** The repository tests use `@DataJpaTest` with an embedded H2, even though your production database is PostgreSQL. The tests pass, but they're testing a different database engine.
- **No Flyway migration.** Claude created the entity with JPA annotations, but didn't generate a `V*__create_speaker_table.sql` migration file. Your team uses Flyway; schema generation from JPA annotations is disabled.
- **No OpenAPI documentation.** Your API is documented with Springdoc, but there are no `@Operation` or `@Schema` annotations on the controller or DTOs.
- **Inconsistent error messages.** The validation messages on the request DTO don't match the style used elsewhere in the project.

The code compiles. The tests pass. But it wouldn't get through code review — not because the *conventions* are wrong (the `CLAUDE.md` handled those), but because the *procedure* is incomplete. Claude didn't know the full checklist.

### Round 2: A Basic Skill — Defining the Procedure

Now let's create a skill. Drop this into `.claude/skills/rest-slice/SKILL.md`:

```yaml
---
name: rest-slice
description: >
  Generate a complete REST endpoint slice for a Spring Boot entity.
  Use when asked to create a new endpoint, add CRUD operations,
  or scaffold a REST API for an entity. Includes controller, service,
  repository, DTOs, mapper, migrations, and tests.
allowed-tools: Bash(mvn *) Read Write Edit Glob Grep
---

# REST Slice Generator

## Step 1: Analyse
- Identify the entity name and fields from the user's request
- Read one existing controller and service to match project patterns
- Check the current Flyway migration numbering in `src/main/resources/db/migration/`

## Step 2: Generate Entity & Migration
- Entity: JPA annotations, UUID primary key, `@Table` with explicit name
- Flyway migration: `V{next_number}__create_{entity}_table.sql`
- Column types must match between JPA annotations and Flyway DDL

## Step 3: Generate DTOs
- `XxxRequest` record with Bean Validation annotations
- `XxxResponse` record
- Validation messages must follow existing pattern: "{Field} is required"

## Step 4: Generate Mapper, Repository, Service
- MapStruct mapper (`componentModel = "spring"`)
- Repository extends `JpaRepository<Xxx, UUID>`
- Service with constructor injection

## Step 5: Generate Controller
- `@Timed` on every method: `api.{entity}.{operation}`
- GET endpoints: public
- POST/PUT/DELETE: `@PreAuthorize("isAuthenticated()")`
- Add `@Operation` and `@ApiResponse` annotations (Springdoc)
- Add `@Schema` annotations on DTOs

## Step 6: Generate Tests
- `XxxServiceTest`: JUnit 5 + Mockito
- `XxxControllerTest`: @WebMvcTest with MockMvc
  - Happy path for each endpoint
  - 401 for unauthenticated POST/PUT/DELETE
  - 400 for invalid request bodies
- `XxxRepositoryTest`: @DataJpaTest with Testcontainers (PostgreSQL)

## Step 7: Verify
- Run `mvn test` and fix any failures
- Verify Flyway migration applies cleanly
- Confirm all test cases pass
```

Same prompt:
```
Generate a complete REST slice for a Speaker entity.
A speaker has a name, bio, company, and a link to their photo.
Include full CRUD and tests.
```

Claude Code detects the match (the description mentions "REST endpoint slice" and "scaffold a REST API"), loads the skill, and now follows the procedure step by step.

The difference is visible immediately:

**Flyway migration appears:**
```sql
-- V4__create_speaker_table.sql
CREATE TABLE speakers (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(255) NOT NULL,
    bio         TEXT,
    company     VARCHAR(255),
    photo_url   VARCHAR(512),
    created_at  TIMESTAMP NOT NULL DEFAULT now(),
    updated_at  TIMESTAMP NOT NULL DEFAULT now()
);
```

**OpenAPI annotations are on the controller:**
```java
@Operation(summary = "List all speakers", 
           description = "Returns all speakers registered for conferences")
@ApiResponse(responseCode = "200", description = "Speakers retrieved successfully")
@GetMapping
@Timed(value = "api.speakers.list", description = "List all speakers")
public List<SpeakerResponse> getAllSpeakers() {
    return speakerService.getAllSpeakers();
}
```

**And `@Schema` annotations on the DTOs:**
```java
@Schema(description = "Request payload for creating or updating a speaker")
public record SpeakerRequest(
    @NotBlank(message = "Name is required")
    @Schema(description = "Speaker's full name", example = "Venkat Subramaniam")
    String name,

    @Size(max = 2000, message = "Bio must not exceed 2000 characters")
    @Schema(description = "Speaker biography", example = "Award-winning author...")
    String bio,

    @Schema(description = "Speaker's company or affiliation", example = "Agile Developer, Inc.")
    String company,

    @Schema(description = "URL to speaker's photo", example = "https://example.com/photo.jpg")
    @Pattern(regexp = "^https?://.*", message = "Photo URL must be a valid HTTP(S) URL")
    String photoUrl
) {}
```

**Testcontainers in the repository test:**
```java
@DataJpaTest
@Testcontainers
class SpeakerRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres = 
        new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private SpeakerRepository speakerRepository;

    @Test
    void shouldSaveAndRetrieveSpeaker() {
        Speaker speaker = new Speaker();
        speaker.setName("Venkat Subramaniam");
        speaker.setCompany("Agile Developer, Inc.");

        Speaker saved = speakerRepository.save(speaker);

        assertThat(speakerRepository.findById(saved.getId()))
            .isPresent()
            .hasValueSatisfying(s -> {
                assertThat(s.getName()).isEqualTo("Venkat Subramaniam");
                assertThat(s.getCompany()).isEqualTo("Agile Developer, Inc.");
            });
    }
}
```

**Consistent validation messages:**
```java
@NotBlank(message = "Name is required")       // matches project pattern
@Size(max = 2000, message = "Bio must not exceed 2000 characters")
```

That's a massive improvement over Round 1. The skill didn't change the conventions (those still come from `CLAUDE.md`). It changed the *completeness* of the output. Claude now follows a checklist instead of improvising.

### Round 3: The Enhanced Skill — Dynamic Context and Reference Files

Let's push it further. We'll add dynamic context injection and a reference file for testing patterns.

First, add a testing reference file at `.claude/skills/rest-slice/references/testing-patterns.md`:

```markdown
# Testing Patterns for Conference Service

## Repository Tests (Testcontainers)
- Use `PostgreSQLContainer` with `postgres:16-alpine`
- Use `@DynamicPropertySource` for datasource config
- Apply `@Testcontainers` and `@Container` annotations
- Test CRUD operations + custom query methods
- Test unique constraints and not-null violations

## Controller Tests (@WebMvcTest)
- Import `SecurityMockMvcRequestPostProcessors.jwt()` for authenticated requests
- Test matrix:
  | Endpoint | Auth Required | Happy Path | Validation | Not Found |
  | -------- | ------------- | ---------- | ---------- | --------- |
  | GET /    | No            | 200 + list | N/A        | N/A       |
  | GET /{id}| No            | 200        | N/A        | 404       |
  | POST /   | Yes           | 201        | 400        | N/A       |
  | PUT /{id}| Yes           | 200        | 400        | 404       |
  | DELETE /{id}| Yes        | 204        | N/A        | 404       |
- Unauthenticated POST/PUT/DELETE must return 401

## Service Tests (Mockito)
- Mock repository and mapper
- Test happy path and exception cases
- Verify interactions: ensure mapper is called, repository is used
```

Now update the `SKILL.md` to use dynamic context injection and reference the file:

```yaml
---
name: rest-slice
description: >
  Generate a complete REST endpoint slice for a Spring Boot entity.
  Use when asked to create a new endpoint, add CRUD operations,
  or scaffold a REST API for an entity. Includes controller, service,
  repository, DTOs, mapper, migrations, and tests.
allowed-tools: Bash(mvn *) Read Write Edit Glob Grep
---

# REST Slice Generator

## Live Project Context

### Current Flyway Migrations
!`ls src/main/resources/db/migration/ 2>/dev/null || echo "No migrations found"`

### Existing Entities (for pattern matching)
!`find src/main/java -name "*Entity.java" -o -name "*.java" -path "*/entity/*" 2>/dev/null | head -10`

### Available Dependencies
!`mvn dependency:tree -DoutputType=text 2>/dev/null | grep -E "(mapstruct|springdoc|testcontainers|flyway|spring-security)" | head -15`

## Procedure

### Step 1: Analyse
- Identify the entity name and fields from the user's request
- Read one existing entity and one existing controller to match patterns
- Note the latest migration version number from the list above
- Confirm MapStruct, Springdoc, Testcontainers, and Flyway are in the dependency tree above

### Step 2: Generate Entity & Migration
- Entity: JPA annotations, UUID primary key, `@Table` with explicit name
- Include `createdAt` and `updatedAt` timestamps with `@CreationTimestamp` and `@UpdateTimestamp`
- Flyway migration: `V{next_number}__create_{entity}_table.sql`
- Column types must match between JPA annotations and Flyway DDL exactly

### Step 3: Generate DTOs
- `XxxRequest` record with Bean Validation
- `XxxResponse` record
- Add `@Schema` annotations on both (Springdoc)
- Validation messages follow pattern: "{Field} is required", "{Field} must not exceed N characters"

### Step 4: Generate Mapper, Repository, Service
- MapStruct mapper (`componentModel = "spring"`)
- Repository extends `JpaRepository<Xxx, UUID>`
- Service with constructor injection — handle `EntityNotFoundException` for not-found cases

### Step 5: Generate Controller
- `@Timed` on every method
- GET: public. POST/PUT/DELETE: `@PreAuthorize("isAuthenticated()")`
- `@Operation` and `@ApiResponse` annotations on every endpoint
- Return `ResponseEntity` with appropriate status codes (200, 201, 204, 404)

### Step 6: Generate Tests
- Follow the patterns in [testing-patterns.md](references/testing-patterns.md)
- Repository tests with Testcontainers (PostgreSQL 16)
- Controller tests with full test matrix (see reference)
- Service tests with Mockito

### Step 7: Verify
- Run `mvn test` and fix any failures
- Verify Flyway migration applies cleanly against Testcontainers PostgreSQL
- Confirm test count: minimum 12 test cases for full CRUD
```

Same prompt, one more time:
```
Generate a complete REST slice for a Speaker entity.
A speaker has a name, bio, company, and a link to their photo.
Include full CRUD and tests.
```

Now before Claude even sees the instructions, the `!` commands have already run. The skill arrives in Claude's context with the actual migration file listing (so it knows the next version number), the actual entity files (so it can pattern-match exactly), and the actual dependency tree (so it knows which versions of Testcontainers, MapStruct, and Springdoc are available).

The reference file gives Claude a complete test matrix. No guessing, no improvising. The controller test now has the full set:

```java
@WebMvcTest(SpeakerController.class)
class SpeakerControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private SpeakerService speakerService;

    // GET /          → 200 (list)
    // GET /{id}      → 200 (found), 404 (not found)
    // POST /         → 201 (valid + auth), 401 (no auth), 400 (invalid body)
    // PUT /{id}      → 200 (valid + auth), 401 (no auth), 404 (not found)
    // DELETE /{id}   → 204 (valid + auth), 401 (no auth), 404 (not found)

    @Test
    void createSpeaker_withValidAuth_returns201() throws Exception {
        when(speakerService.createSpeaker(any())).thenReturn(sampleResponse());
        mockMvc.perform(post("/api/speakers")
                .with(jwt())
                .contentType(MediaType.APPLICATION_JSON)
                .content(validSpeakerJson()))
            .andExpect(status().isCreated());
    }

    @Test
    void createSpeaker_withoutAuth_returns401() throws Exception {
        mockMvc.perform(post("/api/speakers")
                .contentType(MediaType.APPLICATION_JSON)
                .content(validSpeakerJson()))
            .andExpect(status().isUnauthorized());
    }

    // ... 10 more tests following the matrix above
}
```

Twelve test cases in total, covering the full matrix. Happy paths, auth failures, validation errors, and not-found cases for every endpoint that supports them.

The entity now includes timestamps:
```java
@CreationTimestamp
@Column(name = "created_at", nullable = false, updatable = false)
private Instant createdAt;

@UpdateTimestamp
@Column(name = "updated_at", nullable = false)
private Instant updatedAt;
```

And the Flyway migration exactly matches the entity definition, right down to the column types — because Claude could see both the existing migrations and the existing entities *before* it started writing.

Let's recap the progression:

| | Round 1 (no skill) | Round 2 (basic skill) | Round 3 (enhanced skill) |
| --- | --- | --- | --- |
| **Conventions followed** | Yes (from CLAUDE.md) | Yes | Yes |
| **Flyway migration** | Missing | Present | Present, version auto-detected |
| **OpenAPI annotations** | Missing | Present | Present |
| **Testcontainers** | Missing (used H2) | Present | Present |
| **Test coverage** | Partial (3 tests) | Good (8 tests) | Full matrix (12+ tests) |
| **Pattern matching** | Guessed patterns | Followed procedure | Read actual project files |
| **Validation messages** | Inconsistent | Consistent | Consistent |

## Controlling Who Triggers a Skill

By default, both you and Claude can invoke any skill. But not every skill should be triggered automatically.

For the `rest-slice` skill? Automatic triggering is perfect. When you say "add a Speaker endpoint," Claude should pick it up.

But what about a `deploy` skill? Or a `force-push` skill? Or a skill that sends a Slack notification? You absolutely do *not* want Claude deciding to run those on its own.

Two frontmatter fields give you control:

**`disable-model-invocation: true`** — Only you can invoke the skill. Claude doesn't even see the description. Use for anything with side effects.

```yaml
---
name: deploy
description: Deploy the application to the staging environment
disable-model-invocation: true
allowed-tools: Bash(*)
---
```

**`user-invocable: false`** — Only Claude can invoke the skill. It doesn't appear in the `/` menu. Use for background knowledge that Claude should pick up automatically but users shouldn't invoke directly.

```yaml
---
name: legacy-billing-context
description: >
  Background context about the legacy billing module.
  Use when working with files in src/main/java/**/billing/**.
user-invocable: false
paths: "src/main/java/**/billing/**"
---
```

The second example is a nice pattern: a skill that only loads when Claude is editing billing code, and provides context about the legacy system's quirks. Users don't need to invoke it — Claude picks it up automatically when it matters.

## Tips From the Trenches

**Start with one skill.** Don't try to skill-ify everything at once. Pick the procedure you repeat most often — for most Java teams, that's "generate a REST endpoint" or "write tests for this class" — and start there.

**Write the description like a search query.** Claude matches against the description text. Include the verbs users would naturally say: "create," "generate," "scaffold," "add," "write tests for." If the skill isn't firing, the description is probably too vague.

**Use `/skills` to debug.** The `/skills` command lists every installed skill with its description. Check that your skill shows up. If it's not there, it's not in one of the expected directories.

**Iterate on the procedure.** Run the skill, review the output, and tighten the instructions. If Claude keeps forgetting to add Springdoc annotations, make that step more explicit. If the test count is consistently low, add a minimum count. The skill is code — treat it like code and refine it.

**Don't duplicate CLAUDE.md rules in skills.** Your `CLAUDE.md` says "constructor injection only." You don't need to repeat that in every skill. Skills build on top of `CLAUDE.md` — they add procedures, not rules.

**Reference existing files, not hardcoded examples.** Instead of putting a template in the skill, tell Claude to read an existing file in the project and match its style. This way, the skill stays correct even as the project evolves.

## What's Next?

Skills turned my Claude Code setup from "knows the rules" to "knows the playbook." The combination of `CLAUDE.md` for conventions and skills for procedures covers most of what you need for day-to-day work.

But there's one more layer of customisation we haven't touched: **Slash Commands and Parameterised Skills**. In Part 4, we'll look at how to pass arguments to skills with the `$ARGUMENTS` syntax, the built-in commands that ship with Claude Code (`/review`, `/debug`, `/batch`), the legacy `.claude/commands/` directory (still works, now unified with skills), and practical recipes like `/slice Speaker` or `/fix-issue 1234`. Think of it as the power-user layer on top of everything we've built so far.

See you there.

## This is part 3 of a 10 parts series.

Previous: [Part 2 — Configuration: Taming Claude Code with CLAUDE.md](https://ronveen.com/posts/claude-02-claude-md/)

Next: **Part 4 — Slash Commands: Your Personal Shortcut Library** — Built-in commands, the `$ARGUMENTS` syntax, parameterised skills, and practical recipes for Java/Spring Boot workflows.
