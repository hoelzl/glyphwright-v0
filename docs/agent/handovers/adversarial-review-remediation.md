# Adversarial-review remediation — active handover

## Handover metadata

- **Status:** active
- **Owner:** unassigned
- **Created:** 2026-07-24
- **Updated:** 2026-07-24 (Phase 0 complete; Phase 1 not started)
- **Review required:** yes — kernel semantic changes (Phases 1–3), the event-schema bump
  (Phase 4), and the version-policy CI guard (Phase 5) require exact-candidate
  adversarial review before merge, per the house rule
- **Successor:** none
- **Completion:** every finding in the register below is fixed, explicitly deferred with
  owner approval, or rejected with evidence; the eight accepted decisions are recorded in
  their owning design documents; then archive this document

## Purpose and authority

On 2026-07-24 a five-lens adversarial review (kernel, design corpus, harness/replay,
frontends/UE5, test suite) was run against `main` at
`d7c0af7c01123a1234d7cc925c4240529a41bc3e` with a fully green suite (529 passed,
6 skipped — the live-editor tests). Every finding below was novel against that green
suite. The review itself lives only in a non-durable agent session, so this document
records the findings in full and is their sole durable source.

The owner reviewed the design-decision points interactively on 2026-07-24 and **accepted
the eight decisions recorded below**. They are settled; do not re-litigate them. Open
items that were *not* decided are listed under "Open questions".

This is not a status ledger. `docs/agent/design/roadmap.md` remains the sole
authoritative slice-status source; design `0003` remains the specification; `0012` owns
the presentation/oracle contract.

**Relationship to the parallel handover** `docs/agent/handovers/tier2-oracle-next-session.md`
(Tier-2 oracle follow-through): that handover stays authoritative for its three
workstreams and its own pending Decisions 1–3, which this document must not decide or
reopen. Two touch points:

- The unwired session/2 oracle/manifest construction path (finding H3 below) is
  **owned by the Tier-2 handover** (workstream 1, step 6). Do not fix it here.
- Phase 8's UE5 hardening must not reopen the landed contract listed in that handover
  ("Landed contract that must not be reopened accidentally"), and the clickmove rule
  accepted here (Decision 7) is deliberately consistent with — but does not decide —
  that handover's pending Tier-2 consultation-timing decision.

## Verified current state

- Branch `main`, HEAD `d7c0af7` ("Harden UE5 drift-audit diagnostics and coverage (#27)").
- Untracked files at review time: `docs/agent/handovers/tier2-oracle-next-session.md`
  (an active handover that violates the multi-agent tracked-files contract by being
  uncommitted — Phase 0 fixes this) and, once written, this document.
- Suite: 529 passed, 6 skipped (only `tests/test_ue5_e2e.py`, live-editor gated),
  ~90 % branch-inclusive coverage, Ruff clean, mypy clean on both platforms.
- No phase of this handover has been started.

## Accepted owner decisions (2026-07-24)

These are **decided**. Each phase below cites the decision it implements.

### Decision R1 — mode-resume revalidation via a Mode protocol hook

**Context:** fleeing an interrupting battle pops only the battle; a suspended
dialogue/lockpick resumes with its reach precondition broken (confirmed: remote
lockpicking and loot delivery from another room).

**Accepted:** re-validate when a suspended mode is uncovered — and, per the owner's
explicit refinement, **not** as dialogue/lockpick special cases: the `Mode` protocol
gains a resume hook (e.g. `resume(state) -> tuple[Event, ...]`, default no-op) that the
kernel invokes whenever a `ModePopped` uncovers a mode. A mode whose precondition no
longer holds returns its own closing events (e.g. `DialogueEnded`, `MinigameAborted`)
and pops itself. Framework-external modes (games built outside the repository)
implement the same protocol and get the same guarantee. Rejected alternatives:
per-command adjacency checks (breaks grammar honesty — an advertised choice could be
rejected — unless grammar generation also becomes reach-aware) and accept-and-document
(remote looting is indefensible as game semantics).

### Decision R2 — effect params: reject at load, clamp folds

**Accepted:** `validate_params` gains range rules (`amount >= 0`, `spread >= 0`,
duration ≥ 1, and analogous bounds for every primitive), making the mid-step crash and
the negative-heal path unrepresentable from content; the `Healed` fold additionally
clamps at 0 as defense in depth. Rejected: supporting negative heal as damage (real
drain semantics deserve their own design, not a remediation patch) and clamp-only
(silently reinterprets author intent, contradicting the loader's located-diagnostics
philosophy).

### Decision R3 — engine-version policy: checklist rule plus CI guard

**Accepted:** AGENTS.md's completion checklist gains the rule *any behavior change under
`kernel/`, `effects/`, `modes/`, `world/` bumps the package version*; a CI job fails a
PR that touches those paths without a version change, with an explicit override label
for provably behavior-free refactors. Rejected: checklist-only (discipline-only
enforcement is the exact failure mode the review found) and a separate replay-compat
constant (still manual, adds a header term without removing the discipline problem).

### Decision R4 — event schema: bump to v10 now, discriminated per type

**Accepted:** replace the flat `glyphwright.event.v9.json` with a discriminated
`oneOf`-per-event-type schema (required payload fields per type,
`additionalProperties: false`), following the frame schema's existing pattern; retire
v9; and write down the retire-and-replace policy that three documents invoke but none
defines. There are no external consumers yet; this is the cheapest moment the bump will
ever have. Existing recordings invalidate — Phase 4 pairs the bump with a version bump
(Decision R3) so they refuse loudly at the header instead of dying mid-run with
"events diverged".

