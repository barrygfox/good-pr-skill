# good-pr

Standards for well-formed, verifiable pull requests — packaged as a Claude Code skill.

## What this is

A skill that ensures every PR is easy to understand, verify, and maintain. It applies to fixes, features, refactors, external contributions, and automated tooling changes.

## Guiding principles

1. **The reviewer should never have to run code to understand the PR.** Every question the description answers is a question the PR body should address.
2. **Verification is demonstrated, not asserted.** "Tests pass" is not a test plan.
3. **Keep the blast radius small.** Every changed file is a potential regression vector.
4. **Scope is a discipline.** Large PRs get skimmed; small focused PRs get reviewed properly.

## What the skill enforces

- **Title** in conventional-commit form: `type(scope): short imperative description`
- **Body sections** (in order): `## Problem`, `## Solution`, `## Changes`, `## What Does Not Change`
- **Regression tests** that FAIL against the broken baseline and PASS after the fix
- **Clean PR ancestry** — every PR branch starts at the fetched canonical upstream base and
  contains exactly one direct commit on top of it
- **Test plan** structured as unit tests, regression tests, and existing suites that must still pass
- **Breaking changes** called out explicitly when public APIs, config schemas, or CLIs change
- **Blast radius discipline** — minimum viable set of files, no co-mixed refactoring
- **Pre-flight duplicate/overlap check** against open and merged PRs

## Usage

Trigger this skill whenever you're about to create, prepare, or open a PR — or before committing changes that will become a PR. The skill will:

1. Resolve and fetch the canonical upstream base, then create the work branch from its tip
2. Rebuild the work branch from that base when local `main` is ahead
3. Ensure the PR contains exactly one direct commit on top of that base
4. Search for duplicate or overlapping PRs/issues
5. Audit the working diff against the self-review checklist
6. Write missing PR body sections, test plans, and regression test scaffolding
7. Flag gaps so they can be fixed before review
8. Enforce minimum blast radius
9. Validate that a regression test exists which fails on the current baseline

See [`good-pr.md`](good-pr.md) for the full skill definition, checklist, examples, and automation opportunities.

## License

MIT
