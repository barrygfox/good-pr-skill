---
name: good-pr
description: Standards for well-formed, verifiable pull requests. Use when creating, preparing, or opening a PR — or before committing changes that will become a PR. Ensures minimal blast radius, regression tests, and complete PR body sections.
license: MIT
metadata:
  version: "1.1"
---

# Good PR — Standards for Well-Formed, Verifiable Pull Requests

This skill ensures every PR opened is easy to understand, verify, and maintain. It applies to
every PR — fixes, features, refactors, external contributions, and automated tooling changes.

## Guiding Principles

1. **The reviewer should never have to run code to understand the PR.**
   Every question the description answers is a question the PR body should address.
2. **Verification is demonstrated, not asserted.**
   "Tests pass" is not a test plan. State what was tested, how, and what the expected outcomes are.
3. **Keep the blast radius small.** Every changed file is a potential regression vector. Prefer
   surgical, minimal diffs — even when a larger refactor seems tidier. If a change can touch N
   files but only needs to touch 1, touch 1.
4. **Scope is a discipline.** Large PRs get skimmed. Small, focused PRs get reviewed properly.

---

## Clean Base and Single-Commit Invariant

Every PR must be created from the latest tip of the repository's canonical upstream base
branch, normally `upstream/main` or the repository's configured default branch. Local `main`
is not a valid PR base because it may contain unrelated commits that are not present upstream.

Before making or committing the fix:

```bash
git fetch --prune <base-remote> <base-branch>
git switch --create <pr-branch> <base-remote>/<base-branch>
```

Resolve `<base-remote>` from the repository's actual PR target; do not assume that `origin` is
the upstream repository or that a local `main` is current. If the fix was already applied to
local `main`, do not branch from it. Create the PR branch from the fetched upstream tip and
bring over only the intended patch (for example, with `git cherry-pick <fix-commit>` or by
staging the intended diff manually); never cherry-pick the whole local branch.

Before opening the PR, the branch must satisfy all of these conditions:

- `git merge-base --is-ancestor <base-remote>/<base-branch> HEAD` succeeds
- `git rev-list --count <base-remote>/<base-branch>..HEAD` returns exactly `1`
- `git rev-parse HEAD^` equals `git rev-parse <base-remote>/<base-branch>`
- `git diff --check <base-remote>/<base-branch>...HEAD` is clean

The one commit must contain the complete, focused fix and its tests. If there are multiple
intended commits, squash them before opening the PR; if there are unrelated commits, rebuild
the branch from the upstream tip rather than including them. The PR must explicitly target the
same `<base-remote>/<base-branch>` repository and branch. When creating it, pass the base
explicitly (for example, `gh pr create --base <base-branch> --head <pr-branch>`), rather than
relying on a CLI or hosting-provider default.

---

## PR Checklist

### 1. Title — Conventional Commit Format

```
type(scope): short imperative description
```

- `type` is one of: `fix`, `feat`, `refactor`, `test`, `docs`, `chore`, `perf`, `ci`
- `scope` is the subsystem or package (e.g. `llm`, `hooks`, `workers`, `learning`)
- `description` is ≤72 characters, lowercase, no trailing period
- Bug-fix titles begin with the bug: `fix(hooks): skip durable write on stale sync-log hash`

**Anti-patterns:**
- Title only: `fix bug` / `update` / `changes`
- Title that restates the diff instead of the fix: `fix llm.py` / `add tests`

### 2. Body Sections — All PRs Must Include

Every PR body must have these four sections, in order:

#### `## Problem`
What is broken, missing, or wrong? Be specific:
- What user or developer observable behavior is incorrect?
- Include error messages, crash signatures, or regression symptoms if applicable.
- For regressions: when did it break, and what commit or change introduced it?

**Minimum bar:** "there is a bug" is not a problem statement.

#### `## Solution`
How does this PR fix it? This is the "why", not the "what":
- Which architectural or data-flow decisions drive the fix?
- What alternatives were considered and rejected, and why?
- For non-obvious changes: explain the key invariant or constraint that makes this correct.

**Skip for trivial changes** (typo fixes, one-liners). If the diff is self-explanatory,
replace this section with `## Solution` → `Trivial one-liner; diff is self-explanatory.`

#### `## Changes`
Lead with the smallest view that makes the key point, then support it with prose — never
prose alone. Match the sketch to the change:
- Logic or algorithm → pseudocode
- Control flow → call tree
- UI structure → component tree, including state and module boundaries that matter
- Refactor or moves → shallow file tree showing ownership
- Small shape change → diff-sketch against the surrounding shape that already exists