### Decision R5 — ADR-001: implement the claimed machinery

**Accepted:** make the four documents true as written by adding mechanical enforcement
that `glyphwright.*` never imports `termverify.*`. Preferred shape: a dependency-free
pytest that walks the package's import graph (ast-based) and fails on any `termverify`
reference; adding the `import-linter` package is an acceptable alternative if the test
proves awkward. Rejected: amending the docs to admit discipline-only enforcement.

### Decision R6 — tall grids: load-time budget diagnostic

**Accepted:** the loader refuses areas exceeding the documented viewport budget with a
located error naming the limit — the same philosophy as every other pack diagnostic.
The exact row/column budget must be derived from the render code
(`src/glyphwright/frontends/gui/scene.py` / `paint.py` assume 9 rows;
`src/glyphwright/frontends/tui/render.py` has its own region budget) and documented in
design `0005` as part of the pack contract. Viewport scrolling remains a possible later
slice; render-time clamping was rejected (hides world content, per-frontend drift).

### Decision R7 — clickmove mid-path contract: stop and discard

**Accepted:** design `0012` pins the rule: a consumer of `expand_click` executes the
expansion until the first rejection, `MoveBlocked`, or mode push, then **discards the
remainder**; the next click replans from wherever the player stands. Add a test placing
an occupier mid-path. Rejected: replan-on-refusal (oscillation risk against moving
blockers, more macro state, and battle interrupts must abort the walk regardless).

### Decision R8 — golden blessing: explicit rule, keep discipline

**Accepted:** AGENTS.md gains a hard rule — never regenerate goldens to make a red test
pass without the diff being reviewed as a semantic change — and
`docs/knowledge/verification-model.md` is softened to describe the actual mechanism
(the diff is the review) instead of claiming goldens are "updated only by humans".
Rejected: a CI provenance-trailer gate (gameable by the same agent it guards against,
adds friction).

## Findings register

IDs are stable; cite them in commits/PRs. Severity: C critical, M major, m minor.
Status: CONFIRMED = reproduced or traced precisely during the review; PLAUSIBLE =
mechanism verified, scenario not reproduced. Phase = where it is fixed.

### Kernel

| ID | Sev | Status | Finding | Phase |
| --- | --- | --- | --- | --- |
| K1 | M | CONFIRMED | `src/glyphwright/kernel/step.py:48` compares `next_rng != state.rng` instead of the `rng` argument; `step(state, Look(), Rng.from_seed(999))` with a non-state cursor raises `AssertionError` — 0003 §5.1 explicitly promises arbitrary-cursor stepping | 1 |
| K2 | M | CONFIRMED | `src/glyphwright/effects/abilities.py:36` (`Ability.effects`) and `Hook.effects` carry plain-`dict` leaf params; probe mutated `ability_defs['firebolt'].effects[0][1]['amount']` through a snapshot, retroactively rewriting all aliased states — breaks replay if anything ever mutates one | 1 |
| K3 | M | CONFIRMED | `src/glyphwright/modes/battle.py:126-145` `_flee` pops only the battle; resumed dialogue/lockpick never re-checks reach — reproduced remote lockpicking with loot delivery from another room | 2 |
| K4 | M | CONFIRMED | `src/glyphwright/effects/primitives.py:39-57` `validate_params` checks types only; `spread=-1` loads clean then `rng.between(0, -1)` raises mid-step, violating the module's own "load-time diagnostics, never mid-session crashes" promise | 1 |
| K5 | M | CONFIRMED | `src/glyphwright/kernel/state.py:204-212` `Healed` fold has no lower clamp and no death check; `heal` with `amount=-50` left a goblin at hp −44, alive, in `hostile_actors`, no `ActorDied` | 1 |
| K6 | m | CONFIRMED | `src/glyphwright/effects/primitives.py:102-103` `_heal` emits `Healed(amount=0)` at full hp, contradicting the evidence doctrine enforced in `common.use_item` ("'You recover 0 hp.' is not a thing that happened") | 3 |
| K7 | m | CONFIRMED | `src/glyphwright/world/entities.py:258-264` status refresh keeps `max(new, existing)` expiry, but the emitted `StatusApplied.expires` carries the shorter value — the event lies about the resulting clock | 3 |
| K8 | m | CONFIRMED | `src/glyphwright/kernel/scheduler.py:5-6` docstring claims the queue is area-filtered; `_queue` uses unfiltered `hostile_actors` and cross-area chasing is intended — docstring drift | 9 |
| K9 | m | PLAUSIBLE | `src/glyphwright/api.py:229-231` `zip(strict=True)` in `_validate` turns any future verb-arity/domain-count mismatch into a raised `ValueError` instead of a `Rejected`; nothing pins the pairing | 3 |
| K10 | m | PLAUSIBLE | `src/glyphwright/kernel/state.py:131-135` portal tokens overwrite same-named space exits by dict merge; loader reserves only `arena`/`return` (`content/pack.py:251`) — a pack can silently shadow a geometry exit | 3 |
| K11 | m | PLAUSIBLE | `src/glyphwright/content/loader.py:169-172` `Openable.contains` is validated for existence only — may reference an actor, another chest, or an item already in an inventory; the `ItemAcquired` fold would corrupt state | 3 |

### Content loading

