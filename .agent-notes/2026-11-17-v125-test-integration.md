# 2026-11-17 — v1.2.5 test integration: state + regression suites into the runner

Executor lane, phase 1 of 2 (first attempt). Base: the phase 0 working tree —
upstream `39380af` plus the already-integrated `src/probe.c` and adapted
`test/host_test.c`. Nothing committed, nothing pushed, no device touched. The
original `/home/taj/workers/mpc-tributary` checkout was used read-only
(readable via `git show 937a83e...`) and is untouched.

## What changed and why

Two new test files, one runner edit, zero runtime changes.

- **`test/state_test.c` (new)** — adapted from `test/state_test.c` in local
  commit `937a83e5d6fc6d02d13a34db601210a5302b6d07` (read with
  `git show 937a83e...:test/state_test.c`; that commit lives in this repo's
  object store and in the read-only original checkout). Every chunk-format
  section from 937a83e passes verbatim against the integrated engine, so they
  are kept as they were. The adaptation adds what the v1.2.5 integration
  changed since that commit: an inactive restore holds no decode
  (`P->used == 0`, all `g_use[] == 0`) and waking the copy is what hands the
  saved routes to the shared read; the mix_owned rule (a SOURCE write after a
  valid restore, or after a real switch press, is refused; a fresh copy still
  takes a SOURCE); the switches_live re-arm (a loaded copy refuses the
  switch defaults the host replays, tested cold AND mid-session); and the
  legacy `v1 <mix> <level>` input line (loads, never wakes, no decode
  ownership, refuses SOURCE too). This includes the SOURCE-after-valid-state-
  restore coverage the brief asked for.
- **`test/tributary-regression.c` (new)** — adapted from
  `.../mpc-tributary-v125-integration-20261117/artifacts/tributary-regression.original.c`.
  Kept: the delay/reverb/chorus restore scenario, the 8-combination sweep,
  the second-save/third-restore drift checks, the full-state (MAIN + tracks +
  DLY/REV/CHO + IN) restore, and the switches_live CONTROL/CANDIDATE
  diagnostic (now a guard: the restore re-arms the default-switch guard with
  or without a prior render). Corrected — every use_from_mix-derived
  expectation is gone: an inactive restore now asserts `P->used == 0` and an
  all-zero `g_use[]` (the original asserted the restored mix was ON the
  shared read with Main forced in, `1u | mix`); the combos and full-state
  blocks re-save the restored copy and require the identical chunk back, so
  Main provably never re-enters an inactive mix on the save path either. In
  exchange the suite gained the positive half the old one could not have:
  activating a restored copy decodes exactly the saved mix (checked for the
  3-fx case and the 7-switch full state), and stopping gives every refcount
  back. No double-STOP claim anywhere — the header says it stays unverified
  without device evidence.
- **`test/run.sh`** — the four suites now build and run in one `&&` chain
  inside the same gcc:12 container: host_test, capture_test, state_test,
  tributary-regression. A failed build or run stops the chain. The compile
  flags moved into a `$CC` variable (same flags as before) to keep the chain
  readable; the docker invocation is otherwise unchanged. AGENTS.md still
  says the runner prints `ALL PASSED` twice — it now prints it four times
  (once per suite); AGENTS.md was not edited per the brief.
- **`src/probe.c`** — untouched by this phase. No test exposed a runtime bug:
  the two failures on the first run were mistakes in my own new test blocks
  (see "Tricky things" below). `test/host_test.c` — also untouched; phase 0
  already covered the integration behaviours it needs.

## Verification

- `sh test/run.sh` → exit 0, `ALL PASSED` four times and no FAIL lines
  (host_test 83 PASS, capture_test 45, state_test 88, tributary-regression
  72 — 288 checks total) in gcc:12.
- `bash .../gate-tests.sh` → `TEST_GATES_OK`, exit 0 (after the notes file
  existed — see below).
- Scope: porcelain shows only `src/probe.c`, `test/host_test.c`,
  `test/run.sh`, `test/state_test.c`, `test/tributary-regression.c`, and the
  two notes files.

## Tricky things to remember next time

- **`sw_on(inst, e, 0)` is 1 on a fresh copy** — Main is the default switch.
  So a refused-SOURCE check must not use `source=0` as the probe write (a
  successful write to 0 and a refused write look identical through bit 0);
  write a source whose absence is visible, e.g. `source=3` after `s5=1`.
- **Mid-session load order matters in the test itself**: render first, then
  `state`, then the replayed defaults. Rendering after the restore legitimately
  re-enables switch writes (`switches_live`), and the defaults then apply.
  My first draft had the render after the restore and failed until reordered.
- **Fake-live discipline** (from the phase 0 notes, upheld here): `g_alive=1,
  g_exited=0, g_owner=NULL` only inside activation windows, and `g_alive=0`
  again BEFORE any `active=0` or destroy — an owner going through hear_off
  with the flag set would `pthread_join` a thread that was never started.
  All three faked sections in state_test.c and both activation windows in
  the regression follow it.
- The gate (`gate-tests.sh`) `test -f`s BOTH notes files before anything
  else and `set -e` makes that a silent exit 1 — write the notes file before
  reading anything into a silent gate failure.
- Like phase 0: the new notes file is **staged, not committed**
  (`git add .agent-notes/2026-11-17-v125-test-integration.md`) so the gate's
  porcelain scope check sees the exact path; DO NOT COMMIT still holds.
- `git show 937a83e...:test/state_test.c` works in this repo (the object is
  present) — no need to touch the read-only original checkout.

## Rollback

Delete `test/state_test.c` and `test/tributary-regression.c`, restore
`test/run.sh` to its two-suite form (`git checkout -- test/run.sh`), and
`git rm --cached .agent-notes/2026-11-17-v125-test-integration.md` plus
delete the file. `src/probe.c` and `test/host_test.c` were not modified in
this phase, so nothing else to undo. No commits made, no device touched.

## Commands / conventions worth recording

- `sh test/run.sh` — now four suites, four `ALL PASSED` lines, one container.
- Scratch compile inside one container run (from phase 0 notes, still valid):
  `docker run --rm -v "$PWD":/w -v /tmp/opencode:/scratch -w /w gcc:12 sh -c
  'gcc -std=gnu11 -O1 -Wall -Wextra -Wno-unused-parameter -pthread -o /tmp/x
  /scratch/t.c /w/src/capture.c -lm && /tmp/x'` — /tmp does not survive
  between `docker run` invocations.
- Gating a pipeline without masking exit codes: write the gate's output to a
  file and echo `$?` from the command itself, not from `tail`.

## Standard-conflict note (per STANDARD.md self-healing clause)

No material conflict. Same wording tension as phase 0 ("committed with your
work" vs STANDARD rule 2 / the brief's DO NOT COMMIT): followed the stricter
rule — notes written and staged, left uncommitted for Taj's final pass.