One sketch is typical, several are allowed, never all. Keep only the calls, files, props,
states, and boundaries needed to make the point. Place each visual next to the short text
it supports.

Enumerate every meaningful change beneath the sketch. Tables are encouraged:

```
**`src/foo/bar.py`** — what changed here
| Change | Why |
|---|---|
| Added `_cache_key()` | Replaces three ad-hoc hash calls |
| Moved init to `__post_init__` | Guarantees ordering before first call |

**`tests/test_foo/`** — new or modified tests
- `test_cache_key_hit` — verifies same content returns cached id
- `test_cache_key_miss` — verifies different content returns new id
```

**Anti-patterns:** Copy-pasting the git diff into the description.

#### `## What Does Not Change`
Explicitly state what is **not** in scope. This prevents scope creep and reassures reviewers:
- API contracts unchanged? Say it.
- Existing tests unaffected? Say it.
- Performance characteristics unchanged? Say it.
- No new dependencies introduced? Say it (test scripts must use stdlib only — no new `pip install` requirements; if a new package is unavoidable, justify it explicitly).
- For small PRs: `N/A — single-file change, no wider scope.` is acceptable.

### 3. Regression Tests — Required for All Non-Documentation PRs

Every PR that changes behaviour must include **two test checkpoints**:

#### Before the fix — tests that FAIL against the current (broken) code
These demonstrate the regression. They must exist in the PR **as new or modified test code**, not
just as a description. A PR with no failing tests against the baseline is not a fix — it is a
refinement of unknown effect.

```
## Regression Tests

**Test that fails before the fix** (`tests/test_llm/test_backbone.py`):
```python
def test_anthropic_provider_respects_custom_api_base():
    """api_base should be used in _build_anthropic(), not hard-coded."""
    config = LLMConfig(provider="anthropic", api_base="https://proxy.example.com/v1")
    backbone = LLMBackbone(config)
    request = backbone._build_anthropic("hello", max_tokens=10)
    # FAILS before fix: uses https://api.anthropic.com/v1/messages
    assert request["base_url"] == "https://proxy.example.com/v1"
```
Result: **FAIL** — `AssertionError: assert 'https://api.anthropic.com/v1/messages' == 'https://proxy.example.com/v1'`
```

#### After the fix — that same test now PASSES
The failing test is included in the PR diff. When the fix is applied, it passes. This is the
primary evidence that the regression is resolved.

```
## Regression Tests

- `tests/test_llm/test_backbone.py::test_anthropic_provider_respects_custom_api_base`
  - **Before fix:** FAIL — `_build_anthropic()` ignores `api_base`, always uses `https://api.anthropic.com`
  - **After fix:** PASS — `_build_anthropic()` respects `config.api_base`
```

### Test Plan Structure

For every PR, structure the test plan in three layers, and present the result as an evidence
pair the reviewer can check at a glance:

- **Before:** the failing run with its exact failure message, or the before screenshot.
- **After:** the same test passing, or the after screenshot.

For visual changes, before/after screenshots are the evidence. For behaviour changes, the
failing-before/passing-after run is the evidence. "Tests pass" on its own is not evidence.

**1. Unit / module tests** — specific, isolated, fast
List each new or changed test file and what it covers. Prefer explicit `test_foo.py::test_bar`
naming over "added unit tests."

**2. Regression test** (required for bug fixes)
- One or more tests that **fail against the current main branch** and **pass after the fix**.
- Include the exact failure message from the pre-fix run.
- If no regression test is possible (e.g. race-condition that cannot be reproduced in a test),
  state the reason explicitly and note what manual verification was performed instead.

**3. Existing test suites** (confirm no regression)
List existing test suites that should still pass with the specific command to run:
- `pytest tests/` passes locally
- `pytest tests/test_llm_provider.py -k anthropic` — verifies no default-provider regression
- Manual: [project-specific manual verification steps]

```
## Test Plan

**Regression test (fails before fix, passes after):**
- `tests/test_llm/test_backbone.py::test_anthropic_provider_respects_custom_api_base`
  - Before: `AssertionError: assert 'https://api.anthropic.com/v1/messages' == 'https://proxy.example.com/v1'`
  - After: PASS

**Existing test suites (confirm no regression):**
- [ ] `pytest tests/` passes locally

