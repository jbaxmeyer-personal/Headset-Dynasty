# Headset Dynasty — Game Design Document

**Status:** Living draft. This document reflects everything decided in planning conversations so far. Sections marked **TBD** are open and will be filled in as we keep talking. Nothing here is final until we've validated it feels good to build and play — but this is our source of truth for what we're building.

**Last updated:** 2026-09-09

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

## 3. Business Model

- **Premium, one-time purchase.** No ads, no IAP, no ongoing monetization systems to build.
- **Free tier exists as a trial/hook**, not a separate monetization strategy:
  - Free: choose **any school** and **any coaching role** (including Head Coach) from day one — no need to climb the ladder to sample the top job.
  - Free is capped at a small number of total seasons (**2 or 3 — TBD**, needs to be picked and playtested).
  - Paid: unlocks unlimited seasons.
- Open question: exact price point. **TBD.**

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

A **separate stat from reputation**: coach skill points **only accumulate, never decrease**, earned from the same category of positive events (wins, Power 4 wins, signing recruits, players drafted). Spent on a **broad, shared coach skill tree** (not siloed per role) — covering things like:

- Scouting efficiency (see §6.1)
- Recruiting pitch/persuasion
- Player development effectiveness
- Other coaching competencies (**exact tree contents TBD — to be designed together**)

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

### 5.7 Positions & attributes

**Full realistic position breakdown** (all ~20 real CFB positions: QB, RB, WR, TE, LT/LG/C/RG/RT, EDGE/DT, LB, CB/S, K/P/LS, etc.), each with a meaningful set of position-relevant attributes.

**The exact attribute list per position, and precisely how those attributes feed into the play-resolution math, is intentionally left open — this is being co-designed with the user as its own dedicated design pass, not dictated.** See §13.

## 6. Recruiting

Recruiting is intentionally **the deepest system in the game** — the core loop, matching its role in real college football sims.

### 6.1 Scouting

- Recruits have **hidden attributes**; scouting accuracy is purely a function of **points invested**, not a separate "scout quality" stat layered on top.
- However, **coach skill (from the shared skill tree, §4.5) directly changes the cost**: a coach with a poor scouting skill might need to spend, e.g., 50 points to fully scout a recruit; a coach who has invested skill points into scouting might only need 20-30 points for the same recruit.
- Points are drawn from a shared weekly pool that covers both scouting *and* recruiting actions (see §6.2) — spending on one is a tradeoff against the other.

### 6.2 Weekly recruiting interaction

**Action-point driven**, week to week: each week you get a pool of recruiting points/hours to spend across your target list on discrete actions — phone calls, home visits, campus visits, scholarship offers. A recruit's interest shifts based on the actions taken (and by whom — the specific coach engaging matters).

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
- **Program economy:** a meaningful budget/facilities layer — stadium/facility upgrades (which grant recruiting/development bonuses), staff salaries (better coaches cost more), and an NIL-collective-strength stat that factors into recruiting pitches.

## 10. Records & History

- **Deep historical tracking**, not just your current season: career and season stat leaders, program records (most wins, longest streaks), award winners, a Hall of Fame for legendary players/coaches, and a trophy case of championships won.
- This is a deliberate "legacy" payoff for long dynasties, not an afterthought.
- **Reference point:** the team's existing Dynasty Tracker project (built for tracking CFB27 dynasties) is being reviewed for concrete data-model and presentation ideas — findings will be added to this document once that review completes (see §14).

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

- [ ] Exact free-tier season cap: 2 or 3 seasons?
- [ ] Price point for the one-time purchase.
- [ ] **Full position/attribute list and the play-outcome formula** — explicitly flagged by the user as a collaborative design pass, not something to be decided unilaterally.
- [ ] Exact contents of the shared coach skill tree beyond scouting (recruiting pitch, development, play-calling/scheme mastery, program management, etc. were floated as categories but not finalized).
- [ ] Findings from the Cricket Manager architecture review (in progress — see §14).
- [ ] Findings from the Dynasty Tracker records/stats review (in progress — see §14).
- [ ] Detailed screen-by-screen UI/UX design (only the high-level visual style — "clean modern sports app" — has been set).
- [ ] Detailed data schema definitions (tables/interfaces) for teams, players, coaches, recruits, and the custom-DB import format.

## 14. Research Notes

- **Cricket Manager** (`jbaxmeyer-personal/cricket-manager`) — cloned for review; architecture/lessons-learned findings pending, will be appended here.
- **Dynasty Tracker** (`jbaxmeyer-personal/dynasty-tracker`) — cloned for review; records/stats-system findings pending, will be appended here.

## 15. Recommended Build Order (proposed, not yet started)

1. **Data layer & schema** — team/player/coach/recruit schema, including the custom-DB import format, since everything else depends on it.
2. **Core simulation engine, headless** — prove deterministic play resolution works (and reads well as text output) via a script that can simulate a full game/season with no UI. Highest-risk, most novel piece — validate it before investing in screens on top of it.
3. **Vertical slice** — one role, one team, one season, real UI, end-to-end — to validate the full loop feels good before broadening to the entire 130-team world and all three coaching roles.

---

*This document will keep growing as we continue planning. Nothing here should be treated as locked until we've both agreed it's ready to build against.*