| ID | Sev | Status | Finding | Phase |
| --- | --- | --- | --- | --- |
| L1 | C | CONFIRMED | `src/glyphwright/content/pack.py:113-118` checks uniqueness for ability/status/area ids but never entity ids; `api.py:128`'s dict-build keeps only the last duplicate — a copy-pasted `[[entity]]` id vanishes with no diagnostic while `pack_id` hashes both | 1 |
| L2 | m | CONFIRMED | `src/glyphwright/content/pack.py:75-82` validates mp against `0..max_mp` but not hp; `hp = 99, max_hp = 10` and negative hp load silently | 1 |
| L3 | m | CONFIRMED | `src/glyphwright/frontends/gui/tiles.py:36-38` ignores unknown top-level tables in `tileset.toml`, inconsistent with `manifest.py:83-86` which refuses them precisely so a typo is a diagnostic | 7 |
| L4 | M | CONFIRMED | No clamp of grid cells to the GUI's 9-row viewport budget (`gui/scene.py:44,121-139,204-233`, `gui/paint.py:75-83`): a tall pack paints over exits/status/log **and** mints overlapping click targets (cell targets first in `Scene.targets`, so a click on an exit label resolves to a different cell's `Move`); `tui/render.py:98` silently truncates the same case, possibly including the player's glyph | 7 |

### Harness, replay, schemas, wire

| ID | Sev | Status | Finding | Phase |
| --- | --- | --- | --- | --- |
| H1 | M | CONFIRMED | `schemas/glyphwright.event.v9.json` is one flat object: only `schema`/`turn`/`type` required, all payload fields optional and freely mixable — a `Moved` with no actor plus a lockpick `pins` field validates | 4 |
| H2 | M | CONFIRMED | Version `0.1.0` unchanged across ~27 behavior-changing PRs; replay's exact-string check (`harness/recording.py:145-151`) can therefore never fire — skew surfaces as misleading "events diverged", and frame-only behavior changes replay `ok=True` | 5 |
| H3 | M | CONFIRMED | session/2 oracle/manifest terms are write-only: `RunFingerprint.create` (`harness/fingerprint.py:60-62`) takes neither, `Engine.fingerprint()` (`api.py:165-169`) leaves them `None`, `oracle_fingerprint()` has no production caller — **owned by the Tier-2 handover; do not fix here** | — |
| H4 | m | CONFIRMED | `recording.py:140-161` never validates the header against the committed schema (missing `harness`, `"oracle": 42`, extra keys all replay); and `COMPATIBLE_SESSION_SCHEMAS` still accepts session/1 whose schema was deleted in PR #23 — an accepted wire format with no committed schema | 4 |
| H5 | m | CONFIRMED | `harness/query.py:57-58` answers `mp → [0, 0]` for mana-less actors that the frame encoder deliberately omits `mp` for (`frames/frame.py:25`, `wire.py:146`) — two evidence surfaces contradict, a false drift signal for any cross-checking harness; the same file's stats branch refuses to fabricate | 8 |
| H6 | m | CONFIRMED | `frontends/jsonl.py:52-63` surrogateescape-decodes invalid UTF-8 stdin and echoes lone surrogates into emitted rejection JSON — invalid interchange JSON that strict consumers reject | 8 |
| H7 | m | CONFIRMED | `frontends/jsonl.py:61` rejection hint omits most verbs (take/use/equip/attack/flee/talk/open/choose/pick/abort/cast); `jsonl.py:57` hardcodes `"glyphwright.rejection/1"` instead of the shared constant | 8 |
| H8 | m | CONFIRMED | A UTF-8 BOM (Windows editor resave) makes replay refuse with "the session header is not JSON" (`recording.py:137-139`; `cli.py:131` opens plain `utf-8`) — 0008 §1 sells recordings as hand-editable; the diagnosis misleads | 8 |
| H9 | m | PLAUSIBLE | `frontends/presentation/ue5/oracle.py:33` fingerprints positions via `"\n".join(sorted(set(keys)))` — a key containing `\n` collides with a key pair; keys are arbitrary editor strings | 8 |
| H10 | m | CONFIRMED | `tests/test_wire.py:45-69` `_check` skips any spec without a top-level `type` (the viewport `oneOf` validates nothing) and never recurses into cells/actors/grammar — schema-conformance evidence is shallower than docstrings claim | 4 |
| H11 | info | CONFIRMED | The events digest embeds the event-schema tag (`wire.py:169-174`), so a schema bump invalidates prior recordings with "events diverged" at step 1 rather than a version message — the diagnostics gap Phase 4+5 closes by pairing the bumps | 4 |

### UE5 bridge

| ID | Sev | Status | Finding | Phase |
| --- | --- | --- | --- | --- |
| U1 | M | CONFIRMED | `frontends/presentation/ue5/audit.py:112-142` returns only the drift tuple — an empty passable-edge set, wrong grid, or collision-free level yields `()`, indistinguishable from a genuine all-clear (`tests/test_ue5_audit.py:155-159` shows `WALLED` traces nothing) | 8 |
| U2 | M | PLAUSIBLE | `audit` never calls `current_level()` nor compares against the recording's `OracleFingerprint` — wrong map open traces the wrong geometry to a clean PASS; the fingerprint and the audit are two halves nothing composes | 8 |
| U3 | m | PLAUSIBLE | Each edge traced one direction only (canonical a→b, `audit.py:95-104,132-141`); one-sided geometry can pass a→b while blocking the walk b→a | 8 |
| U4 | m | PLAUSIBLE | `ue5/client.py:248-253`: no timeout on any MCP call (hung editor blocks forever); `json.loads` on non-JSON text raises raw `JSONDecodeError` instead of `UE5Error`; `_decode` re-parses string payloads so a tool returning the string `"123"` decodes to `int 123` | 8 |
| U5 | m | PLAUSIBLE | `ue5/oracle.py:48` assumes anchor items are dicts (raw `AttributeError` on malformed anchors); a zero-position fingerprint is well-formed and rides the header with no warning | 8 |
| U6 | m | CONFIRMED | `frontends/presentation/scenegraph.py:60-68,159-173` documents `Transition.kind` ∈ move/strike/spawn/despawn but only `move` is ever produced | 9 |

