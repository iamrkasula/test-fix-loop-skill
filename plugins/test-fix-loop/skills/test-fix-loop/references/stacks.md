# Stacks, CI logs and timings

Always prefer the project's own command (scripts, Makefile, CI config, CLAUDE.md). These are fallbacks.

## Build once, run one test

| Stack | Build once, then test | Run one test |
|---|---|---|
| Xcode (XCTest / Swift Testing) | `xcodebuild build-for-testing …` then `test-without-building` | `-only-testing:Target/Class/testMethod` (Swift Testing: `Target/Suite/test()`) |
| SwiftPM | `swift build --build-tests` then `swift test --skip-build` | `swift test --filter Suite/test` |
| Gradle / Android | `./gradlew assembleDebugAndroidTest` | `--tests 'pkg.Class.method'` (unit) · `-Pandroid.testInstrumentationRunnerArguments.class=pkg.Class#method` |
| Jest / Vitest | — | `npx jest path -t 'name'` · `npx vitest run path -t 'name'` |
| pytest | — | `pytest path::Class::test_name` (`--lf` reruns last failures) |
| Go | `go test -c` | `go test ./pkg -run '^TestName$'` |
| Rust | `cargo test --no-run` | `cargo test test_name` |
| Playwright | — | `npx playwright test file -g 'name'` (`--last-failed`) |

Xcode: one `xcodebuild` per DerivedData folder at a time; give each parallel destination its own
`-derivedDataPath` and `-resultBundlePath`. Failure screenshots and logs are in the `.xcresult`
(`xcrun xcresulttool get test-results tests --path X.xcresult`).

## CI mode

- GitHub Actions: `gh run list --branch <branch> --limit 5`, then `gh run view <id> --log-failed` (only the
  failed steps). For a PR: `gh pr checks <n>`.
- GitLab: `glab ci view` / `glab ci trace <job>`. Other CI: ask for the failing job log or link.
- Read only the failing step's errors. Note CI-only differences (OS image, versions, env vars, secrets,
  parallelism, time zone, locale) before reproducing locally with the same command.
- If it can't be reproduced locally, say so and treat it as an environment issue (with the difference you
  suspect) instead of guessing a code change.

## Timings for estimates

- Xcode: durations in previous `.xcresult` bundles (`xcresulttool get test-results tests`).
- JUnit XML (Gradle, Jest with jest-junit, pytest `--junitxml`, Go via gotestsum): `time` attributes.
- Earlier logs of the project's own script (lines like `passed (54.3 seconds)`).
- `.test-fix-loop.json` from an earlier session.

Use them to estimate the run, run the slowest tests last within a layer, and spread long tests across
destinations when the project allows parallel destinations.
