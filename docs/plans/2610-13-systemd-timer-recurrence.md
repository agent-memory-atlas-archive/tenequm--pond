# Systemd timer recurrence (#239)

## Result and scope

Keep scheduled sync recurring after its first run and after a user-manager
restart. Keep #321's dead-timer detection, repair message, and shared text/JSON
status behavior. Change the Linux systemd registration and its unit tests; launchd, cron, Task Scheduler, sync, and store behavior stay
outside this work.

## Evidence

- [Issue #239](https://github.com/tenequm/pond/issues/239) and its
  [latest owner update](https://github.com/tenequm/pond/issues/239#issuecomment-6134815376)
  distinguish the shipped detection/repair from the remaining timer fix.
- Systemd's [timer documentation](https://www.freedesktop.org/software/systemd/man/latest/systemd.timer.html)
  supports persistent catch-up for calendar timers. The current monotonic
  recurrence instead needs the service's remembered activation timestamp.
- Upstream [timer source](https://github.com/systemd/systemd/blob/db11bab38ccf1ed257f310d29070843d4c58ea01/src/core/timer.c#L416-L451)
  skips the recurring anchor when activation history is absent, while a saved
  persistent stamp can disable the past boot anchor. Reproduce that combination
  in isolated Linux before claiming the root cause experimentally verified.

## Implementation

1. Add unit regressions next to `schedule.rs` for all five generated calendar
   cadences and their status round-trip. Keep reading old monotonic unit files
   so an unrepaired installation still reports its registered cadence.
2. Replace `OnUnitActiveSec` with `OnCalendar`, retaining the initial boot trigger,
   persistent catch-up, oneshot service, pinned environment, and `--no-wait`.
   Cadences become local wall-clock boundaries: every 5/15 minutes, hourly,
   every six hours, and 03:00 daily. Daily timing follows local DST. Add a fixed
   per-host delay capped at half the cadence and one hour, leaving headroom for
   default timer accuracy while spreading shared-store load.
3. Reload, enable, and explicitly restart a changed or broken timer; enabling
   an already active elapsed timer alone is insufficient. Preserve the existing
   asynchronous service kick for a broken registration and the healthy no-op.
4. Preserve the shared last-sync-based next-run estimate in text and JSON. It
   is an approximate interval bound, also used by cron and Task Scheduler, not
   the exact next calendar timestamp. Document only the scheduling invariant
   in the spec. Existing `pond init`, schedule commands, and both status formats
   continue through the same shared code.

## Verification and stop condition

- Show the new unit regression fails against the old generated timer.
- Run scheduler tests, the full Rust test suite, format, and all-target clippy.
- In an isolated Linux user manager, validate all calendar expressions, reproduce
  the old dead timer, and check repeated firing, manager restart/catch-up,
  rewriting an enabled timer, and a oneshot longer than its interval.
- Stop once those checks pass or explicitly report any unavailable check.
  Do not register a schedule on the developer's host or touch a real store.

## Verification results (initial PR commit 64acaa5, 2026-10-13)

- The calendar regression failed against the old timer body and passed after
  replacement. `cargo test --bin pond schedule::` passed all 12 scheduler tests.
- `cargo test --locked` passed the full suite, with two existing ignored tests.
  `cargo fmt --check` and `git diff --check` passed.
- `cargo clippy --locked --all-targets -- -D warnings` passed.
- Source-extracted registration self-checks with a fake `systemctl` passed first
  install, healthy unchanged no-op, old-template replacement, broken unchanged
  repair with an asynchronous service kick, and restart-error propagation.
- An isolated Debian systemd 252 user manager reproduced the old persistent
  monotonic timer as enabled but elapsed with no next firing. All five generated
  calendar expressions validated with `systemd-analyze calendar`.
- Generated calendar units, with test intervals accelerated to two seconds,
  fired repeatedly, repaired an enabled elapsed timer, caught up and kept firing
  after a full user-manager restart, and ran a five-second oneshot without
  overlap or loss of recurrence. The test service wrote only to container files;
  it did not run sync against a store.
- The full patched Linux CLI was not built or run. Linux checks exercised the
  generated unit definitions; Rust tests and source-extracted self-checks covered
  their generation, parsing, and registration commands on macOS.

## PR review follow-up

- Keep the shared approximate next-run bound; remove the systemd-only exclusion
  and corresponding spec/release-note claims.
- Use a fixed per-host delay of half the cadence, capped at one hour, with
  enough headroom for systemd's default accuracy window. FixedRandomDelay
  requires systemd 247 or newer; runtime verification below used systemd 252.
- Align the daily calendar anchor with cron and Task Scheduler at 03:00.
- Keep the restart rationale at the command sequence and use clap's ValueEnum
  variant list in both cadence readers.
- Source-extracted guards failed against the old generated body: midnight daily
  instead of 03:00, and no fixed randomized delay.
- Revised generated units validated on systemd 252; the same measured delay
  persisted across three successive firings and a full user-manager restart.
  Repeated firing, elapsed-timer replacement, catch-up, and non-overlapping long
  oneshots passed with accelerated test intervals.
- Full Rust suite after review fixes: 752 passed, two existing tests ignored.
- All-target clippy, formatting, and diff checks passed; bug-review found no
  verified introduced defects. Gitleaks, semgrep, and clippy reported no leads;
  unrelated-language scanners were skipped because those file types did not change.