### Frontends and CLI

| ID | Sev | Status | Finding | Phase |
| --- | --- | --- | --- | --- |
| F3 | M | PLAUSIBLE | `expand_click`'s mid-path-rejection contract has no owner — no in-repo consumer executes multi-move expansions; a naive loop injects remaining moves into the wrong mode after a refusal or battle push | 7 |
| F4 | m | CONFIRMED | `frontends/presentation/pathfinding.py:48` never validates the start: `find_path` from a wall cell returns a path from inside the wall | 7 |
| F5 | m | CONFIRMED | `frontends/presentation/clickmove.py:52` reads `engine.snapshot()._state` (private kernel state) for topology instead of an api-level accessor | 7 |
| F6 | m | PLAUSIBLE | `frontends/gui/session.py:119` lets `pygame.display.set_mode` failure (headless, no `SDL_VIDEODRIVER`) escape as a raw traceback, unlike the missing-extra case which gets a CLI hint | 7 |
| F7 | m | CONFIRMED | `cli.py:78-81,87-115` entangles `--replay` with frontend checks it never uses: `--frontend tui --replay f` with piped stdin exits 2 with "needs an interactive terminal" (exactly the CI case); `--frontend gui --replay` demands pygame and loads tilesets, then the replay branch returns before using any of it | 7 |

### Tests and CI

| ID | Sev | Status | Finding | Phase |
| --- | --- | --- | --- | --- |
| T1 | C | CONFIRMED | The fold invariant is tested tautologically: production computes the successor *from* the fold (`step.py:38`) and `tests/test_kernel.py:79-83` re-executes it, asserting only player position and turn for one `Move`; the scheduler's incremental composition and the re-applied RNG-stamped `TurnAdvanced` (`step.py:44-46`) have no whole-state `fold(prior, events) == returned_state` test anywhere | 6 |
| T2 | C | CONFIRMED | `.github/workflows/ci.yml:35-38`: nothing proves the extras are *present* in the quality job — if `--all-extras` drifts, `importorskip("pygame")` (`test_gui_e2e.py:21`) and five `importorskip("anyio")` files silently skip the entire GUI e2e and offline UE5 surfaces while CI stays green | 6 |
| T3 | M | CONFIRMED | Coverage is measured, never enforced: no `fail_under` (`pyproject.toml:62-68`), no `--cov-fail-under` in CI; `cli.py` at 35 % and `frontends/jsonl.py` at 52 % (subprocess-tested, invisible to coverage) | 6 |
| T4 | M | CONFIRMED | `tests/test_properties.py:79-113`: wall / known-space / mode-stack invariants asserted only at walk end (transient violations pass); the mode-stack whitelist `("exploration",)\|("exploration","battle")` misfires the first time a walk reaches dialogue; no walk steers toward player defeat or resource exhaustion | 6 |
| T5 | M | PLAUSIBLE | Golden self-blessing is one `regenerate_goldens.py` command away while `verification-model.md:22` claims "updated only by humans" (Decision R8); also `tests/goldens/ue5_capture_probe.png` is an orphan the `*.txt`-only orphan check (`test_goldens.py:34`) cannot see | 6 |
| T6 | m | CONFIRMED | `tests/test_package.py:6-7` compares `importlib.metadata.version("glyphwright")` to itself — vacuous | 6 |
| T7 | m | accepted | Offline UE5 tests verify the client against a hand-written fake of the editor's wire shape; the real-shape tests (`test_ue5_e2e.py`) are env-gated and never run in CI — documented opt-in design (`pyproject.toml:59`), recorded here as a known limitation, no action | — |
| T8 | m | CONFIRMED | No Hypothesis profile/derandomization/`.hypothesis` CI cache — property runs are not run-to-run reproducible except via the printed blob | 6 |
| T9 | m | CONFIRMED | CI's dual-platform mypy coverage is incidental: plain `mypy src tests` relies on the OS matrix; dropping a matrix leg silently drops that platform's narrowing check, while local pre-push hooks pin both `--platform` flags explicitly | 6 |
| T10 | m | CONFIRMED | The bare job (`ci.yml:56`) proves only pygame's absence, not `mcp`/`anyio`'s — an `anyio` leak into the dev group would run UE5 offline tests under a false isolation assumption | 6 |

### Design corpus and documentation

