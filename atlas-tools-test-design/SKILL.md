---
name: atlas-test-design
description: >
  Test-first design workflow for the atlas-tools workspace (Gualidan/atlas-tools)
  and repos structured like it. Covers all three crates: **atlas** (install,
  update, uninstall, dependency resolution, install database, filesystem
  mutation), **sisyphus** (the build side: recipes, sandboxed builds, fetching,
  artifact extraction, payload/metadata generation, signing, .sky file writing,
  repo DB insertion), and **common** (shared types, sky verification, DB
  schemas — the format contract between writer and reader). Use BEFORE any
  implementation whenever the user wants to introduce a change in any of them:
  a new feature, bug fix, refactor, or behavior change. Also trigger when the
  user asks for "test cases" or "fixtures", says "I wanna add/change X",
  describes a change idea and expects tests with it, or asks to review whether a
  change is testable. Produces, in order: a failure audit of the planned change,
  a test scenario matrix, reusable invariant asserts, fixture code, and test
  skeletons — and stops for user confirmation after the scenario matrix. The
  run is vault-integrated: before designing it searches the user's Obsidian
  vault (via the Obsidian MCP server) for notes with the same purpose — prior
  test designs, incident and bug-journal entries — and after the work is
  confirmed it records its full output as a new note placed alongside them.
---

# Atlas-tools test-first design

## Why this workflow exists

In the last quarter, four serious bugs in atlas were caught by AI code review
and **none** by the test suite that shipped with them. Every time, the tests
covered the happy path and the "fails before anything is touched" path, but not
the dangerous middle: *partial success, then failure*. The pattern behind all
four: deciding while mutating, per-item values computed outside their loop,
skip/keep decisions applied one hop instead of transitively, and rollback
journals built while the filesystem was already changing.

The build side adds a second failure family: sisyphus **writes** the format
atlas **consumes**, so a change on one side silently breaks the other — and
parts of sisyphus (libarchive, bubblewrap) cannot run in minimal environments,
so tests can pass locally while never actually running in CI.

This skill exists so that detection happens at design time, not in review.
Follow the steps in order. The order is the point: vault before audit, audit
before scenarios, scenarios before code, user confirmation before anything
long, and the vault note only after the work is confirmed.

## Step 0 — Ground yourself in the repo

The workspace is three crates with one artifact between them. Know which side
of the line the change lives on before designing anything:

- **`atlas/`** — the consumer. Installs, updates, uninstalls packages. Reads
  `.sky` files, writes the install DB. Integration tests in `atlas/tests/*.rs`
  share `atlas/tests/fixtures/mod.rs`: a `Workspace` struct with `publish`,
  `install`, `uninstall`, `update` and the `assert_*` family. New atlas
  fixtures extend this style; never invent a parallel pattern.
- **`sisyphus/`** — the producer. Recipe → fetch (HTTP + archive cache) →
  sandboxed build (bubblewrap) → artifact extraction → payload/metadata
  generation (`sisyphus/src/package/`) → signing → `.sky` file → repo DB
  insert. Its own tests live in `sisyphus/tests/` (`fetch_archive_cache.rs`,
  `keygen.rs`, `repo_db.rs`, `source_root.rs`) — read the relevant one before
  adding to it.
- **`crates/common/`** — the contract. Shared types (metadata, manifest), sky
  verification, DB schemas. `crates/common/tests/package_format.rs` is the
  writer/reader contract test: if the change touches what a `.sky` file *is*,
  this file changes too.

Also: read the module(s) the change touches plus one level of callers, and note
every early exit, database write, and filesystem mutation. If a file you need
is not available, **ask for it** — a test designed against an imagined API is
worse than no test.

**Ground in the vault (Obsidian).** Use the Obsidian MCP server to search the
user's vault for notes with the same purpose as this run: prior test-design
notes, bug-journal and incident entries, and anything about the module or
behavior being changed. Read the hits before designing anything — they supply
scenario ideas, invariants, and incident details that Steps 1 and 2 would
otherwise reinvent from scratch, and sometimes reveal this exact change was
already designed once. Tool names vary by server (search / read / create
note); if the MCP server is unavailable, say so in one line and continue —
never silently skip. Keep every hit you found: Step 6 links to these notes.

