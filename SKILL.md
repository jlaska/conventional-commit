---
name: conventional-commit
description: Generate a Conventional Commits format message based on git diff. Use this skill whenever the user wants to commit, asks for a commit message, says /commit, or mentions conventional commits.
allowed-tools:
  - Bash(git add:*)
  - Bash(git commit:*)
  - Bash(git push:*)
  - Bash(git status:*)
  - Bash(git diff:*)
  - Bash(git log:*)
argument-hint: "[type] [scope]"
disable-model-invocation: true
---

## Context

Dynamic git state information:

- Current git status: !`git status --short`
- Staged changes: !`git diff --cached --stat`
- Changed and untracked files: !`git diff --name-only HEAD && git ls-files --others --exclude-standard`
- Recent commits (for style reference): !`git log --oneline -5`
- Current branch: !`git branch --show-current`

## Your Task

Generate a git commit message following the [Conventional Commits specification](https://www.conventionalcommits.org) based on the current changes, then present it to the user for review before committing.

## User Arguments

The user invoked this command with: $ARGUMENTS

**Argument handling:**
- No arguments: Auto-detect type and scope
- One argument: Force type (e.g., `feat`, `fix`, `chore`)
- Two arguments: Force type and scope (e.g., `feat api`)

## Conventional Commits Format

```
<type>[optional scope]: <description>
```

### Core Types

- `feat` - New feature (MINOR version)
- `fix` - Bug fix (PATCH version)
- `docs` - Documentation changes
- `style` - Formatting, whitespace
- `refactor` - Code restructuring without behavior change
- `perf` - Performance improvements
- `test` - Test changes
- `build` - Build system changes
- `ci` - CI configuration
- `chore` - Maintenance tasks, dependencies
- `revert` - Revert previous commit

### Breaking Changes

Add an exclamation mark (!) after type/scope (e.g., feat!: or fix!:) if changes break existing APIs or remove public functions.

## Instructions

### Phase 1: Analysis & Message Generation

Follow this workflow:

#### 1. Check for Changes

If there are no changes (clean working tree):
- Display: "No changes to commit. Working tree is clean."
- Do not proceed further

#### 2. Analyze Changes & Determine Type

Use this decision tree to determine commit type:

| Pattern | Type | Example |
|---------|------|---------|
| New files/functions/features added | `feat` | New API endpoint |
| Bug fixes (error handling, conditions) | `fix` | Null check fix |
| Only dependencies updated | `chore(deps)` | npm update |
| Only docs/markdown changed | `docs` | README update |
| Only test files changed | `test` | Add unit tests |
| Code restructuring, no behavior change | `refactor` | Extract functions |
| Only formatting/whitespace | `style` | Prettier format |
| Performance optimizations | `perf` | Cache implementation |
| Config/build changes | `build` or `chore` | webpack config |

**User override:** If user provided type argument, use it instead of auto-detection.

**Mixed changes:** Choose the dominant type based on impact. If truly ambiguous, prefer: `feat` > `fix` > `refactor` > `chore`.

#### 3. Detect Scope

Automatically infer scope from file paths:

- **Directory-based**: `src/api/file.js` → `api`, `src/auth/login.js` → `auth`
- **File pattern**: `package.json` deps → `deps`, `*.test.js` → `tests`
- **Common parent**: Multiple files in same dir → use dir name
- **Omit if unclear**: Scope is optional, leave it out if not obvious

**User override:** If user provided scope argument, use it instead of auto-detection.

**Common scopes:**
- `deps` - Dependency updates
- `api` - API changes
- `ui` - UI components
- `auth` - Authentication
- `config` - Configuration files
- `tests` - Test files

#### 4. Detect Breaking Changes

Add an exclamation mark (!) after type/scope if:
- API signature changes (function parameters, return types)
- Removed public functions or exports
- Major version bumps in dependencies
- Database schema changes

#### 5. Generate Description

Create a concise description following these rules:

- **Imperative mood**: Use "add" not "added", "fix" not "fixed"
- **Lowercase**: Start with lowercase after colon
- **No period**: Don't end with period
- **Length**: 50-72 characters ideal
- **Specific**: Mention what changed, not why or how
- **Concise**: Remove unnecessary words

**Examples:**
- ✅ `feat(api): add user authentication endpoint`
- ✅ `fix(auth): resolve token expiration issue`
- ✅ `chore(deps): update express to 4.18.2`
- ❌ `feat(api): Added a new endpoint for user authentication.`
- ❌ `fix: fixed bug`

#### 6. Stage All Relevant Files

Before presenting the message, auto-stage both modified and new files:

**Modified tracked files:**
- Get list from `git diff --name-only HEAD`
- Stage each file individually with specific paths

**Untracked new files:**
- Do NOT auto-stage untracked files. Instead, list them and ask the user which ones to include.
- Display: "Untracked files found:\n- [list]\nWould you like to include any of these in this commit?"

**General rules:**
- **Use specific file paths** — never use `git add -A` or `git add .`
- **Exclude secret files**: .env, .env.*, credentials.json, *.key, *.pem, *secret*, *password*
- If secret files are detected, warn: "Detected potential secret files: [list]. These will NOT be staged."

#### 7. Present Message to User

Display the generated commit message in this format:

```
Generated Conventional Commit message:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
<type>[scope]: <description>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Files staged (X total):
- [list of staged files]

To commit with this message, I can run:
git commit -m "[full message]"

Would you like me to:
1. Commit with this message
2. Commit and push with this message
3. Discuss the message
```

**Important:** Always count and display the total number of files staged. Use `git diff --cached --name-only | wc -l` to get the count.

**Stop here and wait for user response.** Do not commit automatically.

### Phase 2: User Review & Commit

After user reviews the message:

**If user approves:**
- Execute: `git commit -m "<type>[scope]: <description>"`
- Do not include `Co-Authored-By` footer
- Confirm with: `git log -1 --pretty=%B`

**If user provides different message:**
- Use their message exactly as provided
- Do not include `Co-Authored-By` footer

**If user requests changes:**
- Regenerate message with requested modifications
- Present new message for review
- Wait for approval again

## Edge Cases

### No Changes
- Display: "No changes to commit. Working tree is clean."
- Do not proceed with message generation

### Only Staged Changes
- Skip git add step
- Proceed directly with message generation
- Note to user: "Using already staged changes"

### Secret Files Detected
- Never stage: .env, .env.*, credentials.json, *.key, *.pem, *secret*, *password*
- Warn user with file list
- Stage only safe files

### Binary Files
- Detect from diff output (Binary files differ)
- Determine type from context (e.g., assets → `chore`)
- Mention in description if relevant

### Very Large Diffs
- Focus on file names and patterns rather than line-by-line analysis
- Choose safe default type if analysis difficult
- Ask user for clarification if needed

### Mixed Changes (Multiple Concerns)
- Choose dominant type based on impact
- Explain reasoning to user
- Suggest: "Consider splitting into multiple commits for: [list other concerns]"

## Security Considerations

- **Never commit secrets**: Detect and exclude .env, credentials, API keys
- **Selective staging**: Use specific file paths, not `git add -A`
- **Warn on suspicious files**: Alert user if potentially sensitive files detected
- **Tool restrictions**: Only git commands allowed via `allowed-tools`

## Success Criteria

- ✅ Message displayed before committing (two-phase interaction)
- ✅ Format follows Conventional Commits specification
- ✅ Type accurately reflects changes
- ✅ Scope is meaningful or omitted appropriately
- ✅ Description uses imperative mood and is concise
- ✅ Files auto-staged selectively (not git add -A)
- ✅ Secret files excluded from staging
- ✅ User argument overrides work correctly
- ✅ Edge cases handled gracefully
- ✅ User can review and approve before commit
- ✅ Co-Authored-By footer not included
