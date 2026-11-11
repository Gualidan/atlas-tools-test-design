---
name: atlas-tools-test-design
description: >
  Test-first design workflow for the atlas-tools workspace (Gualidan/atlas-tools)
  and repos structured like it. Covers all three crates: atlas (install,
  update, uninstall, dependency resolution, install database, filesystem
  mutation), sisyphus (recipes, sandboxed builds, fetching, artifact extraction,
  payload/metadata generation, signing, .sky writing, repo DB insertion), and
  common (shared types, sky verification, DB schemas). Use before implementation
  for features, bug fixes, refactors, behavior changes, test cases, or fixtures.
  Produce the failure audit, scenario matrix, invariant asserts, fixture code,
  and test skeletons; write the fixtures and tests rather than stopping at a
  plan. Record the audit, matrix, fixtures, tests, and test-failure audit in an
  Obsidian note without asking the user to confirm or approve the work.
---

# Atlas-tools test-first design

## Why this workflow exists

Serious bugs have escaped tests that covered only the happy path and failure
before any state changed. The dangerous gap is partial success followed by
failure: filesystem or database mutations have happened, but rollback is
incomplete. Other recurring risks include deciding while mutating, values
computed outside their per-item loop, one-direction-only graph decisions, and
writer/reader drift between sisyphus and atlas.

Follow the steps in order, but do not stop for approval. The deliverable is
working fixture and test code plus a note recording the design and results—not
a proposal waiting for confirmation.

## Step 0 — Ground yourself in the repo and vault

Know which side of the three-crate workspace the change affects:

- **`atlas/`** — consumer: installs, updates, uninstalls packages, reads `.sky`
  files, and writes the install DB. Integration tests in `atlas/tests/*.rs`
  share `atlas/tests/fixtures/mod.rs` and its `Workspace`; extend that style,
  never invent a parallel pattern.
- **`sisyphus/`** — producer: recipe, fetch, sandboxed build, artifact
  extraction, payload/metadata generation, signing, `.sky` writing, and repo DB
  insertion. Read the relevant test in `sisyphus/tests/` before extending it.
- **`crates/common/`** — shared format contract, types, verification, and DB
  schemas. A `.sky` format change also updates
  `crates/common/tests/package_format.rs`.

Read the changed module and one level of callers. Note early exits, DB writes,
and filesystem mutations. If required source/API details are unavailable, ask
for those details; do not design against an imagined API.

Search the user's Obsidian vault (through the Obsidian MCP server) for related
test designs, incident reports, bug journals, and module notes. Read relevant
hits and retain links for the final note. If the MCP server is unavailable,
say so briefly and continue. If no matching note folder exists, choose a
reasonable existing test/design location or create the note in the vault's
usual notes location; do not pause to ask where it should go.

For sisyphus, account for environment limitations: `compress-tools` needs
system libarchive and builds run under bubblewrap. Identify which tests can run
in minimal CI, isolate environment-gated tests, and state that limitation in
the matrix and note. Do not present compile-only coverage as executed tests.

Context: packages are signed `.sky` files; the repo DB (`packages`, including
`deps`) drives resolution; the install DB (`installed_packages`,
`package_dependencies`, `owned_files`) records installed state and ownership.
Repo DB and sky metadata can intentionally disagree, but name that scenario.

## Step 1 — Failure audit

Before listing tests, walk the change for these risks:

1. **The `?`-walk:** For each early exit after the first filesystem or DB side
effect, state what remains and who restores it. An exit with no rollback is a
finding.
2. **Decide-then-act:** Flag decisions made while mutating the same set being
read. Prefer collecting/validating a complete snapshot before acting.
3. **Loop invariants:** Ensure every per-item field, especially query results,
is computed from the current iteration's bindings.
4. **Contract walk:** For changes to format, metadata, sky structure, signing,
or schemas, identify both writer (sisyphus) and reader (atlas), and include a
contract test in `crates/common/tests/package_format.rs`.

Record each finding as a numbered item with code location, detection question,
and threatened invariant. Treat an empty audit as a result, not a reason to
omit the audit from the note.

## Step 2 — Scenario matrix

Create one row per test:

```text
name | scenario | expected outcome | how it fails on the current code
```

- Use full-sentence snake_case test names. The name is the specification.
- Cover happy path, failure before any changes, and partial success followed by
  failure for each behavior. If partial failure is impossible, explain why.
- For graph decisions, test both incoming dependents and the package's own
  dependencies; include shared dependency, diamond, cycle, and kept/skipped
  packages with their own planned dependencies as applicable.
- Add reversed-order variants for ordering-sensitive passes.
- Consider boundaries relevant to the change, including `/` install root,
  mount-point roots, empty payload, same version with different release,
  explicit versus dependency install reason, and uninstall with no dependents.
  For sisyphus, consider symlink-only/empty payload dirs, unusual modes,
  missing/extra recipe fields, filename-sensitive package/version characters,
  and unsorted archive entries.
- Mark the regression scenario for every audit finding and say exactly how each
  proposed test fails on the current implementation.

## Step 3 — Invariant asserts

Write reusable assertions that tests can call, as appropriate:

1. No dangling dependency edges: every installed dependency remains installed.
2. Root unchanged after failure: the target root is byte-identical to before
   the failed operation (recursive hash or sorted listing).