**Environment reality (affects sisyphus test design).** `compress-tools` needs
system libarchive and builds run under bubblewrap — neither exists in minimal
containers or CI images. Before proposing a test, know whether it can actually
execute in CI: pure logic (recipe parsing, metadata generation, payload
ordering, signing math) runs anywhere; archive round-trips and sandboxed builds
do not. Design so the testable core is testable without system deps, isolate
the rest, and say explicitly which proposed tests are environment-gated.

Context you can rely on: packages are signed `.sky` files; the repo DB
(`packages` table, with a `deps` column) is the source of truth for resolution;
the install DB (`installed_packages`, `package_dependencies` FK edges,
`owned_files`) records what is installed and owns which paths. Resolution
trusts the repo DB; ownership edges are written from sky metadata. The two may
disagree, and a deliberate disagreement is a legitimate test — but it must be
stated in the test name.

## Step 1 — Failure audit of the planned change

Do this before listing any test. Four walks:

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

**The contract walk (sisyphus ↔ atlas).** If the change touches package
format, metadata/manifest types, sky structure, signing, or the DB schemas:
name both ends explicitly — sisyphus's writer (`sisyphus/src/package/`) and
atlas's reader (`verify`, `install`). A field added, renamed, or re-ordered on
the write side must have a matching expectation on the read side, and that is
*a test in `crates/common/tests/package_format.rs`*, not an assumption.

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
  reason, uninstall of a package nothing depends on. On the sisyphus side:
  payload dirs that are only symlinks, empty payload dirs, artifact files with
  unusual modes, recipes with missing/extra fields, a package whose name or
  version contains characters that end up in filenames, archive entries in
  non-sorted order.
- **Mark the regression row** for each Step 1 finding.

## Gate — stop and confirm

Present the audit and the scenario matrix. Wait for the user to confirm scope.
This gate is deliberate: scenarios are cheap to change now and expensive to
change after fixtures exist. Do not write fixture code before this point.
If the vault search found no folder where same-purpose notes live, also ask
here where the Step 6 note should go — not after the work is done.

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

Extend the crate's existing fixture style — `atlas/tests/fixtures/mod.rs`
(`Workspace`) for atlas changes, the corresponding `sisyphus/tests/` helpers
for build-side changes:

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
- **Contract tests are fixtures too**: a format change gets a round-trip case —
  build the package with the sisyphus-side helpers, unpack and verify it with
  the common/atlas-side helpers — in `crates/common/tests/package_format.rs`.
  Writer-only or reader-only tests do not close the contract.
- **Environment-gated tests are isolated and honest**: anything needing
  libarchive or bubblewrap gets its own test file or clear marker, and the
  scenario matrix says which proposed tests will not run in a minimal CI. A
  test that silently compiles-but-never-runs in CI is a vacuous assert with
  extra steps.

### The recipe (atlas side)

One fixture package, end to end, using only what `atlas/tests/fixtures/mod.rs`
already provides. In order:

1. **Workspace**: `let ws = Workspace::new()?;` — tempdir, keypair, runtime
   config, repo DB, install DB and a default target root in one call. The
   struct owns its `ConfigEnv` (it serializes `SISYPHUS_CONFIG_DIR` mutation
   behind a mutex, sets assume-yes, restores env on drop). Only hand-roll a
   standalone `ConfigEnv` if you must — and then never let the guard drop
   early.
2. **Publish**: `ws.publish("base", "1.0.0", 1, &[])?;` writes a signed
   `{name}-{version}-{release}.sky` with a unique payload
   (`{name}/{version}.txt`) *and* seeds the repo DB with the same deps —
   one call, both sources. This is the matched-sources rule enforced by
   construction; prefer it over raw `seed_repo_db_with_deps` whenever a sky
   exists.
3. **Custom payloads**: `ws.publish_with_entries(..., &[PayloadEntry::file(
   "dir/file.txt", "x"), PayloadEntry::symlink("bin/lnk", "../dir/file.txt"),
   ...])` when the test needs real files, modes or links. Give each package
   its own payload dir (`dir/`, `bardir/`, ...): shared payload paths collide
   on install and produce stale-literal asserts.
4. **Hostile payloads**: `write_signed_sky_with_raw_entry(...)` builds one
   fully controlled raw tar entry that bypasses path sanitization — evil paths
   (`/etc/evil`, `dir/../..`), forbidden entry types, exotic modes. One crime
   per test; this is also the closest thing to the poisoned-payload builder.