| ID | Sev | Status | Finding | Phase |
| --- | --- | --- | --- | --- |
| D1 | M | CONFIRMED | 0003 §16.1 (and 0002, `verification-model.md:28`, `testing.md:24`) claim ADR-001 is "enforced mechanically" via import-linter + a bare TermVerify job; neither exists — the only matches for "import-linter" in the repo are those four documents (Decision R5) | 6 |
| D2 | M | CONFIRMED | 0012 §1/§6 revise 0003's terminal-first premise and determinism contract; 0003's Revisions table records nothing, and README:3 plus `docs/knowledge/product-vision.md:14` still assert the retired framing — the single-spec governance mechanism is silently false | 9 |
| D3 | M | CONFIRMED | Determinism-contract statements inconsistent across 0003 §2 Goal 3, README:64, `architecture.md:39`, `verification-model.md:49`, `product-vision.md:20` vs 0012 §6; no document states crisply that the oracle term is *generation provenance*, not a replay-determinism input (corrections live in the recorded commands, so replay determinism still holds on the original four terms) | 9 |
| D4 | m | CONFIRMED | 0003 §5.2/§10 specify a message-log cursor that does not exist in `kernel/state.py:73-101` (`terminology.md:15` propagates it); `WorldState` carries `focus`, `ability_defs`, `status_defs` beyond §5.2's inventory, unrecorded | 9 |
| D5 | m | CONFIRMED | 0003 §11's `SemanticFrame` sketch still uses `list` types after the Revisions table outlawed exactly that for §5.1; code uses tuples | 9 |
| D6 | m | CONFIRMED | `world/space.py:70-82` widened the Space protocol with `contains` and `melee_range` (load-bearing, geometry-dependent semantics) — unrecorded in 0003 §7.1's otherwise meticulous log | 9 |
| D7 | m | CONFIRMED | 0003 §10's menu/inventory mode was never implemented (inventory is exploration verbs) and battle verbs drifted (`attack/use/flee/look/cast` vs "attack/skill/item/flee") — neither deviation recorded | 9 |
| D8 | m | CONFIRMED | 0003 §13's `:save`/`:load`/`:events --since` are unimplemented and `:save`'s semantics are undefined (0008 §2 forbids disk snapshots) — annotate the deferral and the open semantics in 0003 | 9 |
| D9 | m | CONFIRMED | 0003 §16.3 and `verification-model.md:44` name a `glyphwright play` subcommand; the CLI is flat argparse with no subcommands | 9 |
| D10 | m | CONFIRMED | 0011 §8 Q1 still reads "Still open, and now actionable" after 0012 §9 declared it absorbed; 0011 carries no supersession note for its retired §7 claim | 9 |
| D11 | m | CONFIRMED | 0012 §6 and roadmap row 14 claim "seeded tie-breaking" A*; `pathfinding.py:13-16` is deliberately (and correctly) unseeded — the decision exists only in a docstring | 9 |
| D12 | m | CONFIRMED | README carries a second, stale copy of slice status ("All six designed slices…", README:7; "all three slices shipped", README:82) — AGENTS.md forbids exactly this | 9 |
| D13 | m | CONFIRMED | The active Tier-2 handover is untracked — invisible to other agents/clones, violating the multi-agent contract | 0 |
| D14 | m | CONFIRMED | `schemas/glyphwright.rejection.v1.json:12-14` types `reason` as a bare string; modes mint reasons ad hoc — an adapter cannot distinguish "rejected as designed" from "engine fault", the distinction 0003 §6 says the type exists for | open |
| D15 | M | CONFIRMED | The "retire-and-replace" schema policy is invoked (0004 §4, 0009 §4, roadmap rows 2/7/12) but defined nowhere, and prior schema versions are deleted — 0003 §16.4's compatibility-matrix story is unimplementable from the committed corpus | 4 |
| D16 | m | CONFIRMED | Small drift: 0008 cites "0003 §5.2" for the determinism-terms claim (lives in §2/§5.4); CONTRIBUTING:18 and README:72 show single-platform mypy vs AGENTS.md:28's dual run; roadmap row 15 wording overstates session/2 ("implemented" — carriage is schema/type/test-level only, per H3); ADR-007's plain-frontend Unicode tileset flag never implemented nor recorded as deferred | 9 |
| D17 | m | CONFIRMED | 0003 A.2/A.5 state "grammar is topology, not permission" without domain bounds; 0004/0009 correctly adopt the opposite for abilities — the boundary of a load-bearing fuzzing contract lives in a subordinate doc's parenthesis | 9 |

## Phase breakdown

Recommended order: 0 → 1 → 6a → 2 → 3 → 4+5 (one PR or adjacent PRs) → 6b → 7 → 8 → 9.
6a (the fold property test) is deliberately pulled before the kernel-touching Phases 2–3
so the new invariant test guards those changes. Phases 7 and 8 are independent of 2–5
and can interleave. Phase 9 runs last so the doc sweep can reference every change.
Strict TDD throughout: watch each focused test fail for the intended reason first.

### Phase 0 — housekeeping [DONE]

Commit `docs/agent/handovers/tier2-oracle-next-session.md` and this document (D13).
No code. Acceptance: `git status` clean. Done in commit `c26c322` (2026-07-24).

### Phase 1 — confirmed kernel/loader bugs [TODO]

Implements Decision R2. Fixes K1, K2, K4, K5, L1, L2.

1. K1: change `step.py:48` to compare against the `rng` argument. Failing test first:
   `step(state, Look(), Rng.from_seed(999))` must not raise and must return the passed
   cursor untouched.
2. K2: make effect params immutable at construction — deep-freeze the leaf mappings
   (`MappingProxyType` over copies, or a frozen params dataclass) in
   `Ability`/`Hook.__post_init__` or at loader boundary. Test: mutating a loaded
   ability's params raises `TypeError`.
3. K4+K5: range rules in `validate_params` for every primitive (sign, minimum
   durations); `Healed` fold clamps at 0. Tests: negative `spread`/`amount` packs
   refuse at load with located errors; a synthetic `Healed(amount=-50)` fold cannot
   drive hp below 0.
4. L1: entity-id uniqueness in `ContentPack.__post_init__`, same style as
   ability/status ids. Test: duplicate `[[entity]]` ids refuse with a located error.
5. L2: hp bounds check mirroring the existing mp check.

Acceptance: all five fixes land with their failing-first tests; full local gate green.

