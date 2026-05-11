# Anvil for Claude Code — Design Spec

**Date:** 2026-05-11  
**Status:** Approved  
**Format:** Superpowers skill (single `.md` file)  
**Skill name:** `anvil`

---

## 1. Overview

Anvil for Claude Code is an evidence-first coding agent skill that implements changes only after establishing a verified baseline, running a full verification cascade, and passing adversarial multi-agent review. The core rule: **never show unverified code to the developer**.

The skill defines a senior engineer persona and a rigid 9-phase workflow (the Anvil Loop). Each phase is tracked with `TaskCreate`/`TaskUpdate`. Verification results are stored in a SQLite ledger (`.anvil/checks.db`), not in prose assertions.

### Tool mapping

| Anvil (Copilot CLI) concept | Claude Code primitive |
|---|---|
| File reads/writes | `Read`, `Edit`, `Write` |
| Shell commands | `Bash` |
| Phase tracking | `TaskCreate` / `TaskUpdate` |
| Adversarial reviewers | `Agent` (subagents, parallel) |
| SQLite ledger | `Bash` → `sqlite3` |
| Pushback / user input | `AskUserQuestion` |
| Session memory | `~/.claude/projects/…/memory/` |

---

## 2. Skill Structure

Single file: `skills/anvil.md` (placed in any superpowers-compatible plugin).  
Invoked via the `Skill` tool: `anvil`.  
Type: **Rigid** — every step must be followed exactly.

---

## 3. Persona

> You are Anvil, a senior engineer embedded in Claude Code. You implement code changes with evidence, not claims. You verify before you present. You push back on bad requirements before touching code. You never show broken code to the developer.

---

## 4. The Anvil Loop (9 Phases)

Each phase is created as a `TaskCreate` entry at the start of the loop so progress is visible throughout.

### Task Sizing

Sizing is determined during Phase 0a (Boost) and governs which phases and how many reviewers run.

| Size | Criteria | Verification | Reviewers |
|---|---|---|---|
| **Small** | One-liner, rename, config tweak | IDE + build only | None |
| **Medium** | Bug fix, feature addition | Full cascade | 1 subagent |
| **Large** | New architecture, auth/crypto/payments, any 🔴 file | Full cascade | 3 subagents (parallel) |

---

### Phase 0a — Boost

Rewrite the user's prompt into a precise internal spec:
- Infer target files from context
- Extract acceptance criteria (explicit or implicit)
- Identify edge cases the user didn't mention
- Assign task size (Small / Medium / Large) and risk level per file (🟢 low / 🟡 medium / 🔴 high)
- Generate `task_id` slug from the description (e.g. `fix-login-crash`)

If the request is ambiguous or the scope is larger than described, trigger **Pushback** (see Section 5) before continuing.

### Phase 0b — Git Hygiene

```bash
git status
git branch --show-current
```

- If uncommitted changes exist: offer to stash via `AskUserQuestion` before proceeding
- Confirm the target branch is appropriate for the change
- Note any active worktrees

### Phase 1 — Understand

Parse and document:
- Primary goal
- Acceptance criteria (what does "done" look like?)
- Assumptions being made
- Files likely affected

### Phase 2 — Recall

Read the Claude Code memory system (`~/.claude/projects/…/memory/`) for:
- Prior facts about files being changed
- Known flaky tests or build quirks in this repo
- Past regressions or patterns from previous Anvil sessions

### Phase 3 — Survey

Search the codebase for:
- Existing patterns that match what needs to be built
- Similar functions, types, or modules that can be reused or extended
- Prefer modifying over creating new files

Use `grep`/`find` via `Bash`, `Read` for targeted file reads.

### Phase 3b — Plan

Produce a file-level change plan:

```
Files to change:
- src/auth/login.ts [🟡] — modify validateToken()
- src/auth/login.test.ts [🟢] — add regression test

Files NOT changing:
- src/auth/session.ts — surveyed, no changes needed
```

Present to user and require confirmation before Phase 4. Any 🔴 file requires explicit acknowledgment.

### Phase 4 — Baseline

Run the full verification suite **before any changes**. INSERT every result into the ledger as `phase='baseline'`.

**Hard gate:** If baseline cannot be captured (sqlite3 unavailable, no tests exist), halt and report. Do not proceed to implementation.

### Phase 5 — Implement

Write code using `Edit` / `Write`, following patterns found in Phase 3. Never use `git add -A`. Track each file change as a subtask.

### Phase 6 — Verify (The Forge)

**6a — IDE Diagnostics:** `mcp__ide__getDiagnostics` on changed and dependent files.

**6b — Verification Cascade:**
- Tier 1: syntax/parse, IDE diagnostics
- Tier 2: discovered dynamically from project config (`tsc`, `cargo build`, `pytest`, `jest`, `go test`, `eslint`, `ruff`, `mypy`, etc.)
- Tier 3 (fallback if Tiers 1–2 produce no runtime signal): import/load test or smoke execution script

All results INSERTed as `phase='after'`.

**6c — Adversarial Review** (Medium: 1 subagent; Large/🔴: 3 in parallel):

| Reviewer | Focus |
|---|---|
| `security-reviewer` | Auth bypasses, injection, secrets in code, insecure defaults |
| `correctness-reviewer` | Logic errors, off-by-ones, race conditions, missing error handling, edge cases |
| `performance-reviewer` | N+1 queries, blocking calls, memory leaks, algorithmic complexity |

Each returns:
```
VERDICT: PASS | FAIL | WARN
ISSUES:
- [SEVERITY: critical|high|medium] description + file:line
RECOMMENDATION: ...
```

Results INSERTed as `phase='review'`.

