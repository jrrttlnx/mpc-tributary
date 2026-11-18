# 2026-11-17 — v1.2.5 core integration: T1 state persistence onto upstream mix ownership

Executor lane, phase 0 of 2 (attempt 2; attempt 1 failed the mechanical gate on
scope, not on code — see "Gate scope trap" below). Base: upstream
`39380af7937ec3a1c850bf9f84d6f412de391127` (v1.2.5), clean. Nothing committed,
nothing pushed, no device touched. Prior work read from
`/home/taj/workers/mpc-tributary-v125-integration-20261117/artifacts/`
(`local-t1.patch` = commit `937a83e` on the old base, `uncommitted-probe.patch`
= the uncommitted `use_from_mix` round, `upstream-v125.patch` = what upstream
v1.2.5 itself changed). The original `/home/taj/workers/mpc-tributary` checkout
was used read-only and is untouched.

## What changed and why

Upstream 1.2.5 (`39380af`) already stores project state, but as `v1 <mix-hex>
<level>` with a loose `sscanf`, and no active flag. The local lane had built
`T1 <level> <active> <mix>` with strict whole-chunk parsing on the older base.
This phase lands T1 on top of upstream and keeps upstream's protections.

`src/probe.c` (+141/−19):

- `state_num` / `state_field` / `state_save` / `state_load` carried over from
  `local-t1.patch` (commit `937a83e`): strict parser (digits only, no sign, no
  multi-digit leading zeros, one space per separator, range-checked fields,
  trailing junk refuses the whole chunk), and `T1 %d %u %x` (level, active,
  mix) as the canonical saved line. Mix is masked to `TB_MAX` (21), never to
  the connected map: bits below `TB_MAX` but past the map are kept and saved
  again; bits at/past `TB_MAX` are dropped, never remapped.
- **New legacy input path `state_load_legacy`**: a chunk that does not start
  with `T` is tried as upstream's `v1 <mix> <level>` (parsed with the same
  strict helpers, not the old `sscanf`). v1 carried no active flag, so it only
  fills level + routes and never starts or stops the copy — same as upstream's
  own restore. Old projects keep loading.
- Upstream's `mix_owned` protection retained untouched: `set_param`
  `pair`/`source` still early-returns when `mix_owned`; real switch changes
  still set `mix_owned = 1`. `state_load` and `state_load_legacy` both set
  `mix_owned = 1` after a valid restore, so a later SOURCE/pair write (the
  meter-refresh echo) cannot collapse the restored mix.
- The apply block in `state_load` is the local one: `active && !P->active` →
  `hear_on`, `!active && P->active` → `hear_off`, `active && P->active` →
  `use_apply` (re-points the shared read like a switch press). An inactive
  restore asks for nothing.
- **Carried forward from `uncommitted-probe.patch`: only the cold-load
  `switches_live = 0` reset**, applied after every valid restore (T1 and
  legacy). It re-arms the `saw_source && !switches_live` default-switch guard
  so the host's post-load replay of default switch values cannot clobber the
  restored routes even when the project loads after audio started.
- **NOT carried forward: `use_from_mix`** and everything it implied (inactive
  instances taking shared `g_use[]`/`P->used` decode ownership, the bit-0
  always-preserved rule). `use_apply` is byte-identical to upstream and keeps
  the active-only zero-mask invariant (`want = P->active ? mix : 0`).
- `get_param` `"state"` now returns `state_save(...)` (T1, refuses a too-small
  buffer with 0 = "no chunk" instead of truncating); `set_param` `"state"` is
  just `state_load(P, val)`. Upstream's `v1` writer/parser is gone.

`test/host_test.c` (adapted, minimal):

