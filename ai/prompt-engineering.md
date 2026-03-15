# Prompt Engineering Standards

> Practical guidelines for writing effective prompts and instructions for AI-assisted development tools. Focuses on Claude Code but principles apply broadly.

## Purpose

Define consistent practices for communicating with AI tools to get reliable, high-quality results. Good prompts reduce iteration cycles, prevent misunderstandings, and produce code that aligns with project standards.

## Core Principles

1. **Context before instruction** - Provide relevant background before asking for action
2. **Specific over vague** - Concrete constraints produce better results than open-ended requests
3. **Show, don't describe** - Examples and code snippets outperform abstract descriptions
4. **Constrain the scope** - Smaller, focused requests beat large ambiguous ones
5. **State the "why"** - Explaining intent helps AI make better trade-off decisions

## CLAUDE.md as Persistent Context

The most impactful prompt engineering happens in CLAUDE.md files — they provide context on every interaction without repeating yourself.

See [Claude Code Standards](./claude-code-standards) for the full CLAUDE.md template. Key sections that function as persistent prompts:

| Section | What It Tells AI |
|---------|------------------|
| Quick Reference | Ports, commands, stack — prevents wrong assumptions |
| Architecture | Patterns and conventions to follow |
| Quality Gates | What checks to run and when |
| Sub-Agent Strategy | When to delegate and to whom |
| Project-Specific Notes | Gotchas and tribal knowledge |

**Anti-pattern:** Repeating the same instructions in every conversation when they should be in CLAUDE.md.

## Prompt Structure

### The Effective Prompt Formula

```
[Context] + [Constraint] + [Instruction] + [Output Format]
```

**Context** — What the AI needs to know about the current situation:
- What files are involved
- What the current behaviour is
- What's already been tried

**Constraint** — Boundaries and requirements:
- Don't modify file X
- Use existing pattern Y
- Must work with framework Z version N

**Instruction** — The specific action:
- Add, fix, refactor, explain, test

**Output Format** — What you expect back:
- Code changes only, no explanation
- Summary of findings with file paths
- Step-by-step plan before implementation

### Examples

#### Weak Prompt
```
Fix the authentication bug
```

#### Strong Prompt
```
The login form at frontend/src/app/login/page.tsx submits successfully
but the JWT token is not being stored in the cookie. The auth API at
backend/Controllers/AuthController.cs returns the token in the response
body. The cookie should be set as httpOnly with SameSite=Strict.
Follow the existing pattern in backend/Controllers/BaseController.cs
for response formatting.
```

#### Weak Prompt
```
Add tests for the user service
```

#### Strong Prompt
```
Add unit tests for backend/Services/UserService.cs covering:
- CreateUser: valid input, duplicate email, missing required fields
- GetUserById: existing user, non-existent user, wrong tenant
Follow the existing test pattern in Backend.Tests/Services/BookServiceTests.cs.
Use xUnit + Moq + FluentAssertions as established in the project.
```

## Task Decomposition

### When to Break Down Tasks

Break a task into multiple prompts when:
- It spans more than 3 files
- It requires both frontend and backend changes
- It involves architectural decisions AND implementation
- The full scope isn't clear upfront

### Decomposition Pattern

```
1. "Explore the codebase to understand how X currently works"
   → AI investigates and reports findings

2. "Based on what you found, plan how to implement Y"
   → AI produces a plan for review

3. "Implement the backend changes from your plan"
   → AI makes focused changes

4. "Now implement the frontend changes"
   → AI makes focused changes

5. "Run the tests and fix any failures"
   → AI validates the work
```

This mirrors the [Claude Code Standards](./claude-code-standards) workflow: Explore → Plan → Implement → Test → Document.

### Planning Mode

For non-trivial changes, start with a planning prompt:

```
Plan how to add [feature]. Read the relevant files first, then
propose an implementation approach. Don't make any changes yet —
just outline the plan with specific file paths and code locations.
```

This lets you review the approach before any code is written.

## Constraint Techniques

### Positive Constraints (Do This)

```
Follow the existing pattern in src/components/DataTable.tsx
Use the same error handling approach as the BookService
Match the API response format from GET /api/users
```

### Negative Constraints (Don't Do This)

```
Don't modify the database schema — only change the API layer
Don't add new dependencies — use what's already in package.json
Don't refactor surrounding code — just fix the specific bug
```

### Reference Constraints (Like This)

```
Format the new endpoint like GET /api/books in BooksController.cs
Use the same validation pattern as CreateBookDto
Follow the component structure of src/components/BookCard.tsx
```

### Scope Constraints

```
Only modify files in the frontend/src/app/dashboard/ directory
Keep changes to the service layer — don't touch controllers
Limit to under 50 lines of changes
```

## Working with Context Windows

### Provide Relevant Context

The AI can only work with what it can see. Front-load the most important context:

- **File paths** — Specify exact files rather than making the AI search
- **Error messages** — Paste the full error, not a summary
- **Expected behaviour** — Describe what should happen, not just what's wrong
- **Existing patterns** — Point to examples in the codebase

### Avoid Context Pollution

Don't paste irrelevant code, logs, or documentation into prompts. More context isn't always better — irrelevant context dilutes the signal.

**Good:** Paste the specific function that's failing
**Bad:** Paste the entire 500-line file "just in case"

### Use File References

Instead of pasting code, reference it by path:

```
Read frontend/src/hooks/useAuth.ts and explain why the
token refresh isn't triggering when the session expires.
```

The AI can read files directly — no need to copy-paste.

## Iterative Refinement

### Course Correction

If the AI's output isn't right, be specific about what to change:

```
# Weak correction
That's not quite right, try again.

# Strong correction
The error handling is correct but the response format is wrong.
It should return { data: T, error: null } on success, matching
the ApiResponse<T> type in shared/types/api.ts.
```

### Building on Previous Output

Reference earlier work in the conversation:

```
The service layer changes look good. Now add the corresponding
controller endpoint following the same pattern you used for the
CreateBook endpoint above.
```

### Knowing When to Reset

Start a new conversation when:
- The context window is getting large and responses slow down
- The AI has gone down a wrong path and corrections aren't working
- You're switching to an unrelated task
- The conversation has accumulated stale/incorrect context

## Anti-Patterns

| Anti-Pattern | Why It Fails | Better Approach |
|---|---|---|
| "Make it better" | No actionable criteria | Specify what "better" means: faster, more readable, more testable |
| Pasting entire files | Dilutes relevant context | Reference by path or paste only the relevant section |
| Asking for everything at once | Overwhelms scope, produces shallow results | Decompose into focused steps |
| No examples or references | AI guesses at conventions | Point to existing patterns in the codebase |
| Correcting without specifics | "Try again" gives no direction | Explain exactly what's wrong and what you expect |
| Skipping planning | Produces code that needs rework | Plan first for non-trivial changes |
| Repeating CLAUDE.md content | Wastes context window | Put persistent instructions in CLAUDE.md |

## Checklist

When writing a prompt for a non-trivial task:

- [ ] Context provided (relevant files, current state, what's been tried)
- [ ] Constraints specified (scope, patterns to follow, things to avoid)
- [ ] Instruction is specific and actionable
- [ ] Expected output format stated (code only, plan first, explain then implement)
- [ ] References to existing patterns included where applicable
- [ ] Task is appropriately scoped (not too broad)
