---
name: forge
description: Adversarial code hardening. Spawns a Builder agent to implement and a Breaker agent to attack it with tests. Loops until the Breaker can't break it.
argument-hint: "<task description> [--rounds N] [--harden] [--test-dir <path>]"
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

Extract from the arguments:
- **Task description**: everything that is not a flag
- **`--rounds N`**: max rounds (default: 5)
- **`--harden`**: if present, set `harden_mode = true`. Skip the Builder in round 1 and send existing code directly to the Breaker. Round 2+ still uses the Builder to fix what the Breaker found.
- **`--test-dir <path>`**: override the adversarial test output directory (default: auto-detected in Step 1)

Determine whether the current directory is a git repository by checking whether the Git root context line above says "NOT A GIT REPO". Store this as `git_mode` (true/false) — it controls which path Steps 2, 3, and 4 take.

### Step 1: Understand the project

Read the relevant config files to determine:
- Language and framework
- Test framework and how to run tests
- Where source and test files live

Read the CLAUDE.md if one exists. This context will be passed to both agents.

**Detect the adversarial test directory** (unless overridden by `--test-dir`):
Check for existing test directories in this order: `__tests__/`, `spec/`, `test/`, `tests/`. Use the first one found and append `/adversarial/` to it. If none exist, default to `tests/adversarial/`. Store as `adversarial_dir`.

### Step 2: Create a checkpoint

**If `git_mode` is true:**
Check for uncommitted changes (`git status --porcelain`). If the working tree is dirty, automatically stash the changes with `git stash push -m "forge-stash-<timestamp>"` and note that you will restore the stash after forge completes. Then create a lightweight git tag `forge/pre-<timestamp>` at HEAD. Store the stash ref if one was created.

**If `git_mode` is false:**
Create a snapshot backup directory at `/tmp/forge-backup-<timestamp>/` and copy the current project directory into it (`cp -r . /tmp/forge-backup-<timestamp>/`). Inform the user of the backup location.

Print: `[Forge] Checkpoint created. Starting round 1/${max_rounds}.`

### Step 3: Run the forge loop

For each round (1 through max rounds):

#### 3a: Builder phase

Print: `[Forge] Round ${round}/${max_rounds} — Builder starting...`

**If `harden_mode` is true AND this is round 1:** Skip the Builder entirely. The existing code in the repository is the Builder's output. Collect the list of relevant source files from the task description (ask the user to clarify if the task description doesn't name specific files or directories).

**Otherwise:** Spawn a Builder subagent. Give the Builder:
- Round 1: the task description and project context (language, framework, conventions from CLAUDE.md)
- Round 2+: the task description + all of the Breaker's failing tests and issue reports from all previous rounds + the Builder Rules below
- Explicit instruction: "Do not delete or weaken the Breaker's tests — fix the implementation instead."
- The file paths it should work in (for round 1, infer from the task description; for round 2+, use the files from the previous round)

Wait for the Builder to complete. Collect the list of files it created or modified.

**If `git_mode` is true:** Record the current HEAD SHA as `round_${round}_start`. Then commit the Builder's changes: `git add <builder files> && git commit -m "forge: round ${round} Builder"`. This preserves each passing Builder state so intermediate work is not lost if forge is interrupted.

#### 3b: Breaker phase

Print: `[Forge] Round ${round}/${max_rounds} — Breaker starting...`

Spawn a Breaker subagent. Give the Breaker:
- The task description
- The full contents of every source file the Builder touched (always, regardless of git mode)
- **If `git_mode` is true:** the diff of only this round's Builder changes: `git diff ${round_${round}_start} HEAD`
- All adversarial test files written by the Breaker in previous rounds (pass their full contents so the Breaker knows what has already been found and fixed)
- Any existing non-adversarial test files for those source files
- Project context (language, test framework, how to run tests, `adversarial_dir`)
- The Breaker Rules below
- **If round ≥ 2:** also include the Anti-Gaming Check instruction below

Wait for the Breaker to complete.

#### 3c: Run the Breaker's tests

Print: `[Forge] Round ${round}/${max_rounds} — Running Breaker tests...`

Run the test command targeting the Breaker's new test file for this round. Record the results (pass/fail counts and failure messages).

#### 3d: Evaluate

- If **all Breaker tests pass** and the Breaker reported **no issues it couldn't write a test for**: the code survived. Print `[Forge] Round ${round} — Breaker found no breaks. Forge complete.` and exit the loop.
- If **any Breaker tests fail**: print `[Forge] Round ${round} — Breaker found ${N} breaks. Sending to Builder.` and continue to the next round.
- If this was the **last round**: stop and report what remains unfixed.

### Step 4: Report

**If `git_mode` is true and a stash was created in Step 2:** restore it now with `git stash pop`.

Output a summary:

```
Forge complete: <passed|max rounds reached>

Rounds: 3/5
Builder files: src/auth/service.py, src/auth/tokens.py
Breaker tests:
  Round 1: tests/adversarial/test_auth_breaker_r1.py (4 tests)
  Round 2: tests/adversarial/test_auth_breaker_r2.py (2 tests)
  Round 3: tests/adversarial/test_auth_breaker_r3.py (1 test)

Round 1: Builder implemented password reset flow
         Breaker found 4 issues, 3 failing tests
Round 2: Builder fixed 3 issues
         Breaker found 1 new edge case, 1 failing test
Round 3: Builder fixed edge case
         Breaker: all tests pass, no new issues found

Adversarial test suite: tests/adversarial/ (7 tests across 3 files)
Checkpoint: forge/pre-20260324-1430 (rollback with: git reset --hard forge/pre-20260324-1430)
```

In non-git mode, replace the Checkpoint line with:

```
Backup: /tmp/forge-backup-20260324-1430 (restore with: cp -r /tmp/forge-backup-20260324-1430/. .)
```

If a stash was created and restored, append:

```
Stash restored: forge-stash-20260324-1430
```

## Breaker Rules

The Breaker agent must follow these rules exactly. Include them verbatim in the Breaker's prompt:

1. **Your job is to break the Builder's code, not review it.** Do not give opinions or suggestions. Write tests that fail.
2. **Every issue must have a failing test.** If you can't write a test for it, note it separately but it doesn't count as a break.
3. **Write real, runnable test files.** Use the project's test framework. Put them in `${adversarial_dir}`. Name the file `test_<feature>_breaker_r<N>.<ext>` where N is the current round number. Do not modify or overwrite test files from previous rounds.
4. **You will be given the test files from previous rounds.** Do not re-test issues that already have a passing test. Dig deeper — the obvious bugs are gone.
5. **Target these categories in order of severity:**
   - Crashes and unhandled exceptions (null/None/undefined inputs, empty collections, missing keys)
   - Security issues (injection, auth bypass, path traversal, unsafe deserialization)
   - Boundary conditions (zero, negative, very large inputs, empty strings, unicode)
   - Race conditions and concurrency (if applicable)
   - Resource leaks (unclosed files, connections, cursors)
   - Contract violations (does the code do what the task description says?)
6. **Be adversarial, not pedantic.** Focus on bugs that would cause real failures in production. Do not write tests for style, naming, or formatting. Do not test framework internals.
7. **Do not weaken or soften tests to make them pass.** If a test fails, that's the point — it proves a bug.
8. **Run your tests before reporting.** Only report tests that actually fail. If a test passes, it's not a break — don't report it. If the test runner itself fails (missing deps, broken setup), report that as a blocker rather than claiming BREAKS FOUND: 0.
9. **Report format:**
   ```
   BREAKS FOUND: <N>

   1. <one-line description>
      File: <test file>:<test name>
      Error: <actual failure message>

   2. ...

   ISSUES WITHOUT TESTS: <N or "none">
   - <description of anything you noticed but couldn't write a test for>
   ```
10. **If you cannot find any bugs:** Report honestly. Say "BREAKS FOUND: 0". Do not invent issues to justify your existence.

## Anti-Gaming Check (round 2+)

Include this instruction in the Breaker's prompt for rounds after the first:

> **Check for input-specific workarounds.** Examine the Builder's fix for each issue the previous Breaker found. If the fix special-cases the exact inputs from the failing test (e.g., `if x == None: return default` rather than fixing the null-handling path generally), write a new test with different inputs that exposes the same class of bug. A fix that only passes the specific test inputs is not a real fix.

## Builder Rules (Round 2+)

Include these in the Builder's prompt for rounds after the first:

1. **You must fix the issues the Breaker found.** The Breaker's test files are now part of the test suite.
2. **Do not delete, skip, or weaken the Breaker's tests.** If a Breaker test is wrong (testing something outside the task scope), flag it but do not modify it — the orchestrator will decide.
3. **Do not just handle the specific test inputs.** Fix the underlying bug so the class of issue is resolved, not just the exact test case.
4. **Run the Breaker's tests after your fix to confirm they pass.**

## Rules

- In git mode, auto-stash uncommitted changes before proceeding rather than stopping. Restore the stash after forge completes.
- The Builder and Breaker must be separate subagents. Never let the same agent build and break.
- Pass full file contents to the Breaker, not just diffs. The Breaker needs context to write meaningful tests.
- Pass all previous Breaker test files to the Breaker each round so it does not re-find already-fixed bugs.
- Each round's Breaker writes a new, separate test file. Do not overwrite previous rounds' files.
- In git mode, commit the Builder's changes after each Builder pass before running the Breaker.
- If the Builder flags a Breaker test as out-of-scope, review the test yourself. If the Builder is right, remove the test before the next round. If the Breaker is right, keep it.
- Adversarial tests are permanent artifacts. They stay in the codebase as a regression suite.