### Phase 2 — mode-resume protocol [TODO]

Implements Decision R1 (K3). Design-touching: amend 0003 (Mode protocol section) in the
same PR.

- Extend the `Mode` protocol with `resume(state) -> tuple[Event, ...]` (default no-op).
- The kernel invokes it at the seam where `ModePopped` uncovers a mode (the fold or the
  step path — choose the location that keeps the successor state exactly the fold of
  the returned events; the resume hook's events must be part of the step's event tuple,
  not a side channel).
- Dialogue implements: speaker still within reach (per the Space protocol's
  `melee_range`/co-location semantics) else `DialogueEnded` + pop. Lockpick: openable
  reachable else `MinigameAborted` + pop.
- Tests: the two reproduced scenarios (flee from battle-over-dialogue, flee from
  battle-over-lockpick) end with the interactive mode closed and no remote loot; a
  same-room flee resumes normally.

Acceptance: reproduction scenarios fixed; fold property test (Phase 6a) still green;
0003 amended via its Revisions table.

### Phase 3 — kernel evidence and latent-hole polish [TODO]

Fixes K6, K7, K9, K10, K11.

- K6: `_heal` emits no event at full hp (match `common.use_item` doctrine).
- K7: emit `StatusApplied.expires` equal to the value the fold actually records
  (the max), so the event is honest evidence.
- K9: a test pinning verb-arity/domain-count pairing across all registered grammars, so
  the `zip(strict=True)` can never fire at runtime unnoticed.
- K10: loader refuses portal tokens that collide with geometry exit tokens of the
  anchor tile (extend the reserved-token check).
- K11: loader validates `Openable.contains` references an item entity that is not an
  actor, not another openable, and not already in an inventory.

Acceptance: each with failing-first tests; full gate green.

### Phase 4 — event schema v10, header validation, retire-and-replace policy [TODO]

Implements Decision R4. Fixes H1, H4, H10, H11, D15. **Land with or immediately after
Phase 5 so old recordings refuse on version, not digest.**

- Write `schemas/glyphwright.event.v10.json` as a discriminated `oneOf` per event type
  (required payload fields per type, `additionalProperties: false`), following the
  frame schema's viewport pattern. Bump `wire.py`'s event tag; regenerate affected
  goldens (with review, per Decision R8).
