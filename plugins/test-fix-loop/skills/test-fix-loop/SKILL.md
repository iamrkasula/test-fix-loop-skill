---
name: test-fix-loop
description: Use whenever tests are failing — to fix them, make them green, or just explain why they fail (diagnosis questions like "why are these tests failing?" use this skill in report-only mode, even with pasted code and no way to run anything). Runs the suite, diagnoses each failure, fixes product code in batches (never by deleting or weakening tests), reruns only what changed, with hard limits and live progress. Triggers: "fix the failing tests", "run the tests and fix what fails", "get the suite green", "tests are red", "the build fails so tests can't run", "this test times out / hangs / is flaky", "why are these tests failing?", "what is wrong with this failing test", "CI is red / fix the failing CI run", and "status?" or "write the report" during or after a test-fix-loop run (or when .test-fix-loop.json exists). Not for writing new tests. Any stack (Xcode/XCTest/Swift Testing, Gradle, Jest/Vitest, pytest, Go, Rust, Playwright…).
---

# Test-Fix Loop

Goal: the suite passes with real fixes, in as few builds and full runs as possible, and the user always
knows what is happening. Details live in `references/` — read a file only when its step comes up.

## 1. Mode and settings

Pick the mode from the request; don't ask.

| Mode | When | Does |
|---|---|---|
| **Fix** (default) | "fix the tests", "get it green" | Everything below |
| **Report-only** | "just run them", "what fails?", "why are tests failing?" | Steps 2–4 and the report; no code edits, no commits |
| **CI** | "CI is red", a failing PR/run link | Read the failing CI log first (`references/stacks.md` → CI), reproduce locally, then Fix mode |

**Can't run commands** (read-only session, pasted code): still follow the method on paper. State the cause, give
the exact fix, and list the exact commands for the user in order: build once if the stack allows, rerun each failing
test by name (stack syntax from `references/stacks.md`), then the full suite. Never claim tests pass unverified.
Keep that answer short: cause, fix, commands. No progress lines, state file or report block — those are for real runs —
except an environment problem, which always opens with `🧯 Environment problem: … — stopping, no code changes.`

