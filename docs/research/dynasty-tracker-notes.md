# Research Notes: Dynasty Tracker — Patterns for Headset Dynasty's Records System

Full findings from an automated review of `jbaxmeyer-personal/dynasty-tracker` (a companion tool the team built for tracking dynasties while playing EA's College Football 27), conducted specifically to inform Headset Dynasty's records/history system. See `docs/GAME_DESIGN.md` §10 for how these were folded into actual decisions.

## 1. Data Model

Six flat tables per dynasty: `Season` (year, school, prestige, ratings, NIL spend, coordinators, support staff, honors arrays, notes), `Game` (week — including a union type modeling the 12-team CFP: `"CC"|"CFP1"|"CFPQF"|"CFPSF"|"Natty"|"Bowl"` — opponent, score, rank context), `Recruit` (incoming only, no "player left" row type, 44 position archetypes, star rating, dev trait), `SeasonTeamStats` (team-level box-score aggregates — defined but unused/empty even in their own sample data), `SchoolPrestige` (rival program tracking over time, also unused in sample data), and `NationalLandscape` (a full independent per-year national snapshot: playoff bracket, conference champs, Heisman, final Top 25).

**Critical gap: there is no individual player-career entity or per-player stat line.** Players only exist as line items inside honors arrays (`all_americans`, `all_conference`, `draft_picks`) and the `Recruit` table. No passing/rushing/receiving statistics are tracked per player, ever — "records" in this tool means win-loss records, streaks, and awards, not statistical bests.

## 2. What "Records" and "Stats" Actually Mean Here

Everything is computed at render time from raw `games[]`/`seasons[]` (`computedStats.ts`) — nothing is stored precomputed. Concretely: season/career/home-away/conference/bowl/playoff/ranked-opponent records, TV-tier splits, a per-opponent head-to-head ledger with current streak and last-played year, best win (highest-ranked opponent beaten), conference/national championship counts, playoff appearances, and awards (All-American/All-Conference tiers, Heisman — matched by cross-referencing the national awards table against the user's own program/year, draft picks by round).

The "Trophy Case" (Career page) is an icon-tile achievement grid (🏆 titles, 🥇 conference titles, 🎟️ playoff trips, ⭐ All-Americans, 🏈 Heismans, 🎯 draft picks), dimmed when zero. There is no explicit Hall of Fame page — the Trophy Case plus honor lists serve that role.

## 3. Tech Stack & Structure

**Notably close to Headset Dynasty's own planned stack**: React 19 + TypeScript + Vite, React Router (HashRouter specifically because GitHub Pages has no server rewrite rules), no UI component library (plain mobile-first HTML forms, deliberately, for bundle size and easy field extension), installable PWA. Backend is **Firebase Firestore + Firebase Auth** (Google sign-in or email/password) per-user, gated behind a login screen — despite the README describing an earlier "GitHub repo as datastore via direct API commits" design that the current code has evidently moved on from. Team logos: real ESPN CDN logos where a mapping exists, falling back to a deterministic colored-initials badge — closely mirrors Headset Dynasty's own default-fictional-teams-get-initials, custom-DB-gets-real-logos decision.

This is a working, shipped validation that "React + TypeScript static site + Firebase, hosted where GitHub Pages can serve it" is a real, viable path for exactly this kind of project.

## 4. UI/UX Patterns for Presenting History

- **Trophy Case grid**: icon + count + label tiles, dimmed when empty — scannable, screenshot-shareable achievement pattern.
- **Coach record card**: dense label:value stat grid rather than a table, good for compact mobile display.
- **Custom lightweight trend charts** (no charting library): win% by season, an inverted-axis "climb" line chart for final ranking over time (plots `26 - rank` so "up" always reads as "better," then relabels the axis back to real ranks), ratings over time, prestige over time, dynasty points earned vs. spent.
- **Record-by-opponent list**: team logo + record + colored win/loss streak pill + last-played year — a rivalry ledger pattern.
- **Season card grid**: team-color-gradient cards, one-line record/prestige/rank summary, browsable as a timeline.
- **Playoff bracket visualization**: built from flat bracket fields (not a nested tree) rendered as round-columns (First Round → QF → SF → Championship) with seed numbers and winner highlighting — simple to author, still a proper bracket UI.
- **"Generate shareable text recap" feature**: a season-end button produces a full plain-text narrative (record, ratings, schedule, honors, recruiting class, career-to-date) formatted for pasting elsewhere — a low-effort, high-perceived-value feature worth considering.

## 5. Directly Reusable Ideas

- The **derive-everything-at-render** architecture (store only atomic facts, compute every record/streak/split as a pure function over them) is exactly the right shape for a records engine that must stay correct across seasons, coaches, and program-level scope.
- The **`week` union type modeling the 12-team CFP** plus small label/sort helpers is a clean, minimal way to encode Headset Dynasty's own already-decided 12-team playoff structure without a nested bracket tree.
- The **rivalry ledger** (head-to-head record + streak + last-played) is a strong, direct fit for Headset Dynasty's rivalry-games mechanic (already planned as a situational sim factor).
- The **Heisman-matching pattern** (cross-reference a per-user record against an independent national-awards table by year/school) is a good template for "did my program produce this national award winner."

## Recommendations Actually Adopted for Headset Dynasty

1. **Adopt the derive-from-raw-facts architecture** for the records system: store atomic `Game`/`Season`/`PlayerGameStat` rows only, compute every leaderboard/streak/split/title-count via pure functions, scoped flexibly (season / career / program / all-time-across-coaches).
2. **Go further than Dynasty Tracker on individual stats** — add real `Player` and `PlayerSeasonStat`/`PlayerGameStat` entities with actual statistical fields (passing yards, rushing TDs, etc.), since Dynasty Tracker conspicuously has no per-player stat line and Headset Dynasty explicitly wants true career/season statistical leaderboards, not just awards and win-loss records.
3. **Reuse the Trophy Case, rivalry-ledger, and flat-field bracket-view UI patterns** close to as-is.
4. **Extend the single-coach-career aggregate pattern into a full multi-coach program history**: Dynasty Tracker only ever tracks one user's own coaching tenure; Headset Dynasty's coaching-carousel design (coaches moving between schools, AI coaches with their own tenures) needs all-time program records that span many coaches, not just one.
5. **Normalize awards as a proper `Award`/`Honor` table** (type, year, player, tier) rather than embedded per-season arrays, once cross-program/global leaderboards are needed (e.g. "most All-Americans produced" across the whole 130-team world) — Dynasty Tracker's embedded-array approach is fine for its single-dynasty scope but won't scale to Headset Dynasty's league-wide comparisons.
6. **Borrow the shareable-text-recap idea** as a season-end or Hall-of-Fame-induction narrative generator — flagged as a nice-to-have, not core scope.
