# Forge

[![Anthropic Published](https://img.shields.io/badge/Anthropic-Officially%20Published-ff6b35?logo=data:image/svg+xml;base64,PHN2ZyB3aWR0aD0iMjQiIGhlaWdodD0iMjQiIHZpZXdCb3g9IjAgMCAyNCAyNCIgZmlsbD0ibm9uZSIgeG1sbnM9Imh0dHA6Ly93d3cudzMub3JnLzIwMDAvc3ZnIj48cGF0aCBkPSJNMTIgMkw0IDIwaDQuNUwxMiA4bDMuNSAxMkgyMEwxMiAyeiIgZmlsbD0id2hpdGUiLz48L3N2Zz4=)](https://claude.com/plugins)
[![Claude Code Plugin](https://img.shields.io/badge/Claude%20Code-Plugin-blue)](https://claude.com/plugins)

Adversarial code hardening for Claude Code. Spawns a Builder agent to implement and a Breaker agent to attack it with tests. Loops until the Breaker can't break it.

<img width="2816" height="1536" alt="Forge: Builder and Breaker agents in an adversarial loop" src="https://github.com/user-attachments/assets/11b67200-27f3-4419-adba-b658df6de1bb" />


## The fight

You give Forge a task. Two agents enter. One builds. One breaks. 
The loop stops when the Breaker dies.

**The Builder** writes the code. It implements your feature, handles the edge cases it can think of, and calls it done.

**The Breaker** reads everything the Builder wrote and tries to destroy it. It writes adversarial tests — not opinions, not suggestions, *tests that fail*. Null inputs. Malformed tokens. Boundary values. Auth bypasses. Every test is a punch. Every failure is proof.

The Builder gets the failing tests back. It can't delete them. It can't weaken them. It has to *fix the code* until every test passes.

Then the Breaker goes again. Harder this time, because the obvious bugs are gone. It digs deeper — race conditions, resource leaks, unicode edge cases, things you'd never think to test.

This continues until one of two things happens:
- **The Breaker can't produce a failing test.** The code survives. Forge is complete.
- **Max rounds hit.** The fight is over. Whatever's left unfixed gets reported.

The Breaker's tests stay in your repo forever. They're not throwaway review comments — they're a permanent adversarial test suite that protects your code from here on out.

## Quick start

```
/forge implement JWT authentication with refresh tokens --rounds 5
```

## What the Breaker attacks

In order of severity:
- Crashes and unhandled exceptions
- Security issues (injection, auth bypass, path traversal)
- Boundary conditions (zero, negative, huge, empty, unicode)
- Race conditions and concurrency
- Resource leaks
- Contract violations (does the code actually do what was asked?)

## Why not just use code review?

Code review gives opinions. Forge gives proof.

A review comment says "this might have a null pointer issue." A Breaker test calls the function with `None` and shows you the traceback. The Builder can't wave it away — it has to make the test pass.

## Example fight

```
Forge complete: passed

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

## Installation

```
/plugin install forge@claude-plugins-official
```

## Usage

```
/forge <task description> [--rounds N] [--harden] [--test-dir <path>]
```

- `--rounds N` — max rounds in the fight (default: 5)
- `--harden` — skip the Builder in round 1 and attack existing code directly; use this to harden code that already exists rather than implementing something new
- `--test-dir <path>` — override where adversarial tests are written (default: auto-detected from project conventions)
- **Git projects:** auto-stashes any uncommitted changes before starting and restores them after; creates a `forge/pre-<timestamp>` tag so you can roll back with `git reset --hard`
- **Non-git projects:** snapshots the project to `/tmp/forge-backup-<timestamp>/` before round 1; restore with `cp -r /tmp/forge-backup-<timestamp>/. .`
- If max rounds are hit without the Breaker running dry, Forge stops and reports what remains unfixed

## Supported stacks

Auto-detects from config files:
- Python (pytest)
- Node.js/TypeScript (jest, vitest, mocha)
- Rust (cargo test)
- Go (go test)
- Ruby (rspec, minitest)

## License

MIT
