---
name: git-conventions
description: Git commit message format, branching strategy, and PR conventions. Use when committing code, creating branches, writing PR descriptions, or managing the git workflow.
---

All git operations must follow established conventions for commits, branches, and pull requests.

## Scope

**In scope:** Commit message format, branch naming, PR conventions, git workflow, and version control best practices.

**Out of scope:** Code implementation, architecture, CI/CD pipeline design — defer these to the relevant agents.

## Commit Message Format

```
<Type> - <Short description>
```

### Types

| Type       | Usage                                        |
| ---------- | -------------------------------------------- |
| `Feature`  | New feature or capability                    |
| `Fix`      | Bug fix                                      |
| `Refactor` | Code improvement without behavior change     |
| `Docs`     | Documentation changes                        |
| `Style`    | Formatting, whitespace, no code logic change |
| `Test`     | Adding or updating tests                     |
| `Chore`    | Build scripts, tooling, configuration        |
| `Agent`    | Claude Code agent definitions                |
| `Skills`   | Claude Code skill definitions                |

### Examples

```
Agent - Added Control Agents
Agent - Added dev agents
Skills - Added skills
Feature - Added streak CRUD controllers
Feature - Added Google OAuth flow
Fix - Fixed token refresh race condition
Refactor - Simplified auth middleware chain
Docs - Updated API documentation
Chore - Added CI workflow
```

### Rules

- **Start** with the type, followed by `-` and a description
- **Capitalize** the first word of the description
- **Keep** the subject line under 72 characters
- **Use** imperative mood: "Added", "Fixed", "Updated" (matching project convention)
- **Do not** end with a period
- **Reference** the affected area when helpful: `Feature - Added streak CRUD controllers`

## Branch Naming

```
<type>/<short-description>
```

### Examples

```
feature/streak-crud
fix/auth-token-refresh
refactor/controller-validation
chore/eslint-setup
docs/api-documentation
test/streak-unit-tests
```

### Rules

- **Use** lowercase kebab-case
- **Prefix** with type: `feature/`, `fix/`, `refactor/`, `chore/`, `docs/`, `test/`, `wip/`
- **Keep** descriptive but concise
- **Branch from** `main` for new work

## Pull Request Conventions

### PR Title

```
<Type> - <Short description>
```

Same format as commit messages. Keep under 70 characters.

### PR Body

```markdown
## Summary

- Brief description of what this PR does
- Why this change is needed
- Key decisions made

## Changes

- List of significant changes
- Files/modules affected

## Test Plan

- [ ] How to verify the changes work
- [ ] Manual testing steps
- [ ] Any automated tests added

## Notes

- Breaking changes (if any)
- Dependencies added/removed
- Related issues or PRs
```

## Git Workflow

1. **Create branch** from `main`: `git checkout -b feature/my-feature main`
2. **Make commits** following the message format
3. **Push** to remote: `git push -u origin feature/my-feature`
4. **Create PR** against `main`
5. **Review** — use code-reviewer agent or manual review
6. **Merge** — squash or merge commit depending on PR size

## Constraints

- **ALWAYS** follow the commit message format: `<Type> - <Description>`
- **ALWAYS** use the branch naming convention: `<type>/<description>`
- **NEVER** commit directly to `main`
- **NEVER** commit `.env` files, secrets, or `node_modules`
- **NEVER** force push to `main`
- **NEVER** use `--no-verify` to skip hooks
- **PREFER** small, focused commits over large monolithic ones
- **PREFER** one concern per commit
