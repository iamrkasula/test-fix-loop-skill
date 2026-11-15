# Changelog

All notable changes to test-fix-loop. Versions follow [Semantic Versioning](https://semver.org).

## 1.1.0 — 2026-11-15

### Added
- **Report-only mode**: "why are the tests failing?" diagnoses and reports without editing code.
- **CI mode**: "CI is red" reads the failed CI log first (`gh run view --log-failed`, `gh pr checks`), then
  reproduces locally.
- **Busy check** before running: lock files, test processes on the same device/simulator/build folder, and
  another session's state file.
- **State file** `.test-fix-loop.json` (excluded via `.git/info/exclude`): phase, counts, failures with
  cause/attempts/status, timings. Makes "status?" exact and resuming precise.
- **Updates on change**: phase start/end, first and new failures, crash, device lost, run finished, and a
  possible-hang warning when nothing moves for ~2× the slowest test.
- **Real wake-up mechanism** for heartbeats: background run + `Monitor` watcher script (with a fallback of
  time-capped waits), so updates arrive while tests run.
- **Status emoji** on every update line with a progress bar; plain-text fallback.
- Timing estimates from earlier result files (`.xcresult`, JUnit XML, logs).
- **Three-level test suite**: 35 read-only eval cases (triggers, non-triggers, 8 stacks, every rule, progress
  format, report), 11 real-run cases against a generic bash-only fake test runner and fake CI client (fix,
  report-only, flaky, hang, device lost, lint first, busy device, CI log, unfixable + resume note, heartbeats,
  estimates), and a live git harness (branch + commits, no-commit, baseline, worktree). No toolchains, devices or
  CI servers needed. Kept outside the plugin, so it is not installed with it.
- **Read-only sessions**: when commands can't be run, the skill still gives the cause, the fix and the exact
  commands in order (single test by name, then the full suite), briefly.
- GitHub Actions validation workflow.

### Changed
- **Shorter SKILL.md** (~6 KB, was ~11 KB) with details in `references/` (stacks, progress, state) that load
  only when needed.
- "Affected tests" defined precisely: failed tests, the changed files' test counterparts, tests referencing
  changed symbols, and the same suite.
- Build errors are read in full and fixed in one pass.
- Never deploys even when asked to "ship it"; says shipping is the user's step.
- Environment problems always open with a 🧯 line and no code changes.
- Final report always lists every line ("none" when empty).
- Broader triggers: build failures blocking tests, timeouts/hangs/flaky tests, "status?" and "write the report".
- Generic examples (no project-specific names).
- Always reruns each failure once by name, even when the cause looks obvious; posts ⚠️ for flaky tests.
- Exact emoji legend in SKILL.md; 🐢 posted before stopping a hung run.
- Progress lines and state file even for quick runs; resume note whenever the run ends with failures left.
- State file never committed, also when Claude Code blocks edits to `.git/info/exclude` (files staged by path).
- Positioned as a process skill: any stack, no dependencies.

## 1.0.0 — 2026-11-15

- First release.
