---
name: tdd-test-writer
description: >
  Test-first workflow where the agent writes and verifies one failing,
  behavior-focused test, then stops for the user to implement it. After the user
  reports the implementation is ready, verify the test passes, update the
  feature's Obsidian note, and continue with the next test. Use for new features,
  bug fixes, regression tests, and explicit TDD or red-green workflows where the
  user wants to write production code.
---

# TDD Test Writer

Use this skill to complete the RED phase of TDD: define behavior with tests first, verify they fail for the right reason, then hand off implementation with objective pass criteria. The user, not the agent, implements production behavior. Work one test at a time and wait at every RED-to-GREEN boundary.

## When to use

Use this skill when the user asks for:

- Test-first development or a red-green-refactor workflow.
- Tests that define a feature before the user implements it.
- Regression tests for a reported bug.
- Incremental feature development where the agent writes the next test only after the user completes the current implementation step.

## Required rules

1. **Never modify production code.** This skill may create or edit test files only. Do not implement the behavior, even to make a test pass.
2. Write behavior-focused tests, not placeholders or tests that merely mirror an implementation detail.
3. Every test must be run and shown to fail before handoff. The failure must demonstrate the missing or incorrect behavior, not a syntax, setup, or unrelated environment problem.
4. Work in the smallest useful behavior slice: add one test, verify RED, hand it off, and stop. Do not write a batch of future tests up front.
5. After handoff, wait for the user to implement the behavior and tell you it is ready. Do not start another test until the current test has been verified GREEN.
6. Do not weaken, delete, or rewrite a handed-off test to accommodate an implementation. Change a test only if the user changes the behavior requirement or the test itself is demonstrably invalid; explain the reason first.
7. Prefer deterministic, targeted test commands over full-suite runs. Avoid network or time randomness; use existing mocks and fixtures where appropriate.
8. For bug fixes, include a regression test that captures the reported failure mode.
9. Use a `tdd_test_writer` subagent for RED-phase test authoring when that role is available. Require tests-only edits and evidence of an expected RED result. If unavailable, author the test directly and state `FALLBACK: tdd_test_writer unavailable`.
10. Never claim a test was run or passed without observing its result.

## Workflow

### 1. Define the behavior contract

- Inspect the relevant code, public behavior, test layout, and existing test conventions. Do not assume APIs or test helpers that are not present.
- Convert the request into explicit acceptance criteria and identify the smallest next behavior slice.
- Consider happy-path, edge, and negative-path behavior as relevant. For bug fixes, capture the reported failure.
- If a behavior-critical detail is unclear, state an `ASSUMPTION:` or ask a focused question rather than inventing a requirement.

### 2. Discover test conventions

- Identify the existing test framework, runner, naming style, fixtures, and the narrowest command that can run the relevant test.
- Reuse existing helpers and patterns. Avoid introducing new dependencies or parallel test infrastructure without a clear need.
- Identify the exact test file and intended observable behavior before authoring.

### 3. Author exactly one RED-phase test

- Add one small, behavior-focused test for the next slice; do not edit production files.
- Use a regression case for a bug fix.
- Keep assertions specific enough to distinguish correct behavior from false positives.
- If a test needs setup fixes, modify only the test and rerun it. Do not mask a missing behavior with mocks or weaken the assertion.

### 4. Verify RED

- Run the narrowest relevant test command.
- Confirm the test fails for the expected missing or incorrect behavior.
- If it passes immediately, investigate whether the behavior already exists or the test is ineffective. Do not hand off a test that has not demonstrated the expected RED state.
- If it fails for an unrelated syntax, setup, or environment reason, fix only the test/setup issue within the tests-only boundary and rerun. If the expected failure cannot be established, report the blocker and stop.

### 5. Hand off and wait for the user

Return the required `TDD RED PHASE COMPLETE` format below, including the exact test path, verification command and result, failure summary, and implementation contract. Then stop. Do not implement production code, author the next test, or continue autonomously.

### 6. Verify GREEN after the user implements

When the user says their implementation is ready:

- Review the change only as needed to understand the implementation boundary; do not edit production code.
- Run the handed-off test and report the observed result.
- If it fails, describe the failure and wait for the user to make changes. Do not alter the test to make it pass.
- If it passes, record the result in the feature's Obsidian note, then select and author only the next smallest behavior test. Verify that test fails as expected, update the same note, and hand it off. Repeat this cycle until the requested behavior is covered.
- Do not report the feature complete until the relevant tests pass and the user has asked to finish or no additional behavior slices remain.

## Required RED-phase output

Use this format after each new test has been verified failing:

```text
TDD RED PHASE COMPLETE

## Authoring Mode
- Subagent: tdd_test_writer
- Fallback: [only if the subagent was unavailable]

## Test Files
- [path]

## Verification
- Command: [exact command]
- Result: FAIL (expected)
- Failure reason: [1-2 lines tied to missing behavior]

## Implementation Contract (for the user)
1. Do not modify this test: [path]
2. Implement production changes only in: [paths or modules]
3. Completion gate: [exact command] passes without weakening the test.
4. Broader safety check: [secondary command, if appropriate]
5. Return evidence: changed files and a concise summary of command output.

## Assumptions
- ASSUMPTION: [only if needed]
```

On a later GREEN checkpoint, report the exact verification command and observed result in chat, then proceed with only the next test as described above.

## Obsidian feature notes

Maintain a code-free Obsidian record of the test-driven work:

1. Create **one new Obsidian note for each new feature or bug-fix effort**. Do not reuse a note for a different feature.
2. After each test is added, expand that feature's existing note with the test's behavior, test name and file location, RED result, and the implementation handoff status. Do not create a separate note for every test.
3. After the user implements a slice and its test passes, update the same note with the GREEN result before adding the next test.
4. Include implementation recommendations in plain-language prose: the intended behavior, relevant module or boundary, constraints, and a suggested approach at a conceptual level. Recommendations must not include source code or dictate an unverified implementation detail.
5. **Never put code in an Obsidian note.** Do not include source snippets, patches, pseudocode, or code blocks. Keep the note readable as prose; test names and file locations may be recorded as test metadata, not as executable examples.
6. Use the Obsidian MCP server when available and save the note without an approval gate. If it is unavailable, say so plainly and provide a code-free note draft in chat; do not claim that a note was saved or guess a vault path.

Suggested note sections: feature summary and acceptance criteria; assumptions; one dated entry per test with behavior, RED/GREEN status, and failure/pass summary; and implementation recommendations in prose. Keep the note focused on the current feature and expand it as the test sequence advances.

## Quality bar

- Each RED failure is reproducible locally and attributable to the behavior gap.
- Tests specify observable behavior and have meaningful, discriminating assertions.
- Regression coverage captures the reported bug.
- Only one new behavior test is handed off at a time.
- The user remains responsible for all production implementation.
- Obsidian notes are created per feature and updated after every test addition and verified GREEN step, contain implementation recommendations, and never contain code.
