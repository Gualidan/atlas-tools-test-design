---
name: atlas-test-design
description: >
  Test-first design workflow for the atlas package manager (Gualidan/atlas-tools)
  and repos structured like it. Use BEFORE any implementation whenever the user
  wants to introduce a change: a new feature, bug fix, refactor, or behavior
  change touching install, update, uninstall, dependency resolution, the repo or
  install databases, or filesystem mutation. Also trigger when the user asks for
  "test cases" or "fixtures", says "I wanna add/change X", describes a change
  idea and expects tests with it, or asks to review whether a change is testable.
  Produces, in order: a failure audit of the planned change, a test scenario
  matrix, reusable invariant asserts, fixture code, and test skeletons — and
  stops for user confirmation after the scenario matrix.
---

# Atlas test-first design

## Why this workflow exists

In the last quarter, four serious bugs in atlas were caught by AI code review
and **none** by the test suite that shipped with them. Every time, the tests
covered the happy path and the "fails before anything is touched" path, but not
the dangerous middle: *partial success, then failure*. The pattern behind all
four: deciding while mutating, per-item values computed outside their loop,
skip/keep decisions applied one hop instead of transitively, and rollback
journals built while the filesystem was already changing.

This skill exists so that detection happens at design time, not in review.
Follow the steps in order. The order is the point: audit before scenarios,
scenarios before code, user confirmation before anything long.

## Step 0 — Ground yourself in the repo

Before designing anything:

1. Read `atlas/tests/fixtures/mod.rs`. Know the `Workspace` struct and its
   helpers (`publish`, `install`, `uninstall`, `update`, the `assert_*`
   family). New fixtures extend this style; never invent a parallel pattern.
2. Read the module(s) the change touches, plus one level of callers. Note every
   early exit, every database write, every filesystem mutation.
3. If a file you need is not available, **ask for it**. Do not guess at APIs —
   a test designed against an imagined API is worse than no test.

Context you can rely on: packages are signed `.sky` files; the repo DB
(`packages` table, with a `deps` column) is the source of truth for resolution;
the install DB (`installed_packages`, `package_dependencies` FK edges,
`owned_files`) records what is installed and owns which paths. Resolution
trusts the repo DB; ownership edges are written from sky metadata. The two may
disagree, and a deliberate disagreement is a legitimate test — but it must be
stated in the test name.

## Step 1 — Failure audit of the planned change

Do this before listing any test. Three walks:

**The `?`-walk.** For every early exit (`?`, `return Err`) that sits *after*
the first filesystem or database side effect, answer: what is left on disk and
in the DB, and who restores it? Any exit where the answer is "nobody" is a
finding. (This exact gap once permanently deleted a user file's original
content: the rollback journal was a local variable dropped on error, and the
tempdir holding the swapped-out original was deleted by its own `Drop`.)

**The decide-then-act scan.** Flag every place that makes decisions while
mutating the same set it reads from. The fix shape is always: collect or
validate everything first, then act on the snapshot. A pass whose output
depends on input order is wrong even when it currently passes.

**The loop-invariant scan.** For every loop that builds per-item records, check
that *every* field of every record originates inside the current iteration —
especially DB query results. A query bound to the wrong name, or a variable
declared before the loop, silently gives every record the same value. (This
gave every update candidate the root package's sky path; verification then
failed three modules away from the real bug.)

Output: a numbered list of findings, each with the code location, the question
that exposed it, and the invariant it threatens.

## Step 2 — Scenario matrix

One row per test:

```
name | scenario | expected outcome | how it fails on the current code
```

- **Names** are full-sentence snake_case, e.g.
  `skipping_a_package_also_keeps_its_own_planned_dependencies`. The name is the
  specification; if it needs an "and", it is two tests or one bigger idea —
  decide deliberately.
- **Every behavior gets three phases**: the happy path; the failure that
  happens *before anything is touched*; and **partial success, then failure**.
  If the change truly has no partial-failure path, say why in one sentence.
- **Every graph decision gets both directions.** Who points *at* the package
  (external dependents, kept packages) and what the package *itself* points to
  (its own deps that must be kept too). Cover: shared dependency, diamond,
  cycle, a kept/skipped package that has its own planned dependencies. A suite
  that only walks one direction once missed that keeping a package required
  keeping its dependencies — five tests, all green, bug live.
