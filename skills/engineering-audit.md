---
name: engineering-audit
description: Use when the user wants to perform a software engineering audit, release readiness review, pre-beta or pre-launch technical review, or quality gate assessment on a codebase. Walks through repository discovery, technical audit, product decision gates, remediation, independent verification, validation, user journey review, and a final release report.
---

# Software Engineering Audit & Release Readiness

Follow these stages in order. Do not skip or compress stages. Update the todo list as you
progress. Never declare work complete until Stage 6 validation passes.

---

## Operating Principles (enforce throughout)

- **Never trust prior summaries.** Verify every finding directly against current source code.
- **Verify before fixing.** Confirm each issue exists in the actual file and on the actual line before writing any fix.
- **Separate product decisions from software defects.** Route each finding to the correct stage.
- **Identify false positives.** Challenge your own findings — a finding that cannot be reproduced is a false positive; mark it as such.
- **Implementation and verification are separate phases.** After implementing fixes, act as an independent auditor in Stage 5 — do not rely on memory of what you just wrote.
- **Never declare work complete until type checks, builds, tests, and regression checks pass.**
- **Prioritize release blockers and user-impacting problems** over finding-count reduction.

---

## Stage 1 — Repository Discovery

**Goal:** Produce an architecture map before opening any source file.

1. Use `list_files` and `glob` to survey the top-level structure.
2. Identify layers present:
   - Frontend (framework, router, state management, component library)
   - Backend / API (framework, language, transport — REST/GraphQL/tRPC)
   - Database (engine, ORM/query builder, schema tool, migration strategy)
   - Authentication and session handling
   - Third-party integrations and external dependencies
   - Test infrastructure (unit, integration, e2e)
   - Build pipeline (bundler, compiler, CI)
   - Deployment configuration (hosting, environment variables, secrets management)
   - Documentation
3. Detect:
   - Package manager (npm / pnpm / yarn / cargo / pip / etc.)
   - Build commands (from `package.json`, `Makefile`, `Cargo.toml`, etc.)
   - Test runner commands
   - Type-check commands
4. Produce a concise **Architecture Map** in a markdown table:

   | Layer | Technology | Entry Point | Notes |
   |-------|-----------|-------------|-------|
   | ...   | ...       | ...         | ...   |

5. Read the README and any architecture docs before proceeding to Stage 2.

---

## Stage 2 — Technical Audit

**Goal:** Produce a classified finding list. Read source files; do not speculate.

Work through the checklist in `audit-checklist.md` (in this skill directory). For each area,
open the relevant source files and read the actual code before writing a finding.

Classify every finding using exactly one label:

| Label | Meaning |
|-------|---------|
| **Critical** | Causes data loss, security breach, crash at runtime, or complete feature failure |
| **High** | Significant user impact, data inconsistency, or missing feature with no workaround |
| **Medium** | Degraded UX, minor data issues, inconsistent state that users may notice |
| **Low** | Code quality, documentation gaps, naming inconsistencies |
| **False Positive** | Initially appeared to be an issue; source code disproves it |
| **Product Decision Required** | Cannot be resolved without a business/product choice |
| **Known External Dependency** | Caused by a third-party constraint outside this codebase |

Format each finding as:

```
### [SEVERITY]-[NN] · Short title
**Severity:** [label]
**Location:** [file:line]
**Steps to reproduce:** [concrete steps a tester can follow]
**Root cause:** [exact mechanism — reference actual code]
**Recommended fix:** [smallest safe change]
**Effort:** [time estimate]
```

Do not report the same issue twice. If two symptoms share a root cause, file one finding and note
the surface areas it affects.

---

## Stage 3 — Product Decision Gate

**Trigger:** Any finding classified as **Product Decision Required**.

**Do not proceed with implementation until each product decision is resolved.**

For each pending decision, present to the user:

1. **Current behavior** (what the code does today)
2. **Available options** (enumerate at least two)
3. **Tradeoffs** per option (user impact, engineering effort, data migration risk)
4. **Recommended choice** with rationale

Wait for an explicit confirmation from the user before moving to Stage 4.
Document the confirmed decision alongside its finding.

---

## Stage 4 — Remediation

**Trigger:** All Critical and High findings that are not False Positives, Product Decisions, or
External Dependencies. Medium and Low findings as time permits.

For each confirmed issue:

1. **State the root cause** (copy from Stage 2 finding).
2. **List the files to be changed** before touching anything.
3. **Implement the smallest safe fix.** Do not refactor surrounding code. Do not add features.
4. **Add or update a regression test** for the fix where a test is feasible.
5. **Mark the finding** as `[FIXED]` in the audit document and note the commit or change summary.

