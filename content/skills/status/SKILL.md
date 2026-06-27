---
name: status
description: The club-ops dashboard — a read-only roll-up across every entity in state/. Surfaces member counts (players/parents) by age_group × gender, staff by role, teams by age_group × gender × season, roster fill (set/forming/locked), upcoming tryouts, and (phase 2, graceful) upcoming schedules + open registrations. Iterates state/ without mutating it; the headline output is the GAP REPORT — age_group/gender cells with no team, unrostered players, teams with no coach, tryouts with no evaluators. The director's at-a-glance club view. Triggered when the user wants the big picture — e.g. "/status", "club status", "where do we stand", "show me the dashboard", "what's the state of the club", "what gaps do we have", "club health check".
---

# /status — The club-operations dashboard

The director's single-screen view of the whole club. `/status` walks
every record under `state/` — members, staff, teams, rosters, tryouts —
and rolls them up into counts, breakdowns, and (the load-bearing part)
a **gap report**: the places where the club's operational data has a
hole that a human needs to close.

This is the orchestrator doing its actual job. The role-domains
(`domain.soccer.{player,coach,parent,director,board,team,league}` + the
live `domain.soccer.goalkeeping`) give each person their toolkit;
`/status` answers the question those toolkits can't — *how is the club,
as an organization, doing right now?* Members, units, and events in one
frame.

Per CLAUDE.md ethos: **calibrated, no narratives.** Report the counts
that are true, flag the gaps that are real, and don't dress a thin club
up as a healthy one. An age group with zero teams is a fact, not a
failure to soften. The signal is the gap.

## Behavior contract

- **Read-only. Always.** This skill opens and parses records under
  `state/`; it never creates, edits, moves, or deletes one. Every
  number it shows is derived live from the files on disk at read time.
  Mutations are the job of `/member`, `/staff`, `/team`, `/roster`,
  `/tryout` — `/status` only reports. If reading the club surfaces a fix
  that's needed, name it in the gap report and point the user at the
  skill that owns the change; do not make the change here.
- **Iterate the source of truth, don't cache it.** The dashboard is
  recomputed from `state/<entity>/*.md` on every run. There is no
  stored "status" record and you must never write one. If a record is
  malformed or a field is missing, surface that as a data-integrity gap
  (see Operation 7) rather than guessing or silently dropping it.
- **The gap report is the headline, not a footnote.** Counts orient;
  gaps are the actionable signal. Lead the closing summary with what's
  missing or inconsistent, then show the breakdowns underneath. A
  director scanning `/status` should see the holes first.
- **Use the canonical field sets verbatim.** Read the exact fields
  defined in `.claude/club-ops-rules.md` and the per-entity templates.
  Members carry `type` (player|parent), `age_group`, `gender`, `status`,
  `teams[]`, `registrations[]`; Staff carry `role`, `teams[]`,
  `certifications[]`, `status`; Teams carry `age_group`, `gender`,
  `season`, `coaches[]`, `roster_id`, `home_complex`, `status`; Rosters
  carry `team`, `players[]`, `from_tryout`, `max_size`, `status`;
  Tryouts carry `age_group`, `gender`, `date`, `location`,
  `evaluators[]`, `candidates[]`, `status`. Cross-tabs MUST use these
  field names — never invent a parallel taxonomy.
- **Dimensions come from `state/club.md`.** The valid `age_groups`,
  `genders`, and `registration_systems` are defined there. The "every
  cell" view (age_group × gender) is computed against the club's
  declared dimensions — that's how you detect an age_group/gender
  combination with *no* team, not just under-filled ones. If
  `state/club.md` is absent or empty, fall back to the dimension values
  actually present in the records and flag that the dimension catalog
  is undefined.
- **Phase-2 entities degrade gracefully.** Schedules, fields, complexes,
  tournaments, leagues, and registrations are defined in the rules and
  have `state/` directories, but their skills are not authored in this
  build. Where `/status` touches them — upcoming schedules, a team's
  `home_complex`, open registrations across `registration_systems` — it
  reports what's present if the directory has records and prints a
  one-line "(phase 2 — not yet tracked)" stub if it's empty or absent.
  Never error on a missing phase-2 directory.