- Replay validates the session header against the committed session schema before
  accepting it (closing H4's missing-`harness` / `"oracle": 42` / extra-keys holes),
  with typed `Replay` errors, never raises.
- Recommit `schemas/glyphwright.session.v1.json` — see policy below.
- Deepen `tests/test_wire.py::_check` to recurse into `oneOf` branches and nested
  specs (H10).
- Improve the digest-mismatch diagnosis: when the recording's event tag differs from
  the engine's, report a version-skew message, not "events diverged" (H11).
- **Write the retire-and-replace policy** into 0003 §16.4 (or a new ADR): *a schema
  version remains committed for as long as any code path accepts it; only versions no
  code accepts are retired; the first external consumer freezes the then-current
  versions and flips the policy to additive evolution.* This single rule makes the
  session/1 recommit mandatory and defines the flip criterion three docs currently
  hand-wave.

Acceptance: malformed events (wrong-type payload mix) fail validation in tests; a v9
recording refuses with a version message; policy recorded; goldens regenerated with
reviewed diffs.

### Phase 5 — version policy and CI guard [TODO]

Implements Decision R3 (H2).

- Add the bump rule to AGENTS.md's completion checklist.
- CI job: fail when a PR's diff touches `src/glyphwright/{kernel,effects,modes,world}/`
  without a `version =` change in `pyproject.toml`; support an override label
  (e.g. `no-behavior-change`) for provably behavior-free refactors.
- Bump the version now (first real bump), paired with Phase 4's schema bump.
- Record the policy in 0008 (which currently promises version fragility it never had).

Acceptance: the guard demonstrably fails a kernel-touching test PR without a bump;
docs updated.

### Phase 6 — verification hardening [TODO]

Implements Decisions R5, R8. Fixes T1–T6, T8–T10, D1. Split: **6a** (fold property
test) lands before Phases 2–3; the rest (**6b**) any time after.

6a:
- T1: property test asserting **whole-state** equality `fold(state_before, events) ==
  next_state` after every step of grammar-driven random walks, including battle rounds
  (the scheduler path) — full structural equality, RNG cursor included.

6b:
- T4: move wall/known-space/mode-stack assertions inside the walk loop (per step);
  replace the mode-stack whitelist with "stack is always a prefix-valid chain rooted at
  exploration"; add a walk steered toward player defeat and consumable exhaustion
  asserting the grammar collapses correctly and no advertised command is rejected.
- T2: extras-presence guard in the quality CI job (`python -c "import pygame, anyio"`
  or a collection/skip-count assertion) so a drifted `--all-extras` fails loudly.
- T10: extend the bare job to assert `mcp`/`anyio` absence alongside pygame's.
- T3: set a coverage floor (`fail_under` near the current 90 %, minus a small margin)
  and enforce it in CI.
- D1: the ADR-001 import test — ast-walk every module under `src/glyphwright/` and fail
  on any `termverify` import (Decision R5); wire into the normal suite so both CI jobs
  run it. Update the four documents to describe the actual mechanism.
- T5: AGENTS.md golden rule per Decision R8; soften `verification-model.md:22`; extend
  the orphan check in `test_goldens.py` to all files in `tests/goldens/`, with an
  explicit allowlist for non-test evidence artifacts (or relocate
  `ue5_capture_probe.png` out of the goldens directory).
- T6: replace `test_package.py`'s self-comparison with an assertion that pins
  something falsifiable (e.g. version matches `pyproject.toml`).
- T8: add a derandomized Hypothesis CI profile (or `.hypothesis` cache) so CI property
  failures are reproducible; keep randomized exploration locally or in a scheduled job.
- T9: pin explicit `--platform` mypy runs in CI (matching pre-push) instead of relying
  on the OS matrix incidentally.

Acceptance: 6a in place before Phase 2 merges; deliberately breaking the fold (a
scratch mutation) makes 6a fail; removing pygame from the env makes the quality-job
guard fail; the ADR-001 test fails on a scratch `import termverify`.

### Phase 7 — loader/frontend contracts [TODO]

Implements Decisions R6, R7. Fixes L3, L4, F3–F7.

- L4/R6: derive the exact viewport budget from the GUI and TUI render code, refuse
  oversized areas at load with a located error, and document the budget in 0005 as
  pack contract. Remove nothing from the renderers — the diagnostic makes the overflow
  unreachable.
- F3/R7: pin stop-and-discard in 0012; add the mid-path-occupier test; if the GUI
  mouse handler is upgraded to multi-move expansion in this phase, it implements the
  rule — otherwise the rule waits, pinned, for the first consumer.
- L3: `tiles.py` refuses unknown top-level tables, mirroring `manifest.py`.
- F7: `cli.py` short-circuits `--replay` before frontend/tiles/interactivity checks.
  Test: `--frontend tui --replay <file>` with piped stdin replays successfully.
- F4: document (or validate) `find_path`'s start precondition — a `ValueError` on a
  non-passable start is the honest cheap contract.
- F5: add an api-level read-only topology accessor and migrate `clickmove.py:52` off
  `_state`.
- F6: catch `pygame.error` from `set_mode` and exit with a diagnostic naming
  `SDL_VIDEODRIVER=dummy` for headless use.

Acceptance: tall-grid pack refuses with located error; replay-under-pipe works; bare
suite still green (this phase touches the optional-extra boundary — run the bare-suite
round trip from AGENTS.md).

### Phase 8 — harness and bridge robustness [TODO]

Fixes H5–H9, U1–U5. **Coordinate with the Tier-2 handover; do not reopen its landed
contract or implement its H3.**

- H5: `query.py` refuses `mp` for mana-less actors exactly as its stats branch refuses
  (same wording pattern), restoring query/frame agreement.
- H6: `jsonl.py` rejects undecodable input with a clean typed rejection (ascii-safe
  `command` echo, e.g. backslash-replace) instead of emitting lone surrogates.
- H7: build the hint from the live grammar (or drop it) and use the shared
  rejection-tag constant.
- H8: open recordings tolerant of a BOM (`utf-8-sig`) or detect it and refuse with a
  message naming the BOM.
- H9: length-prefix or hash-per-key the oracle positions fingerprint join (protocol
  detail of an optional term with no production writer yet — change now, before the
  Tier-2 workstream wires it; note the change in 0012).
- U1: `audit` returns (or exposes) the traced-edge count; zero-edge audits are a
  distinct explicit outcome, never a bare clean pass.
- U2: `audit` cross-checks `current_level()` against the expected level (parameter or
  recording fingerprint) and refuses on mismatch.
- U3: trace each edge in both directions (2× cost, closes one-sided geometry).
- U4: timeouts on all MCP calls; wrap `JSONDecodeError` in located `UE5Error`; guard
  `_decode`'s string re-parse against non-object results.
- U5: validate anchor item shape before `.get`; warn (in the returned structure) on a
  zero-position fingerprint.

Acceptance: fake-transport tests for every item; live-editor evidence not required
(the Tier-2 handover owns live proof); all offline UE5 suites green.

### Phase 9 — documentation trueing sweep [TODO]

Fixes D2–D12, D16, D17, K8, U6. One PR; use each document's own revision mechanism.

- D2/D3: fold 0012's premise revisions into 0003's Revisions table with pointers;
  sweep README and `product-vision.md` off the retired terminal-first framing; add one
  crisp paragraph (0008 or 0012) stating the oracle term is generation provenance and
  replay determinism holds on `(engine, pack, seed, commands)`.
- D4–D9, D16, D17: the itemized corrections from the register (phantom log cursor,
  list→tuple sketch, Space protocol log entry, mode/verb deviations, §13 deferral
  annotation, `play` subcommand, citation fixes, dual-platform mypy in
  CONTRIBUTING/README, roadmap row 15 wording, ADR-007 deferral note, grammar-boundary
  statement in 0003).
- D10: mark 0011 §8 Q1 absorbed; add the supersession note for §7.
- D11: record the unseeded-A* decision in 0012 §6 and fix roadmap row 14.
- D12: replace README's slice-status sentences with a pointer to roadmap.md.
- K8: fix the scheduler docstring; U6: fix the `Transition.kind` doc or emit the
  documented kinds.

Acceptance: walk AGENTS.md's documentation checklist explicitly; grep spot-checks for
each retired phrase ("terminal-first" in the swept files, "seeded tie-break",
"glyphwright play", "updated only by humans", "six designed slices") return clean.

## Open questions (not decided — do not implement as settled)

1. **D14 — rejection-reason vocabulary.** Enumerating a closed `reason` vocabulary in
   the rejection schema (v2 bump) would give adapters the designed/fault distinction
   0003 §6 promises, but freezes a surface that is still growing. Recommendation when
   raised: enumerate in documentation first (0003 appendix), bump the schema only when
   the first external consumer needs to assert on it. Needs an owner decision at
   Phase 4 or later; deliberately excluded from the Phase 4 bump.
2. **Viewport camera** (rejected-for-now alternative of Decision R6) — a candidate
   future slice, owner-scheduled via roadmap, not this handover.

## Current status

- **Completed:** the adversarial review itself; the eight owner decisions (recorded
  above); Phase 0 (both handovers tracked, commit `c26c322`). No remediation code
  exists yet.
- **In progress:** nothing.
- **Blockers:** none. Every phase can start now; no live UE5 fixture is required
  (Phase 8 is fake-transport only).
- **Tests:** suite green at HEAD `d7c0af7` (529 passed, 6 skipped live-editor).

## Next steps

Phase 1. Phase 1's five fixes are independent
of each other; K1 is a one-token change whose test is described above. Gotchas:

- K2's freeze must not change `pack_id` hashing (`content/pack.py:252-282` hashes
  canonical `asdict` content — verify the hash of the reference pack is unchanged, or
  document the bump as a deliberate consequence).