**Manual verification steps** (if applicable):
- [ ] Dashboard → Settings → LLM Provider → "Train model now" button: daemon does not crash
- [ ] Hook POST fires to correct daemon port on a non-default-user machine
```

**For external contributors:** A regression test is required even if it is a single `pytest`
invocation demonstrating the bug. If the contributor cannot run tests, state:
`"[Contributor did not run tests — maintainer to add regression test on merge]"` — do not leave blank.

**Note on dependencies in test scripts:** Regression tests and any test helper scripts
(`scripts/validate_pr_body.py`, `scripts/check_*.py`, etc.) must not introduce new `pip install`
requirements. Use Python stdlib only (`json`, `subprocess`, `pathlib`, `re`, etc.). If a new
package is genuinely needed, it must be justified in the PR body and approved separately.

### 4. Breaking Changes — Required When Applicable

If the PR changes any public API, config schema, or CLI interface, add:

```
## Breaking Changes

- `LLMConfig(api_base=...)` no longer accepts bare strings; must be valid URL with scheme.
  Callers passing bare hostnames must wrap in `urlparse()` first.
```

If no breaking changes: `## Breaking Changes` → `None.`

### 5. Screenshots and Demos — Required for UI/Visual Changes

For any change that affects the dashboard, a screenshot comparing before/after is required.
For terminal output changes, include sample output in the PR body.

---

## Reviewer Guidance

If a PR needs specific reviewer expertise or context, add a `## Reviewer Notes` section:

```
## Reviewer Notes

- @kenyonxu — the RRF fusion change in `retrieval/reranker.py:42` interacts with your
  scope-weighting change in #44; please confirm the interaction is correct.
- This PR depends on #43 (schema migration) — merge order matters.
```

---

## Blast Radius Discipline

Every changed file is a potential regression vector. Before finalising the diff, ask:

**"If I reverted this PR tomorrow, would anything unrelated break?"** If the answer is unclear,
the blast radius is too wide.

### Rules

1. **Touch only the minimum viable set of files.** If a fix touches 5 files and 3 of them are
   "nice to clean up while you're here", cut those 3. Open a follow-up PR.
2. **Do not co-mix refactoring with behaviour changes.** Moving a function to a new file + fixing
   a bug in that function = two separate concerns. Split them.
3. **Verify impact before changing shared code.** Before editing any function in `core/`,
   `infra/`, `storage/`, or `retrieval/`, run the project's impact analysis tool (GitNexus
   `gitnexus_impact` or equivalent) and report blast radius to the PR body.
4. **Flag shared-file changes in the PR body.** When a change touches a file used by many
   call-sites, list them: `"`src/utils/cache.py`" is used in 14 call-sites; regression-tested
   at the two most heavily used: `engine.py:210` and `daemon.py:88`."

### Door and Blast-Radius Summary (Reviewer-Facing)

The rules above keep the diff small; the body must also state the merge risk in two lines a
reviewer can read without opening the diff:

- `Door:` one-way or two-way, with the reason. A PR that is cheap to revert is a two-way
  door. Destructive actions and hard-to-reverse decisions are one-way doors.
- `Blast radius:` the potential impact in reviewer terms — affected surfaces, consumers,
  call-sites — not the file count.

### If a PR Touches ≥5 Files

Consider:
- Can the change be staged as a small fix PR + a follow-up cleanup PR?
- Is there a shared dependency (e.g. a new utility function) that should be in a PR of its own?
- Are the file changes all in one subsystem, or are they scattered across unrelated parts of the
  codebase? Scattered changes across unrelated subsystems are the strongest signal the diff is
  too large.

### Anti-Patterns

- "While I was in there I also fixed X, Y, and Z"
- Renaming a file + fixing an unrelated bug in the same PR
- Formatting / import-cleanup mixed with functional changes
- Adding a new utility used in 8 places — the utility is a separate PR, its usage is a follow-up

---

## Pre-Flight: Duplicate and Overlap Check

Before opening a PR, search for duplicates. Submitting a PR that overlaps with an existing one
wastes reviewer time and creates merge conflicts.

**Check both the local repo and upstream:**
- Search open issues for the same symptom or feature request (`gh issue list --search "..."`)
- Search open PRs for the same fix or approach (`gh pr list --search "..."`)
- For the local upstream (`origin`): check both open and merged PRs — a closed PR that was
  never merged may still represent work-in-progress that overlaps
- For the upstream remote (`upstream`): check if a fix exists on `upstream/main` that is not yet
  in the local branch

**If a duplicate or overlap is found:**
- Link the existing issue/PR in the new PR body: `See also #123`
- If the existing PR is open and better-scoped, consider closing this one and contributing to that
  instead
- If the existing PR is merged but the fix is not in the local branch, rebase or cherry-pick rather
  than duplicating the commit

