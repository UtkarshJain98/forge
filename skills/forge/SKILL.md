---
name: forge
description: Adversarial code hardening. Spawns a Builder agent to implement and a Breaker agent to attack it with tests. Loops until the Breaker can't break it.
argument-hint: "<task description> [--rounds N]"
allowed-tools: [Read, Glob, Grep, Bash, Edit, Write, Agent]
---

# Forge

Harden code through adversarial Builder/Breaker iterations.

## Arguments

$ARGUMENTS

## Context

- Git root: !`git rev-parse --show-toplevel 2>/dev/null || echo "NOT A GIT REPO"`
- Current branch: !`git branch --show-current 2>/dev/null`
- Project stack signals: !`ls package.json pyproject.toml Cargo.toml go.mod Gemfile Makefile 2>/dev/null || echo "no config files found"`
- HEAD commit: !`git log --oneline -1 2>/dev/null`

## Instructions

### Step 0: Parse arguments

Extract the task description and optional `--rounds N` flag from the arguments. Default max rounds to 5 if not specified.

### Step 1: Understand the project

Before spawning agents, read the relevant config files to determine:
- Language and framework
- Test framework and how to run tests
- Where source and test files live

Read the CLAUDE.md if one exists. This context will be passed to both agents.

### Step 2: Create a checkpoint

Before any changes, create a lightweight git tag `forge/pre-<timestamp>` at HEAD so the user can roll back the entire forge session if needed. If there are uncommitted changes, warn the user and stop — Forge needs a clean working tree to isolate its changes.

### Step 3: Run the forge loop

For each round (up to max rounds):

#### 3a: Spawn Builder agent

Spawn a subagent with `subagent_type` appropriate for implementation. Give the Builder:

- The task description (round 1) or the task description + Breaker's failing tests and issue report (round 2+)
- Project context (language, framework, conventions from CLAUDE.md)
- Explicit instruction: "Implement the feature/fix. If this is round 2+, you must also make all of the Breaker's failing tests pass. Do not delete or weaken the Breaker's tests — fix the implementation instead."
- The file paths it should work in

Wait for the Builder to complete. Collect the list of files it created or modified.

#### 3b: Spawn Breaker agent

Spawn a subagent. Give the Breaker:

- The task description (what was supposed to be built)
- The full diff of what the Builder changed: `git diff forge/pre-<timestamp> HEAD`
- The source files the Builder touched (read them in full)
- Any existing test files for those source files
- Project context (language, test framework, how to run tests)
- Explicit instructions (see Breaker Rules below)

Wait for the Breaker to complete. Collect its output: a list of issues and the adversarial test files it wrote.

#### 3c: Run the Breaker's tests

Run the test command targeting the Breaker's test files. Record the results.

#### 3d: Evaluate

- If **all Breaker tests pass** and the Breaker reported **no issues it couldn't write a test for**: the code survived the round. Forge is complete.
- If **any Breaker tests fail**: the Builder has work to do. Continue to the next round.
- If this was the **last round** (max rounds reached): stop and report what remains unfixed.

### Step 4: Report

After the loop ends, output a summary:

```
Forge complete: <passed|max rounds reached>

Rounds: 3/5
Builder files: src/auth/service.py, src/auth/tokens.py
Breaker tests: tests/adversarial/test_auth_breaker.py

Round 1: Builder implemented password reset flow
         Breaker found 4 issues, 3 failing tests
Round 2: Builder fixed 3 issues
         Breaker found 1 new edge case, 1 failing test
Round 3: Builder fixed edge case
         Breaker: all tests pass, no new issues found

Adversarial test suite: tests/adversarial/test_auth_breaker.py (7 tests)
Checkpoint: forge/pre-20260324-1430 (rollback with: git reset --hard forge/pre-20260324-1430)
```

## Breaker Rules

The Breaker agent must follow these rules exactly. Include them verbatim in the Breaker's prompt:

1. **Your job is to break the Builder's code, not review it.** Do not give opinions or suggestions. Write tests that fail.
2. **Every issue must have a failing test.** If you can't write a test for it, note it separately but it doesn't count as a break.
3. **Write real, runnable test files.** Use the project's test framework. Put them in a `tests/adversarial/` directory (create it if needed). Name files `test_<feature>_breaker.<ext>`.
4. **Target these categories in order of severity:**
   - Crashes and unhandled exceptions (null/None/undefined inputs, empty collections, missing keys)
   - Security issues (injection, auth bypass, path traversal, unsafe deserialization)
   - Boundary conditions (zero, negative, very large inputs, empty strings, unicode)
   - Race conditions and concurrency (if applicable)
   - Resource leaks (unclosed files, connections, cursors)
   - Contract violations (does the code do what the task description says?)
5. **Be adversarial, not pedantic.** Focus on bugs that would cause real failures in production. Do not write tests for style, naming, or formatting. Do not test framework internals.
6. **Do not weaken or soften tests to make them pass.** If a test fails, that's the point — it proves a bug.
7. **Run your tests before reporting.** Only report tests that actually fail. If a test passes, it's not a break — don't report it.
8. **Report format:**
   ```
   BREAKS FOUND: <N>

   1. <one-line description>
      File: <test file>:<test name>
      Error: <actual failure message>

   2. ...

   ISSUES WITHOUT TESTS: <N or "none">
   - <description of anything you noticed but couldn't write a test for>
   ```
9. **If you cannot find any bugs:** Report honestly. Say "BREAKS FOUND: 0". Do not invent issues to justify your existence.

## Builder Rules (Round 2+)

Include these in the Builder's prompt for rounds after the first:

1. **You must fix the issues the Breaker found.** The Breaker's test files are now part of the test suite.
2. **Do not delete, skip, or weaken the Breaker's tests.** If a Breaker test is wrong (testing something outside the task scope), flag it but do not modify it — the orchestrator will decide.
3. **Do not just handle the specific test inputs.** Fix the underlying bug so the class of issue is resolved, not just the exact test case.
4. **Run the Breaker's tests after your fix to confirm they pass.**

## Rules

- Do not proceed if the working tree is dirty. Forge needs to isolate its changes.
- The Builder and Breaker must be separate subagents. Never let the same agent build and break.
- Pass full file contents to the Breaker, not just diffs. The Breaker needs context to write meaningful tests.
- If the Builder flags a Breaker test as out-of-scope, review the test yourself. If the Builder is right, remove the test before the next round. If the Breaker is right, keep it.
- Adversarial tests are permanent artifacts. They stay in the codebase as a regression suite.
