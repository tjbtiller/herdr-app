# Upstream issues + fork patches — runtime gotchas from the chat-mod build (2026-11-12)

Six runtime gotchas hit while building the native chat mod were re-verified
against upstream master (`17eac390`, 2026-11-11) with isolated repro mods
(scratch `HERDR_APP_HOME`, `/tmp/herdr-repros` fixtures). Five are real at
HEAD and filed upstream; one claim from our README was a misdiagnosis and
filed as the transport gap it actually is.

## Issues filed (herdrdev/herdr-app)

| # | Gotcha | Verdict at HEAD | Issue | Our workaround |
|---|--------|-----------------|-------|----------------|
| 8 | vlist row renderers forbid all interactive handlers | Real **by design and enforced** (`interactive_inline_layout` guard, engine/quickjs/mod.rs; typings state it). Not a bug — a capability gap: one click target per row, no sub-targets, no row geometry for popup anchoring. Kills hover-action patterns for chat/feed mods | [#942](https://github.com/herdrdev/herdr-app/issues/942) (feature request: declarative action zones / opt-in stateful rows / row-geometry callback) | list-level `on_item_click` key routing + popup mods (`chat-msg-menu`, `chat-row-menu`) where handlers are legal |
| 9 | `VirtualListScrollHandle` exposes no position readback to JS | Real **binding gap**: Rust-side `ScrollbarHandle` impl has `offset()`/`content_size()`/`viewport_bounds()` (and state tracks `last_content_size`) but JS binds only `scroll_to_item`/`scroll_to_bottom`/`release`. No at-bottom detection → chat scroll-follow degenerates to wheel-direction guessing | [#944](https://github.com/herdrdev/herdr-app/issues/944) (feature request: bind offset_y/content_height/viewport_height/at_bottom) | intent-based follow: wheel-up exits, End key + jump-latest pill resume; wheel-down never yanks |
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

## Gotcha 7 (2026-11-12, found in the chat-mod adversarial audit)

**`crypto.webcrypto.subtle` is declared but has no backend — calling it silently
kills the calling task.**

Verdict at HEAD (`4c8c622`, checked against `upstream/master`): real and new.
`vendor/gpui-kit/crates/shell/src/typings.rs:3781` declares
`export const webcrypto: Crypto;`, and the `crypto` module is registered from
`quickjs_jit_stdlib::crypto::CryptoModule`, but nothing in `vendor/` or `src/`
implements `subtle` (`git grep -n "subtle"` finds only a palette field). There
is no issue filed for it (checked #926–#931 — none covers crypto).

Minimal repro (a 20-line mod, scratch `HERDR_APP_HOME`, no network):

```js
import { webcrypto } from "crypto";
cx.spawn(async () => {
  await w("1 start");
  await w("2 plain await OK");                  // logged
  const g = webcrypto.getRandomValues(new Uint8Array(4));
  await w("3 getRandomValues OK " + g.length);  // logged
  await w("4 about to call subtle.importKey (NOT awaited)");
  const pr = webcrypto.subtle.importKey("raw", new Uint8Array(32), "AES-GCM", false, ["encrypt"]);
  await w("5 returned a promise: " + (pr && typeof pr.then)); // NEVER LOGGED
});
```

Observed: the log stops on line 4. The call does not throw, does not return,
and does not reject — the task simply ends. Same stall via
`globalThis.crypto.subtle`, and `subtle.digest("SHA-256", …)` behaves
identically.

Second finding from the same probe: **`setTimeout` never fires in the mod
runtime.** `await new Promise((r) => setTimeout(() => r("yes"), 100))` never
settles, so a hanging `subtle` call cannot even be timed out from a mod. Use
`cx.sleep`, which is host-backed.

Failure mode is the #926 class (a task stranded with no exception and no log),
but the trigger is different: it is not a reload, it is the call itself.

Impact: any mod wanting AES-GCM / hashing in-process is silently dead. The chat
mod's live collab attach needs exactly this, so it now takes an explicit
`INPROC_CRYPTO = false` constant and a read-only fallback rather than parking
a task; sends ride an exec'd node helper where real `webcrypto` exists.

Filed status: **[#934](https://github.com/herdrdev/herdr-app/issues/934)**
(2026-11-12, re-verified live at 0.1.15 / upstream `17eac39` in an isolated
`HERDR_APP_HOME` window before filing).

Correction from the live repro: `setTimeout` is not "inert/never settles" —
it **throws** `TypeError: \`setTimeout\` is not available in the shell: use
cx.timer.after(ms, callback)` (catchable). The subtle strand is exactly as
claimed: the awaited promise never settles, the scheduler's unhandled-rejection
report does not fire, the app log stays empty, and sibling tasks keep running.

Fork-patch column: **not patchable our-side** — a real `subtle` backend is a
runtime-engine feature, and carrying a JS crypto shim in the fork just to
convert "strand" into "throw" would be heavier than the honest
`INPROC_CRYPTO = false` + exec'd-node-helper workaround already in the chat
mod. Collapses when upstream implements `subtle` (or makes it reject loudly).