3. No staging leftovers under the root.
4. DB and disk agree about owned files and install directories.

Every scenario test must call at least one relevant invariant. Avoid generic
assertions that cannot detect a specific regression.

## Step 4 — Write the fixtures

**Actually implement the fixture helpers in the repository. Fixture code is a
required deliverable, not an optional recommendation or a follow-up after user
approval.** Extend the existing crate test helpers:

- Atlas: `atlas/tests/fixtures/mod.rs` (`Workspace`). Use its `new`, `publish`,
  `publish_with_entries`, install/uninstall/update methods, and `assert_*`
  helpers where applicable. Read the actual helper signatures before use.
- Sisyphus: extend the relevant local helpers in `sisyphus/tests/`; if multiple
  files need a helper, promote it to `sisyphus/tests/fixtures/mod.rs` rather
  than copying it.
- Common/format: use the existing package-format test helpers and cover both
  writer and reader for a contract change.

Fixture requirements:

- Use unique payload paths and sky filenames per package/version/release;
  avoid collisions and stale-path assertions.
- Keep repo DB seed data matched to sky metadata unless disagreement is the
  scenario, and state that explicitly in the test name.
- Build a reusable poisoned payload for mid-commit failure (valid entry followed
  by an entry that fails), where applicable.
- Read back fixture setup (counts, versions, dependencies) so a silent no-op
  cannot make a test vacuous.
- Isolate and clearly mark tests requiring libarchive or bubblewrap; identify
  them as environment-gated in the matrix and note.
- Use hostile raw entries only for the specific security/validation case being
  tested; one malformed condition per test where practical.

For atlas, the common flow is `Workspace::new()`, publish matched repo/sky
metadata with `publish` or `publish_with_entries`, assert the arrange, perform
the operation, check concrete DB/disk outcomes, then check invariants and
staging cleanup. Use `install_into`/`create_target_root` for alternate roots.
Repo-DB-only fixtures may use `seed_repo_db_with_deps`; if a sky also exists,
its deps must match unless the test deliberately exercises disagreement.

## Step 5 — Write test skeletons

**Implement the arrange/act/assert test skeletons for the scenario matrix using
the fixtures from Step 4. Do not stop after describing proposed tests.**
Tests should use concrete values: exact versions, file lists, and error
messages, not just `is_ok()` where the result can be checked.

For each test, record the red check: what assertion fails on current code and
which bug/regression it detects. If possible in the available environment, run
the focused tests and record the exact command and outcome. Do not claim tests
were run if they were not. If implementation is explicitly out of scope or the
repository is unavailable, state the blocker and still record the ready-to-use
fixture/test plan in the note; do not request approval as a substitute.

## Step 6 — Write the Obsidian note automatically

Write or update the note through the Obsidian MCP server as part of this run.
Do not ask the user to confirm the matrix, approve fixture writing, or choose a
note location. Do not defer note creation until approval. Chat output alone is
not the deliverable.

Search before creating: extend a related note with a dated section if it covers
the same change; otherwise create a new note beside related test/design notes.
Follow the vault's naming/frontmatter convention, or use `Test design — <short
change name>` with crate/module tags and today's date.

Include, in order:

1. Change summary and scope.
2. Numbered failure-audit findings (including an explicit no-findings result if
   applicable).
3. Scenario matrix as a Markdown table: name, scenario, expected outcome, and
   how it fails on current code.
4. Fixtures written/extended and their file paths.
5. Invariant asserts used.
6. Test skeletons/implementations, file paths, and test function names.
7. Test-failure audit: each test's red check, expected/current failure behavior,
   and the test command/result if actually run (or why it could not be run).
8. Environment-gated tests and limitations.
9. Wiki-links to relevant vault notes found in Step 0.

If a planned fixture or test could not be written, document the precise blocker
and leave no ambiguity about what remains. Never imply that unimplemented work
was completed.

## Recurring failure patterns

| Pattern | Detection question |
| --- | --- |
| Loop-invariant field/query | Is every record field computed from this iteration's bindings? |
| Decide while mutating | Does a decision read a set this pass also edits? |
| Unrolled-back partial mutation | Who restores state if this line fails after prior mutations? |
| Half-walked decision | Does this decision have two directions, and are both tested? |
| Stale copy-paste literal | Does every test literal derive from its own package? |
| Vacuous assert | What code change would make this assertion fail? |
| Source disagreement assumed away | Do repo DB and sky metadata agree here, intentionally or otherwise? |
| Cross-crate format drift | Do both the sisyphus writer and atlas reader cover this format change? |
| Environment-gated test that never runs | Did this test execute in CI, or only compile? |

## Hard rules

- No vacuous asserts, copy-pasted fixture identities, or unchecked `is_ok()`
  where a value can be checked.
- Do not fake unavailable environment behavior or use `#[ignore]` to make CI
  green. Redesign for testability where possible and state limitations.
- Shared format/type changes include a contract-test update in
  `crates/common/tests/package_format.rs` covering writer and reader.
- Audit and scenario design precede fixture and test implementation, but there
  is **no user-confirmation gate** between those steps.
- Write the fixtures and test skeletons; do not return only a matrix or proposal.
- Record the audit, scenario matrix, fixture work, tests, red checks, and run
  results in an Obsidian note automatically, without asking for confirmation.
- Extend the existing fixture system; do not invent a parallel one.