- Phase 6a before Phases 2–3: the fold property test is the guard rail for the mode
  and evidence changes.
- Phase 4+5 pairing is deliberate: bump the engine version in the same PR (or the
  immediately preceding one) as the event-v10 bump so pre-bump recordings refuse at
  the header with a version message.
- Any phase touching `frontends/gui` or the extras boundary (Phase 7, parts of 6)
  must run the bare-suite round trip from AGENTS.md, not just the normal gate.

## Key files and architecture notes

Files to be modified, by phase (no files exist yet for this work):

- Phase 1: `src/glyphwright/kernel/step.py`, `src/glyphwright/effects/abilities.py`,
  `src/glyphwright/effects/hooks.py`, `src/glyphwright/effects/primitives.py`,
  `src/glyphwright/kernel/state.py`, `src/glyphwright/content/pack.py`; tests
  `test_kernel.py`, `test_loader.py`, `test_stats.py`/`test_abilities.py`.
- Phase 2: `src/glyphwright/modes/__init__.py` (protocol), `modes/dialogue.py`,
  `modes/lockpick.py`, `kernel/state.py` or `kernel/step.py` (resume seam),
  `kernel/events.py` if a new closing event is needed; design 0003.
- Phase 3: `effects/primitives.py`, `world/entities.py`, `api.py` (test only),
  `content/loader.py`, `content/pack.py`.
- Phase 4: `schemas/` (new v10, recommitted session.v1), `frontends/wire.py`,
  `harness/recording.py`, `tests/test_wire.py`, goldens; design 0003 §16.4.
- Phase 5: `pyproject.toml`, `.github/workflows/ci.yml`, `AGENTS.md`, design 0008.
- Phase 6: `tests/test_properties.py`, `tests/test_kernel.py`, new ADR-001 import
  test, `tests/test_goldens.py`, `tests/test_package.py`, `.github/workflows/ci.yml`,
  `pyproject.toml`, `AGENTS.md`, `docs/knowledge/verification-model.md`.
- Phase 7: `content/pack.py` or `content/loader.py` (budget), `frontends/gui/tiles.py`,
  `cli.py`, `frontends/presentation/{clickmove,pathfinding}.py`, `api.py` (topology
  accessor), `frontends/gui/session.py`; designs 0005 and 0012.
- Phase 8: `harness/query.py`, `frontends/jsonl.py`, `harness/recording.py`,
  `frontends/presentation/ue5/{oracle,audit,client}.py`; design 0012 (H9 note).
- Phase 9: documentation only (files per the register).

Patterns to preserve: located diagnostics (`PackError` with file/table/key), typed
rejections/`Replay` errors instead of raises at every boundary, the single
`canonical_json` choke point, sorted-id iteration, events as the only state channel
(the resume hook's events must ride the step's event tuple), and TDD with
failing-first evidence.

## Testing approach

Strict TDD per AGENTS.md. Standard gate for every phase:

```bash
uv --no-config run pytest --cov --cov-report=term-missing   # SDL_VIDEODRIVER=dummy headless
uv --no-config run ruff check .
uv --no-config run ruff format --check .
uv --no-config run mypy --platform win32 src tests
uv --no-config run mypy --platform linux src tests
uv --no-config run pre-commit run --all-files
uv --no-config run pre-commit run --hook-stage pre-push --all-files
```

For phases touching the extras boundary (7, parts of 6), additionally the bare-suite
round trip:

```bash
uv --no-config sync --all-groups --locked
uv --no-config run pytest -q
uv --no-config sync --all-groups --all-extras --locked
```

No live UE5 evidence is required by any phase here; Phase 8 is fake-transport only.

## Session notes

- The review's overall verdict, for context: the architecture is sound and the
  determinism/replay core is genuinely strong (90+ seeded walks found zero fold
  violations and zero advertised-command rejections). The dominant defect *pattern* is
  guarantees enforced by construction or discipline rather than by falsifiable checks,
  plus documented enforcement machinery that was never built. Phases 5 and 6 are the
  structural answer; the rest are point fixes.
- The owner accepted every recommended option, with one refinement: Decision R1's
  resume revalidation must be a general `Mode`-protocol capability usable by
  framework-external modes, not a dialogue/lockpick special case.
- PLAUSIBLE findings (K9–K11, F3, F6, U2–U5, H9, T5) were mechanism-verified but not
  reproduced; write their failing tests first — if a test cannot be made to fail, the
  finding is wrong and should be rejected with evidence in the register, not silently
  dropped.
