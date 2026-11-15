---
name: test-fix-loop
description: Run a project's test suite, fix the failures in batches, and rerun only what changed, with hard limits on attempts and full runs. Use when asked to "run the tests and fix what fails", "get the suite green", "fix the failing tests", or before a release when the whole suite must pass. Works for any stack (Xcode/XCTest/Swift Testing, Gradle, Jest/Vitest, pytest, Go, Rust, Playwright…).
---

# Test-Fix Loop

Goal: the whole suite passes, with real fixes, in as few builds and full runs as possible.

## 1. Settings

Use the recommended settings and start. Ask only if the user explicitly asked to choose, or the project's
instructions (CLAUDE.md, a release checklist) set different values; project instructions win.

| Setting | Recommended | Alternatives |
|---|---|---|
| Fix level | All issues; minors fixed in one batch at the end | Blockers + critical only (log minors) · Blockers only |
| Bug still failing after 3 attempts | Log it, move on, report at the end | Stop and ask |
| Max full-suite runs | 2 | 3 |
| Commits | One commit per fix batch on the current branch (branch first if on the default branch) | No commits |
| Time limit for the whole loop | 60 minutes | Custom |
| Parallel test runs | On where the stack supports it safely; UI/end-to-end tests serial unless the project says otherwise | Off |
| Coverage | Off (slows runs) | On |

Never push, upload, release or deploy as part of this skill.

## 2. Learn the project first (once, cheaply)

1. Find how tests run: project instructions, README, Makefile/justfile, package.json scripts, CI config,
   existing test scripts. Prefer the project's own command over a generic one.
2. Find how to run **one** test or one file by name, and how to build once and test many times (see the
   table below). Note the test-time limits or per-test timeouts the project sets.
3. Note project rules that affect tests: which simulator/emulator/browser to use, required env vars, test
   data/fixtures, network or no-network, parallelism, ports.

| Stack | Build once, then test | Run one test |
|---|---|---|
| Xcode (XCTest / Swift Testing) | `xcodebuild build-for-testing …` then `test-without-building` | `-only-testing:Target/Class/testMethod` (Swift Testing: `Target/Suite/test()`) |
| SwiftPM | `swift build --build-tests` then `swift test --skip-build` | `swift test --filter Suite/test` |
| Gradle / Android | `./gradlew assembleDebugAndroidTest` | `--tests 'pkg.Class.method'` (unit) · `-Pandroid.testInstrumentationRunnerArguments.class=pkg.Class#method` |
| Jest / Vitest | — | `npx jest path -t 'name'` · `npx vitest run path -t 'name'` |
| pytest | — | `pytest path::Class::test_name` |
| Go | `go test -c` | `go test ./pkg -run '^TestName$'` |
| Rust | `cargo test --no-run` | `cargo test test_name` |
| Playwright | — | `npx playwright test file -g 'name'` |

4. Baseline: note uncommitted changes (`git status`) and confirm the project builds before running tests.
   If it doesn't build or the working tree has unrelated edits, say so first; failures that existed before
   your changes are reported as "already failing", not as regressions you caused.

## 3. Run

1. Cheap checks first, in order: build/compile → lint/type check (if the project uses them) → unit tests
   → integration → UI/end-to-end. Don't start a slow layer while a faster one fails for a blocker.
   Run the full suite once this way. Stop early only for a blocker (crash, build failure, or a failure that
   stops other tests running).
2. Rerun each failed test once by name. Pass on rerun = **flaky**: treat as minor, and still look for the
   cause (timing, shared state, order dependence).
3. Classify: **blocker** (crash, build/compile error, suite can't run) > **critical** (core feature wrong,
   data loss, security, many tests failing) > **minor** (edge case, cosmetic, flaky).
4. Environment, not code: if the cause is outside the code (simulator/emulator or browser broken, device
   missing, network down, missing secret or env var, disk full, signing/provisioning, port in use), stop
   and report it as an environment issue. Fix it only if it is clearly local and safe (e.g. reboot a
   simulator, free a port); never change system or security settings.
5. Diagnose before fixing: for each failure write one line "fails because …", and reproduce it with the
   single test by name before editing code. No guess-fixes; a fix without a stated cause doesn't count as
   an attempt worth making.
