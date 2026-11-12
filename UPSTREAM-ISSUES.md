# Upstream issues + fork patches — runtime gotchas from the chat-mod build (2026-11-12)

Six runtime gotchas hit while building the native chat mod were re-verified
against upstream master (`17eac390`, 2026-11-11) with isolated repro mods
(scratch `HERDR_APP_HOME`, `/tmp/herdr-repros` fixtures). Five are real at
HEAD and filed upstream; one claim from our README was a misdiagnosis and
filed as the transport gap it actually is.

## Issues filed (herdrdev/herdr-app)

| # | Gotcha | Verdict at HEAD | Issue | Our workaround |
|---|--------|-----------------|-------|----------------|
| 1 | `list()` renders zero rows | Real, but **not a list bug**: zero-height container + a module-export trap (`list` lives in `gpui-kit`, `v_virtual_list` in `gpui-base`); both failures are silent | [#930](https://github.com/herdrdev/herdr-app/issues/930) | import `list`/`uniform_list` from `gpui-kit`; give list containers explicit height (`h(N)`) |
| 2 | `websocket` "capability-gated" | Misdiagnosis: TCP `ws://` connects fine (verified, incl. loopback). The real gap: **no Unix-domain socket transport** (`ws+unix://` rejected) — omp collab endpoints are unix sockets | [#928](https://github.com/herdrdev/herdr-app/issues/928) | exec'd node guest (`guest.mjs`) bridging the unix socket, link via file |
| 3 | nested async cx starves | Real at HEAD, exact mechanism found: a **mod reload silently strands pending sleeps/spawned tasks** (generation retirement → promise never settles, no log). Empirically isolated (E-loop died at reload, control timer re-registered by init) | [#926](https://github.com/herdrdev/herdr-app/issues/926) | register background work idempotently from `init()`; never let a waiter cross a reload; **fork patch B logs the strand** |
| 4 | `track_scroll` ignored on lists | Real (by design, but typings imply otherwise; runtime warn invisible to mod authors) | [#931](https://github.com/herdrdev/herdr-app/issues/931) | none needed (Scrollbar pairs by name) |
| 5 | flex columns shrink-wrap | Real + undocumented: **mod content gets no definite height from its block** (px or weight); `flex_1`/`h_full` chains collapse silently (measured 25px vs explicit 99px; `h_full` is %-of-auto = no-op) | [#929](https://github.com/herdrdev/herdr-app/issues/929) | explicit `h(N)` on the topmost container that must fill |
| 6 | `check` loads phantom copies | Real, narrower than feared: fs-device writes + shared.json ARE clamped (#515 fixed that); **exec() and `herdr.writeTextFile` still run for real during check** (marker-verified; children orphaned; CAS writes bypass) | [#927](https://github.com/herdrdev/herdr-app/issues/927) | per-boot data files; **fork patch A adds `herdr.checking()` + warn logs** |

## Fork patches (branch `tyler`, rebase-managed by `~/tools/herdr-bridge-sync.sh`)

| # | Commit | Class | Collapses when |
|---|--------|-------|----------------|
| A | `cb76d1d` `feat(mods): surface check mode to mods (herdr.checking) + warn on exec/write side effects` — `src/mods.rs`, `defaults/herdr.d.ts` | droppable | upstream lands #927 (check clamp or `checking()` equivalent) |
| B | `ca8e90e` `fix(shell): report script tasks stranded by an application reload` — `vendor/gpui-kit/.../scheduler.rs` | droppable | upstream lands #926 (settle-or-log at generation retirement) |

Both are diagnostics-only (no behavior change): A adds a flag + log lines
+ one read-only export; B adds one `tracing::warn!` naming the stranded
task APIs at generation retirement.

## Verification notes (2026-11-12, repro artifacts under /tmp/herdr-repros)

- Built `17eac390` (== tyler == upstream master) with user-local Zig 0.16;
  ran a scratch-home app on the real display, drove it with `herdr-app do`
  / `shot`, pixel-measured screenshots.
- ws probes: `ws://127.0.0.1:7990` (bun echo server) connected in 6 ms;
  `ws+unix://` → TypeError; dead port → clean Connection refused.
- r3cx: A (init timer) 42 ticks, B (init spawn+sleep loop) 43, C
  (timer-callback spawn) 41, D (await-fs-then-sleep) fired; after `touch
  main.js`: A/B/C re-registered and resumed, the once-only E loop died
  silently — zero log lines.
- r6: `herdr-app check` on a live scratch home exec'd a child (marker
  file; `sleep 5` orphaned), `writeTextFile` wrote outside the app home,
  `fs/promises` writes correctly refused ("outside every granted write
  root").
- r5: pixel-measured — flex-chain column 25 px vs explicit `h(100)` 99 px
  in the same block; `h(400)` proves the chain fills when height exists.
