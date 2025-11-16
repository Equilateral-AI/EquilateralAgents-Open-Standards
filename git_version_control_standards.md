# Git and Version Control Standards

**Purpose**: Establish consistent git practices for atomic commits, clear history, and collaborative development across all Equilateral projects.

**Scope**: All projects using git for version control, including application code, infrastructure as code, documentation, and configuration.

**Core Principle**: Git history should tell a clear story of how the codebase evolved, with each commit representing a complete, working, atomic change.

---

## 1. Atomic Commits

### Commit Frequently and Atomically

**Standard**: Commit whenever you complete an atomic unit of functionality — a discrete feature, bug fix, or substantial logical chunk — regardless of how many files changed.

**Atomic Unit Definition**:
- A single feature addition (e.g., "Add user authentication endpoint")
- A single bug fix (e.g., "Fix null pointer in date validation")
- A single refactoring (e.g., "Extract database connection logic to helper")
- A single documentation update (e.g., "Document API authentication flow")
- A single configuration change (e.g., "Update Lambda memory to 512MB")

**Anti-Pattern**: Waiting until "everything is done" to commit
```bash
# ❌ BAD: One massive commit at end of day
git add .
git commit -m "Fixed bugs and added features"

# ✅ GOOD: Multiple atomic commits throughout development
git commit -m "Add email validation to user registration"
git commit -m "Fix CORS headers in API Gateway response"
git commit -m "Extract credential management to UnifiedCredentialManager"
```

### Each Commit Should Be a Working State

**Standard**: Every commit should represent a coherent, working state where:
- Code compiles/builds without errors
- Tests pass (or are updated to pass)
- Application functionality is preserved or enhanced
- No broken references or missing dependencies

**Rationale**:
- Enables safe `git bisect` for debugging
- Allows reverting to any commit without breaking the system
- Facilitates code review by showing logical progression
- Supports cherry-picking commits to other branches

**Exception**: Work-in-progress branches may have intermediate non-working commits, but these should be squashed before merging to main branches.

---

## 2. Commit Message Standards

### Imperative Mood

**Standard**: Write commit messages in imperative mood (as if giving a command), not past tense.

**Pattern**: "This commit will [commit message]"

```bash
# ✅ GOOD: Imperative mood
git commit -m "Add multi-stack architecture standards"
git commit -m "Fix credential expiration monitoring"
git commit -m "Update API Gateway route configuration"
git commit -m "Remove deprecated Lambda functions"

# ❌ BAD: Past tense
git commit -m "Added multi-stack architecture standards"
git commit -m "Fixed credential expiration monitoring"
git commit -m "Updated API Gateway route configuration"
```

**Why Imperative Mood**:
- Matches git's own message format (e.g., "Merge branch 'feature'")
- Reads naturally with "This commit will..."
- Industry standard followed by Linux kernel, Git itself, and major projects

### Commit Message Structure

**Standard**: Use structured commit messages with summary, body, and footer.

```
<type>: <summary> (50 characters or less)

<body - explain what and why, not how> (wrap at 72 characters)

<footer - references, breaking changes, co-authors>
```

**Commit Types**:
- `feat`: New feature
- `fix`: Bug fix
- `refactor`: Code restructuring without behavior change
- `docs`: Documentation only
- `style`: Formatting, missing semicolons, etc.
- `test`: Adding or updating tests
- `chore`: Maintenance tasks, dependencies, build config
- `perf`: Performance improvements
- `security`: Security fixes or improvements

**Examples**:

```bash
# Simple commit (summary only)
git commit -m "fix: Resolve null pointer in date validation"

# Detailed commit (summary + body)
git commit -m "$(cat <<'EOF'
feat: Add ADP Marketplace webhook endpoints

Implement 5 webhook handlers for ADP Marketplace subscription events:
- OAuth token endpoint for bidirectional authentication
- Subscription create, cancel, change, and status handlers
- Integration with UnifiedCredentialManager for secure credential storage

All endpoints use system-wide credentials (SYSTEM_COMPANY_ID) rather than
hardcoded environment variables to support automatic credential rotation.

Refs: #123
EOF
)"

# Full structured commit (summary + body + footer)
git commit -m "$(cat <<'EOF'
feat: Add multi-stack architecture standards

Comprehensive standards for managing multi-stack CloudFormation deployments
when AWS service limits require splitting infrastructure across multiple
stacks and API Gateways.

Key features:
- Multi-stack best practices for 4+ CloudFormation stacks
- Persistent vs ephemeral resource separation patterns
- API_BASE and API_BASE2 dual API Gateway configuration
- Cross-stack reference patterns using CloudFormation exports/imports

Reference implementation from HappyHippo project demonstrating 4 stacks,
2 API Gateways, and 255+ Lambda functions.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

### Summary Line Guidelines

**Standard**: First line (summary) should:
- Be 50 characters or less (hard limit: 72)
- Start with lowercase (unless proper noun or acronym)
- Not end with a period
- Complete the sentence "This commit will..."

```bash
# ✅ GOOD
fix: resolve CORS error in API Gateway
feat: add credential health monitoring
docs: document multi-stack architecture
refactor: extract response formatting to helper

# ❌ BAD
Fix: Resolve CORS error in API Gateway.  # Capitalized, has period
Added credential health monitoring       # Past tense, no type
Updated some stuff                       # Vague, no type
This commit adds multi-stack architecture standards  # Too wordy
```

### Body Guidelines

**Standard**: Commit body should:
- Wrap at 72 characters
- Explain **what** and **why**, not **how** (code shows how)
- Include context for future maintainers
- Reference related issues, tickets, or documentation

**Good Body Content**:
- Why the change was necessary
- What problem it solves
- What alternatives were considered
- Any side effects or migration requirements
- References to related commits or issues

**Example**:
```
feat: Migrate to UnifiedCredentialManager for ADP credentials

Previously, ADP Marketplace credentials were stored as environment
variables in Lambda functions, requiring redeployment whenever
credentials rotated. This created operational overhead and potential
downtime during credential updates.

Now credentials are stored encrypted in database and retrieved at
runtime via UnifiedCredentialManager. This allows credential rotation
without code deployment and provides audit trail for compliance.

Breaking Change: Requires running migration script to move existing
credentials from environment variables to database.

Refs: #456
```

---

## 3. Branching Strategy

### Branch Naming Convention

**Standard**: Use descriptive branch names with type prefix:

```
<type>/<short-description>

Types:
- feature/  - New functionality
- fix/      - Bug fixes
- refactor/ - Code restructuring
- docs/     - Documentation updates
- chore/    - Maintenance, dependencies, build
- hotfix/   - Urgent production fixes
```

**Examples**:
```bash
feature/adp-marketplace-webhooks
feature/multi-stack-architecture
fix/cors-header-missing
fix/credential-expiration-alert
refactor/extract-credential-manager
docs/api-authentication-guide
chore/update-lambda-runtime-nodejs18
hotfix/production-api-timeout
```

### Branch Lifecycle

**Standard**:
1. **Create**: Branch from `main` for new work
2. **Develop**: Commit atomically and frequently
3. **Review**: Create PR when feature is complete
4. **Merge**: Squash or rebase before merging to main
5. **Delete**: Remove branch after successful merge

**Main Branch Protection**:
- Never commit directly to `main` in shared repositories
- Require pull request reviews for main branch
- Maintain clean, linear history on main branch

---

## 4. Incremental Commits

### Commit as You Complete Meaningful Pieces

**Standard**: Don't wait until everything is done; commit incrementally as you complete meaningful pieces.

**Incremental Commit Example** (ADP Marketplace implementation):
```bash
# Commit 1: Infrastructure
git commit -m "feat: Add SAM template for ADP webhook endpoints"

# Commit 2: Core handler
git commit -m "feat: Add OAuth token handler for ADP authentication"

# Commit 3: Webhook handlers
git commit -m "feat: Add subscription event webhook handlers"

# Commit 4: Integration
git commit -m "feat: Integrate webhooks with UnifiedCredentialManager"

# Commit 5: Monitoring
git commit -m "feat: Add credential health monitoring to Super Dashboard"

# Commit 6: Documentation
git commit -m "docs: Document ADP Marketplace webhook setup"

