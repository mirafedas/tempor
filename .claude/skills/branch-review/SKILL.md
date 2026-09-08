---
name: branch-review
description: Deep MAS-aware PR review for Milo or MAS pull requests using parallel domain agents (security, correctness, MAS contracts, coverage). Default to multi-agent fan-out; degrade to single-pass for narrow PRs. Activates on "/branch-review".
tags: [pr, review, multi-agent, mas, milo]
triggers:
  - "/branch-review"
---

# MAS-aware branch review

## Purpose

One skill for reviewing MAS PRs. Default behavior fans out four parallel domain agents (security, correctness/races, MAS-side integration, coverage/dead-code), dedupes and verifies their findings, then outputs their findings in the markdown format grouped by files.

Falls back to a single-agent pass for genuinely narrow PRs.


## Phase 1 — Fetch

Identify the current branch name. Fetch the diff and changed-files list, and cache them to a local path for all agents to read. Run via Bash:
BRANCH=$(git rev-parse --abbrev-ref HEAD)
git diff main..."$BRANCH" > /tmp/mas-review/diff.patch
git diff --name-only main..."$BRANCH" > /tmp/mas-review/changed-files.txt
echo "$BRANCH" > /tmp/mas-review/branch.txt

Capture the changed-files list and total LOC. **Do not read the full diff into your own context** — let agents read from the cached file path.

Do not ask for permission for every agent to read diff.patch - it is always allowed as they won't be able to do the job without reading the diff.

## Phase 2 — Pick the mode

**Default to multi-agent.** Use single-pass only when ALL of these are true:
- Total LOC under ~300 (excluding `dist/`, generated bundles, `.md`)
- Files in a single subsystem (e.g. only `libs/blocks/foo/`, or only `studio/src/fields/`)
- No file in MAS-bridge globs (see below)
- No new exported functions / public API surface

Override flags take precedence over the heuristic.

The asymmetry: picking single-pass when you should have gone multi-agent misses bugs. Picking multi-agent when single-pass would have sufficed costs ~400K tokens. Lean thorough.

## Phase 3 — Conditional gate: MAS-integration agent

Skip the MAS-integration agent if no file in the diff matches `libs/blocks/{merch,merch-card*,caas}/`, `libs/features/{personalization/preview*,mas}/`, `libs/deps/mas/`, or `mas/web-components/src/`. Saves ~100K tokens on Milo-only PRs that don't cross into MAS territory. This is the only conditional.

## Phase 4 — Fan out the agents

Dispatch all selected agents in a **single message with multiple `Agent` tool calls** (concurrent, `run_in_background: true`). Each agent gets the cached diff path, the PR URL, and live source paths. Each agent prompt is self-contained — they do not see this skill or session history.

**Model routing:** dispatch every finder agent (correctness, security, MAS-integration, coverage) with `model: sonnet` — they read a scoped slice of the diff against current source and report findings, which sonnet handles well. Keep this skill's own reasoning on the session model (Opus) for Phase 5+ synthesis, false-positive verification, and severity resolution — that judgment is where quality matters and must not be downgraded. (For a high-stakes security-critical PR, you may raise an individual finder to `model: opus`; do not blanket-upgrade.)

Source paths to include in every agent prompt:
- Milo: `/Users/cod07261/Desktop/milo/libs/` (current main, may differ from PR branch)
- MAS web-components: `/Users/cod07261/Desktop/mas/web-components/src/`
- MAS studio: `/Users/cod07261/Desktop/mas/studio/src/`
- MAS IO: `/Users/cod07261/Desktop/mas/io/`

**Rot resistance**: agent prompts must instruct verification against *current source*, not against any contracts enumerated in this skill. The skill encodes WHERE to look, not WHAT is there.

### Agent A — Security

```
You are auditing the current branchfor security only.

Scope: XSS / HTML injection in template-literal HTML construction; URL-open / navigation hijack; clipboard injection; prototype pollution in tree-walks / Object.assign patterns; preventDefault timing in capture-phase click handlers.

Diff: <DIFF_PATH>
Live source: /Users/cod07261/Desktop/milo/libs/ (Milo main) and /Users/cod07261/Desktop/mas/ (MAS main).

Constraints: author-only / preview-only code paths are still real attack surface if URL params trigger them. Rate severity accordingly but don't dismiss. Skip style, perf, dead code, observer hygiene, test coverage — other agents cover those.

Output (under 400 words): findings as [SEV] file:line — issue, exploitability, fix. SEV ∈ {CRITICAL, HIGH, MEDIUM, LOW}. One-line verdict at the end.
```

### Agent B — Correctness + races

```
You are auditing the current branch for correctness and race conditions only.

Scope: MutationObserver / event-listener hygiene (attach/detach symmetry, capture flag matched on add/remove); event ordering races; WeakMap lifecycle; debounced timer cleanup; idempotence under self-triggered observers; click delegation hit-box / preventDefault scoping including RTL; lifecycle gates (does the gate fire at the right phase — page load vs runtime toggle?).

Diff: <DIFF_PATH>
Live source: /Users/cod07261/Desktop/milo/libs/ and /Users/cod07261/Desktop/mas/.

Critical caution: verify any race-condition claim against current source before including it. Walk through the actual code path (when does flag X get set? when is it read?). False-positive races are the #1 failure mode of correctness audits.

Skip security, MAS contracts, test coverage — other agents cover those.

Output (under 500 words): findings as [SEV] file:line — bug, repro/scenario, fix. SEV ∈ {HIGH, MEDIUM, LOW}. One-line verdict.
```

