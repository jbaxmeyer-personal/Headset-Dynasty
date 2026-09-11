# Research Notes: Cricket Manager — Lessons for Headset Dynasty

Full findings from an automated review of `jbaxmeyer-personal/cricket-manager` (the team's earlier desktop sports management sim), conducted to extract concrete architecture lessons before building Headset Dynasty. See `docs/GAME_DESIGN.md` for how these were folded into actual decisions.

## 1. Tech Stack

Godot 4.6 (GDScript), Windows Desktop + Android exports — a native game-engine project, not web/React-Native-adjacent. Persistence is a single in-memory `Dictionary` tree serialized to a flat JSON file per save slot — no database, no schema migrations. Data-generation tooling (Python scripts for player generation/statistical calibration, plus two legacy Node.js scripts) lives outside the engine and hand-duplicates rating-formula math from GDScript — two implementations of the same logic kept in sync manually.

## 2. Architecture — Simulation vs. UI Separation

**The ball-by-ball match engine (`match_engine.gd`) is cleanly isolated** — pure data-in/data-out functions (`simulate_ball()`, `simulate_innings()`), no UI or scene-tree coupling. This validates that a framework-agnostic core is achievable in practice.

**Everything outside the live match is not separated.** The career simulation (auctions, scouting, training, aging, injuries, AI teams, board confidence) all lives inside one 6,658-line autoloaded singleton (`career_state.gd`) that is simultaneously the global state store *and* the simulation logic. The UI layer (`hub.gd`, 8,787 lines) contains 406 direct references into that state, including 23 direct field mutations from UI event handlers, and even generates some game data (random scout/staff candidates) directly inside UI-building functions.

**Bottom line:** the "isolate the pure engine" pattern worked for match resolution but was never extended to the season/career layer, and that's exactly what created the tangle that made the project hard to build and would block porting to a second UI.

## 3. Data Model

Players are flat dictionaries with 5 grouped attribute categories (~30+ granular 1-20 stats: technical batting/bowling/fielding, mental, physical), feeding **derived summary ratings** (1-100, e.g. `batting_rating`) via documented weighted formulas — with an explicit code comment: *"display-only and must never drive match outcomes."* The granular attributes drive the sim; the summary rating is UI sugar only.

All 220 players are **hardcoded as GDScript source literals**, generated offline by Python tooling and manually pasted in — no runtime data file, no import pipeline. This was tenable for a fixed 10-team fantasy league; it would not scale to a much larger, evolving roster set.

## 4. Complexity Hot Spots

Three files exceed 3,700-8,800 lines each (`hub.gd` 8,787, `career_state.gd` 6,658, `match_screen.gd` 3,752) — insufficient modularization of UI/state code specifically (the match engine, by contrast, stayed a reasonable 1,024 lines).

A five-session bug-fix log (`PLAN.md`) documents concrete production bugs directly traceable to the architecture: a new career inheriting ~45 unreset state variables from a previous save (autoload singleton never fully reset), two scouts independently picking the same player (no dedup), a wicket-replacement bug corrupting roster state, and a class of "freed object" crashes from UI closures capturing state that outlived its scope — a direct symptom of UI-owns-closures-over-global-state.

Repo-wide "Phase N" comments run from Phase 2 to Phase 25, and a release-readiness doc candidly frames the main remaining risk as "systems complete, but nobody confirmed it *feels* like the sport" — corroborating a long, iterative, genuinely difficult build even though git history was squashed to one commit.

## 5. What Worked Well

- Pure-function match engine, no UI coupling.
- Granular sim attributes → derived display rating via a documented formula, with an explicit rule that the display number never feeds back into the sim.
- Small, clean metadata files kept separate from big data blobs (`team_data.gd`, 70 lines).
- **Statistical calibration against real-world benchmarks**: an offline Python harness runs thousands of simulated matches and auto-tunes engine weights until output matches real IPL scoring distributions, plus a 30-season long-run check for rating inflation/collapse.
- **Headless in-engine regression harness**: a GDScript test runner simulates 30 full seasons through the real auction/match-engine code paths (not mocks) and exports CSV balance reports.

## 6. Project Size / Scope Signal

~24,500 lines of GDScript + ~4,250 lines of Python/JS tooling — a substantial, feature-complete solo build (full career mode, auction system, drag-and-drop squad management, animated match screen, scouting, finances, awards, tutorial), not a prototype. Internal evidence (25 build phases, a structured multi-session bug-fix plan, calibration/regression tooling) confirms a genuinely long, iterative effort consistent with "awesome but difficult to build."

## Recommendations Actually Adopted for Headset Dynasty

1. Extend the "pure data-in/data-out engine" discipline to the **entire simulation layer**, not just play resolution — recruiting, development, coaching AI, economy all need the same treatment as the match engine got, or the same tangle will recur when porting to React Native.
2. Keep a hard separation between attributes that drive the sim (hidden 0-100 numbers) and what the player sees (letter grades) — already the plan, and this research confirms the pattern works and the "never feeds back" rule matters enough to state explicitly.
3. Build a headless, multi-season statistical calibration + regression-testing harness early, running the real engine code (not a mock) — folded into the recommended build order.
4. Treat player/team data as external, structured, runtime-loaded data from day one (already the plan via the custom-DB architecture) — this research is a direct cautionary tale for why that matters at scale.
5. Keep individual files/modules reasonably scoped — avoid single files exceeding a few thousand lines, since that's where the concrete, documented bugs came from.
6. Build in an early external playtest checkpoint rather than validating "does this feel like the sport" only near the end.