# NOT: One commit "feat: Add ADP Marketplace integration" with all changes
```

**Benefits**:
- Easier code review (smaller, focused diffs)
- Clearer understanding of implementation progression
- Ability to revert specific parts without losing everything
- Better collaboration (smaller merge conflicts)
- Natural documentation of thought process

### When to Commit

**Commit After**:
- ✅ Completing a handler function and its tests
- ✅ Adding a new database migration
- ✅ Updating infrastructure template
- ✅ Fixing a specific bug
- ✅ Adding comprehensive documentation section
- ✅ Refactoring a single module
- ✅ Updating dependencies (per dependency or related group)

**Don't Commit**:
- ❌ Half-written functions that don't compile
- ❌ Commented-out debugging code
- ❌ Uncommitted merge conflict markers
- ❌ Temporary test files (unless gitignored)
- ❌ API keys or secrets (should be gitignored)

---

## 5. Working with Claude Code / AI Assistants

### AI-Generated Code Commits

**Standard**: When Claude Code or other AI assistants generate code:
1. Review all changes before committing
2. Use the standard commit message format
3. Include "Co-Authored-By: Claude" in footer
4. Commit atomically (don't batch unrelated AI changes)

**Claude Code Commit Pattern**:
```bash
git commit -m "$(cat <<'EOF'
feat: Add multi-stack architecture standards

Comprehensive standards for managing multi-stack CloudFormation deployments
when AWS service limits require splitting infrastructure across multiple
stacks and API Gateways.

Includes best practices for persistent vs ephemeral resource separation,
dual API Gateway configuration, and cross-stack reference patterns.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

### AI Assistant Workflow

**Standard**: When working with AI assistants:
1. AI generates code → **Review** → **Test** → **Commit**
2. Don't blindly commit AI-generated code
3. Ensure each commit still represents a working state
4. Break large AI-generated changes into atomic commits

**Example**:
```bash
# AI generated 10 files for new feature
# Don't commit all at once - break into logical pieces:

git add src/backend/src/handlers/marketplace/*.js
git commit -m "feat: Add ADP Marketplace webhook handlers"

git add IAC/cloudformation/lambda-stacks/lambda-integrations-adp-marketplace-additions.yaml
git commit -m "feat: Add CloudFormation template for webhook infrastructure"

git add src/frontend/src/features/super/components/ADPMarketplaceCredentialHealth.tsx
git commit -m "feat: Add credential health monitoring UI component"

git add docs/ADP_MARKETPLACE_WEBHOOK_SETUP.md
git commit -m "docs: Document ADP Marketplace webhook setup process"
```

---

## 6. Commit Hygiene

### Before Committing

**Checklist**:
- [ ] Code builds/compiles without errors
- [ ] Tests pass (or are updated to pass)
- [ ] No commented-out debugging code
- [ ] No console.log statements (unless intentional logging)
- [ ] No hardcoded credentials or API keys
- [ ] Files properly formatted (run linter/formatter)
- [ ] Commit message follows standards
- [ ] Only related changes included (no accidental file changes)

### Git Add Best Practices

**Standard**: Use selective staging, not `git add .` blindly

```bash
# ❌ BAD: Adds everything, including unintended changes
git add .

# ✅ GOOD: Add specific files or patterns
git add src/backend/src/handlers/marketplace/
git add IAC/cloudformation/lambda-stacks/lambda-integrations-adp-marketplace-additions.yaml
git add docs/ADP_MARKETPLACE_WEBHOOK_SETUP.md

# ✅ GOOD: Interactive staging for partial file commits
git add -p src/backend/src/helpers/responseUtil.js

# ✅ GOOD: Review what will be committed
git status
git diff --cached
```

### Amending Commits

**Standard**: Only amend commits that haven't been pushed to shared branches

```bash
# ✅ SAFE: Amend local commit before pushing
git commit -m "feat: Add webhook handler"
# (realize you forgot to add a file)
git add missed-file.js
git commit --amend --no-edit

# ❌ DANGEROUS: Amending after pushing to shared branch
git push
git commit --amend  # This rewrites history others may have pulled
```

**When to Amend**:
- Fix typo in commit message before pushing
- Add forgotten file to commit before pushing
- Adjust commit based on pre-commit hook feedback

