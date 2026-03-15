# Code Review Guidelines

> Authoritative code review standards for consistent, constructive, and efficient review processes.

## Purpose

Establish effective code review practices that improve code quality, share knowledge across the team, and maintain a healthy engineering culture.

## Core Principles

1. **Improve the codebase** - Every change should leave the code better than before
2. **Be respectful** - Review the code, not the person
3. **Be timely** - Reviews should not block progress unnecessarily
4. **Be thorough** - But not pedantic about style (let linters handle that)
5. **Share knowledge** - Reviews are learning opportunities for both parties
6. **Maintain velocity** - Don't let perfect be the enemy of good

## Roles and Responsibilities

### Author Responsibilities

- Write clear PR descriptions explaining what and why
- Keep changes focused and reasonably sized
- Self-review before requesting review
- Respond to feedback constructively
- Run all tests and checks before requesting review

### Reviewer Responsibilities

- Review within 1 business day (ideally within 4 hours)
- Provide constructive, specific feedback
- Distinguish between blocking issues and suggestions
- Approve once blocking concerns are addressed
- Learn from the code being reviewed

## PR Size Guidelines

| Size | Lines Changed | Review Time | Recommendation |
| ---- | ------------- | ----------- | -------------- |
| XS | < 50 | < 15 min | Ideal for quick fixes |
| S | 50-200 | 15-30 min | Good for features |
| M | 200-500 | 30-60 min | Maximum recommended |
| L | 500-1000 | 1-2 hours | Consider splitting |
| XL | > 1000 | 2+ hours | Must split |

### Splitting Large Changes

```
# Instead of one large PR:
"Add user authentication system" (2000 lines)

# Split into:
1. "Add User entity and database schema" (200 lines)
2. "Add password hashing service" (150 lines)
3. "Add JWT token service" (200 lines)
4. "Add authentication controller" (300 lines)
5. "Add auth middleware and guards" (200 lines)
6. "Add login/register UI components" (400 lines)
```

## PR Description Template

```markdown
## Summary
Brief description of what this PR does and why.

## Changes
- Added X to handle Y
- Modified Z to support W
- Removed deprecated V

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] Manual testing performed

## Screenshots (if UI changes)
Before | After
--- | ---
[image] | [image]

## Related Issues
Closes #123
Related to #456

## Deployment Notes
Any special deployment considerations or migrations.
```

## Review Checklist

### Functionality

- [ ] Code does what the PR description says
- [ ] Edge cases are handled
- [ ] Error handling is appropriate
- [ ] No obvious bugs or logic errors

### Design

- [ ] Changes fit the overall architecture
- [ ] No unnecessary complexity
- [ ] Code is in the right location
- [ ] Abstractions are appropriate

### Code Quality

- [ ] Code is readable and self-documenting
- [ ] Names are clear and meaningful
- [ ] No duplicated code
- [ ] Functions/methods are focused (single responsibility)

### Testing

- [ ] Tests cover the changes
- [ ] Tests are meaningful (not just coverage)
- [ ] Edge cases are tested
- [ ] Tests are maintainable

### Security

- [ ] No obvious security vulnerabilities
- [ ] Input validation where needed
- [ ] No secrets in code
- [ ] Authentication/authorization correct

### Performance

- [ ] No N+1 queries or inefficient loops
- [ ] Appropriate caching considered
- [ ] No memory leaks
- [ ] Database indexes considered

## Comment Guidelines

### Comment Types

| Prefix | Meaning | Blocks Approval |
| ------ | ------- | --------------- |
| (none) | Blocking issue | Yes |
| `nit:` | Minor suggestion | No |
| `optional:` | Take it or leave it | No |
| `question:` | Seeking clarification | Maybe |
| `suggestion:` | Alternative approach | No |
| `praise:` | Positive feedback | No |

### Good Comments

```markdown
# Blocking issue
This query will cause an N+1 problem in production. Consider using
`include` or a separate query with `IN` clause.

# Suggestion
suggestion: Consider extracting this into a separate function for reusability.

# Question
question: What happens if `user` is null here? Should we add a guard?

# Nitpick
nit: This variable name could be more descriptive. Maybe `userEmailAddress`?

# Praise
praise: Great use of the strategy pattern here! This makes adding new
payment methods much cleaner.
```

### Bad Comments

```markdown
# Too vague
"This is wrong"
"Fix this"

# Personal attack
"Why would you do it this way?"

# Pedantic without tooling
"Add a space here"  # Let the linter handle this

# Overwhelming
[50 comments on a 100-line PR about stylistic preferences]
```

## Response Guidelines

### As an Author

```markdown
# Acknowledging feedback
"Good catch! Fixed in abc123."

# Explaining decisions
"I considered that approach, but went with this because [reason].
Would you like me to reconsider?"

# Asking for clarification
"I'm not sure I understand the concern. Could you elaborate?"

# Respectfully disagreeing
"I see your point, but I think the current approach is better because
[reason]. Happy to discuss further if you'd like."
```

### As a Reviewer

```markdown
# Approving with suggestions
"LGTM! A few optional suggestions, but feel free to merge as-is."

# Requesting changes
"Please address the blocking comments before merging. Happy to
re-review once updated."

# Being flexible
"I have a preference for X, but Y also works. Your call."
```

## Review Turnaround Times

| Priority | Target | Maximum |
| -------- | ------ | ------- |
| Urgent (production fix) | 30 minutes | 2 hours |
| High (blocking work) | 2 hours | 4 hours |
| Normal | 4 hours | 1 business day |
| Low (nice to have) | 1 business day | 2 business days |

## Approval Requirements

### Standard Changes

- 1 approval from team member
- All CI checks pass
- No unresolved blocking comments

### Critical Changes

Require additional scrutiny:

- Security-sensitive code: +1 security-focused reviewer
- Database migrations: +1 DBA or senior engineer
- API changes: +1 API design reviewer
- Infrastructure changes: +1 DevOps reviewer

## Automated Checks

Let automation handle:

- [ ] Code formatting (Prettier, Black, etc.)
- [ ] Linting (ESLint, Pylint, etc.)
- [ ] Type checking (TypeScript, mypy, etc.)
- [ ] Test coverage thresholds
- [ ] Dependency vulnerabilities
- [ ] PR size limits

## Anti-Patterns to Avoid

### Reviewer Anti-Patterns

- **Nitpicking**: Focusing on trivial issues while missing important ones
- **Gatekeeping**: Blocking PRs for personal preferences
- **Drive-by reviewing**: Leaving comments without following up
- **Delayed reviews**: Letting PRs sit for days

### Author Anti-Patterns

- **Mega PRs**: Changes too large to review effectively
- **No context**: PRs without description or testing evidence
- **Ignoring feedback**: Not responding to reviewer comments
- **Force pushing**: Rewriting history during review

## Checklist

### Process Setup

- [ ] PR template configured
- [ ] Required reviewers defined
- [ ] Branch protection rules enabled
- [ ] CI checks required before merge

### Review Culture

- [ ] Team aligned on review expectations
- [ ] Turnaround times communicated
- [ ] Comment guidelines shared
- [ ] Regular retros on review process

### Tooling

- [ ] Automated formatting checks
- [ ] Automated linting
- [ ] Test coverage reports on PRs
- [ ] PR size warnings

## References

- [Google Engineering Practices - Code Review](https://google.github.io/eng-practices/review/)
- [How to Do Code Reviews Like a Human](https://mtlynch.io/human-code-reviews-1/)
- [Conventional Comments](https://conventionalcomments.org/)
- [The Standard of Code Review](https://google.github.io/eng-practices/review/reviewer/standard.html)
