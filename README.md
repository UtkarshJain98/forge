# Forge

Adversarial code hardening for Claude Code. Spawns a Builder agent to implement and a Breaker agent to attack it with tests. Loops until the Breaker can't break it.

<img width="2816" height="1536" alt="Gemini_Generated_Image_y1to7ly1to7ly1to" src="https://github.com/user-attachments/assets/11b67200-27f3-4419-adba-b658df6de1bb" />

## How it works

```
/forge implement password reset flow --rounds 5
```

1. **Builder** implements the feature
2. **Breaker** reads the implementation and writes adversarial tests targeting edge cases, security issues, and boundary conditions. Tests must fail to count as breaks.
3. **Builder** receives the failing tests and must fix the implementation without deleting or weakening the tests
4. Repeat until the Breaker can't produce a failing test, or max rounds reached
5. Adversarial tests stay in the repo as a permanent regression suite

## What the Breaker targets

In order of severity:
- Crashes and unhandled exceptions
- Security issues (injection, auth bypass, path traversal)
- Boundary conditions (zero, negative, very large, empty, unicode)
- Race conditions and concurrency
- Resource leaks
- Contract violations (does the code do what was asked?)

## What makes this different from code review

Code review plugins give opinions. Forge produces **evidence** — every issue must have a failing test that proves the bug. The Builder can't just dismiss feedback; it has to make the test pass.

## Example output

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

From the Claude Code plugin marketplace:

```
/plugin install forge@claude-plugins-official
```

Or load locally for a single session:

```bash
claude --plugin-dir ~/forge
```

## Usage

```
/forge <task description> [--rounds N]
```

- `--rounds N` — max Builder/Breaker iterations (default: 5)
- Requires a clean git working tree (no uncommitted changes)
- Creates a checkpoint before starting so you can roll back the entire session

## Supported stacks

Auto-detects from config files. Works with any stack that has a test runner:
- Python (pytest)
- Node.js/TypeScript (jest, vitest, mocha)
- Rust (cargo test)
- Go (go test)
- Ruby (rspec, minitest)

## Requirements

- Claude Code
- Git

## License

MIT
