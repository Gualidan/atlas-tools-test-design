# Gualidan's Agent Skills

A collection of reusable skills for AI coding agents.

## `tdd-test-writer`

A human-in-the-loop TDD workflow: the agent writes and verifies one behavior-focused failing test, then stops so you can implement the behavior. Once you report that your implementation is ready, the agent checks that test passes, records the result, and writes the next test. It never writes production code.

### Install

```sh
npx skills add Gualidan/skills --skill tdd-test-writer
```

### How to use it

Ask the agent to use `tdd-test-writer`, describe the feature or bug, its expected behavior, and any constraints. Point the agent at the relevant repository. It will inspect the project's test conventions, write one test, verify that it fails for the expected reason, and hand it to you with pass criteria. It then waits for your implementation before checking GREEN and moving on to the next test.

For each feature or bug fix, the skill also creates a separate code-free Obsidian note and updates it each time a test is added or verified. The note includes implementation recommendations in prose, never code. If Obsidian MCP is unavailable, the agent must say so and provide a note draft rather than claiming it saved one.

### Example prompt

```text
Use the tdd-test-writer skill to add [feature or bug fix]. Expected behavior: [describe outcomes]. Constraints: [list constraints or say none known]. Write and verify one failing test, then stop so I can implement it. Do not modify production code. Keep a code-free Obsidian note for this feature and update it after each test.
```
