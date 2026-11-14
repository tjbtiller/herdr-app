# Upstream issues + fork patches — runtime gotchas from the chat-mod build (2026-11-12)

Six runtime gotchas hit while building the native chat mod were re-verified
against upstream master (`17eac390`, 2026-11-11) with isolated repro mods
(scratch `HERDR_APP_HOME`, `/tmp/herdr-repros` fixtures). Five are real at
HEAD and filed upstream; one claim from our README was a misdiagnosis and
filed as the transport gap it actually is.

**2026-11-13 dev sweep**: #930/#931 closed COMPLETED via docs PR #1006;
#944 is being implemented by the dev team (PR #971, open). Verdicts,
teachings, and closure triage: sections at the bottom of this doc.

## Issues filed (herdrdev/herdr-app)

| # | Gotcha | Verdict at HEAD | Issue | Our workaround |
|---|--------|-----------------|-------|----------------|
| 8 | vlist row renderers forbid all interactive handlers | Real **by design and enforced** (`interactive_inline_layout` guard, engine/quickjs/mod.rs; typings state it). Not a bug — a capability gap: one click target per row, no sub-targets, no row geometry for popup anchoring. Kills hover-action patterns for chat/feed mods | [#942](https://github.com/herdrdev/herdr-app/issues/942) **open**, no response yet (feature request: declarative action zones / opt-in stateful rows / row-geometry callback) | list-level `on_item_click` key routing + popup mods (`chat-msg-menu`, `chat-row-menu`) where handlers are legal |
| 9 | `VirtualListScrollHandle` exposes no position readback to JS | Real **binding gap**: Rust-side `ScrollbarHandle` impl has `offset()`/`content_size()`/`viewport_bounds()` (and state tracks `last_content_size`) but JS binds only `scroll_to_item`/`scroll_to_bottom`/`release`. No at-bottom detection → chat scroll-follow degenerates to wheel-direction guessing | [#944](https://github.com/herdrdev/herdr-app/issues/944) **open — dev team building it**: PR [#971](https://github.com/herdrdev/herdr-app/pull/971) (jaklimoff, open) binds `offset()`/`content_size()`/`viewport_size()` on both handles; covers readback half of #828 | intent-based follow until #971 ships: wheel-up exits, End key + jump-latest pill resume; then switch to real at-bottom (teachings §3) |
| 1 | `list()` renders zero rows | Real, but **intended** — dev verdict: the missing piece was documentation, not a runtime fix (zero-height container + module-export trap both now documented) | [#930](https://github.com/herdrdev/herdr-app/issues/930) **CLOSED COMPLETED 2026-11-13** via docs PR [#1006](https://github.com/herdrdev/herdr-app/pull/1006) (be6cede) | import `list`/`uniform_list` from `gpui-kit`; give list containers explicit height (`h(N)`) — now the documented pattern |
| 2 | `websocket` "capability-gated" | Misdiagnosis: TCP `ws://` connects fine (verified, incl. loopback). The real gap: **no Unix-domain socket transport** (`ws+unix://` rejected) — omp collab endpoints are unix sockets | [#928](https://github.com/herdrdev/herdr-app/issues/928) **open**, no response yet | exec'd node guest (`guest.mjs`) bridging the unix socket, link via file |
| 3 | nested async cx starves | Real at HEAD, exact mechanism found: a **mod reload silently strands pending sleeps/spawned tasks** (generation retirement → promise never settles, no log). Empirically isolated (E-loop died at reload, control timer re-registered by init) | [#926](https://github.com/herdrdev/herdr-app/issues/926) **open**, no response yet. Dev signal: b990a7a (#760) reworks reload machinery (mods move to a new script VM per window) but does not settle retired sleeps | register background work idempotently from `init()`; never let a waiter cross a reload; **fork patch B logs the strand** |
| 4 | `track_scroll` ignored on lists | Real — **intended and now documented**: typings now say `list`/`uniform_list` always keep their own position; a passed handle is ignored (app-log warn); pair a `Scrollbar` of the same id | [#931](https://github.com/herdrdev/herdr-app/issues/931) **CLOSED COMPLETED 2026-11-13** via docs PR [#1006](https://github.com/herdrdev/herdr-app/pull/1006) (be6cede) | none needed (Scrollbar pairs by name — now the dev-blessed pattern) |
| 5 | flex columns shrink-wrap | Real + undocumented: **mod content gets no definite height from its block** (px or weight); `flex_1`/`h_full` chains collapse silently (measured 25px vs explicit 99px; `h_full` is %-of-auto = no-op). The list-flavored slice got documented by PR #1006; the general block-anchoring rule + zero-height warn remain asked | [#929](https://github.com/herdrdev/herdr-app/issues/929) **open**, no response yet | explicit `h(N)` on the topmost container that must fill |
| 6 | `check` loads phantom copies | Real, narrower than feared: fs-device writes + shared.json ARE clamped (#515 fixed that); **exec() and `herdr.writeTextFile` still run for real during check** (marker-verified; children orphaned; CAS writes bypass) | [#927](https://github.com/herdrdev/herdr-app/issues/927) **CLOSED COMPLETED 2026-11-14** via upstream PR [#1033](https://github.com/herdrdev/herdr-app/pull/1033) merged @`75fabc7` (our patch A adopted, cosmetic rewording) | per-boot data files; ~~fork patch A~~ upstream-owned |
| 10 | synthetic pointer motion dead (hands-verify) | Real at HEAD (`69ba1ca`-era, fork patches diagnostics-only): uinput/ydotool **motion** never reaches the app — hover zero-effect, mid-drag zero motion (portal cursor metadata unchanged after a 732 px injection, held and unheld, not cadence-sensitive) — while clicks, press/release, and wheel land on the same widgets, and identical motion injections move GTK apps 1:1 in the same session. Automation can click but never drag/hover | [#1032](https://github.com/herdrdev/herdr-app/issues/1032) **open** (filed 2026-11-13, no response yet). Possibly-related upstream: #698 (real-mouse motion laggy — arrives but stutters, distinct), #5 (mod `on_hover` never dispatched), #157 (Windows-only motion-report gap) | keyboard carry for drag semantics (move/reorder via keys); clicks fully usable. Recorded in SKILL.md §5/§9 |

## Fork patches (branch `tyler`, rebase-managed by `~/tools/herdr-bridge-sync.sh`)

| # | Commit | Class | Collapses when |
|---|--------|-------|----------------|
| A | ~~`ac245a6`~~ **merged upstream** `75fabc7` (`feat: mods can see a check run (herdr.checking) (#1033)`, merged 2026-11-14 13:56Z, closes [#927](https://github.com/herdrdev/herdr-app/issues/927)) | **merged** | dropped at next rebase: upstream owns this logic. Semantic drift = cosmetic only — dev's warn strings are shorter (`check: mod exec'd \`{command}\`; the child runs for real` vs our `herdr-app check: … — the child runs for real during check`) and upstream dropped the `pub fn set_checking()` export (sets the flag inline instead). Same flag, same `herdr.checking()` export, same three warn sites. `tyler`'s patch A will conflict textually (same hunk regions) — take upstream's side wholesale. |
| B | `ff83cf2` `fix(shell): report script tasks stranded by an application reload` — `vendor/gpui-kit/.../scheduler.rs` | **PR open** (unchanged, 0 review/comment) | upstream PR [#1034](https://github.com/herdrdev/herdr-app/pull/1034) (2026-11-13, branch `fix/reload-stranded-tasks`) — no dev movement yet; master `5e578a0` touches the same file (window-side recovery, different crate) — no textual conflict |

Both are diagnostics-only (no behavior change): A adds a flag + log lines
+ one read-only export; B adds one `tracing::warn!` naming the stranded
task APIs at generation retirement.

Rebase receipt 2026-11-13: `tyler` rebased onto upstream master `69ba1ca`
(brings the dev's #1006 docs into our tree); both patches carried clean,
worktree clean, `upstream/master..tyler` = exactly the 5 fork commits,
nothing pushed, nothing deployed. Ledger SHAs above are post-rebase (were
`cb76d1d`/`ca8e90e`).

Upstream PR receipts 2026-11-13: no equivalent at upstream HEAD `b67715e`
(`checking` hits are update-check machinery only; `src/mods.rs` and the
scheduler unchanged since `69ba1ca`). Both patches re-cut onto `b67715e` as
single commits in upstream house style (subject-only, `feat:`/`fix:`,
`Co-Authored-By: omp` trailer) — PR wording adapted, not fork-lifted:
A inlines the flag-set in `install_check_policy`, upstream warn tone
(`check: mod exec'd ...`), and adds two lifecycle tests
(`herdr_checking_is_false_in_a_live_load` /
`herdr_checking_is_true_while_a_check_loads_the_mod`;
`cargo test --locked --bins herdr_checking` → 2 passed). B live-verified on
the real display (scratch home `HC_SESSION`-isolated): a placed mod parking
`cx.sleep(600000)` warns `(1 cx.sleep(ms))` when its layout switches away;
frame-mod timers warn at the same rebuild; a check run reports at exit.
Both PRs OPEN + MERGEABLE. When they merge: drop A/B from `tyler` and adopt
upstream's wording wholesale (the fork commits' wording differs).

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

## Verification note — synthetic motion deadness (2026-11-13, hands-verify round, receipts /tmp/gpui-flows-receipts.jsonl)

- Full-screen staged herdr-app on the real display (Wayland/niri), driven
  by a uinput/ydotool synthetic pointer; screenshots pixel-measured;
  pointer position cross-checked via xdg-desktop-portal screen-cast
  cursor metadata.
- Clicks reach GPUI widgets: card press/select produced a ~13.6k px
  render diff, detail popup on release. Wheel reaches GPUI: 119k px
  scroll over the chat pane.
- Motion never does: hover zero-effect; mid-drag zero motion — portal
  cursor position unchanged after a single 732 px injection (held and
  unheld), and a fast 6×40 px @ 60 ms probe changed nothing either (not
  cadence-sensitive).
- Contrast control: identical injections against a GTK app in the same
  session log held-motion 1:1 — the compositor and synthetic device are
  fine; the gap is in herdr-app's input path.
- Source read at HEAD (`69ba1ca`-era; the 5 `tyler` fork commits are
  diagnostics-only): `vendor/gpui-pre-linux/src/linux/wayland/client.rs`
  forwards `wl_pointer::Motion` unconditionally as
  `PlatformInput::MouseMove` (plus a synthesized MouseMove on Enter), and
  the gpui core `handle_input` path applies no motion filter — so the
  block is not an obvious motion filter; suspected deeper in
  hover/grab/pointer-seat state interaction with synthetic motion.
  Filed as [#1032](https://github.com/herdrdev/herdr-app/issues/1032);
  dupe sweep found only distinct neighbors (#698 laggy real-mouse motion,
  #5 mod on_hover never dispatched, #157 Windows motion reports).

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

## What the dev taught us (2026-11-13)

Sweep of herdrdev/herdr-app across our eight reports (#926–#931, #934, #942,
#944) plus the wider tracker, open and closed. The dev's answers ship as
commits and PRs, not issue comments — there is not a single comment on any
of our issues; the verdicts live in closure reasons and merged diffs.

**1. The dev answers by shipping.** Two of our issues (#930/#931) were
closed COMPLETED 36 h after filing, by a merged docs PR (#1006); our #944
feature request got a dev-team implementation PR (#971) within 24 h. Silence
on the rest ≠ rejection — the open backlog is days deep across all
reporters and barely uses labels. Read verdicts from `stateReason` + the
closing commit/PR, never by asking for an update.

**2. Silent-zero-height lists and track_scroll-on-`list` are INTENDED
(#930/#931).** The dev's fix is documentation, not runtime: PR #1006
(be6cede, "docs: explain lazy-list modules, height and track_scroll") added
to `defaults/AGENTS.md` and the typings, verbatim:

- "`list` and `uniform_list` come from `gpui-kit`, where `div` is;
  `v_virtual_list` and `h_virtual_list` come from `gpui-base`."
- "They need a definite height from their container: a `flex_1()` chain
  with no explicit `h(n)` anywhere lays out at zero and draws nothing."
- "`track_scroll` works on a plain scroll area and the virtual-list pair;
  `list` and `uniform_list` keep their own position, so pair a `Scrollbar`
  of the same id instead."

No runtime warn shipped for the zero-height case. Our workarounds are now
the documented patterns. → RUNTIME-LAWS.md Law 17 carries these with
be6cede as upstream authority.

**3. Scroll readback is coming, and its units are mod-pixels (#944 →
PR #971).** The dev's surface (open PR, not yet merged): `offset()` →
`{x, y}` (counts up from 0, never negative), `content_size()` and
`viewport_size()` → `{width, height}`, on BOTH `ScrollHandle` and
`VirtualListScrollHandle`. Rules that change our mod code:

- **Lengths are in the pixels a mod writes, not window pixels** — a row
  written `.h(30)` moves `offset()` by 30 at any zoom. Deliberately
  different from event positions and `window.viewport_size()`, so
  comparisons against our own row heights/`item_sizes` are zoom-safe.
- Values are the **last frame's**, all zero before the first frame —
  guard first-frame reads.
- Rounded to hundredths (no float dust across zoom division).
- NOT in the PR: `set_offset` and an `on_scroll` event (#828's other
  halves) — jump-to-unread still needs more than this.
- Their binding rule: one host function per reader.

Mod-side plan: probe (`typeof this.scroll.offset === "function"`) and
switch chat's scroll-follow from wheel-direction heuristics to real
at-bottom the day the binary carries #971. Use the dev's names/shapes
(`offset().y`, `content_size().height`) — our proposed
`offset_y()/content_height()` names lost.

**4. Reload machinery is under active investment, but the #926 strand is
untouched.** b990a7a (#760) moves a window's mods to a new script VM
before reloads exhaust it — the dev cares about reload semantics; our
settle-or-log ask stands. Fork patch B stays until they land it.

**5. The dev's NOT_PLANNED line** (from other reporters' closures, sweep
2026-11-13): shared-component behavior changes get declined — #814
(Radix-style popup flip/reposition in the shared `Popup`) NOT_PLANNED
2026-11-10; #834 (macOS MacRoman clipboard mangling) NOT_PLANNED. A report
framed "change the shared X so everything gets it" without a maintainer go
is dead on arrival. #814 matters to our popup mods: popups clamp into the
window, cover their trigger near an edge, and do not follow the trigger
when it scrolls — permanent. → RUNTIME-LAWS.md Law 18.

**6. Tracker vocabulary**: COMPLETED = fix shipped and the closing commit
references the issue; NOT_PLANNED = deliberate wontfix (rare); DUPLICATE;
"filed by mistake". Labels (bug/enhancement/…) exist but are barely
applied — `stateReason` is the signal. Also noted in passing: upstream
added omp itself as a built-in program (9e7b754, #730).

## Closure triage (2026-11-13)

For each closed item that touches our estate — fork-patch / reopen /
accept:

- **#930, #931 — closed COMPLETED (docs).** ACCEPT. Both were accurate and
  the dev turned them into the exact documentation we asked for; our tree
  has it via the rebase (be6cede in, mods swept: zero violations — no
  `list`/`uniform_list` import from `gpui-base`, no `track_scroll` on
  `list()` in any live mod).
- **#944 — open, dev PR #971 pending.** ACCEPT-WAIT. Nothing to do fork
  side; carrying the dev's own open PR would be reconciling churn.
- **Our issues closed NOT_PLANNED: none.** Nothing of ours was declined.
- **#814 (not ours, NOT_PLANNED, touches our popup mods).** ACCEPT AND
  DOCUMENT, no fork patch: the maintainer explicitly declined changing the
  shared `Popup`; a fork-side flip implementation would be a permanent
  divergence in a churned vendor file fighting upstream forever. The real
  need is met mod-side: size popup content so edge-clamping stays usable,
  close menus on scroll. Law 18 records it.
- **Reopens: none justified.** No closure left a door open — #1006 and
  #971 answered more than we asked; re-arguing anything would spend dev
  time against Tyler's bar.
- **New fork patches: none justified.** A and B remain the entire
  divergence ledger (both verified to carry clean on `69ba1ca`). A
  fork-side zero-height warn (#929's ask) would collide with any upstream
  warn; a `subtle` backend (#934) is engine-scale; unix-socket WebSocket
  (#928) is a transport feature upstream may take. Workarounds hold.

## Gotcha 8 (2026-11-13, audit pass-III residual R1 — closed in our estate)

**The bundled `agents`/`spaces`/`tabs` mods parse persisted localStorage values
with unguarded `JSON.parse` in `init()`: one corrupted value silently kills the
mod at every start.**

Verdict at upstream master `5e578a0`: real. Five sites — `agents/main.js:86-87`
(`changed`, `pins`), `spaces/main.js:84,86` (`machines.reconnecting`,
`machines.folded`), `tabs/main.js:18` (`panels`). The `?? "{}"` fallback covers
only a missing key, not a value whose payload isn't JSON. A malformed store
*file* is already rejected loudly by the engine (storage.rs, "not a valid store
file"); this class is one level up — a valid file with a corrupt value string.

Live repro (scratch `HERDR_APP_HOME`, upstream `tabs` init line verbatim,
0.1.15): corrupted `panels` value → cold start → no `probe` marker in the store,
no render, nothing in the app log or `herdr-app status` (silent). Same start
with the one-line try/catch guard → marker lands, mod loads with defaults.
Red-green marker receipts under `/tmp/r1-live`; red/green JS-expression probe
matches. Note: the engine reads the store once at startup — hot reload never
sees a corrupted file; only a cold start exercises the class.

Filed: **[#1028](https://github.com/herdrdev/herdr-app/issues/1028)**
(2026-11-13). No dupe of #926–#944/#1020 (keyword sweep: parse/corrupt/store/
localStorage/crash — nothing covers mod-side value parsing).

**Our estate already fixed** (same day, before filing): guarded at the source
per each mod's build workflow — `our-agents/main.js.bak-20261113:227-228`
(rebuilt with esbuild `--minify-whitespace`, `"use strict";` re-prepended;
shipped `main.js` sha256 `363e58f7…d4590181305ff8d26`), `our-spaces/main.js:106,108`,
`our-tabs/main.js:32` (plain sources, edited in place). `herdr-app check` PASS
×3; the shipped `our-agents` artifact carries all three guards
(`changed`, `pins`, plus the pre-existing `agents.cmdseen`). Collapses when
upstream lands #1028: our three mods are clones of the parents and can adopt
the upstream diff wholesale.

## Reporting discipline — the bar before filing anything upstream

(Learned 2026-11-13: two of our "bugs" were intended behavior. The
receipts made them fast wins, but the bug-vs-docs class call is ours to
get right — the dev's time is the scarce resource.)

1. **Repro against upstream HEAD**, not our fork: verify `tyler ==
   upstream/master` by rev, else check out and build upstream master;
   scratch `HERDR_APP_HOME`; the minimal repro mod actually executed in a
   live window (or `herdr-app check`), expected-vs-actual OBSERVED —
   marker files, pixel-measured screenshots, log lines — never assumed.
2. **Docs read first**: `defaults/AGENTS.md` + the `.d.ts` truth files
   (`defaults/herdr.d.ts`, `gpui-kit` typings) + source comments. Behavior
   that is documented or visibly intended is not a bug. Runtime coherent
   but typings imply otherwise → bug (docs/typings class, #931 pattern).
   Coherent but undiscoverable → documentation issue (#930 class — these
   get fixed fast). Capability missing → feature request with a concrete
   real-mod use case (#942/#944 class).
3. **Existing-issue sweep**, open AND closed, by symptom keywords AND by
   API name; check closure reasons — a NOT_PLANNED predecessor means do
   not re-file (see #814).
4. **Strip our-fork context**: no fork patches in the repro, no mod-estate
   specifics; a maintainer must be able to paste-run it. Version + commit
   named.
5. **One issue per defect**, title stating the observable behavior, not
   our diagnosis of the mechanism.
