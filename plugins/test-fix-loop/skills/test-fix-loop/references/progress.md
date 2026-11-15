# Progress updates

## Emoji legend (one per line, at the start, always the same meaning)

| Emoji | Line |
|---|---|
| ▶️ | Starting a run (command, destinations, test count, estimate) |
| 🔨 | Building |
| ✅ | A layer or phase passed, a bug fixed, the final result green |
| ⏳ | Heartbeat while tests run |
| ❌ | A test failed / first-run failures summary |
| 🔁 | Rerunning failures once (flaky check) |
| ⚠️ | Flaky test found, or a warning that doesn't stop the run |
| 🔍 | Diagnosing a failure |
| 🔧 | Applying a fix batch |
| 🧯 | Environment problem (simulator, device, network, secret, signing) |
| 🐢 | Possible hang: no progress for unusually long |
| ⏸️ | Waiting for another run to free the device/build folder |
| 🛑 | Stopped (limit reached, blocker, or asked to stop) |
| 🏁 | Final run started / final report |

The words must make sense without the emoji. Plain text (same words, bar as `[####------]`) when the user
or project asks for no emoji, or output goes to a CI log or file. No raw logs or code in updates.

## Examples

```text
▶️ Starting: scripts/test.sh on 2 devices — 40 tests, ~45m (UI tests on real devices are slow)
⏸️ Device busy (another test run holds build/.test.lock) — waiting
🔨 Building for testing…
✅ Build done (3m). Unit 120/120 passed. Running UI tests…
⏳ First run  ▓▓▓▓░░░░░░ 14/40 · ❌ 1 · 18m elapsed · ~25m left
❌ First run: 34/40 passed — 1 blocker, 3 critical, 2 minor
🔁 Rerunning 6 failures once…
⚠️ SyncTests.testRetry passed on rerun — flaky (shared mock)
🔍 Diagnosing 2/5: login crash
🔧 Fixing 2/5: nil token after logout
✅ Fixed 2/5: login crash. Related tests pass.
🐢 No progress for 12m on testCheckout — checking
🧯 Device B disconnected — stopping; reconnect it and say "resume"
🏁 Final run: 40/40 passed · 2 runs · 52m
```

## When to post

- Start; each phase start/end; before/after each fix batch; final run; any stop (with the reason).
- **On change, at once**: first failure, a new failure in a different test (group ones arriving together),
  crash or build error, device/simulator lost or app not launching, run finished, no new result for about
  2× the slowest test so far.
- Heartbeat about every 10 minutes (or every 25% of tests) during runs longer than ~5 minutes. If nothing
  changed, say so briefly; never repeat an identical line.
- "status?" → immediate answer from the live log and `.test-fix-loop.json`.

## Getting woken up while tests run

You can only post between tool calls, so a long blocking command means silence. Instead:

1. Start the run in the background, output to a log (Claude Code: Bash with `run_in_background`).
2. Start a watcher that prints a line only on change plus a heartbeat, and exits when the run ends.
   In Claude Code use the `Monitor` tool; each printed line wakes you to post one update. Example watcher
   for an XCTest-style log (adapt the patterns to the stack):

   ```bash
   LOG=build/test.log; PID=<pid of the test run>; last=""; t0=$(date +%s); beat=$t0
   while kill -0 "$PID" 2>/dev/null; do
     p=$(grep -c "' passed (" "$LOG"); f=$(grep -c "' failed (" "$LOG")
     now=$(date +%s); state="$p passed, $f failed"
     if [ "$state" != "$last" ] || [ $((now-beat)) -ge 600 ]; then
       echo "$(( (now-t0)/60 ))m: $state"; last="$state"; beat=$now
     fi
     sleep 20
   done
   echo "finished: $(grep -c "' passed (" "$LOG") passed, $(grep -c "' failed (" "$LOG") failed"
   ```

   Stall check: if the counts haven't moved for ~2× the slowest test, post 🐢 and look at the log tail.
3. Without a watcher tool: wait in steps of at most ~9 minutes that end early when the run finishes or a
   new failure appears (a time-capped `until … ; do sleep 15; done`), then post a line.
4. Never block in one foreground command for longer than ~10 minutes.

## Task list

With a task list tool: one item per phase (First run, Fix blockers, Fix critical, Fix minors, Final run);
add one item per bug only if there are 5 or fewer.
