---
title: "Claude Code - 02 - Configuration"
date: 2026-05-21
draft: false
tags: ["Claude", "Claude Code", "Anthropic", "AI", "Agent Coding"]
cover:
  image: "/images/Claude-02-configuration.png"
  alt: "Configuring Claude Code with CLAUDE.md"
series: ["Claude Code"]
series_order: 2
---
In [Part 1](https://ronveen.com/posts/claude-01-introduction/), I showed you what Claude Code is and what it can do. You saw an AI agent that reads your project, writes code, runs your build, and fixes its own mistakes. Impressive stuff.

But here's the thing. If you just fire up Claude Code and start prompting, the output you get is… generic. It's *correct*, sure. But it doesn't know your team uses MapStruct instead of manual mapping. It doesn't know your POST endpoints need authentication. It doesn't know you measure everything with Micrometer and that a controller without metrics is considered incomplete.

Claude Code is like a brilliant new hire on their first day. They're smart, motivated, and ready to go. But they don't know the house rules yet.

That's where `CLAUDE.md` comes in.

## What Is CLAUDE.md?

`CLAUDE.md` is a plain markdown file you drop in the root of your project (or in a few other strategic locations — more on that shortly). Claude Code reads it at the start of every session, before it looks at a single line of your code.

Think of it as a briefing document. It tells Claude Code:
- How to build your project
- What conventions your team follows
- What patterns to use (and which to avoid)
- What "done" looks like in your codebase

If you read Part 1, you already saw a basic example. But that was just the appetiser. In this article, we're going to turn `CLAUDE.md` into a precision instrument.

## Why Should You Care?

Here's a question I get a lot: *"Can't Claude Code just figure this out from looking at the code?"*

Sometimes, yes. If your project has 200 files all following the same pattern, Claude Code will pick up on that. But there are things it *can't* infer:
- That your team decided last sprint to switch from `@Autowired` field injection to constructor injection
- That every REST endpoint must expose Micrometer metrics
- That POST and PUT endpoints require JWT authentication, but GET endpoints don't
- That your DTO naming convention is `XxxResponse` for outbound and `XxxRequest` for inbound

These are *decisions*. They live in your team's collective memory, in your wiki, in your code review comments. Claude Code doesn't have access to any of that — unless you write it down in `CLAUDE.md`.

## The Scope Hierarchy: Where CLAUDE.md Files Live

This is the part most people get wrong on their first try. `CLAUDE.md` isn't a single file — it's a hierarchy. Claude Code loads them in order, from broadest to most specific:

| Scope | Location | Who it's for |
| --- | --- | --- |
| **Managed policy** | `/Library/Application Support/ClaudeCode/CLAUDE.md` (macOS) | Your entire organisation — managed by IT/DevOps |
| **User (global)** | `~/.claude/CLAUDE.md` | You, across all your projects |
| **Project** | `./CLAUDE.md` or `./.claude/CLAUDE.md` | Your team, committed to version control |
| **Local** | `./CLAUDE.local.md` | You, for this project only — gitignored |

Claude Code walks *up* the directory tree from your working directory, picking up every `CLAUDE.md` and `CLAUDE.local.md` it finds. It concatenates all of them into one block of context, with the most specific files loaded last.

Within each directory, `CLAUDE.local.md` is appended *after* `CLAUDE.md`, so your personal notes are the last thing Claude reads at that level.

**For Java developers**, this hierarchy maps nicely to how we already think about configuration. It's like Spring's property resolution:
- Managed policy = your `application.yml` baked into the company base image
- User global = your `~/.m2/settings.xml`
- Project = `application.yml` in `src/main/resources`
- Local = `application-local.yml` — your overrides, not committed

The mental model is the same: more specific wins.

### What Goes Where?

Here's how I split things in practice:

**`~/.claude/CLAUDE.md`** (global, personal): things that follow *you* across projects.
```markdown
# Global Preferences

## Style
- I prefer explicit imports over wildcards
- Use `var` for local variables when the type is obvious from the right-hand side
- Favour readability over cleverness

## Workflow
- Always explain your plan before writing code
- Run the build after making changes
```

**`./CLAUDE.md`** (project, shared): things your team agrees on. This is the one you commit.
```markdown
# Conference Service

## Build
mvn clean verify

## Test  
mvn test

## Architecture
- Spring Boot 3.4 with Java 21
- Layered architecture: Controller → Service → Repository
- Constructor injection only (no field injection)
- MapStruct for DTO mapping
- Flyway for database migrations

## Conventions
- Controllers: `XxxController` in `com.example.conferences.controller`
- Services: `XxxService` in `com.example.conferences.service`
- Repositories: `XxxRepository` in `com.example.conferences.repository`
- DTOs: `XxxRequest` (inbound), `XxxResponse` (outbound)
- Entities: JPA entities in `com.example.conferences.entity`
```

**`./CLAUDE.local.md`** (local, personal): things only you need.
```markdown
# Local Overrides
- My local PostgreSQL runs on port 5433 (not default 5432)
- I use `mvn test -pl conference-api` to run just the API module tests
- Skip integration tests locally: `mvn test -DskipITs`
```

## The Token Tax: Why Size Matters

Here's something that catches people off guard.

Your `CLAUDE.md` is loaded into the context window at the start of *every single session*. It sits there, consuming tokens, before you've even typed your first prompt. And it's not just at the start — it persists across turns.

Let me put some numbers on that. Say your `CLAUDE.md` is 3,000 tokens. That's a decent-sized file — maybe 150 lines of markdown. Over a 40-turn session, that's 120,000 tokens of "tax" you're paying just for the briefing document.

Now multiply that by however many sessions you run in a day.

The official recommendation from Anthropic is to keep each `CLAUDE.md` file under 200 lines. I'd go further and say: **treat your CLAUDE.md like a tweet thread, not a wiki article.** Every line should earn its place.

Here are some practical tips:

**Be terse.** You're writing for an LLM, not for a new hire's onboarding document.

```markdown
# Bad (wastes tokens)
When writing controller classes, please ensure that you always use 
constructor-based dependency injection rather than field-based injection 
with the @Autowired annotation. This is because constructor injection 
makes dependencies explicit and enables easier testing.

# Good (same information, fewer tokens)
- Constructor injection only. No @Autowired on fields.
```

**Use HTML comments for human-only notes.** Block-level HTML comments are stripped before injection into Claude's context. They cost you zero tokens.

```markdown
<!-- 
  Maintainer note: we switched to MapStruct in Q2 2025.
  See ADR-047 for the decision rationale.
-->
- Use MapStruct for all DTO mapping. No manual mapping methods.
```

**Offload details to path-scoped rules.** If a rule only matters for your API layer, don't burn tokens loading it when Claude is editing a database migration. We'll cover this later in the article.

## Progressive Example: From Generic to Production-Quality

This is where it gets fun. Let's use the same scenario from Part 1 — adding a Conference entity and REST endpoint — and watch how the output improves as we refine our `CLAUDE.md`.

### Round 1: No CLAUDE.md — The Blank Slate

You fire up Claude Code in a fresh Spring Boot project. No `CLAUDE.md`, no rules, nothing. You type:

```
Add a REST endpoint GET /conferences that returns a list of conferences. 
Include a Conference entity with typical fields.
```

Claude Code produces something like this:

```java
@RestController
@RequestMapping("/conferences")
public class ConferenceController {

    @Autowired
    private ConferenceService conferenceService;

    @GetMapping
    public List<Conference> getAllConferences() {
        return conferenceService.getAllConferences();
    }
}
```

```java
@Entity
public class Conference {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private String location;
    private LocalDate startDate;
    private LocalDate endDate;

    // getters and setters
}
```

```java
@Service
public class ConferenceService {

    @Autowired
    private ConferenceRepository conferenceRepository;

    public List<Conference> getAllConferences() {
        return conferenceRepository.findAll();
    }
}
```

It works. But look at what's wrong from a "house rules" perspective:
- **Field injection** with `@Autowired` — your team banned this last quarter
- **Returns entities directly** — no DTO layer, no MapStruct
- **No metrics** — production endpoints need Micrometer counters
- **No authentication** — this endpoint should be open, but there's no security configuration at all
- **`Long` for the ID** — your team uses `UUID`
- **No DTOs at all** — just raw entities over the wire, no request/response separation
- **No tests** — not a single test class

It's the "brilliant new hire" problem. Technically correct, organisationally wrong.

### Round 2: Basic CLAUDE.md — Teaching the Conventions

Now let's add a `CLAUDE.md` with your team's core conventions:

```markdown
# Conference Service

## Build
mvn clean verify

## Test
mvn test

## Architecture
- Spring Boot 3.4, Java 21
- Layered: Controller → Service → Repository
- Constructor injection only. No @Autowired on fields.
- MapStruct for DTO mapping
- UUID for all entity primary keys

## Naming
- Controllers: `XxxController` in `controller` package
- Services: `XxxService` in `service` package  
- Repositories: `XxxRepository` in `repository` package
- DTOs: `XxxResponse` (outbound), `XxxRequest` (inbound) in `dto` package
- Mappers: `XxxMapper` in `mapper` package

## Testing
- Write unit tests for services (JUnit 5 + Mockito)
- Write integration tests for controllers (@WebMvcTest)
```

Same prompt:
```
Add a REST endpoint GET /conferences that returns a list of conferences. 
Include a Conference entity with typical fields.
```

Now the output looks like this:

```java
@RestController
@RequestMapping("/api/conferences")
public class ConferenceController {

    private final ConferenceService conferenceService;

    public ConferenceController(ConferenceService conferenceService) {
        this.conferenceService = conferenceService;
    }

    @GetMapping
    public List<ConferenceResponse> getAllConferences() {
        return conferenceService.getAllConferences();
    }
}
```

```java
@Entity
@Table(name = "conferences")
public class Conference {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;
    
    @Column(nullable = false)
    private String name;
    
    @Column(nullable = false)
    private String location;
    
    @Column(name = "start_date", nullable = false)
    private LocalDate startDate;
    
    @Column(name = "end_date", nullable = false)
    private LocalDate endDate;

    @Column(length = 1000)
    private String description;

    // constructors, getters, setters
}
```

```java
public class ConferenceResponse {

    private UUID id;
    private String name;
    private String location;
    private LocalDate startDate;
    private LocalDate endDate;
    private String description;

    // constructors, getters, setters
}
```

```java
@Mapper(componentModel = "spring")
public interface ConferenceMapper {
    ConferenceResponse toResponse(Conference conference);
    List<ConferenceResponse> toResponseList(List<Conference> conferences);
}
```

```java
@Service
public class ConferenceService {

    private final ConferenceRepository conferenceRepository;
    private final ConferenceMapper conferenceMapper;

    public ConferenceService(ConferenceRepository conferenceRepository,
                             ConferenceMapper conferenceMapper) {
        this.conferenceRepository = conferenceRepository;
        this.conferenceMapper = conferenceMapper;
    }

    public List<ConferenceResponse> getAllConferences() {
        return conferenceMapper.toResponseList(conferenceRepository.findAll());
    }
}
```

Much better. Constructor injection, a separate DTO layer, MapStruct mapper, UUID primary keys, proper package structure. Claude Code also generated a `ConferenceControllerTest` with `@WebMvcTest` and a `ConferenceServiceTest` with Mockito.

Notice, though, that the DTO is a plain class with getters and setters — we never told Claude to use records. It's following our naming convention (`XxxResponse`), but it defaulted to a traditional POJO. We'll fix that in Round 3.

But we're not done yet.

### Round 3: The Full Briefing — Metrics, Security, and Production Standards

Now let's add the rules that make the difference between "it compiles" and "it passes code review":

```markdown
# Conference Service

## Build
mvn clean verify

## Test
mvn test

## Architecture
- Spring Boot 3.4, Java 21
- Layered: Controller → Service → Repository
- Constructor injection only. No @Autowired on fields.
- MapStruct for DTO mapping
- Use records for all DTOs — no mutable POJOs
- UUID for all entity primary keys
- Bean Validation on all request DTOs (@Valid + constraint annotations)

## Naming
- Controllers: `XxxController` in `controller` package
- Services: `XxxService` in `service` package
- Repositories: `XxxRepository` in `repository` package
- DTOs: `XxxResponse` (outbound), `XxxRequest` (inbound) in `dto` package
- Mappers: `XxxMapper` in `mapper` package

## Security
- All POST, PUT, DELETE endpoints require JWT authentication
- GET endpoints are public unless specified otherwise
- Use Spring Security with @PreAuthorize where needed
- Configure SecurityFilterChain — no extends WebSecurityConfigurerAdapter

## Observability
- Every controller method must have a @Timed annotation: `api.<entity>.<operation>`
- Example: `api.conferences.list`, `api.conferences.create`
- @Timed provides both timing and call-count metrics automatically
- Add a health indicator for external dependencies

## Error Handling
- Use @ControllerAdvice for global exception handling
- Return ProblemDetail (RFC 9457) for all error responses
- Map ConstraintViolationException to 400 with field-level errors

## Testing
- Unit tests for services (JUnit 5 + Mockito)
- Integration tests for controllers (@WebMvcTest)
- Include tests for error cases (404, 400, 401)
- Test security: verify unauthenticated POST returns 401
```

Same prompt again:
```
Add a REST endpoint GET /conferences that returns a list of conferences.
Also add POST /conferences to create a new conference.
Include a Conference entity with typical fields.
```

Now Claude Code produces a full slice. Here are the highlights of what changed:

**The controller now has metrics and security baked in:**
```java
@RestController
@RequestMapping("/api/conferences")
public class ConferenceController {

    private final ConferenceService conferenceService;

    public ConferenceController(ConferenceService conferenceService) {
        this.conferenceService = conferenceService;
    }

    @GetMapping
    @Timed(value = "api.conferences.list", description = "List all conferences")
    public List<ConferenceResponse> getAllConferences() {
        return conferenceService.getAllConferences();
    }

    @PostMapping
    @PreAuthorize("isAuthenticated()")
    @Timed(value = "api.conferences.create", description = "Create a conference")
    public ResponseEntity<ConferenceResponse> createConference(
            @Valid @RequestBody ConferenceRequest request) {
        ConferenceResponse response = conferenceService.createConference(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }
}
```

**The request DTO has Bean Validation**
```java
public record ConferenceRequest(
    @NotBlank(message = "Conference name is required")
    String name,

    @NotBlank(message = "Location is required")  
    String location,

    @NotNull(message = "Start date is required")
    @FutureOrPresent(message = "Start date must be in the future")
    LocalDate startDate,

    @NotNull(message = "End date is required")
    LocalDate endDate,

    @Size(max = 1000, message = "Description must not exceed 1000 characters")
    String description
) {}
```

**The DTOs are now records** — remember the plain class from Round 2? Gone.
```java
public record ConferenceResponse(
    UUID id,
    String name,
    String location,
    LocalDate startDate,
    LocalDate endDate,
    String description
) {}
```

**There's a global exception handler:**
```java
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ConstraintViolationException.class)
    public ProblemDetail handleValidationException(ConstraintViolationException ex) {
        ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        problem.setTitle("Validation Failed");
        problem.setProperty("violations", ex.getConstraintViolations().stream()
            .map(v -> Map.of("field", v.getPropertyPath().toString(), 
                             "message", v.getMessage()))
            .toList());
        return problem;
    }
}
```

**And the security configuration is there:**
```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(sm -> sm.sessionCreationPolicy(STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(HttpMethod.GET, "/api/**").permitAll()
                .requestMatchers("/actuator/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .build();
    }
}
```

**The test suite now includes security assertions:**
```java
@WebMvcTest(ConferenceController.class)
class ConferenceControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private ConferenceService conferenceService;

    @Test
    void getAllConferences_returnsOk() throws Exception {
        when(conferenceService.getAllConferences()).thenReturn(List.of());
        mockMvc.perform(get("/api/conferences"))
            .andExpect(status().isOk());
    }

    @Test
    void createConference_withoutAuth_returns401() throws Exception {
        mockMvc.perform(post("/api/conferences")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {"name": "Devoxx", "location": "Antwerp",
                     "startDate": "2026-10-12", "endDate": "2026-10-16"}
                    """))
            .andExpect(status().isUnauthorized());
    }

    @Test
    void createConference_withInvalidBody_returns400() throws Exception {
        mockMvc.perform(post("/api/conferences")
                .with(jwt())
                .contentType(MediaType.APPLICATION_JSON)
                .content("{}"))
            .andExpect(status().isBadRequest());
    }
}
```

That's the progression. Same prompt, three completely different levels of output. The code in Round 3 would actually pass a code review. The code in Round 1 would get sent back with a dozen comments.

The `CLAUDE.md` didn't change the prompt. It changed the *baseline*.

Now that you've seen the impact, let's look at some techniques to keep your `CLAUDE.md` maintainable as it grows.

## The @import Trick: Keeping CLAUDE.md Lean

Your `CLAUDE.md` supports an `@path/to/file` import syntax. This is incredibly useful for referencing existing project files without copy-pasting their content.

```markdown
# Conference Service

## Project Overview
@README.md

## Dependencies  
@pom.xml

## Architecture Decision Records
@docs/adr/
```

Imported files are expanded at launch and loaded into context alongside the `CLAUDE.md` that references them. Both relative and absolute paths work. You can even nest imports up to five levels deep.

A word of caution, though: imports still consume tokens. If your `pom.xml` is 500 lines, that's 500 lines loaded into every session. Use this for files Claude genuinely needs to reference, not as a "let me dump everything" mechanism.

A pattern I find useful: reference your `pom.xml` so Claude knows exactly what dependencies are available, and reference a short architecture doc so it understands the high-level design. But don't import your entire `docs/` folder.

```markdown
## Context
@pom.xml
@docs/architecture-overview.md

## Git Workflow
@docs/git-instructions.md
```

For personal overrides that shouldn't be committed, use `CLAUDE.local.md` and import from your home directory:

```markdown
# My Personal Preferences
@~/.claude/my-spring-boot-rules.md
```

This way, your personal rules follow you across worktrees of the same repo.

## Path-Scoped Rules: Load Only What's Needed

As your `CLAUDE.md` grows, you'll start noticing that some rules only matter for specific parts of the codebase. Your API validation rules are irrelevant when Claude is editing a Flyway migration. Your testing conventions don't apply to Docker configuration.

This is where `.claude/rules/` comes in. You can create markdown files in this directory, and optionally scope them to specific file patterns using YAML frontmatter:

```
your-project/
├── .claude/
│   ├── CLAUDE.md
│   └── rules/
│       ├── api-design.md
│       ├── testing.md
│       └── database.md
```

Here's what `api-design.md` might look like:

```markdown
---
paths:
  - "src/main/java/**/controller/**"
  - "src/main/java/**/dto/**"
---

# API Design Rules

- All POST/PUT endpoints require @Valid on the request body
- Return ProblemDetail (RFC 9457) for errors
- Every controller method needs a Micrometer counter
- GET endpoints are public; POST/PUT/DELETE require authentication
- Use records for DTOs
```

And `testing.md`:

```markdown
---
paths:
  - "src/test/**"
---

# Testing Rules

- Use @WebMvcTest for controller tests, not @SpringBootTest
- Mock dependencies with @MockitoBean
- Test the happy path, validation errors, and authentication (401)
- Use Testcontainers for repository integration tests
```

These rules *only* load when Claude reads files matching the glob pattern. So when you're working on a controller, Claude sees the API design rules. When you're writing tests, it sees the testing rules. When you're editing `docker-compose.yml`, neither of these loads, saving you tokens and keeping Claude's context focused.

This is the equivalent of conditional `@Profile` beans in Spring — configuration that activates only when it's relevant.

## CLAUDE.md vs Auto Memory: Who Writes What?

Claude Code has two memory systems, and it's worth understanding the difference.

**CLAUDE.md** is what *you* write. It's your rules, your conventions, your instructions. You control every word.

**Auto memory** is what *Claude* writes for itself. When you correct it — "No, we use `BigDecimal` for monetary values, not `double`" — Claude can save that lesson to `~/.claude/projects/<project>/memory/MEMORY.md`. Next session, it remembers.

Think of it this way:
- `CLAUDE.md` = the employee handbook you hand to the new hire
- Auto memory = the notes the new hire scribbles in their notebook after their first week

Both are loaded at session start. Your project-root `CLAUDE.md` survives compaction (when Claude trims its context window) — it gets re-read from disk and re-injected. Auto memory lives on disk too, so it's always available at the start of the next session. They serve different purposes.

**Use CLAUDE.md for:**
- Architectural decisions (constructor injection, MapStruct, layered architecture)
- Build and test commands
- Naming conventions
- Security requirements
- Non-negotiable standards

**Let auto memory handle:**
- Build quirks Claude discovers ("running `mvn test` on this project requires Docker to be running")
- Debugging patterns ("the Flyway migration error usually means the local DB is out of sync")
- Your personal workflow habits ("Ron prefers to see the plan before any code is written")

You can check what Claude has memorised by running `/memory` in a session. Everything is plain markdown you can read, edit, or delete. If Claude learned something wrong, just fix the file.

## Putting It All Together: My Recommended Setup

Here's the setup I use across my Spring Boot projects. It's opinionated, but it works.

```
my-project/
├── .claude/
│   ├── CLAUDE.md              # Core conventions (committed)
│   └── rules/
│       ├── api-design.md      # Path-scoped: controllers + DTOs
│       ├── testing.md         # Path-scoped: test files
│       ├── database.md        # Path-scoped: entities + migrations
│       └── security.md        # Path-scoped: security config
├── CLAUDE.local.md            # My local overrides (gitignored)
└── src/
    └── ...
```

The main `CLAUDE.md` stays under 100 lines: build commands, project architecture, and the most critical conventions. Everything else goes into path-scoped rules that only load when relevant.

## Tips From the Trenches

After months of using `CLAUDE.md` in real projects, here are the things that made the biggest difference:

**Start small.** Don't try to write the perfect `CLAUDE.md` on day one. Start with your build command and three conventions. Add rules as you find yourself correcting Claude.

**Be specific, not vague.** "Use 2-space indentation" beats "format code properly." "Every controller method must have a `@Timed` annotation" beats "add observability." If you can't verify it, Claude can't follow it reliably.

**Use `/init` to bootstrap.** Run `/init` in your project and Claude will analyse your codebase and generate a starting `CLAUDE.md`. It won't be perfect, but it's a solid starting point. You can set `CLAUDE_CODE_NEW_INIT=1` for an interactive flow that asks follow-up questions.

**Review it like code.** Your `CLAUDE.md` lives in version control. Treat changes to it with the same scrutiny you'd give a pull request. A bad rule in `CLAUDE.md` can silently degrade every piece of code Claude generates for your team.

**Watch for contradictions.** If your project `CLAUDE.md` says "use records for DTOs" but a subdirectory `CLAUDE.md` says "use classes with Lombok for DTOs," Claude might pick either one. Periodically review your files and remove outdated rules.

**Use HTML comments as changelogs.** Since block-level HTML comments are stripped from the context, you can leave notes for your team without any token cost:

```markdown
<!-- Updated 2026-05-15: switched from Lombok to records for DTOs. See PR #287. -->
- Use records for all DTOs. No Lombok @Data or @Value.
```

## What's Next?

You now have the tools to make Claude Code behave less like a generic code generator and more like a team member who *actually read the wiki*. A well-crafted `CLAUDE.md` is the single highest-leverage thing you can do to improve your Claude Code experience.

But conventions are just the start. In Part 3, we'll look at **Skills** — the `SKILL.md` architecture that lets you teach Claude Code how to perform complex, multi-step tasks like generating a full REST slice, running a code review, or scaffolding a new microservice. Think of skills as the difference between telling someone the rules and showing them how it's done.

See you there.

## This is part 2 of a 10 parts series.

Previous: [Part 1 — Introduction: What Is Claude Code?](https://ronveen.com/posts/claude-01-introduction/)

Next: **Part 3 — Skills: Teaching Claude Code Your Standards** — The SKILL.md architecture, auto-triggered vs explicit skills, writing your own, and best practices for skill design.