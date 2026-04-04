# Forge

Adversarial code hardening for Claude Code. Spawns a Builder agent to implement and a Breaker agent to attack it with tests. Loops until the Breaker can't break it.

<img width="2816" height="1536" alt="Gemini_Generated_Image_y1to7ly1to7ly1to" src="https://github.com/user-attachments/assets/11b67200-27f3-4419-adba-b658df6de1bb" />


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
Breaker tests: tests/adversarial/test_auth_breaker.py

Round 1: Builder implemented password reset flow
         Breaker found 4 issues, 3 failing tests
Round 2: Builder fixed 3 issues
         Breaker found 1 new edge case, 1 failing test
Round 3: Builder fixed edge case
         Breaker: all tests pass, no new issues found

Adversarial test suite: tests/adversarial/test_auth_breaker.py (7 tests)
Checkpoint: forge/pre-20260324-1430
```

## Installation

```
/plugin install forge@claude-plugins-official
```

## Usage

```
/forge <task description> [--rounds N]
```

- `--rounds N` — max rounds in the fight (default: 5)
- Needs a clean git working tree
- Creates a checkpoint before round 1 so you can roll back everything

## Supported stacks

Auto-detects from config files:
- Python (pytest)
- Node.js/TypeScript (jest, vitest, mocha)
- Rust (cargo test)
- Go (go test)
- Ruby (rspec, minitest)

## License

MIT
