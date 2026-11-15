# State file, resume note and final report

## `.test-fix-loop.json` (project root, not committed)

Create it before the first run and update it after every run, rerun, fix batch and stop. It must never be
committed: add it to `.git/info/exclude` (not `.gitignore`, which would change the project); if editing `.git/` is
blocked, stage files by path instead of `git add -A`.

```json
{
  "version": 1,
  "mode": "fix",
  "started": "2026-11-15T11:37:00Z",
  "updated": "2026-11-15T12:05:00Z",
  "phase": "fix-loop",
  "command": "scripts/test.sh",
  "destinations": ["device A", "device B"],
  "settings": {"fixLevel": "all", "fullRuns": 2, "timeLimitMin": 60, "commits": true},
  "baseline": {"builds": true, "alreadyFailing": ["ExportTests.testLargeFile"], "uncommitted": false},
  "fullRunsUsed": 1,
  "log": "build/test.log",
  "counts": {"total": 40, "passed": 34, "failed": 6, "flaky": 1},
  "failures": [
    {"test": "LoginTests.testLogout", "severity": "blocker", "cause": "nil token after logout",
     "attempts": 1, "status": "fixed", "commit": "a1b2c3d"},
    {"test": "CheckoutUITests.testApplePay", "severity": "critical", "cause": "Apple Pay sheet unavailable on simulator",
     "attempts": 0, "status": "environment"}
  ],
  "timings": {"CheckoutUITests.testTourDefault": 408}
}
```

`status` is one of `open`, `fixing`, `fixed`, `flaky`, `environment`, `unresolved`, `already-failing`,
`logged`. Another session that finds this file with a recent `updated` time and a live test process is
treated as "busy" (wait or ask).

## Resume note `.test-fix-loop.md` (on any stop before green)

- Why it stopped (limit, blocker, environment, user)
- Remaining failures: test, severity, cause guess, attempts, what was tried
- Exact commands to rerun each one, and the full-suite command
- Commits made so far

A later session reads the note and the JSON and continues from there.

## Final report

```text
🏁 test-fix-loop report
Settings       recommended defaults (or what changed)
Mode           fix | report-only | CI (run link)
Baseline       builds ✓ · already failing: ExportTests.testLargeFile
Result         39/40 passed · 2 full runs · 52m
Fixed          LoginTests.testLogout — nil token after logout → guard in SessionStore.signOut (a1b2c3d)
Tests changed  none (or: which test and why the test was wrong)
Flaky          SyncTests.testRetry — shared mock not reset; fixed in setUp
Environment    CheckoutUITests.testApplePay — Apple Pay sheet unavailable on simulator
Unresolved     — (or: test, what was tried, likely cause)
Logged         minors outside the chosen fix level
```

Report-only mode: same header, then failures grouped by severity with the one-line cause for each and the
command to rerun it; no Fixed/Tests changed lines.