- The two upstream regression suites are kept as they were — they assert
  behaviour, not chunk text: the SOURCE-after-switch block ("an old SOURCE
  restores only that output" / "a later SOURCE write does not clear the
  switches") and the closing save/reload round-trip ("the project can store
  the switches" / "a reloaded project keeps the switches and the level"). Both
  pass unchanged under T1: the reload test survives because a restored copy
  refuses the replayed defaults (`s0=1`, `s2=0`) via the `saw_source` +
  `switches_live` guard while SOURCE is refused via `mix_owned`.
- Added `T1`-specific coverage, all behavioural: the saved chunk starts with
  `T1 `; an ACTIVE T1 restore decodes exactly the saved output (`g_use[5]`
  taken, `g_use[0]` left alone) and returns the decode on destroy; a SOURCE
  write after a T1 restore cannot replace the routes; a load mid-session
  re-arms the default-switch guard (render → real switch → T1 restore →
  replayed defaults suppressed); a legacy `v1` chunk restores level + routes
  without waking the copy and takes no decode ownership; six damaged T1
  chunks (trailing junk, missing field, wrong version, doubled space,
  out-of-range level) are refused whole.

## Verification

- `sh test/run.sh` → exit 0, `ALL PASSED` twice (host_test 82 PASS lines,
  capture_test 44), in gcc:12.
- `bash /home/taj/workers/mpc-tributary-v125-integration-20261117/gate-core.sh`
  → `CORE_GATES_OK`, exit 0.
- `use_from_mix` greps clean in `src/probe.c`; the only remaining `v1`
  mentions are comments and the legacy parser itself.

## Gate scope trap (this is what killed attempt 1)

`gate-core.sh` checks `git status --porcelain=v1` against an allowlist of
`src/probe.c`, `test/host_test.c`, and the notes file **path**. Porcelain v1
collapses an untracked directory to its name, so a new `.agent-notes/` tree
shows up as `.agent-notes/` — which is not on the allowlist → `OUT_OF_SCOPE`,
even though the file inside it is exactly the one the brief asked for. The
gate author clearly expected the path itself to appear in porcelain, which
only happens once the file is staged. So: **`git add
.agent-notes/2026-11-17-v125-core-integration.md` at the end of the phase**
(staging, not committing — the DO NOT COMMIT rule stays respected; Taj's
final pass does the actual commit). Same trick will be needed for phase 1's
`2026-11-17-v125-test-integration.md`.

## Rollback

Two files changed on a clean base: `git checkout -- src/probe.c
test/host_test.c` returns the tree to upstream v1.2.5, and
`git rm --cached .agent-notes/2026-11-17-v125-core-integration.md` (plus
deleting the file) undoes the staging. No commits made, no other tracked file
differs, no device touched.

## Tricky things to remember next time

- The strict parser is what makes whole-chunk refusal work: each field's
  separator is consumed by exactly one `state_field`, so a doubled/missing
  space can never silently skip into the next field. Don't "relax" it by
  adding space-skipping.
- Dispatch is `*p == 'T'` → T1, else legacy `v1`, else refuse. A chunk
  starting with `T` that fails T1 parsing is refused outright — it must not
  fall through to the legacy parser.
- Legacy `v1` is masked to `TB_MAX` (not `src_lim()`) for uniformity with T1;
  harmless in practice because upstream always masked to the map before
  writing a v1 line.
- `switches_live = 0` after restore is deliberate: `render()` sets it to 1 as
  soon as host audio runs, and a project loaded mid-session would otherwise
  let the host's default switch replay clobber the restored mix. The new
  "load mid-session re-arms the guard" test pins exactly this.
- Upstream `state` mask-to-lim at save is gone on purpose: T1 keeps bits below
  `TB_MAX` but past the connected map (same output returns when that machine
  model is connected again).
- Do not "simplify" `state_save`'s small-buffer refusal to truncation — the
  wrapper treats return 0 as "no chunk", and a cut line would be refused on
  load anyway.
- Host-test blocks that wake a copy must set `g_exited = 0; g_alive = 1`
  first (join path, no real thread) and must **not** let a copy that owns the
  read hear_off — owner-off calls `hear_join_locked`, which would
  `pthread_join` a thread that was never started. Destroy the copy instead
  (destroy only stops the read for the last instance) and clear `g_alive`
  by hand, like the existing blocks do.

## Commands / conventions worth recording

- `sh test/run.sh` runs natively here (Docker 29.9.3 + the gcc:12 image are
  on this host); it must print `ALL PASSED` twice.
- Scratch compile inside the same image run.sh uses:
  `docker run --rm -v "$PWD":/w -v /tmp/opencode:/scratch -w /w gcc:12 sh -c
  'gcc -std=gnu11 -O1 -Wall -Wextra -Wno-unused-parameter -pthread -o
  /tmp/x /scratch/t1_scratch.c /w/src/capture.c -lm && /tmp/x'`
  (`/tmp` does not survive between `docker run` invocations — compile and run
  in one container).
- The mechanical gates for this run live in
  `/home/taj/workers/mpc-tributary-v125-integration-20261117/` (`gate-core.sh`
  now, `gate-tests.sh` in phase 1); run them yourself before finishing.

## Standard-conflict note (per STANDARD.md self-healing clause)

No material conflict. One wording tension, reported not acted on: the brief
(and STANDARD's run-notes footer) says the notes file is "committed with your
work", while STANDARD's own rule 2 and the brief's DO NOT COMMIT say executors
never commit. Followed the stricter rule: notes written and **staged** (so the
mechanical scope check sees the exact path), left uncommitted for Taj's final
pass. STANDARD wins either way, so nothing to fix in the project docs.