**When NOT to Amend**:
- After pushing to shared branch (use new commit instead)
- To combine unrelated changes (make separate commits)
- To fix bugs found later (make new commit with "fix:" type)

---

## 7. Special Cases

### Documentation-Only Commits

**Standard**: Documentation changes warrant their own commits

```bash
# ✅ GOOD: Separate documentation commits
git commit -m "docs: Add API authentication guide"
git commit -m "docs: Update deployment procedures for multi-stack"
git commit -m "docs: Document credential rotation process"

# ❌ BAD: Mixing docs with code changes (unless directly related)
# If updating docs because API changed, include in same commit
# If updating docs independently, separate commit
```

### Refactoring Commits

**Standard**: Refactoring should be separate from behavior changes

```bash
# ✅ GOOD: Separate refactor from feature
git commit -m "refactor: Extract credential management to helper"
git commit -m "feat: Add ADP credential rotation support"

# ❌ BAD: Mixing refactor with new feature
git commit -m "Add credential rotation and refactor credential management"
```

**Rationale**: If new feature has bugs, you want to keep refactoring benefits without reverting the refactor too.

### Emergency Hotfixes

**Standard**: Hotfixes can be less atomic if time-critical, but should be cleaned up

```bash
# Initial hotfix (may be messy)
git commit -m "hotfix: Disable failing ADP webhook temporarily"

# Follow-up cleanup commits
git commit -m "fix: Resolve ADP webhook authentication issue"
git commit -m "feat: Re-enable ADP webhook with proper error handling"
git commit -m "test: Add integration tests for ADP webhook failure scenarios"
```

### Database Migrations

**Standard**: Database migrations should be committed separately

```bash
# ✅ GOOD: Migration separate from implementation
git commit -m "chore: Add database migration for marketplace subscription tables"
git commit -m "feat: Add marketplace subscription handlers using new tables"

# Rationale: Migrations may need to run independently in production
```

---

## 8. Collaboration Best Practices

### Pull Request Commits

**Standard**: Before creating pull request:
1. Review all commits for atomic, meaningful units
2. Consider squashing "fix typo" and "oops forgot file" commits
3. Ensure commit messages follow standards
4. Rebase on latest main to avoid merge conflicts

**Interactive Rebase** (for cleaning up local commits):
```bash
# Rebase last 5 commits interactively
git rebase -i HEAD~5

# Options:
# pick   = use commit
# squash = merge with previous commit
# reword = change commit message
# edit   = amend commit content
# drop   = remove commit
```

### Merge Strategies

**Standard**: Choose merge strategy based on context

**Squash Merge** (feature branches → main):
- Use for feature branches with many small commits
- Maintains clean linear history on main
- Single commit represents entire feature

**Rebase Merge** (keeping individual commits):
- Use when commits already represent meaningful atomic units
- Preserves detailed implementation history
- Good for complex features with logical progression

**Regular Merge** (avoid):
- Creates merge commits that clutter history
- Use only for long-running release branches

```bash
# Squash merge
gh pr merge --squash --delete-branch

# Rebase merge
gh pr merge --rebase --delete-branch
```

---

## 9. Anti-Patterns to Avoid

### Don't

❌ **Batch unrelated changes**
```bash
git commit -m "Various fixes and updates"  # Too vague
```

❌ **Commit broken code** (except WIP branches)
```bash
git commit -m "WIP: Half-finished feature"  # Don't push to main branch
```

❌ **Use vague messages**
```bash
git commit -m "Updates"
git commit -m "Fixed stuff"
git commit -m "Changes"
```

❌ **Commit sensitive data**
```bash
git add .env  # Should be in .gitignore
git add credentials.json  # Should never be committed
```

❌ **Rewrite public history**
```bash
git push --force  # Only acceptable on personal feature branches
```

❌ **Wait days between commits**
```bash
# Working for 3 days...
git add .
git commit -m "Finished entire feature"  # Should be many atomic commits
```

---

## 10. Validation and Enforcement

### Pre-Commit Hooks

**Standard**: Use pre-commit hooks to enforce commit standards