5. **Act**: `ws.install(name)` / `ws.install_into(name, root)` /
   `ws.uninstall(name)` / `ws.update(name)`; `ws.create_target_root("alt")`
   for a second install root.
6. **Read back the arrange, then assert**: `ws.assert_installed(name, version,
   release)?` and `ws.assert_deps(name, &["base"])?` *before* the act phase;
   afterwards `ws.row()`, `ws.deps()`, `ws.owned_paths()`,
   `ws.get_update_log()`, `read_target_file(root, path)` and
   `count(&conn, sql, params)` for exact numbers. `ws.set_metadata()` forces
   `installed_at`/`install_reason` when a test needs a specific reason.
7. **Invariants**: end every test with `ws.assert_no_staging_dirs()?` plus
   whatever Step 3 helpers the scenario demands.

Repo-DB-only rows (no sky on disk) use `seed_repo_db_with_deps(&path,
&[(name, version, release, sky_path, deps)])`. If a sky *also* exists, its
deps must match — otherwise the arrange dies three steps later as FK 787. A
deliberate mismatch is built by publishing normally and then reseeding with
different deps, and the test name must end in
`_when_repo_db_and_sky_metadata_disagree`.

On the sisyphus side there is no shared fixtures module yet: each test file
(`repo_db.rs`, `fetch_archive_cache.rs`, `source_root.rs`) carries its own
local `package()` / `metadata()` / `count()` helpers. Extend them in place;
if a second file needs the same helper, promote it to a shared
`sisyphus/tests/fixtures/mod.rs` — never copy it.

## Step 5 — Test skeletons

Full arrange/act/assert for each confirmed scenario, using only the fixtures
from Step 4.

- Asserts are concrete: exact versions, exact file lists, exact error strings.
  No `is_ok()` where a value can be checked.
- **Red check**: for each test, state precisely where it fails on the current
  code. A test that cannot fail is decoration, not a test.
- Every assert must name what code change would make it fail. If you cannot
  name one, delete the assert.

## Step 6 — Record the output in the vault

Every completed run ends as a note in the user's Obsidian vault, written via
the Obsidian MCP server. A run whose output lives only in chat is lost.

- **Search before creating.** You already searched in Step 0. If one of those
  notes covers this same change, extend it — append a dated section — instead
  of creating a near-duplicate; otherwise create a new note.
- **Place it with its own kind.** Put the note in the folder where the
  same-purpose notes from Step 0 live. If the vault has no such folder, use
  the location the user gave at the Gate.
- **Title and metadata.** Follow the vault's existing convention if the Step 0
  hits show one; otherwise title it `Test design — <short change name>` with
  frontmatter tags for the crate and module and today's date.
- **Content — what this run outputted**, in this order: the change in one
  sentence; the numbered failure-audit findings; the scenario matrix as a
  Markdown table (name | scenario | expected | how it fails); the fixtures
  added or extended, with file paths; the invariant asserts used; the skeleton
  locations (file + test function names); and each test's red check.
- **Link, don't summarize away.** Wiki-link every vault note this run drew on
  (`[[...]]`), especially incident and journal notes — the note is the graph
  edge between this change and its history.

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
| Cross-crate format drift | "Does *both* the writer (sisyphus) and the reader (atlas) know about this format change?" | The writer/reader split is why `crates/common/tests/package_format.rs` exists — drift is caught there or not at all |
| Environment-gated test that never runs | "Did this test actually execute in CI, or only compile there?" | sisyphus's libarchive/bubblewrap deps make whole suites unrunnable in minimal containers — easy to mistake 'compiles' for 'passes' |

## Hard rules

- No vacuous asserts; no copy-pasted literals; no `is_ok()` where a value can
  be checked.
- Environment CI cannot provide (mount points, real `/`, libarchive,
  bubblewrap) is not faked. Redesign so the rule is enforced by construction —
  e.g. staging inside the install root — and say so. Never `#[ignore]` a test
  to make CI green.
- A change to shared format or types always lands with a contract-test update
  in `crates/common/tests/package_format.rs` — writer and reader together.
- Scenarios and audit before any code; confirmation gate between them.
- Every deliverable references the fixtures that exist; the skill never
  invents a second fixture system.
- Search the vault before designing; write the Step 6 note after the work is
  confirmed. A run without its vault note did not happen.