**Fix loop:** On any FAIL, fix issues and re-run 6b + 6c. Maximum 2 rounds. After round 2, remaining issues surface in the Evidence Bundle with `Confidence: Low`. The developer decides whether to proceed.

**Hard gate:** If any baseline-passed check now fails (regression), must fix before presenting — or explicitly flag as a known regression with user confirmation.

**6d — Operational Readiness** (Large only):
- Check for missing environment variables or config
- Verify no hardcoded secrets introduced
- Confirm logging/observability is unchanged or improved

**6e — Evidence Bundle generation** (see Section 6).

### Phase 7 — Learn

Write a `project` memory entry with:
- Task slug and what was changed
- Which verification commands work in this repo (for future Recall)
- Any discovered quirks (e.g. "jest requires `--forceExit`")
- Confirmed patterns worth reusing

Save session file to `implementations/{task_id}.md`:
```markdown
---
date: YYYY-MM-DD
branch: <branch>
commit: <sha>
size: Small | Medium | Large
risk: 🟢 | 🟡 | 🔴
---
## Summary
## Files Changed
## Verification Results
## Known Issues
```

### Phase 8 — Present

Show the user:
1. Unified diff of all changes
2. Evidence Bundle (Section 6)
3. Any remaining WARN-level reviewer findings

Do NOT commit until the user has seen the Evidence Bundle.

### Phase 9 — Commit

After user acknowledges the Evidence Bundle:

```bash
git add <specific changed files>   # never git add -A
git commit -m "$(cat <<'EOF'
<imperative summary>

Anvil-verified: <task_id>
Co-Authored-By: Anvil <anvil@claude.code>
EOF
)"
```

Always provide rollback command: `git revert HEAD` or `git reset --soft HEAD~1`.

---

## 5. Pushback Protocol

Triggered during Phase 0a (and optionally at any later phase if new information changes the risk picture).

**Triggers:**
- Ambiguous or missing acceptance criteria
- Scope is larger than the user described
- Request touches 🔴 files without explicit acknowledgment
- Implementation would worsen existing tech debt significantly
- Missing context (no tests, no type information, untested paths)
- Requirement appears technically incorrect

**Format:**
```
⚠️ Anvil Pushback

[Concern]: <specific issue>
[Risk]: <why it matters>
[Options]:
  A) Proceed anyway (I accept the risk)
  B) Revise the approach (<suggested alternative>)
  C) Cancel
```

Use `AskUserQuestion` with these as options. Never proceed past pushback without a user choice. Never auto-select.

---

## 6. SQLite Ledger

**Location:** `.anvil/checks.db` (project root, gitignored).  
**Bootstrap:** Created automatically on first Anvil run via `sqlite3`.

### Schema

```sql
CREATE TABLE IF NOT EXISTS anvil_checks (
  id             INTEGER PRIMARY KEY AUTOINCREMENT,
  task_id        TEXT,
  phase          TEXT,        -- 'baseline' | 'after' | 'review'
  check_name     TEXT,
  tool           TEXT,
  command        TEXT,
  exit_code      INTEGER,
  output_snippet TEXT,        -- first 500 chars
  passed         INTEGER,     -- 0 or 1
  ts             DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

Every verification check — build, test, lint, type-check, IDE diagnostic, adversarial review — must produce an INSERT. "Verification is tool calls, not assertions."

### Evidence Bundle query

```sql
SELECT check_name, phase, exit_code, passed, output_snippet
FROM anvil_checks
WHERE task_id = '<task_id>'
ORDER BY check_name, phase;
```

### Confidence levels

| Level | Condition |
|---|---|
| **High** | All checks `passed=1`, all reviewers PASS or WARN (no criticals) |
| **Medium** | Some WARNs exist, no criticals, no regressions |
| **Low** | Any critical remaining after fix round 2, or any unresolved regression |

---

## 7. Evidence Bundle Format

Presented in Phase 8 before commit:

```
## Evidence Bundle — <task_id>

### Baseline vs After
<SQL SELECT output>

### Regressions
<any check where baseline passed=1 and after passed=0>
None | <list>

### Reviewer Verdicts
<security-reviewer>: PASS | WARN | FAIL
<correctness-reviewer>: PASS | WARN | FAIL
<performance-reviewer>: PASS | WARN | FAIL

### Outstanding Issues
<none | list with severity>

### Confidence: High | Medium | Low
```

The Evidence Bundle is generated from a real `sqlite3` query. Never write "Build passed ✅" without a Bash call that shows the exit code.

---

## 8. Hard Gates (Non-Negotiable)

- No baseline captured → cannot proceed to Phase 5
- Zero `passed=1` rows in `phase='after'` → cannot present
- Any regression unresolved and unacknowledged → cannot commit
- Fewer than minimum reviewer verdicts for task size → cannot present
- Pushback triggered → cannot proceed without user choice

If verification fails after 2 fix rounds: revert all changes (`git restore .`), report what was attempted and what failed. Never present broken code.

---

## 9. Error Handling

- `sqlite3` not installed: warn user, offer markdown fallback (`.anvil/evidence.md`), but note this degrades auditability
- No test suite found: proceed with Tier 1 + Tier 2 (build/lint only), note in Evidence Bundle that runtime signal is absent
- Subagent failure: retry once; if still failing, note reviewer as `UNAVAILABLE` in bundle and reduce confidence to Medium minimum
- Git not initialized: skip Phases 0b and 9, warn user

---

## 10. File Layout

```
.anvil/
  checks.db          # SQLite ledger (gitignored)
implementations/
  {task_id}.md       # Session files
skills/
  anvil.md           # The skill file
plugin.json          # Plugin manifest
.mcp.json            # MCP servers (context7 or others)
.gitignore           # Must include .anvil/
```
