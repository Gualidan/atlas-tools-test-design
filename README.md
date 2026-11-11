# A collection of skills I created mainly to work on [Atlas Tools](https://github.com/Gualidan/atlas-tools)

## `atlas-tools-test-design`

Use this skill **before implementing** a feature, bug fix, refactor, or other behavior change in the Atlas Tools workspace. It covers the `atlas`, `sisyphus`, and `crates/common` crates, and is also useful when you want test cases or fixtures designed for a proposed change.

### How to use it

1. Make the `atlas-tools-test-design` skill available to your coding assistant (for example, by adding this repository to the assistant's skills setup).
2. Start a request with the skill name and describe the change you want to make. Include the relevant crate or module, expected behavior, constraints, and any known failure cases. Point the assistant to the repository/workspace so it can inspect the actual code and existing test helpers.
3. The skill first searches your Obsidian vault for related test designs, incident notes, and bug-journal entries, if the Obsidian MCP server is available. It then audits the proposed change and presents a test scenario matrix.
4. **Review the audit and matrix and confirm or adjust the scope.** The skill intentionally stops here before writing fixture code or test skeletons.
5. After confirmation, it develops the reusable invariants, fixtures, and test skeletons. Once the work is confirmed, it records the completed design in the relevant vault location. If Obsidian MCP is unavailable, the assistant should say so and continue without silently claiming it searched or saved a note.

### What it outputs

In order, the skill produces:

- A failure audit identifying risky exits, mutation ordering, loop-specific values, and cross-crate format-contract concerns.
- A test scenario matrix (`name | scenario | expected outcome | how it fails on current code`), including regression and failure-path coverage.
- After you confirm the matrix: reusable invariant assertions, fixture recommendations or code, and arrange/act/assert test skeletons with a precise explanation of how each test catches a defect in the current code.
- After the work is confirmed: a full design note in your Obsidian vault, linked to relevant notes found during the initial search.

### Example prompt

Copy this prompt and replace the bracketed sections to fit your change:

> Use the `atlas-tools-test-design` skill before implementation. In **[crate/module, e.g. `atlas` / update flow]**, I want to **[describe the behavior change or bug fix]**. The expected behavior is **[specific outcome]**. Important constraints or known edge cases are **[constraints, failure cases, or “none known”]**. Please inspect **[repository/workspace or relevant files]** and the existing test fixtures; do not assume APIs that are not in the code. Search my Obsidian vault for related designs and incidents if the Obsidian MCP server is available. First give me the failure audit and scenario matrix, then stop for my confirmation before writing fixtures or test skeletons.

You can adapt the bracketed details, but keep the confirmation gate: it lets you adjust the scenarios before the skill does the more detailed test-design work.
