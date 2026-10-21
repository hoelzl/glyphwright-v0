# Tier-2 oracle follow-through — active handover

## Handover metadata

- **Status:** active
- **Owner:** unassigned
- **Created:** 2026-07-19
- **Updated:** 2026-07-19 after PR #27
- **Review required:** yes — Tier-2 click semantics, collision-envelope rules, and protocol/evidence changes require owner approval plus exact-candidate adversarial review
- **Successor:** none
- **Completion:** complete the Tier-2 corrected-run proof and dispose every remaining issue #25/art-pass item as completed, explicitly deferred, or transferred to a named successor; then archive this document

## Purpose and authority

This document replaces the earlier post-Windows-update handover. The old reconstruction and completed-work narrative is intentionally removed; only current continuation context remains.

It transfers three bounded workstreams:

1. implement and record a real Tier-2 collision-corrected click-to-move run;
2. resolve the two remaining semantic items in GitHub issue #25;
3. retain the deferred human-approved CC0 tileset pass without guessing asset mappings.

This is not a second status ledger. `docs/agent/design/roadmap.md` is the sole authoritative slice-status source, GitHub is authoritative for issue/PR state, and design `0012` owns the presentation/oracle contract.

## Verified current state

### Repository and remote

Verified on 2026-07-19 from Git and GitHub:

