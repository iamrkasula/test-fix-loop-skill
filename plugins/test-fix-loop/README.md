# test-fix-loop

A Claude Code skill that turns a failing test suite green with real fixes. It records a baseline, runs cheap
checks before slow UI tests, reruns each failure once to separate flaky tests from real bugs, states the cause
before changing code, batches fixes with one build per batch, and reruns only the affected tests before one final
full run. It never deletes, skips or weakens tests, stops at hard limits (3 attempts per bug, 2 full runs,
60 minutes), reports environment problems instead of patching around them, and keeps you informed with short
progress lines and a final report.

Works with any stack: Xcode (XCTest, Swift Testing), SwiftPM, Gradle/Android, Jest, Vitest, pytest, Go, Rust
and Playwright. It is a single instruction file with no scripts, hooks or network calls.

Install:

```text
/plugin marketplace add iamrkasula/test-fix-loop-skill
/plugin install test-fix-loop@test-fix-loop-skill
```

Full documentation: https://github.com/iamrkasula/test-fix-loop-skill

License: MIT © 2026 Rajesh Kasula
