# AI Assistant Configuration Standards

**Configuring AI coding assistants for automatic standards enforcement**

---

## 1. Use "Personality Prompts"

**Rule:** Configure AI assistants with project standards, conventions, and war stories in a config file they read on startup.

**Why:**
- Standards enforced automatically on every code generation
- AI references existing patterns instead of reinventing solutions

### What to Include

**War stories with context:**
```markdown
### "No Mocks, No Fallback Data"
**Why:** Mocks hid real failures. Tests passed. Production failed.
**Rule:** Fail fast in dev, fail loud. Never hide integration failures.
```

**Mandatory workflow checklist:**
```markdown
Before ANY code change:
1. Check standards - does pattern exist?
2. Follow existing patterns
3. Design errors first
4. Document what you checked
```

**Banned patterns:**
```javascript
// ❌ Mock fallbacks - hides failures
if (apiCallFails) return mockData;

// ❌ Vague errors - not actionable
throw new Error('Bad request');
```

**Trigger words:**
```
"mock" → Hiding integration? Don't.
"fallback" → Masking failure? Don't.
"any"/"unknown" → Escaping types? Use proper types.
```

## 2. Frame AI as a Role

**Rule:** Give AI a specific role that aligns with your goals.

**Why:** Activates relevant training patterns and sets behavioral expectations.

**Examples:**
```markdown
# Quality-focused developer → Standards enforcement
# Technical team lead → Agent coordination
# Security-conscious engineer → Security reviews
```

## 3. Include Specific Examples

**Rule:** Show good vs bad code examples. AI learns from patterns better than rules.

**Example:**
```javascript
// ❌ Bad - silent failure
try { riskyOp(); } catch (e) { console.log(e); }

// ✅ Good - fail loud with context
try {
    riskyOp();
} catch (error) {
    console.error('❌ Operation failed:', { error: error.message });
    throw new Error(`Operation failed: ${error.message}`);
}
```

## 4. Configuration File Formats

**Rule:** Use AI-native config format in project root, git-tracked.

**Formats:**
- Claude Code: `CLAUDE.md`
- Cursor: `.cursorrules`
- Continue: `.continuerc.json`
- GitHub Copilot: `.github/copilot-instructions.md`

**Best practices:**
- ✅ Project root, git-tracked, one canonical file
- ❌ Hidden in subdirectories, not version controlled

## 5. Make It Living Documentation

**Rule:** Update config after incidents, new patterns, or standards changes.

**When to update:**
- After major incidents (add war story)
- When adding new patterns
- When standards evolve

**Example:**
```markdown
### "Never Use DefaultAuthorizer" - CORS Disaster
**What Happened:** Broke CORS preflight. 2 days debugging.
**Rule:** Use explicit per-function authorization only.
**Date:** 2025-09-03
```

**Process:** Incident → Add war story → Git commit → Team pulls → AI learns

## 6. Balance Detail with Readability

**Rule:** Be specific enough to guide behavior, concise enough to scan quickly.

**Target length:** 300-500 lines

**Structure:**
```
1. Role (1 paragraph)
2. Critical standards (5-10 principles with why)
3. Mandatory workflow (4-6 steps)
4. Banned patterns (5-10 examples)
5. Trigger words (10-15 words)
```

**Writing style:**
- ✅ Direct, imperative, actionable
- ❌ Vague, theoretical, wishy-washy

## 7. Test and Iterate

**Rule:** Verify config actually changes AI behavior with before/after tests.

**Test method:**
1. Ask AI to implement feature WITHOUT config
2. Note violations (mocks, vague errors, etc.)
3. Add config
4. Ask same question in new session
5. Verify AI now follows standards

**Common fixes:**
- AI ignores prompt → Make instructions more direct, add war stories
- AI over-applies rules → Add "when NOT to apply" sections
- AI references wrong patterns → Be more specific about file locations

---

## Summary

1. **Use Personality Prompts** - Standards, conventions, war stories in config file
2. **Frame as a Role** - Quality-focused developer, team lead, security engineer
3. **Include Examples** - Good vs bad code
4. **Choose Right Format** - CLAUDE.md, .cursorrules, etc. in project root
5. **Keep It Living** - Update after incidents and pattern changes
6. **Balance Detail** - 300-500 lines, direct and actionable
7. **Test and Iterate** - Verify behavior changes

See [CLAUDE.md.template](./CLAUDE.md.template) for complete template.

---

**Questions?** Contact info@happyhippo.ai