- **Every ordering-sensitive pass gets a reversed-order variant.**
- **Boundaries**: `/` as an install root, a root that is a mount point, empty
  payload, same version with different release, explicit vs dependency install
  reason, uninstall of a package nothing depends on.
- **Mark the regression row** for each Step 1 finding.

## Gate — stop and confirm

Present the audit and the scenario matrix. Wait for the user to confirm scope.
This gate is deliberate: scenarios are cheap to change now and expensive to
change after fixtures exist. Do not write fixture code before this point.

## Step 3 — Invariant asserts

Write reusable helpers any test can call at the end. The standard four:

1. **No dangling dependency edges** — every installed package's dependencies
   are still installed.
2. **Root unchanged after failure** — after a *failed* operation the target
   root is byte-identical to before (recursive hash or sorted listing).
3. **No staging leftovers** — no temp/staging dirs remain under the root.
4. **DB and disk agree** — recorded owned files/install dirs match reality.

One invariant beats ten hand-invented scenarios: it catches violations in
situations nobody imagined. Every test in the matrix must call at least one. A
test that calls none and asserts no invariant is probably not worth writing.

## Step 4 — Fixtures

Extend `atlas/tests/fixtures/mod.rs` in the existing `Workspace` style:

- **Unique identity everywhere**: payload path per package *and* version
  (`{name}/{version}.txt`), sky filename per package+version+release. Two
  packages never share a filename or payload path — shared ones cause path
  collisions during install and, worse, tests that pass vacuously against stale
  paths.
- **Matched sources**: repo DB seeding must agree with sky metadata unless the
  test is deliberately about the disagreement (then the name says so — e.g.
  `..._when_repo_db_and_sky_metadata_disagree`). A silent mismatch surfaces
  later as an unrelated FK error far from the cause.
- **Poisoned payload builder**: one helper producing a package whose manifest is
  `[valid entry, entry that fails mid-commit]`. This is the reusable embodiment
  of the dangerous middle; build it once, use it in every partial-failure test.
- **Assert your arrange**: seeding helpers read back counts; install helpers
  read back versions. An arrange that silently no-ops produces tests that
  assert nothing about the code under test.

## Step 5 — Test skeletons

Full arrange/act/assert for each confirmed scenario, using only the fixtures
from Step 4.

- Asserts are concrete: exact versions, exact file lists, exact error strings.
  No `is_ok()` where a value can be checked.
- **Red check**: for each test, state precisely where it fails on the current
  code. A test that cannot fail is decoration, not a test.
- Every assert must name what code change would make it fail. If you cannot
  name one, delete the assert.

## Recurring failure patterns

These are atlas's actual incident history. When designing, actively hunt for
each one; when reviewing an implementation, ask its detection question.

| Pattern | Detection question | Incident |
| --- | --- | --- |
| Loop-invariant field/query | "Is every record field computed from this iteration's bindings?" | Every update candidate got the root package's `sky_path` |
| Decide while mutating | "Does any decision read a set that this pass also edits?" | Skip propagation missed kept packages' deps (PR #7) |
| Unrolled-back partial mutation | "Who restores state if this line fails, given the three lines before it succeeded?" | Mid-commit failure deleted original file content (PR #8) |
| Half-walked decision | "This decision has two directions — did both get a test?" | Five shared-dep tests, all leaf-only, cascade bug live |
| Stale copy-paste literal | "Does every literal in this test derive from its own package?" | Disk asserts checked paths from a renamed package — vacuously true |
| Vacuous assert | "What code change makes this assert fail?" | Asserts on paths that never existed |
| Source disagreement assumed away | "Do repo DB and sky metadata agree in this arrange — on purpose?" | FK 787 during install, three steps from the cause |

## Hard rules

- No vacuous asserts; no copy-pasted literals; no `is_ok()` where a value can
  be checked.
- Environment CI cannot provide (mount points, real `/`) is not faked.
  Redesign so the rule is enforced by construction — e.g. staging inside the
  install root — and say so.
- Scenarios and audit before any code; confirmation gate between them.
- Every deliverable references the fixtures that exist; the skill never
  invents a second fixture system.