Use `apply_diff` for surgical edits. Use `write_file` only for new files.
After all fixes in a batch are applied, proceed immediately to Stage 5.

---

## Stage 5 — Independent Verification

**Mindset:** You are a new auditor who has never seen this codebase. Do not rely on memory of
what you implemented in Stage 4.

For **each finding** that was marked `[FIXED]`, re-open the original file and verify:

- Read the specific file and line cited in the finding.
- Confirm the fix is present and correct.
- Confirm no regression was introduced nearby.

Report each finding with one of:

| Verdict | Meaning |
|---------|---------|
| ✅ Verified Fixed | Source code evidence confirms the fix |
| ⚠️ Partially Fixed | Fix is present but incomplete |
| ❌ Still Reproducible | Issue remains in source |
| 🔵 False Positive | The original finding was incorrect |
| 📋 Product Decision | Behavior is intentional per Stage 3 decision |
| 🔗 Known Dependency | Blocked by an external constraint |

Use the verification checklist in `verification-checklist.md` (in this skill directory).

Include actual source-code snippets as evidence. Do not write "I verified this" without a
code excerpt.

---

## Stage 6 — Validation

Run all available validation commands. Report each one explicitly.

Minimum required checks (where available):

1. **Backend type check** — e.g. `tsc --noEmit`, `cargo check`, `mypy`
2. **Frontend type check** — same tool as above for the frontend workspace
3. **Lint** — e.g. `eslint`, `clippy`, `flake8`
4. **Production build** — e.g. `npm run build`, `cargo build --release`
5. **Unit tests** — e.g. `vitest run`, `jest --ci`, `cargo test`
6. **Integration tests** — if configured
7. **End-to-end / smoke tests** — if configured

For each check, report:

```
[PASS] npm run typecheck — 0 errors
[FAIL] npm run build — 3 errors (see below)
[SKIP] e2e tests — not configured in this project
```

**Do not report "all tests pass" if tests were skipped, excluded, or not configured.**
Report those limitations explicitly as `[SKIP]` with a reason.

If any check fails, return to Stage 4 and fix before proceeding.

---

## Stage 7 — First-Time User Journey Audit

**Mindset:** You are a new user who has never seen this product.

Use the journey checklist in `user-journey-checklist.md` (in this skill directory) to trace
each flow. For each flow, evaluate:

- Does it complete without a dead end?
- Is every success message accurate (no false positives)?
- Are error messages actionable?
- Are there missing recovery paths?
- Is pricing, terminology, and copy consistent?
- Would a new user trust this screen?

Identify **trust-breaking UX** — anything that would cause a reasonable user to abandon the
product.

Report each flow as:

```
### [Flow name]
**Status:** ✅ Complete / ⚠️ Incomplete / ❌ Broken
**Issue:** [description, or "None"]
**Severity:** [Critical / High / Medium / Low / None]
```

---

## Stage 8 — Beta Readiness Triage

Collect all open findings from Stages 2–7. Classify each remaining issue:

| Priority | Meaning |
|----------|---------|
| **Must Fix Before Beta** | Blocks core user flows, causes data loss, breaks trust |
| **Should Fix During Beta** | Degrades experience but has a workaround |
| **Post-Beta** | Polish, optimization, low-traffic edge cases |
| **False Positive** | Confirmed not an issue |
| **External Dependency** | Blocked by third party |

For each **Must Fix Before Beta** item include:

- User impact (who is affected and how)
- Engineering effort (estimated hours)
- Technical risk (low / medium / high)
- Dependencies on other fixes

Present findings in priority order. Produce a recommended fix sequence.

---

## Stage 9 — Release Report

Use the template in `release-report-template.md` (in this skill directory).

Populate every section. Do not leave placeholder text.

Conclude with a **Go / Go With Conditions / No-Go** recommendation.

- **Go:** All Critical and High findings resolved; validation passes; no Must Fix items open.
- **Go With Conditions:** Critical findings resolved; High findings documented with mitigation; validation passes.
- **No-Go:** Any unresolved Critical finding, failing build, or must-fix blocker.

Include a recommended Git commit message and tag name for the release.

---

## Supporting Files

The following files are in this skill directory and should be referenced at the appropriate stage:

| File | Used in Stage |
|------|---------------|
| `audit-checklist.md` | Stage 2 |
| `verification-checklist.md` | Stage 5 |
| `user-journey-checklist.md` | Stage 7 |
| `release-report-template.md` | Stage 9 |
| `example-invocations.md` | Reference |