### Agent C — MAS-side integration

```
You are auditing the current branch for MAS-side integration impact only.

Scope: cross-repo contracts that the diff touches. For each MAS-owned element, event, or data-* attribute the diff reads or stamps, verify the contract against *current* MAS source.

Diff: <DIFF_PATH>
MAS source: /Users/cod07261/Desktop/mas/web-components/src/ (web components, constants, hydrate)
MAS IO: /Users/cod07261/Desktop/mas/io/ (fragment pipeline, placeholder resolution)

Methodology: do NOT rely on memory or prior reviews for what MAS does. For every claim the PR makes about MAS (event payload shape, selector list, attribute name, lifecycle), grep current MAS source and confirm. If the PR's claim has drifted from current MAS source, flag it.

Skip security, races, test coverage.

Output (under 500 words): findings as [SEV] file:line — contract concern, what could break, mitigation in Milo. SEV ∈ {HIGH, MEDIUM, LOW}. Verdict on whether MAS contracts are respected.
```

### Agent D — Coverage + dead code

```
You are auditing the current branch for test coverage gaps, and dead code only.

Scope: untested branches in changed production files; **new exported functions/constants with zero tests**; test structure (one-behavior-per-test, AAA-in-spirit); unused imports/exports/constants introduced by the diff; orphan CSS classes / selectors with no JS assignment; test-quality smells (real timers in debounce tests, config singleton mutation, unkeyed snapshot-style assertions).

Diff: <DIFF_PATH>
Live source: /Users/cod07261/Desktop/milo/libs/ and /Users/cod07261/Desktop/mas/.

Methodology: focus on **changed files only**. Don't survey the whole test landscape. For each new exported function/branch, check whether the test file added in the same PR exercises it.
Tests that assert multiple unrelated behaviors in one `it()`, or that lack a clear setup→action→assert flow, are **LOW** nits — note them, don't block. Do NOT flag absence of `// Arrange/Act/Assert` comments (the repo forbids inline comments).

Constraints: "alive" means used by production code OR by tests. Test-only usage counts as alive. Dead code findings must cite the symbol name and declaration file:line.

Skip security, correctness, MAS contracts.

Output (under 400 words): three sections — Coverage gaps (incl. untested new exports as HIGH), Test structure nits, Dead code. Each finding cited with file:line. SEV ∈ {HIGH, MEDIUM, LOW}. Verdict on adequacy for the size of the change.
```

## Phase 5 — Synthesize

When all agents return:

1. **Dedupe** — same file:line raised by multiple agents merges into one entry, citing both angles. Convergence is signal — call it out.

2. **Verify every MEDIUM+ finding** against current source before including. Read the cited file:line, check the surrounding context, confirm the claim. **Mandatory** — this is the false-positive filter. If verification fails, drop the finding or downgrade to LOW with note "unverified — author should check."

3. **Resolve severity disagreements** by taking the **producing agent's** severity (the one whose domain it's in), not the max. A coverage agent flagging an untested *branch* is LOW unless the path itself is exploitable; severity comes from the security agent's read of that path, not the coverage agent's anxiety. **Exception:** a new exported function with *zero* tests is the coverage agent's own HIGH (TDD block) — keep it HIGH; this is a deliberate policy, not anxiety.

4. **Generate the output** using the exact structure below:

## 1. 📂 File-by-File Feedback
For each file that contains feedback, create a dedicated heading with the file path. Inside each file section, list all issues found in the file in a numbered list. Each list item should include:
  - **Line**: [Line number or range]
  - **Issue**: [Clear explanation of the bug, security flaw, data leak, or breaking issue; Performance hits, unhandled errors, missing edge cases, or maintainability debt; Styling, naming conventions, minor readability tweaks, or dead code]. If the issue is critical, mark it with a 🔴. If it's medium, mark it with a 🟡. If it's low/nit, mark it with a 🟢.
  - **Fix**: [Actionable steps to resolve it. Provide a concise, diff-style or standard code snippet showing the fix if applicable]

# Execution Rules
1. **File Grouping**: Do not mix comments from different files. Every comment must live under its respective file heading.
2. **Strict Sorting**: Within a file, a 🔴 CRITICAL issue must always appear before a 🟡 MEDIUM, which must always appear before a 🟢 LOW/NIT. 
3. **No Issues, No Section**: If a file has no issues, do not list it in the feedback section. If the entire PR is perfect, state that in the Summary Dashboard with a '✅ No issues found.'.
4. **Caveman mode**: Do not add any additional text or formatting. List only files and issues that need to be addressed. Do not list what went well. Use caveman speak. Keep it short. Be direct.