---

## Self-Review Checklist (Run Before Opening PR)

- [ ] Title follows `type(scope): description` convention
- [ ] PR branch was created from the fetched canonical upstream base tip, not local `main`
- [ ] PR branch contains exactly one direct commit on top of the upstream base
- [ ] `HEAD^` equals the upstream base tip and `git diff --check <base>...HEAD` is clean
- [ ] PR body has `Problem`, `Solution`, `Changes`, `What Does Not Change` sections
- [ ] `## Changes` leads with a shape sketch, not prose alone
- [ ] Test plan reads as a before/after evidence pair
- [ ] Body states the merge door (one-way/two-way plus reason) and blast-radius summary
- [ ] Test plan lists concrete verification steps (not just "tests pass")
- [ ] New code has tests; modified code has updated tests
- [ ] Bug-fix PRs include a regression test that FAILS against the current (broken) baseline and PASSES after the fix
- [ ] No secrets, credentials, or sensitive data in the diff
- [ ] Diff is focused — if >10 files, consider splitting into a scoped fix + follow-up
- [ ] Blast radius is minimal — only the minimum viable set of files was touched
- [ ] No co-mixed refactoring or "while I was here" changes
- [ ] `git diff --stat` reviewed — no unexpected large files or binary blobs
- [ ] No new dependencies introduced — test scripts use stdlib only; new `pip install` requirements are explicitly justified
- [ ] Commit messages follow conventional format
- [ ] Searched for duplicate or overlapping issues/PRs (both open and recently-merged)
- [ ] Linked any related issues or PRs in the body

---

## Automation Opportunities

The following checks are **not yet in CI** but should be added:

| Check | Purpose | Implementation |
|---|---|---|
| PR body has all required sections | Enforce structure | `scripts/validate_pr_body.py` — reads PR body, fails if sections missing |
| Title matches `type(scope):` pattern | Enforce conventional commits | `commit-msg` git hook + `ruff`/`commitlint` |
| Bug-fix PRs include a regression test | Ensure fix is demonstrated, not just asserted | GitHub Action validates regression test exists in diff |
| Test plan present and non-empty | Prevent untested PRs | GitHub Actions job parsing PR body |
| No large binary blobs in diff | Prevent LFS abuse | `git diff --check` in CI |

---

## Usage

Trigger this skill whenever the user asks to create, prepare, or open a PR — or before
committing changes that will become a PR. Use it to:

1. **Resolve and fetch the PR base** — identify the canonical target repository and branch,
   fetch its latest tip, and create the work branch from that tip; never use local `main` as
   the base
2. **Enforce one commit** — carry over only the intended fix and tests, squash if necessary,
   and verify the branch has exactly one direct commit over the fetched base
3. **Duplicate and overlap check** — search `gh issue list --search` and `gh pr list --search`
   against both local and upstream remotes before opening the PR
4. **Audit the working diff** against the self-review checklist before the PR is created
5. **Apply missing sections** — write the PR body, test plan, and regression test content
6. **Flag gaps** — report which required sections are missing or malformed so they can be fixed
7. **Enforce minimal blast radius** — check the diff touches only the minimum viable set of files
8. **Validate test-first discipline** — confirm a regression test exists that fails against the
   current baseline and passes after the fix

---

## Wrapup — After the PR Is Opened

Once the PR URL is returned, output a checklist summary showing what was included. One row per
checklist item, ✅ for satisfied, ❌ for any gap that was accepted as-is (explain why inline).

Example format:

```
PR: https://github.com/org/repo/pull/N

| Check | Status |
|---|---|
| Title names the bug, not the diff | ✅ |
| `## Problem` with error cause and mechanism | ✅ |
| `## Solution` with Alternatives Considered table | ✅ |
| `## Changes` table | ✅ |
| Shape sketch leads `## Changes` | ✅ |
| Evidence pair (before/after) | ✅ |
| Merge door plus blast-radius summary | ✅ |
| `## What Does Not Change` | ✅ |
| `## Breaking Changes: None.` | ✅ |
| Regression test (fails before fix, passes after) | ✅ |
| `## Regression Tests` — timing test not possible, explained | ✅ |
| `## CI Notes` — pre-existing failures documented | ✅ |
| Duplicate/overlap check — none found | ✅ |
| Blast radius: N files, M lines | ✅ |
```

Keep it tight — the table is a receipt, not a re-explanation. If an item was skipped for a valid
reason (trivial one-liner, no CI failures, no breaking changes), mark ✅ with a note in the
Status cell rather than omitting the row.