- **Honor the orchestrator/domain boundary.** `/status` reports
  organizational facts (who's rostered, which teams lack a coach). It
  does NOT report on coaching quality, player development, or training
  load — those live in the role-domains and are out of scope. The
  domain references are soft: if a domain isn't installed, the roll-up
  still runs.

## Operations

A `/status` run executes these in order. Each is a read pass over one or
more `state/` directories; nothing writes back.

### 1. Resolve the dimensions

Read `state/club.md` and load the declared `age_groups` (e.g. U6..U19,
Adult), `genders` (boys|girls|coed), and `registration_systems` (the
club's own system + any state/regional/national orgs). These define the
grid every cross-tab is computed against. Hold them for Operations 3, 4,
and 6. If `state/club.md` is missing, derive the dimensions from the
values present in records and note the catalog is undefined.

### 2. Member roll-up (players + parents)

Walk `state/member/*.md`. For each record read `type`, `age_group`,
`gender`, `status`, `teams[]`, `registrations[]`. Produce:

- Total members, split by `type` (player vs parent).
- **Players by `age_group` × `gender`** — the core cross-tab. Count only
  `status: active` for the live picture; note `prospect` and `inactive`
  counts separately so the director sees the pipeline and the attrition.
- Parents linked vs unlinked — parents whose `players[]` is empty are a
  data-integrity gap (Operation 7), not a count.

### 3. Staff roll-up (by role)

Walk `state/staff/*.md`. For each read `role` (coach|director|admin|
board), `teams[]`, `certifications[]`, `status`. Produce a count by
`role` for `status: active` staff. Hold the set of coaches and which
`teams[]` they're assigned to — Operation 5 needs it to find teams with
no coach and coaches who are over-allocated.

### 4. Team roll-up (age_group × gender × season)

Walk `state/team/*.md`. For each read `age_group`, `gender`, `season`,
`coaches[]`, `roster_id`, `home_complex`, `status`. Produce:

- Teams by `season`, then within a season the **age_group × gender**
  grid — one cell per declared dimension combination.
- For each team: coach count (`len(coaches[])`), roster state (resolved
  in Operation 5), and `home_complex` (phase-2 — print the value if set,
  "(phase 2)" if not).
- **The empty-cell scan:** for the active season, walk every declared
  `age_group` × `gender` combination from Operation 1 and flag the cells
  with zero teams. This is a primary gap.

### 5. Roster fill + linkage integrity

For each team's `roster_id`, read `state/roster/<roster_id>.md`:
`team`, `players[]`, `from_tryout`, `max_size`, `status`
(forming|set|locked). Produce:

- Roster state per team — `forming` vs `set` vs `locked`, and fill ratio
  `len(players[]) / max_size`.
- **Unrostered active players:** cross-reference members whose `status`
  is `active` and `type` is `player` against the union of all rosters'
  `players[]`. A player on no roster is a primary gap.
- **Linkage integrity (read-only check):** a roster's `players[]` should
  match each member's `teams[]`, and the team's `roster_id` should point
  back. Report any mismatch as an integrity gap — `/roster` is the skill
  that repairs it; `/status` only flags it.

### 6. Upcoming tryouts

Walk `state/tryout/*.md`: `age_group`, `gender`, `date`, `location`,
`evaluators[]`, `candidates[]`, `status` (scheduled|complete). Produce:

- Upcoming tryouts (`status: scheduled`, `date` in the future), sorted
  by `date`, with `age_group` × `gender`, location, evaluator count, and
  candidate count.
- **Tryout gaps:** a scheduled tryout with empty `evaluators[]`, or with
  no `candidates[]`, or whose `location` references a phase-2
  complex/field that isn't tracked yet. Each is actionable.
- Recently `complete` tryouts whose outcomes haven't produced a roster
  yet (no roster has `from_tryout` pointing at them) — a follow-up gap
  that routes to `/roster`.

### 7. Data-integrity sweep

While iterating, collect anything malformed: a record missing a required
field, a `teams[]`/`roster_id` reference that points at a nonexistent
record, a `gender`/`age_group` value not in the declared dimensions, a
parent with no linked players. These are reported as their own gap class
— the club's data has a hole regardless of headcount.

### 8. Phase-2 surfaces (graceful)

Best-effort, never-error:

- **Upcoming schedules** — if `state/schedule/` has records, show the
  next practice/game entries; otherwise "(phase 2 — schedules not yet
  tracked)".
- **Open registrations** — tally member/team `registrations[]` against
  each `registration_system` from Operation 1; show coverage if data
  exists, else "(phase 2 — registration tracking pending)".
- **Complexes/fields, tournaments, leagues** — one-line presence note
  each; full roll-up is a phase-2 skill.

### 9. Render the dashboard

Assemble Operations 1–8 into the single output below. **Gap report
first**, then the breakdowns. Cross-reference the drill-down skill for
every gap class so the director's next click is obvious.

## Output structure

The whole point is one screen. Gaps lead; counts support. Cite the
record path (`state/<entity>/<id>.md`) for anything a gap points at, so
the director can jump straight to the drill-down skill.

```markdown
# Club Status — <club name from state/club.md>
_Read-only roll-up · <N> members · <N> staff · <N> teams · <season>_

## ⚠ Gaps (action needed)
- **Empty cells:** <age_group> <gender> has no team this season → /team
- **Unrostered players:** <N> active players on no roster → /roster
  - <player name> — state/member/<id>.md
- **Teams without a coach:** <team name> (coaches[] empty) → /staff, /team
- **Tryouts without evaluators:** <age_group> <gender> on <date> → /tryout
- **Completed tryout, no roster:** <tryout id> → /roster
- **Data integrity:** <member id> parent has no linked players → /member
_(If no gaps: "No open gaps — every declared cell has a team, every
active player is rostered, every team has a coach.")_

## Members
| age_group | boys | girls | coed | total |
|---|---|---|---|---|
| U8 | 14 | — | — | 14 |
| ... | | | | |
Active players: <N> · prospects: <N> · inactive: <N>
Parents: <N> (<N> linked, <N> unlinked)

## Staff
coach <N> · director <N> · admin <N> · board <N>   (active)

## Teams — <season>
| age_group | gender | team | coaches | roster | home_complex |
|---|---|---|---|---|---|
| U10 | girls | <name> | 1 | set 12/14 | <complex / phase 2> |
| ... | | | | | |
Empty cells (no team): <list>

## Upcoming tryouts
| date | age_group | gender | location | evaluators | candidates |
|---|---|---|---|---|---|
| <date> | U12 | boys | <loc / phase 2> | 2 | 9 |

## Phase 2 (graceful)
- Schedules: (phase 2 — not yet tracked)
- Registrations: <coverage if present, else phase-2 stub>
- Complexes / fields / tournaments / leagues: <presence note>
```

Adapt the columns to the dimensions actually declared in
`state/club.md`; drop a `gender` column the club doesn't use; collapse a
section to a one-liner when its directory is empty.

## Style rules

- **Gaps first, every time.** Even when the club is healthy, lead with
  the gap section so the director's eye lands on "what needs me" before
  "what's fine." A clean run says so explicitly.
- **Cite records as `state/<entity>/<id>.md`.** Every gap that points at
  a specific entity gets a path so the next click is one hop away.
- **Cross-reference the owning skill by `/name`.** Each gap names the
  drill-down that fixes it: empty cells → `/team`, unrostered players →
  `/roster`, missing coach → `/staff` + `/team`, tryout gaps →
  `/tryout`, member-data gaps → `/member`.
- **Counts use the live `status` filter.** Headline numbers count
  `active` records; show `prospect`/`inactive`/`forming` as secondary so
  the director sees pipeline and attrition without them inflating the
  "now" picture.
- **Empty is a value, not an omission.** A zero-team cell, an
  unrostered player, an evaluator-less tryout — print them. Silence
  reads as "fine," and the whole skill exists to prevent that.
- **No sign-offs.** End on the last section of the dashboard.

## What you must NOT do

- **Don't write anything to `state/`.** Not a status cache, not a
  derived index, not a "last run" stamp. `/status` is read-only; a
  single write is a contract violation. All mutations belong to the
  drill-down skills.
- **Don't fix gaps inline.** Spotting an unrostered player is the job;
  rostering them is `/roster`'s job. Name the gap, route the user, stop.
- **Don't invent fields or dimensions.** Use the canonical field sets
  and the dimensions from `state/club.md`. If a record uses an undeclared
  `age_group` or `gender`, that's a data-integrity gap to report — not a
  new column to silently add.
- **Don't error on phase-2 directories.** Empty or absent
  `state/{schedule,field,complex,tournament,league,registration}/`
  degrades to a one-line stub. The foundation roll-up must always
  complete.
- **Don't report on coaching, training, or player development.** Those
  are role-domain concerns. `/status` is organizational, not athletic.
- **Don't soften the picture.** A thin club reads as thin. No narrative
  spin in either direction.

## When NOT to use this skill

- **You want to change a record** (add a member, assign a coach, build a
  roster) → use `/member`, `/staff`, `/team`, `/roster`, or `/tryout`.
  `/status` only reads.
- **You want the detail on one entity** (a single team's full record, a
  member's history) → open that entity's drill-down skill; `/status` is
  the aggregate, not the deep dive.
- **You want practice/game scheduling, field assignment, tournament or
  league standings, or registration submission** → those are phase-2
  skills. `/status` shows their presence but doesn't operate them.
- **You want coaching/training/development insight** → that's the
  role-domain's job, not the orchestrator's.

## What "done" looks like for a /status session

The director has a single rendered dashboard: total members (players +
parents) cross-tabbed by `age_group` × `gender`, staff by `role`, teams
by `age_group` × `gender` × `season` with coach and roster state,
upcoming tryouts, and graceful phase-2 presence notes — led by a gap
report that names every empty cell, unrostered player, coachless team,
under-staffed tryout, and data-integrity break, each cited to its
`state/` record and routed to the drill-down skill that closes it.
Nothing in `state/` changed; the club just got seen clearly.