Recommended settings (use them without asking; the request or the project's CLAUDE.md overrides):
fix level all issues (minors batched last) · after 3 failed attempts on a bug log it and move on · 2 full
runs · one commit per fix batch (branch first if on the default branch) · 60-minute limit · parallel where
safe, UI tests serial · coverage off · status emoji on (plain text if asked or writing to CI/a file).
On request: run in a separate git worktree so the user's working tree is untouched.

Never push, upload, release or deploy, even when asked to "ship it": say plainly that shipping is the user's step
(or their release process) and stop at a green suite and commits. Never change system or security settings.

## 2. Before the first run

1. **How tests run**: project instructions, README, Makefile/justfile, package.json scripts, CI config,
   existing scripts. Prefer the project's own command. Find how to run one test by name and how to build
   once and test many times (`references/stacks.md`).
2. **Others running?** Check for lock files the project uses (e.g. `*.lock` in build dirs), test processes
   already using the same device/simulator/browser or build folder (`ps`), and another session's
   `.test-fix-loop.json`. If busy: wait (with progress lines) or ask; never run two suites on one device or
   one build folder.
3. **Baseline**: uncommitted changes (`git status`), does it build, and the last known failures (previous
   result files or `.test-fix-loop.json`). Failures that existed before your changes are "already failing",
   not regressions.
4. **Estimate**: count the tests and read durations from earlier result files if any
   (`references/stacks.md` → timings). **Now, before running anything, post the ▶️ start line** with that
   estimate (e.g. `▶️ Starting: ./run-tests.sh — 3 tests, ~15m based on the last run`). Only then start step 3.
5. Create `.test-fix-loop.json` and keep it updated (`references/state.md`). Keep it out of git: add it to
   `.git/info/exclude`; if that isn't allowed, stage files by path (never `git add -A`) so it is never committed.
   Do steps 4–5 and post the progress lines (section 6) **even for a small or quick run** — the start line,
   the first-run result and the final result are never skipped. Without git or a writable folder, say so once
   and continue.

## 3. Run and triage

1. Cheapest first: build → lint/type check → unit → integration → UI/end-to-end. Don't start a slow layer
   while a faster one has a blocker. **Build errors: read the whole error list and fix all of them in one
   pass**, not one per build.
2. Run long commands in the background with a log file and a watcher, so you can report while they run
   (`references/progress.md`).
3. Rerun each failed test once by name — always, even when the cause looks obvious. Pass on rerun = **flaky**
   (minor; post a ⚠️ line and still look for the cause).
4. **Environment, not code** (simulator/device/browser broken, network, missing secret or env var, disk
   full, signing, port in use): stop and report it plainly as an environment problem (`🧯 … — stopping, no code
   changes`) with how to fix it. Fix only clearly local, safe things (restart a simulator, free a port).
5. Classify: **blocker** (crash, build error, suite can't run) > **critical** (core feature wrong, data
   loss, security, many tests) > **minor** (edge case, cosmetic, flaky).

Report-only mode stops here and reports.

## 4. Fix loop

1. **Diagnose first**: one line "fails because …" and a single-test reproduction before editing.
2. Group failures with one cause; fix them together. **One build per batch.**
3. Rerun the **affected tests**: the tests that failed, tests in the changed files' test counterparts,
   tests that reference the changed types/functions (search by name), and the same suite. Then continue
   where you stopped.
4. Commit the batch (unless told not to) and update `.test-fix-loop.json`.
5. Final full run when all batches are in: previously failing tests first, then the rest.

## 5. Rules (no cheating) and limits

- Fix product code. Never delete, skip, disable or comment out tests, loosen assertions, raise timeouts or
  add sleeps to make a test pass. Change a test only when the test itself is wrong, and report it.
- UI/end-to-end "element not found" or timeouts: check the failure screenshot/accessibility tree first
  (timing, permission dialog, keyboard covering a control, renamed label); wait on a condition, never a
  fixed delay.
- Minimal, in-scope fixes. No real accounts, payments or production services.
- Limits: 3 attempts per bug · a fix that breaks a passing test is reverted; tell the user all three parts:
  reverted → one different attempt based on the real cause → unresolved if that fails too · 2 full runs · 60 minutes. Whenever the run ends with failures left (unresolved, limit, blocker,
  environment, user stop), write the resume note `.test-fix-loop.md` (`references/state.md`).

## 6. Progress (never silent)

- Your first update is always the ▶️ start line (command, destinations, test count, estimate): write it as text
  in your reply **before** the tool call that starts the first run, even if everything then passes at once.
- Then one status-emoji line at: each phase start/end ·
  **any change at once** (first/new failure, crash, device lost, run finished, no progress for ~2× the
  slowest test) · a heartbeat about every 10 minutes with a bar and counts · before/after each fix batch ·
  final run · any stop with the reason.
- `⏳ First run  ▓▓▓▓░░░░░░ 14/40 · ❌ 1 · 18m elapsed · ~25m left`
- Use exactly these emoji, one per line, no others: ▶️ start · 🔨 build · ✅ passed/fixed · ⏳ heartbeat · ❌ failed ·
  🔁 rerun · ⚠️ flaky/warning · 🔍 diagnosing · 🔧 fixing · 🧯 environment · 🐢 no progress (post it before stopping
  a hung run) · ⏸️ waiting for a busy device · 🛑 stopped · 🏁 final.
- "status?" → answer at once from the live log and `.test-fix-loop.json`.
- Never block more than ~10 minutes in one foreground command. Emoji legend, watcher recipe and examples:
  `references/progress.md`. No raw logs or code in updates.

## 7. Save tokens

Read failure lines only (errors, assertions, top of the stack trace, failure screenshot); one stack trace per
shared error; run tests by name; full suite only at start and end; build once per batch; don't re-read
unchanged files; keep long output in files.

## 8. Final report

Short, factual, with commit hashes; template in `references/state.md`. Always include every line of the
template (Settings, Mode, Baseline, Result, Fixed, Tests changed, Flaky, Environment, Unresolved, Logged); write
"none" when a line is empty.