**Example `.git/hooks/pre-commit`**:
```bash
#!/bin/bash

# Run linter
npm run lint || exit 1

# Run formatter check
npm run format:check || exit 1

# Run tests (or at least fast unit tests)
npm run test:unit || exit 1

# Check for console.log statements
if git diff --cached | grep -E "console\.(log|debug)" > /dev/null; then
    echo "Error: Found console.log statements. Remove before committing."
    exit 1
fi

# Check for secrets
if git diff --cached | grep -E "(password|secret|key).*=.*['\"].*['\"]" > /dev/null; then
    echo "Warning: Possible hardcoded secret detected. Review before committing."
    exit 1
fi
```

### Commit Message Validation

**Standard**: Use commit message hooks to enforce format

**Example `.git/hooks/commit-msg`**:
```bash
#!/bin/bash

commit_msg=$(cat "$1")

# Check for type prefix
if ! echo "$commit_msg" | grep -qE "^(feat|fix|refactor|docs|style|test|chore|perf|security):"; then
    echo "Error: Commit message must start with type prefix (feat, fix, refactor, docs, etc.)"
    echo "Example: feat: Add new feature"
    exit 1
fi

# Check summary length
first_line=$(echo "$commit_msg" | head -n1)
if [ ${#first_line} -gt 72 ]; then
    echo "Error: Commit summary must be 72 characters or less"
    exit 1
fi

# Check for imperative mood (basic check)
if echo "$first_line" | grep -qE "(added|fixed|updated|created|deleted)"; then
    echo "Warning: Use imperative mood (add, fix, update) not past tense (added, fixed)"
fi
```

---

## 11. Git Configuration

### Recommended Git Config

```bash
# Set your identity
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# Use main as default branch name
git config --global init.defaultBranch main

# Enable color output
git config --global color.ui auto

# Set default editor
git config --global core.editor "code --wait"  # VS Code
# or
git config --global core.editor "nano"  # Nano

# Enable helpful aliases
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.unstage 'reset HEAD --'
git config --global alias.last 'log -1 HEAD'
git config --global alias.visual 'log --oneline --graph --decorate --all'

# Better diff output
git config --global diff.algorithm histogram

# Automatically prune deleted remote branches
git config --global fetch.prune true

# Use rebase instead of merge for git pull
git config --global pull.rebase true
```

---

## 12. Tools and Resources

### Recommended Tools

**Git GUI Clients**:
- GitHub Desktop (beginner-friendly)
- GitKraken (visual, powerful)
- Tower (macOS, professional)
- Built-in IDE git tools (VS Code, IntelliJ)

**Commit Message Helpers**:
- Commitizen (`npm install -g commitizen`)
- Conventional Commits CLI
- Git commit templates

**Pre-commit Framework**:
- [pre-commit](https://pre-commit.com/) - Multi-language pre-commit hooks
- Husky (for Node.js projects)

### Learning Resources

- [Conventional Commits](https://www.conventionalcommits.org/)
- [Git Book](https://git-scm.com/book/en/v2)
- [GitHub Flow](https://docs.github.com/en/get-started/quickstart/github-flow)
- [Semantic Versioning](https://semver.org/)

---

## Quick Reference

### Commit Checklist

Before every commit:
- [ ] Changes represent atomic, complete unit of work
- [ ] Code compiles/builds without errors
- [ ] Tests pass or are updated
- [ ] Commit message uses imperative mood
- [ ] Commit message has type prefix (feat, fix, docs, etc.)
- [ ] No debug code, console.logs, or secrets
- [ ] Only related files staged

### Commit Message Template

```
<type>: <summary in imperative mood, ≤50 chars>

<body explaining what and why, wrapped at 72 chars>

<footer with references, breaking changes, co-authors>
```

### Common Commit Types

- `feat:` New feature
- `fix:` Bug fix
- `docs:` Documentation
- `refactor:` Code restructure
- `test:` Tests
- `chore:` Maintenance
- `perf:` Performance
- `security:` Security

---

**Related Standards**:
- `development_principles.md` - Overall development philosophy
- `testing_principles.md` - Testing and quality standards
- `api_design_standards.md` - API development practices

**Last Updated**: 2025-11-16
**Version**: 1.0.0