6. Group failures with the same cause and fix them together. One build per batch, not one per bug.
7. After a batch, rerun only the affected tests and their neighbours (same file/suite/feature). Continue
   from where you stopped.
8. When every test has run and the batches are in, do one final full run: previously failing tests
   first, then the rest, so a remaining failure shows up early.
9. New failures in scope during the loop: fix the same way, inside the run limit.

## 4. Rules for fixes (no cheating)

- Fix the product code. Do **not** delete, skip, disable, comment out, loosen assertions, raise timeouts or
  add sleeps just to make a test pass.
- Change a test only when the test itself is wrong (outdated expectation after an intended change, wrong
  fixture, real race in the test code). Say so in the report, with the reason.
- UI/end-to-end failures like "element not found" or timeouts: first look at the failure screenshot,
  recording or accessibility tree. It is often timing, an unexpected alert/permission dialog, keyboard
  covering a control, or a renamed label. Fix with proper waits on a condition, not fixed sleeps.
- Keep fixes minimal and in scope; no unrelated refactors.
- Never use real accounts, real payments or production services to make a test pass.

## 5. Loop limits

- 3 attempts per bug, then follow the setting.
- A fix breaks something that passed: revert it, try once more differently, then count the bug as
  unresolved.
- Full-run limit or time limit reached with failures left: stop and report.
- Whenever you stop before the suite is green, leave a resume note (`.test-fix-loop.md` in the project
  root, not committed, or the last commit message): remaining failures, cause guesses, what was tried,
  the exact commands to rerun them. A later session starts from that note.

## 6. Progress updates

- With a task list tool: one item per phase (First run, Fix blockers, Fix critical, Fix minors, Final
  run); add one item per bug only if there are 5 or fewer.
- The user must always know what is happening. Never go silent for more than ~10 minutes.
- One-line updates at these points:
  - Before starting: `Starting: <command or script> on <destinations> — <N> tests, roughly <estimate> (UI tests on real devices are slow).`
    Count the tests (or say "count unknown") and give a rough time from earlier logs if any.
  - At the start of each layer or phase: `Build done (2 min). Running unit tests…` / `Unit 120/120 passed. Running UI tests on 2 phones…`
  - During any run longer than ~5 minutes: a heartbeat every ~10 minutes or every 25% of tests, whichever
    comes first, read from the live log (count passed/failed lines, never paste them):
    `First run in progress: 14/40 done, 1 failure so far, 18 min elapsed, ~25 min left.`
    Run long commands in the background (or with output to a log file) so you can check and report
    between heartbeats instead of blocking until they finish.
  - After the first run: `First run: 34/40 passed. 6 failures (1 blocker, 3 critical, 2 minor). Fixing now.`
  - Before each fix batch: `Fixing 2/6: login crash (cause: nil token after logout).`
  - After each bug is fixed or given up: `Fixed 2/6: login crash. Related tests pass.`
  - Before the final run: `All fixes in. Starting final run (~<estimate>).`
  - When stopping for any reason: say why in one line.
- If the user asks "status?", answer at once from the live log: what is running, how far, failures so far,
  time elapsed and left.
- No logs, code or step-by-step narration in updates.

## 7. Save tokens and time

- Read only failure output: error lines, assertion messages, the top of the stack trace, the failure
  screenshot when there is one. Never load full logs; filter them (grep for the failing test, "error:",
  "failed", "Assertion").
- Many failures with the same error line: read one stack trace, treat the rest as the same bug until
  the fix proves otherwise.
- Run tests by name; the full suite only at the start and the end.
- Build once per batch; reuse build products where the stack allows.
- Don't re-read files you haven't changed.
- Put long outputs in a temporary file and read only the lines you need.

## 8. Final report (short)

- Settings used (say if they were the recommended ones)
- Baseline: builds yes/no, already-failing tests before any change
- Passed / total (final full run), full runs used, time taken
- Environment issues found (not code)
- Fixed: one line each (cause → fix), plus commit hashes if committed
- Tests changed (not product code): which and why
- Flaky: which, and the suspected cause
- Unresolved: what failed, what was tried, likely cause
- Logged, not fixed (outside the chosen fix level)