- Checkout: `C:\Users\tc\Programming\Python\Projects\glyphwright`.
- Branch: `main`, tracking `origin/main`.
- HEAD: `d7c0af7c01123a1234d7cc925c4240529a41bc3e` — `Harden UE5 drift-audit diagnostics and coverage (#27)`.
- Only registered worktree: the primary checkout.
- Only local/remote branch: `main` / `origin/main`.
- Open pull requests: none.
- Open follow-up: [issue #25](https://github.com/hoelzl/glyphwright-v0/issues/25), now limited to floor→wall drift semantics and the start-cell self-hit rule.
- This handover is the sole untracked file. Do not reset, delete, or accidentally stage unrelated files if that changes.

### PR #27 transition

PR #27 merged after its first exact-candidate adversarial review found an overly broad `float(...)` coercion boundary. The revised candidate:

- accepts only exact built-in JSON `int`/`float` trace distances;
- rejects booleans, non-numbers, custom coercibles, non-finite values, and negative distances with a located `UE5Error`;
- pins vertical-edge, multi-row/multi-edge, and non-default-tile audit behavior;
- fixes passable-edge canonicalization to numeric reading order beyond single-digit coordinates.

The revised exact candidate was **MERGE-READY** with no blocking findings. All eight current-head CI jobs passed: Ubuntu and Windows on Python 3.12–3.14, the core suite without extras, and package distributions. Final local evidence was `529 passed, 6 skipped`, Ruff clean, mypy clean on win32 and Linux, and both pre-commit stages green. The six skips were live-editor tests; no live evidence was claimed.

### Live editor and art fixture

Re-probed on 2026-07-19:

- `http://127.0.0.1:8000/` returned HTTP `000`: UE5/MCP is unreachable.
- `C:\Users\tc\Tmp\kenney_roguelike-rpg-pack.zip` is still present.

To restore the live fixture:

1. launch UE 5.8 with `G:\Unreal\Projects\Tests\MCPTest`;
2. load `/Game/Maps/EmptyOpenWorld1` unless the test deliberately names another fixture;
3. run `ModelContextProtocol.StartServer` if auto-start is disabled;
4. expect `curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8000/` to return `404` when ready;
5. inspect existing `gw_` actors before spawning anything and clean up every actor by its full `refPath` in `finally` blocks.

## Landed contract that must not be reopened accidentally

Design `0012` and merged 15A work establish:

1. GlyphWright's deterministic grid A* is Tier 1 and remains the backbone.
2. UE5 is an optional Tier-2 navigation/collision oracle.
3. Collision correction occurs in the presentation click macro, upstream of `step`.
4. The macro emits ordinary corrected `Move` commands; recording stores those commands.
5. Replay re-executes recorded commands and event digests without consulting UE5.
6. No oracle dependency, correction event, ambient I/O, or wall clock enters the pure kernel.
7. New recordings use `glyphwright.session/2`; replay accepts `/1` and `/2`.
8. Oracle and manifest fingerprints are optional and omitted for Tier-1 runs.
9. The coarse oracle fingerprint identifies level, plugin version, and semantic-position-key set; collision geometry remains in the separate on-demand audit.

Relevant seams:

- `src/glyphwright/frontends/presentation/clickmove.py`: synchronous Tier-1 expansion.
- `src/glyphwright/frontends/presentation/pathfinding.py`: pure seeded A*.
- `src/glyphwright/frontends/presentation/ue5/audit.py`: coarse passable-edge audit.
- `src/glyphwright/frontends/presentation/ue5/oracle.py`: live oracle fingerprint.
- `src/glyphwright/harness/recording.py`: recording factory currently writes the engine fingerprint immediately and still lacks a construction path for supplied oracle/manifest terms.
- `src/glyphwright/frontends/presentation/manifest.py`: `PresentationManifest.hash` is the canonical manifest fingerprint.

## Pending owner decisions

These recommendations have been explained but are **not yet accepted**. Do not implement them as settled decisions merely because they are recommended here.

### Decision 1 — Tier-2 consultation timing

- **Recommended: full-path preflight.** Compute the Tier-1 route, serially trace all candidate edges, exclude blocked edges, deterministically reroute, and emit no kernel command until a clear route exists.
- **Alternative: per-step consultation.** Trace immediately before each emitted move and reroute from the current cell when blocked.

The recommendation gives one click atomic semantics and prevents a click from partially executing before a later blocker is discovered. Per-step consultation reacts better to late geometry changes but permits partial execution. Dynamic per-step revalidation can be added later if real evidence requires it.

### Decision 2 — missing-wall direction in issue #25

- **Recommended: explicitly defer floor→wall auditing for now.** The current audit honestly detects only “semantic floor is blocked by geometry.”
- **Alternative: add a separate floor→wall pass now.** It must first define where collision is expected when tracing from a floor cell toward a semantic wall cell; arbitrary meshes may collide at the cell boundary, center, or elsewhere.

Do not widen `passable_edges`; its contract is semantic floor↔floor edges. A reverse-direction audit is a separate pass with separate result semantics.

### Decision 3 — start-cell self-hit rule in issue #25

- **Recommended: explicit probe-height contract.** `probe_height` must be strictly above the highest collision surface considered traversable floor; callers with taller floor geometry must choose a higher value.
- **Alternatives:** ignore very short hits, offset starts toward the destination, or model a richer movement volume.

Ignoring short hits can hide a real blocker immediately in front of the actor. A horizontal start offset does not reliably escape a raised floor whose collision surface remains above the ray. The probe-height rule is the narrowest honest contract for the deliberately coarse ray audit.

## Workstream 1 — Tier-2 corrected recording

**Entry gate:** owner accepts Decision 1. The live editor is required for final evidence, but fake-transport implementation can proceed first.

**Objective:** prove end to end that a human click is planned from GlyphWright topology, corrected by UE5 collision, compiled to ordinary moves, recorded with oracle and manifest fingerprints, and replayed without UE5 to the same final state.

**Execution:**

1. Record the accepted consultation semantics in design `0012` if they add specificity beyond its current text.
2. Use strict RED→GREEN→REFACTOR. The first failing test must put a UE5 blocker on the direct Tier-1 route while leaving a valid deterministic grid detour.
3. Add a presentation-layer Tier-2 macro with an injected fake `UE5Client`; never import MCP or UE5 into the kernel.
4. Extend pathfinding minimally to exclude oracle-blocked edges while preserving deterministic tie-breaking.
5. Serialize editor traces; never overlap game-thread calls.
6. Add a recording construction path that supplies `oracle=await oracle_fingerprint(client)` and `manifest=manifest.hash`. Tier-1 recording must remain unchanged and omit both terms.
7. Record each corrected move through `RecordingEngine.step`; do not record a click pseudo-command or correction event.
8. Replay with UE5 unavailable and assert event-digest success plus exact final snapshot equality.
9. Add an opt-in live e2e with real collision geometry and deterministic cleanup: prove the direct edge is blocked, record the detour, replay it, and remove every spawned actor in `finally`.
10. Update design `0012` and `roadmap.md` only for an actual contract/status transition.

**Acceptance:**

- the direct Tier-1 route crosses a scripted and live blocker;
- Tier 2 emits one deterministic alternate `Move` sequence avoiding the blocked edge;
- every emitted command is accepted by the kernel;
- the header is `glyphwright.session/2` with the live oracle object and `manifest == PresentationManifest.hash`;
- Tier-1 headers still omit both optional terms;
- replay succeeds with no editor and ends at the exact recorded snapshot;
- the live fixture is restored after success and failure;
- exact-candidate adversarial review checks kernel purity, command/replay soundness, serialized editor calls, and that the live test genuinely exercises correction rather than fingerprint population alone.

## Workstream 2 — finish issue #25

The coverage/diagnostic part is complete in PR #27. Remaining work begins only after Decisions 2 and 3 are accepted.

1. For an accepted probe-height contract, update the owning audit documentation and add tests that distinguish a valid configured envelope from an explicitly unsupported one. Do not imply the default 50 cm is universally sufficient.
2. If floor→wall auditing is accepted now, design it as a separate pass and pin deterministic report order and collision expectation before implementation.
3. Use live tests only where fake transport cannot establish the behavior; preserve actor cleanup.
4. Close issue #25 only after every original item is fixed, explicitly deferred, or rejected with evidence.

## Workstream 3 — deferred CC0 tileset pass

This remains blocked on human art direction and does not block Tier 2 or issue #25.

- Inspect the Kenney archive with the owner and select semantically correct floor, wall, character, hostile, item, and fixture assets.
- Do not infer character/item mappings from an architectural sheet merely to fill the table.
- Record source/license provenance, update stable crops/mappings, and obtain human review of visual baselines.
- Because this touches the GUI/optional-extra boundary, run both the all-extras gate and the bare-suite round trip from `AGENTS.md`, then restore the all-extras environment.

## Validation commands

```bash
export SDL_VIDEODRIVER=dummy
uv --no-config run pytest tests/test_clickmove.py tests/test_recording.py tests/test_ue5_oracle.py tests/test_ue5_audit.py -q
GLYPHWRIGHT_UE5_URL=http://127.0.0.1:8000/mcp \
  uv --no-config run pytest tests/test_ue5_e2e.py -q
uv --no-config run pytest --cov --cov-report=term-missing
uv --no-config run ruff check .
uv --no-config run ruff format --check .
uv --no-config run mypy --platform win32 src tests
uv --no-config run mypy --platform linux src tests
uv --no-config run pre-commit run --all-files
uv --no-config run pre-commit run --hook-stage pre-push --all-files
```

For tileset/GUI work, additionally:

```bash
uv --no-config sync --all-groups --locked
uv --no-config run pytest -q
uv --no-config sync --all-groups --all-extras --locked
```

## Non-negotiables

- Read `README.md`, designs `0003` and `0012`, relevant knowledge pages, and `docs/developer-guide/testing.md` before non-trivial edits.
- Use strict TDD for behavior changes.
- Never let UE5/MCP, filesystem, clock, network, or ambient entropy enter the kernel.
- Never make replay consult UE5.
- Never put collision geometry into the coarse oracle fingerprint without a new reviewed design/protocol decision.
- Treat UE5 as a persistent user fixture: inspect before spawning, capture full handles, and clean up on every path.
- Keep editor calls serialized on the game thread.
- Do not bless visual baselines without human review.
- Merge only after exact-candidate adversarial review and current-head CI. Material review-driven edits require a fresh review.
- The patch tool's parser may reject valid Python 3.12 PEP 695 test helpers; project Python, Ruff, mypy, and pytest are authoritative.

## Transition rules

- Mark **blocked** when a pending owner decision or unavailable UE5 fixture prevents all remaining safe work.
- Mark **complete** only when every workstream is completed or explicitly transferred/deferred with owner approval and all retained acceptance criteria pass.
- Mark **superseded** only when a named successor preserves every unresolved item.
- On completion or supersession, move this file to `docs/agent/handovers/archive/`, add the archive’s non-authoritative retirement banner, and repair references. Do not update `glyphwright-implementation-archive.md`.
