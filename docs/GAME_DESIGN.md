# Headset Dynasty — Game Design Document

**Status:** Living draft. This document reflects everything decided in planning conversations so far. Sections marked **TBD** are open and will be filled in as we keep talking. Nothing here is final until we've validated it feels good to build and play — but this is our source of truth for what we're building.

**Last updated:** 2026-09-12

---

## 1. Vision

Headset Dynasty is a mobile college football management sim in the spirit of Football Manager — but for college football, and built mobile-first from day one instead of ported from desktop. The defining principle, set explicitly against the lesson learned from the team's earlier Cricket Manager project: **plan extensively before building**, and build a **real simulation, not randomized outcomes dressed up as one**.

Core promise to the player: you are a football coach with a specific job (position coach, coordinator, or head coach) at a specific school, competing in a living, 130-team fictional FBS world, climbing a real coaching career ladder through recruiting, player development, and on-field results that are calculated from real player and team data — not dice rolls.

## 2. Platform & Technical Architecture

- **Frontend (now):** React + TypeScript, static site hosted on GitHub Pages.
- **Simulation engine:** Built as a framework-agnostic, pure TypeScript package with no UI/DOM dependencies. It must be able to run headless (e.g., simulate a full season and print results to a console) with zero React code involved. This is the single most important architectural constraint — it's what lets us port to React Native later by rewriting only the UI layer, not the game logic.
- **Backend:** Firebase (Firestore + Auth). Needed from day one because saves are cloud-synced, not local-only, despite the frontend being "just" a static site — the static site talks to Firebase over the network for auth and data.
- **Auth:** Social login only — Google and Apple. No email/password.
- **Future port:** React Native, targeting iOS first, Android after. The sim engine and Firebase data layer should require no rewrite for this; only the presentation layer changes.
- **Data schema:** Teams, conferences, players, coaches, and recruits are modeled as clean, documented data structures (see §8) from day one, because the custom-DB import feature depends on the "default" fictional world and any "custom" imported world being loadable by the exact same engine.
- **Architectural rule, learned the hard way from Cricket Manager** (see §14): the "pure, data-in/data-out engine, no UI coupling" discipline must apply to **the entire simulation layer** — recruiting, player/coach development, the coaching AI, and the program economy — not just play resolution. Cricket Manager's match engine was cleanly isolated, but its career/season simulation was fused into one giant autoloaded singleton that the UI mutated directly in hundreds of places, which is exactly what made it hard to build and would have blocked a second UI. Every system in Headset Dynasty's engine should be reachable only through plain functions taking/returning data, never through a UI screen reaching into shared mutable state.
- **Display vs. simulation values must stay separate**: attributes that drive the sim (hidden 0-100 numbers, §5.7) must never be replaced in formulas by whatever's shown to the player (letter grades, or any future composite "Overall" rating) — the display value is derived *from* the sim value, one direction only, never the reverse. This mirrors a rule Cricket Manager enforced explicitly (and a comment in its code warning display ratings "must never drive match outcomes").

## 3. Business Model

- **Premium, one-time purchase.** No ads, no IAP, no ongoing monetization systems to build.
- **Free tier exists as a trial/hook**, not a separate monetization strategy:
  - Free: choose **any school** and **any coaching role** (including Head Coach) from day one — no need to climb the ladder to sample the top job.
  - Free is capped at **2 total seasons**.
  - Paid: unlocks unlimited seasons.
- Exact price point: deliberately deferred — decide closer to launch once the game's actual depth/feel is known, rather than guessing this early.

## 4. Core Gameplay Loop & Coaching Roles

### 4.1 Role types

The player controls **one coach** on **one team**, in one of three role tiers:

- **Position Coach** — owns one position group.
- **Coordinator** (Offensive or Defensive) — owns one side of the ball.
- **Head Coach** — owns the whole program.

Role difficulty to obtain scales with school prestige: **any non-Power-4 school will readily hire you as a position coach with zero reputation required.** Every step up (bigger school, higher role) requires more reputation.

### 4.2 Authority per role

Whichever role you hold, **you have real, direct authority over your slice of the program**, not just advisory input:

- As a **position coach**: you recruit for your position, set your position's depth chart and in-game usage percentage, and manage your position group's player development (in-season and offseason point allocation).
- As a **coordinator**: full play-calling authority for your side of the ball (via gameplan/tendencies, see §5.2), plus recruiting/depth-chart/development authority across your whole side of the ball.
- As **Head Coach**: sets overall program direction, manages the budget/economy layer (§9), hires/fires assistant coaches, and has final program-wide authority — but does not micromanage the coordinators' individual play-calling.

Everything and everyone above and below your role is run by **full AI coaches** — not abstracted formulas. Every coach in the world (yours and every rival school's) is a simulated individual with their own skills, tendencies, and career.

### 4.3 Career ladder (job market)

Modeled on real coaching carousels, not a simple in-place promotion tree:

- Moving up is always **job-market driven** — you apply for openings, like Football Manager's job market, and compete against AI candidates.
- **Exception:** when your boss (your HC, or your coordinator if you're a position coach) leaves for a new job, and they liked your work, you can be:
  1. **Taken with them** to their new school, or
  2. **Promoted in place** by your current school to fill the vacancy, or
  3. Free to **apply elsewhere** on the open market instead.
- When **you** move up and take a new coordinator or HC job, you get to choose which of your subordinate staff to bring with you (a coordinator can bring position coaches; a new HC can bring both coordinators and position coaches).
- **AI coaches behave the same way** — the entire league is a living coaching carousel: AI coaches get fired for poor performance, get poached by bigger programs, and generate the job openings you compete for.

### 4.4 Reputation

Reputation gates which jobs you're eligible for. It **can go up or down** (unlike coach skill points, §4.5) and is a composite of:

- Overall win/loss record
- **Power 4 conference wins specifically weighted higher**
- Recruiting class rankings/success
- Player development (players drafted/developed into stars)
- **Prestige-weighted results** — beating a higher-prestige/ranked opponent matters more than beating a weak one

### 4.5 Coach skill points (progression)

A **separate stat from reputation**: coach skill points **only accumulate, never decrease**, earned from the same category of positive events (wins, Power 4 wins, signing recruits, players drafted). Spent as a **flat list of efficiency multipliers** — no role-gating (any coach can invest in any skill regardless of current role) and no prerequisite tree. Locked in so far:

| Skill | Effect |
|---|---|
| **Scouting** | Reduces the point cost to fully scout a recruit (§6.1). |
| **Recruiting** | Increases the effectiveness of your recruiting pitches — how much recruit interest your weekly actions generate (§6.2). |
| **Player Development** | Increases development points earned per practice (in-season drip and offseason bump, §7) for players in your unit. |
| **Tactics** | Boosts the in-game effective performance of whichever players/side of the ball you have direct authority over (§4.2) — a small positive nudge to your side's matchup differential (§5.1) in plays your role controls. |

**Deliberately left open-ended** — more categories may be added as they come up; this is not meant to be a complete/final list yet.

### 4.6 School / Program Prestige

Distinct from personal coach Reputation (§4.4): **Prestige is a program-level stat**, not a coach-level one. It belongs to the school, persists independent of who's coaching (including under AI coaches), and is inherited by whoever takes the job next — a coach walking into a blue-blood program starts with that program's existing Prestige, for better or worse.

**1-5 star scale** (more stars = better) — not the letter-grade scale used elsewhere (§5.7), matching the familiar recruiting-star convention instead. Driven by a similar input set to personal Reputation, but tracked at the program level: overall win/loss record, Power 4 wins/losses (weighted higher), conference win/loss record, recruiting class ranking, end-of-season team ranking, and players drafted.

**Mechanical effects:**
- **Sizes the weekly recruiting action-point pool** (§6.2) — a higher-Prestige program generates more recruiting points to spend on scouting and pitching each week, mirroring how blue-blood programs run bigger recruiting operations in real life.
- **Is the concrete implementation of the "program prestige & recent success" 25% factor** in the recruit commitment formula (§6.3) — that factor isn't an abstract number, it's this stat.
- **Complements, rather than duplicates, the NIL-collective-strength lever** (§9): Prestige represents brand/tradition, NIL represents money — both feed recruiting success as separate, stackable levers, matching how both actually matter in real college football recruiting.

## 5. Simulation Engine

### 5.1 Design philosophy

**Deterministic, ratings-driven outcomes — not random events.** Every play's outcome is calculated from the actual attributes of the players and coaches involved (matchup math: e.g., OL rating vs. DL rating, scheme fit, fatigue, coach tendencies) producing a weighted probability distribution that is then rolled — this is "legit sim," not coin-flips dressed up as football.

Critically, the engine outputs **structured play-by-play data** (ball position, players involved, yards gained, key events) rather than just a final score or flavor text. This means:

- Today: that structured data is rendered as text/stat lines.
- Later: if/when a 2D visual layer is added, it just **renders the data the engine already computed** — no rework of the simulation logic itself.

### 5.2 Play-calling

**Gameplan/tendency-based, not per-down play selection.** Before a game, the play-caller (whichever coach controls that side of the ball) sets a gameplan — run/pass balance, tempo, aggressiveness, formation tendencies — and can adjust it live during the game (e.g., "get more aggressive," "establish the run") rather than picking an individual play every single down. This keeps sessions fast (see §11.1) while still giving real strategic control.

### 5.3 Schemes

**Real, named offensive/defensive schemes with actual identity** (e.g., Air Raid, Spread Option, Pro-Style, 4-3, 3-4), each with genuine strengths/weaknesses. Schemes require certain player archetypes to run well — scheme fit matters both for recruiting (a recruit's play style should fit or not fit your scheme) and for how effectively your existing roster performs.

### 5.4 Situational factors

The following affect sim outcomes:

- **Home-field advantage** — real, if modest, edge; bigger for tougher road environments.
- **Weather** — modifies relevant attributes/play types (e.g., passing accuracy in rain, kicking in wind).
- **Rivalry games & momentum** — designated rivalry games carry extra unpredictability/upset potential; in-season momentum/morale can swing performance.

### 5.5 Special teams & situational realism

**Full realism**, not abstracted: real overtime rules, meaningful 4th-down/2-point/onside-kick decisions, kicking/punting attributes that matter.

### 5.6 Injuries

**Realistic, play-level risk model**: every play carries injury risk (weighted by play type/position), with severity/duration ranging from "shake it off" to season-ending, modified by a player durability attribute.

### 5.7 Attribute scale & display

Every attribute is a **hidden 0-100 number** under the hood, displayed to the player as a **letter grade** (A+ down to F, same shape as recruiting grades) so it reads like a scouting report, not a spreadsheet. Scouting investment (§6.1) determines what's actually visible:
- Under-scouted recruit → a range shown (e.g. "B- to A-")
- Fully scouted → the exact letter grade

### 5.8 Position groups (not rigid per-position schemas)

Rather than ~20 separate position-specific attribute schemas, players are modeled in **four broad groups**, each sharing one attribute set. This mirrors real recruiting/coaching practice (ATH designations, position "projections," in-career position changes) and deliberately **supports repositioning a player** during their career — something a rigid per-position schema couldn't do cleanly.

| Group | Covers | Group-specific attributes |
|---|---|---|
| **Passer** | QB | Accuracy Short, Accuracy Medium, Accuracy Deep |
| **Specialist** | K, P | Power, Accuracy, Clutch |
| **Line** | OL, DL | Run Block, Pass Block, Pass Rush, Run Defense, Tackling |
| **Athlete** | RB, WR, TE, LB, CB, S | Elusiveness, Break Tackle, Ball Security, Receiving, Route Running, Catching, Catch-in-Traffic, Release, Run Block, Run Defense, Man Coverage, Zone Coverage, Press, Ball Skills, Tackling, Pass Rush |

Long snapper is **not modeled as a distinct role/attribute** — snapping competence is abstracted away rather than tracked.

Every player also carries **7 universal attributes**, regardless of group: **Speed, Strength, Agility, Awareness, Durability** (injury risk modifier), **Stamina** (in-game fatigue), and **Potential** (hidden ceiling, never shown directly — only inferred from development rate).

Universal attributes deliberately absorb several traits that would otherwise be separate (per direct design decisions):
- QB: no separate Arm Strength (covered by Strength), no Play-Action/Pocket Presence/Decision Making (covered by Awareness), no separate Scramble/Mobility (covered by Speed + Agility).
- RB: no separate Vision (covered by Awareness).
- WR/TE: no separate YAC (covered by Speed + Agility).

**A player's full group-wide attribute set exists at all times**, but only the subset relevant to their **currently assigned position** is treated as "active" for play math and shown prominently in the UI — the rest are secondary/background (e.g., a starting WR's Man Coverage and Pass Rush grades exist but sit low and out of the spotlight until/unless he's ever moved to defense). **This "active attributes" presentation needs to be validated with an actual UI mockup before being treated as finalized** — noted as an open item, see §13.

### 5.9 Positional Familiarity

Because Athlete- and Line-group attributes are fully portable across the real positions within their group, an unrestricted "move anyone anywhere instantly" mechanic would be unrealistic and exploitable (no one would ever value drafting a true CB if you could just freely reslot your best raw athlete). To prevent that:

- Every player has a **Familiarity** rating (0-100) **per real position** within their group — starts high at their recruited/assigned position, starts low or at zero for any position they've never played.
- Low Familiarity **suppresses effective attribute values in play math** (not the underlying grades) — e.g., a Safety just moved to Linebacker plays like a worse linebacker than his raw numbers alone would suggest, until Familiarity rises.
- Familiarity **climbs through playing time at the new position**, and can be **accelerated by spending development points there** — modeling the real "he needs more reps at the new spot" reality of a position change.

This makes repositioning a real, weighty coaching decision (worth it for a great athlete stuck behind a starter, or to fix a poor fit) rather than a free respec.

### 5.10 Physical profile (height & weight)

Players have **realistic height and weight**, generated per their actual real-world position projection (e.g., a projected Linebacker generates in realistic LB size ranges, distinct from a projected Cornerback) even though both are in the shared Athlete group underneath.

Height/weight are **not just flavor** — they matter in two ways:
1. **Generation-time correlation**: body type realistically constrains which attributes a player tends to roll (a 340 lb lineman is very unlikely to also roll elite Speed).
2. **Situational play math**: size acts as a targeted modifier in the specific real-football moments where it matters most — contested catches/jump balls (favors height), goal-line/short-yardage power (favors mass), trench push (favors weight/strength combined) — rather than being woven into every formula.

**Weight can change over a career** through development investment (a strength-and-conditioning track, tied to development points) — modeling real freshman weight-room gains, a coach intentionally bulking up a lineman, or slimming down a player as part of a position conversion (tying directly into §5.9 Familiarity). Height is fixed.

### 5.11 Recruiting position tags

Despite the shared-attribute-group model underneath, recruits and players still carry a **projected/assigned position tag** (e.g., "CB," not just "Athlete") for recruiting-board organization, depth-chart purposes, and to drive realistic height/weight generation (§5.10). The group model is an internal data/flexibility architecture — it does not remove the concept of "what position is this guy" from the player-facing experience.

## 6. Recruiting

Recruiting is intentionally **the deepest system in the game** — the core loop, matching its role in real college football sims.

### 6.1 Scouting

- Recruits have **hidden attributes**; scouting accuracy is purely a function of **points invested**, not a separate "scout quality" stat layered on top.
- However, **coach skill (from the shared skill tree, §4.5) directly changes the cost**: a coach with a poor scouting skill might need to spend, e.g., 50 points to fully scout a recruit; a coach who has invested skill points into scouting might only need 20-30 points for the same recruit.
- Points are drawn from a shared weekly pool that covers both scouting *and* recruiting actions (see §6.2) — spending on one is a tradeoff against the other.

### 6.2 Weekly recruiting interaction

**Action-point driven**, week to week: each week you get a pool of recruiting points/hours to spend across your target list on discrete actions — phone calls, home visits, campus visits, scholarship offers. A recruit's interest shifts based on the actions taken (and by whom — the specific coach engaging matters, boosted by the Recruiting coach skill, §4.5).

**The size of that weekly pool is driven by School Prestige** (§4.6) — a higher-Prestige program simply has more recruiting points to work with each week than a lower-Prestige one, before any coach skill is even factored in.

### 6.3 Commitment decision

When a recruit decides, the outcome is weighted:

- **50%** — the recruit's own priorities/personality (distance from home, academics, NFL pipeline reputation, playing style preference — weighted differently per individual recruit, not a single formula for everyone)
- **25%** — fit & need (position competition, scheme fit, realistic odds of early playing time)
- **25%** — program prestige & recent success

### 6.4 Custom DB integration

Recruiting (and the whole simulation) must work identically whether running on the default fictional world or a user-imported custom DB (§8.3) — there is no special-cased "real teams" logic anywhere in the engine.

## 7. Player Development

- **Development itself is automatic** — practices are not manually managed play-by-play.
- Players earn **development points** based on their (and their position coach's) development-related skill, at two paces:
  - A **small, steady drip** during the season.
  - A **large bump** in the offseason.
- The **position coach decides how to spend** those development points on their players (which attributes to improve). This is the actively-managed part of player development — you don't run practices, but you do direct growth.
- Development points can also be spent on **Familiarity at a new position** (§5.9) and on a player's **weight** via a strength-and-conditioning track (§5.10) — covering position conversions and physical development, not just raw skill attributes.

## 8. World & Data

### 8.1 League scale & structure

- **Full FBS scale: ~130 fictional teams across 10+ conferences.**
- **Postseason mirrors the current real-world format**: conference championships feed a 12-team playoff bracket; non-playoff teams play in bowl games.

### 8.2 Team identity (default fictional world)

- **Hand-crafted, not procedurally generated** — a fixed roster of ~130 authored fictional schools, conferences, mascots, and colors, designed collaboratively rather than name-banked.
- **Visual style:** clean, modern sports-app aesthetic (cards, stats, ESPN-app-like), not a retro text-terminal look.
- **Art assets for the default fictional teams: initials/programmatic badges only** — colors + team initials, no illustrated logos or mascot art. This keeps 130 teams achievable without an art production pipeline.

### 8.3 Custom DB import (real teams, or any user-created world)

- Users can import a **custom database** to play with real teams/conferences/players (or any other fictional set they want) instead of the default world.
- **Format:** structured file import (JSON/CSV) against a schema we publish and document — not an in-app editor (at least not initially).
- **Custom DBs support full visual identity**: real logos, colors, and mascots (unlike the default fictional world's initials-only badges) — since these are user-supplied/user-responsibility assets, not something we need to produce ourselves.
- **Shareable:** players can export a custom DB file and share it with others (Discord, forums, etc.) to import — no in-game marketplace/hosting needed, just file portability.
- **Licensing note:** using real team names/logos in a custom DB is entirely the user's own responsibility/risk, not something the shipped game or its default content includes.

## 9. Coaching Staff & Program Economy (Head Coach layer)

As Head Coach, you manage additional systems that don't exist for lower roles:

- **Staff hiring:** a full AI coach market. Every coach in the league (including your own eventual assistants) is a real, simulated individual with their own skill/reputation/salary expectations. As HC you scout, negotiate, and hire within your staff budget — and rival schools can poach your assistants right back.
- **Program economy:** a meaningful budget/facilities layer — stadium/facility upgrades (which grant recruiting/development bonuses), staff salaries (better coaches cost more), and an NIL-collective-strength stat that factors into recruiting pitches alongside, and separately from, School Prestige (§4.6).

## 10. Records & History

- **Deep historical tracking**, not just your current season: career and season stat leaders, program records (most wins, longest streaks), award winners, a Hall of Fame for legendary players/coaches, and a trophy case of championships won.
- This is a deliberate "legacy" payoff for long dynasties, not an afterthought.

### 10.1 Architecture: derive from raw facts, don't store precomputed records

Based directly on reviewing the team's Dynasty Tracker project (§14): store only **atomic facts** — `Game`, `Season`, `PlayerGameStat` rows — and compute every leaderboard, streak, split, and title count as a **pure function over those facts**, scoped flexibly (single season / career / whole program across coaches / all-time league-wide). Nothing about "records" should be a value written and maintained in place; it should always be re-derivable from the raw history, the same way §5 requires play outcomes to be computed rather than looked up. This avoids an entire class of "the record didn't update / disagreed with the underlying games" bugs.

### 10.2 Individual player statistics (a deliberate gap Dynasty Tracker left, closed here)

Dynasty Tracker — reviewed as a direct reference point — turned out **not to track individual player statistics at all**: no passing/rushing/receiving stat lines, only team-level aggregates and awards. Headset Dynasty needs real `Player` and `PlayerSeasonStat`/`PlayerGameStat` entities with actual statistical fields, because true career/single-season statistical leaderboards (not just win-loss records and awards) are part of the ask in this section. This is the main way Headset Dynasty's records system needs to go further than the reference project, not just copy it.

### 10.3 Program history spans many coaches, not just yours

Because Headset Dynasty's coaching-carousel design (§4.3) has coaches — yours and every AI coach — moving between schools over a career, program-level all-time records must aggregate across **every coach who ever held the job**, not just the current user's tenure. (Dynasty Tracker didn't need this — it only ever tracks one user's own coaching career at one identity.) A program's record book is a first-class thing independent of who's currently coaching it; a coach's personal career record (their own tenure record across every school they've coached at) is a separate, second view over the same underlying game history.

### 10.4 Awards as a normalized table

Track awards/honors (All-American, All-Conference, Heisman-equivalent, draft picks) as a proper `Award`/`Honor` table (type, year, player, tier) rather than embedding them inside season records — needed once cross-program and league-wide leaderboards matter (e.g. "most All-Americans produced, all-time, across all 130 programs"), which is squarely in scope here given the AI-coach-driven living league (§4.3, §10.3).

### 10.5 UI patterns to reuse

Several presentation patterns from Dynasty Tracker's UI are worth carrying forward close to as-is:
- **Trophy Case**: an icon-tile achievement grid (titles, playoff trips, All-Americans, Heismans, draft picks), visually dimmed when empty.
- **Rivalry ledger**: per-opponent head-to-head record, current streak, and last-played year — a natural fit for the rivalry-game mechanic already in §5.4.
- **Playoff bracket view**: modeled as flat fields per round (not a nested tree) — directly matches the 12-team CFP structure already decided in §8.1.
- **Season-end shareable text recap** (nice-to-have, not core scope): an auto-generated plain-text narrative of the season (record, honors, recruiting class, career-to-date) formatted for easy copy/paste elsewhere.

**Full research reports:** `docs/research/cricket-manager-lessons.md` and `docs/research/dynasty-tracker-notes.md`.

## 11. Accounts, Saves & Social

### 11.1 Session pacing

Target: a typical week (recruiting actions + game sim + reports) should be playable in **2-5 minutes** — quick and bite-sized, suited to mobile play during a commute/break.

### 11.2 Saves

- **Cloud-synced from day one** via Firebase — saves are not local-only, even though the frontend starts as a static site.
- **Multiple save slots per account** — a player can run several independent dynasties at once (e.g., HC at School A in one save, position coach at School B in another).

### 11.3 Social features

**Light social only, for now**: leaderboards/comparisons (e.g., reputation, championships, career wins vs. friends or global players). No direct player interaction, no shared/multiplayer universe (that idea was considered and explicitly deferred, not ruled out forever).

### 11.4 Onboarding

**Light touch** — contextual tooltips/help available throughout the UI, but **no mandatory guided tutorial**. Trusts players to explore; the free tier's "jump straight into any role" design already serves as a low-friction on-ramp.

### 11.5 Difficulty

**No difficulty/customization sliders.** One well-tuned default experience for everyone (no injury-frequency toggles, AI-aggressiveness sliders, etc.) — keeps balance/QA surface area contained.

## 12. Branding

**"Headset Dynasty" is the real, final title** — used in the UI, App Store listing, and all branding, not a placeholder.

## 13. Open Questions (TBD)

These are known-open items, not forgotten — to be resolved in future planning sessions:

- [ ] Price point for the one-time purchase (deliberately deferred to closer to launch).
- [ ] **"Active attributes" UI presentation** (§5.8) needs an actual mockup/prototype before being treated as validated — user explicitly wants to see it in practice, not just approve it in the abstract.
- [ ] Exact contents of the shared coach skill tree beyond scouting (recruiting pitch, development, play-calling/scheme mastery, program management, etc. were floated as categories but not finalized).
- [ ] Exact numeric shape of the situational size modifiers (§5.10) — e.g. how much a height advantage should shift a contested-catch probability — needs real formula work, not just the qualitative direction agreed so far.
- [ ] Exactly how weekly recruiting actions/interest (§6.2) mathematically feed into the 50/25/25 commitment-decision weights (§6.3) — e.g. whether accumulated interest is a threshold to make a recruit's shortlist at all, or a continuously blended factor. Directionally settled, precise formula still open.
- [ ] Coach skill tree (§4.5) is intentionally incomplete — Scouting, Recruiting, Player Development, and Tactics are locked in; more categories may be added later.
- [ ] Detailed screen-by-screen UI/UX design (only the high-level visual style — "clean modern sports app" — has been set).
- [ ] Detailed data schema definitions (tables/interfaces) for teams, players, coaches, recruits, the custom-DB import format, and the records/stats entities from §10 (`Player`, `PlayerSeasonStat`/`PlayerGameStat`, `Award`).

## 14. Research Notes

- **Cricket Manager** (`jbaxmeyer-personal/cricket-manager`) — reviewed. Godot/GDScript desktop project. Its match-resolution engine was cleanly isolated as pure functions (validates the framework-agnostic-core goal), but everything else (recruiting-equivalent, development, AI, economy) was fused into one giant mutable singleton the UI reached into directly hundreds of times — the direct cause of several documented production bugs and the likely root of "awesome but difficult to build." Also confirmed the value of separating sim attributes from display ratings, and the risk of hardcoding roster data instead of loading it externally. Full report: `docs/research/cricket-manager-lessons.md`.
- **Dynasty Tracker** (`jbaxmeyer-personal/dynasty-tracker`) — reviewed. A React + TypeScript + Firebase web app (a close real-world precedent for Headset Dynasty's own planned stack) built for tracking CFB27 dynasties. Strong "derive records from raw facts" architecture and reusable UI patterns (Trophy Case, rivalry ledger, flat-field bracket view) — but notably has **no individual player statistics**, only team/program-level records and awards, which is the main gap Headset Dynasty's records system needs to close rather than replicate. Full report: `docs/research/dynasty-tracker-notes.md`.

## 15. Recommended Build Order (proposed, not yet started)

1. **Data layer & schema** — team/player/coach/recruit schema, including the custom-DB import format, since everything else depends on it.
2. **Core simulation engine, headless** — prove deterministic play resolution works (and reads well as text output) via a script that can simulate a full game/season with no UI. Highest-risk, most novel piece — validate it before investing in screens on top of it.
   - **Build a statistical calibration + regression-testing harness alongside it**, not after — directly adopted from what worked in Cricket Manager (§14): a headless runner that simulates many games/seasons through the real engine code (never a mock or a hand-ported copy of the formulas) and checks output against real college-football statistical benchmarks, plus long-run stability across many simulated seasons (watching for rating inflation/collapse).
3. **Vertical slice** — one role, one team, one season, real UI, end-to-end — to validate the full loop feels good before broadening to the entire 130-team world and all three coaching roles.

---

*This document will keep growing as we continue planning. Nothing here should be treated as locked until we've both agreed it's ready to build against.*
