---
name: team
description: Create and manage a club's TEAMS — the fielded competitive units. A team is an age_group + gender + season + assigned coach(es) + a roster + a home_complex + the leagues/tournaments it competes in. Use this skill to create a team (set age_group, gender, season, assign coach(es) from staff), assign or replace coaches, set a home_complex (phase 2), enter a league or tournament (phase 2 — record the intent), update facts, list/filter teams (by age_group, gender, season), and archive a team at season's end. Writes state/teams/<id>.md and keeps coach assignments consistent with each coach's staff record. Triggered when the user wants to stand up or operate a team — e.g. "/team create U12 boys fall 2026", "make the U14 girls team", "assign coach Maria to the U10 team", "replace the U16 boys coach", "set the U12 home complex", "enter the U14 girls in the Fall Classic", "list all U10 teams", "archive last season's teams".
---

# /team — create and manage club teams

The fielded competitive unit of the club. A **team** binds the club's
cross-cutting taxonomy (an `age_group` + a `gender`) to a `season`,
puts one or more **coaches** (drawn from `staff`) in charge of it,
points at a **roster** of players, calls a **home_complex** home, and
enters the **leagues** and **tournaments** it competes in. This skill
owns the team record end to end: it creates teams, moves coaches on and
off them, records where they play and what they compete in, and retires
them at season's end — and at every step it keeps the team's facts and
the affected staff records pointing at each other.

The team record is club **operational data**: it lives in `state/teams/`
in the umbrella repo, the orchestrator owns it, and every other
club-ops skill — `/roster`, `/tryout`, `/staff`, `/schedule` (phase 2),
`/league` (phase 2) — reads it. A team that is not in `state/teams/`
does not exist to the club. `/team` is how it comes to exist and how it
changes.

Per the load-bearing rule: **the orchestrator manages the unit; it does
not do the unit's work.** `/team` decides that the U12 boys team exists,
that Maria coaches it, and that it plays in the Fall League. It does
**not** coach the team, build the practice plan, or train a single
player — that is the coach's own toolkit (`rasa.domain.soccer.coach`)
and the team's own toolkit (`rasa.domain.soccer.team`). Those domain
references are **soft**: a team is fully managed here whether or not any
domain Element is present. Only `rasa.domain.soccer.goalkeeping` is live
so far; everything else degrades gracefully to "absent, no behavior
lost at the orchestrator level."

## Behavior contract

- **`state/teams/<id>.md` is the source of truth for a team.** A team
  exists if and only if it has a record there. This skill creates,
  updates, and archives that record from `state/teams/_template.md`.
  The `id` is the primary key — stable, lowercase, no spaces (e.g.
  `u12-boys-fall-2026`). It never changes once assigned; facts change,
  the id does not.

- **The canonical Team field set is fixed. Use it exactly.** Every team
  record carries these fields and only these for its machine zone —
  matching the names every other skill and the `_template.md` use:

  | Field | What it is | Example |
  |---|---|---|
  | **id** | Primary key = record filename | `u12-boys-fall-2026` |
  | **name** | Human display name | "U12 Boys — Fall 2026" |
  | **age_group** | One of the club's `age_groups` (state/club.md) | `U12` |
  | **gender** | One of the club's `genders` | `boys` |
  | **season** | The season this team is fielded for | `Fall 2026` |
  | **coaches[]** | `staff` ids of the assigned coach(es) | `[maria-lopez, dev-shah]` |
  | **roster_id** | The `roster` record this team draws from | `roster-u12-boys-fall-2026` |
  | **home_complex** | The `complex` this team plays/practices at (phase 2) | `north-fields` |
  | **leagues[]** | `league` ids the team competes in (phase 2) | `[city-fall-league]` |
  | **tournaments[]** | `tournament` ids the team enters (phase 2) | `[fall-classic-2026]` |
  | **status** | Lifecycle state | `active` \| `archived` |

  `age_group` and `gender` MUST be drawn from the dimensions defined in
  `state/club.md` (`age_groups`, `genders`). Do not invent a U13 if the
  club's ladder is U12/U14 — ask, or add the dimension to `club.md`
  first. `gender` is one of `boys`, `girls`, `coed`.

- **Coach assignment is two-sided. Keep both sides consistent.** A
  coach is `staff` with `role: coach`. When a coach is assigned to a
  team, the assignment lands in **two** surfaces: the team's
  `coaches[]` gains the staff id, AND that staff member's `teams[]`
  gains this team's id. When a coach is removed, both sides drop the
  reference. **Both land or neither does** — a team that lists a coach
  whose staff record doesn't list the team (or the reverse) is drift,
  and this skill exists in part to prevent it. On entry, if you detect
  a one-sided assignment, reconcile it rather than erroring.

- **Only real staff coach a team.** Every id in `coaches[]` MUST resolve
  to an existing `state/staff/<id>.md` record with `role: coach`. Do
  not put a name in `coaches[]` — put the staff id. If the person isn't
  in staff yet, that's a `/staff` operation first; do not fabricate a
  staff record from inside `/team`. A team may have zero coaches
  transiently (just created, coach TBD) — that is valid `active` state,
  not an error.

- **`roster_id` points at the roster; `/roster` owns the players.**
  `/team` records *which* roster a team draws from (`roster_id`) and may
  create the empty roster shell so the pointer resolves. It does **not**
  add or remove players — the `players[]` list, `max_size`, the
  `forming`/`set`/`locked` lifecycle all belong to `/roster`, which in
  turn keeps each player-member's `teams[]` consistent. Keep the split
  clean: `/team` ↔ `roster_id`; `/roster` ↔ roster contents + member
  back-refs.

- **Phase-2 fields are recorded, not operated.** `home_complex`,
  `leagues[]`, and `tournaments[]` reference entities whose own skills
  (`/complex`, `/league`, `/tournament`) are **not authored in this
  build**. `/team` records the *intent* — the string id the team will
  attach to — and degrades gracefully: it stores the value, notes it is
  a phase-2 reference, and does **not** try to validate it against a
  `state/complexes/` or `state/leagues/` record that may not exist yet.
  When those skills ship, they reconcile against what `/team` recorded.
  Never block a team operation because a phase-2 entity isn't modeled.

- **`state/` is the orchestrator's; `.claude/` is the toolkit's.** Team
  records and all the records they cross-reference live under `state/`.
  The rules this skill obeys live in `.claude/club-ops-rules.md`. Keep
  them straight: `state/` is operational data the orchestrator mutates;
  `.claude/` is installed toolkit.

## Operations

1. **create** — stand up a new team. Required: `age_group`, `gender`,
   `season`. Validate `age_group` + `gender` against `state/club.md`
   dimensions (ask or extend `club.md` if absent). Derive a stable `id`
   (`<age_group-lowercased>-<gender>-<season-slug>`, e.g.
   `u12-boys-fall-2026`) and a display `name`. Copy
   `state/teams/_template.md` → `state/teams/<id>.md` and fill the
   machine zone. Optionally assign coach(es) in the same pass (see
   **assign coach**) and create the paired roster shell so `roster_id`
   resolves (delegate the players to `/roster`). Set
   `status: active`. Refuse to create a duplicate id — if it exists,
   this is an **update**, not a create.

2. **assign coach** — put one or more `staff` coaches in charge.
   Resolve each named coach to a `state/staff/<id>.md` with
   `role: coach` (if not found, stop and point at `/staff` — do not
   invent the record). Add each id to the team's `coaches[]` AND add
   this team's id to each staff member's `teams[]`. Both surfaces or
   neither. Report who was assigned and confirm both sides agree.

3. **replace coach** — swap a coach off and another on. This is a
   remove + an assign, kept atomic: drop the outgoing id from the
   team's `coaches[]` and drop the team id from the outgoing coach's
   `teams[]`; add the incoming id to `coaches[]` and add the team id to
   the incoming coach's `teams[]`. End state: exactly the intended
   coaches on both sides, no orphaned back-references. Removing a coach
   without a replacement (coach steps down, TBD) is the same operation
   minus the add — leave `coaches[]` shorter, do not block.

4. **set home_complex** *(phase 2 — graceful)* — record which complex
   the team calls home. Store the complex id in `home_complex`. Do
   **not** validate it against a `state/complexes/` record (the
   `/complex` skill isn't authored in this build); record the intent
   and note it as a phase-2 reference. When `/complex` ships it
   reconciles against this value.

5. **enter league / tournament** *(phase 2 — graceful)* — record that
   the team competes in a league or enters a tournament. Append the
   league id to `leagues[]` or the tournament id to `tournaments[]`.
   This records the **intent** only — registration into the league's
   actual system, schedule generation, and the league/tournament record
   itself are `/league` and `/tournament` (and `/registration`) work,
   none authored in this build. Store the id, mark it phase-2, do not
   fabricate the league record.

6. **update** — change a team's facts in place (e.g. correct the
   `season`, fix the display `name`, repoint `roster_id`). Edit
   `state/teams/<id>.md`, overwriting changed machine fields and
   preserving the human-authored prose zones the template doesn't own.
   The `id` is immutable — a team whose `age_group`/`gender`/`season`
   identity genuinely changed is a different team (create a new one,
   archive the old). Update carries forward every fact you didn't
   change.

7. **list / filter** — enumerate teams from `state/teams/`. Support
   filtering by `age_group`, by `gender`, and by `season` (and
   combinations — "all U12 boys teams", "every team this fall"). Read
   the records, apply the filter, and present a scannable table:
   `id`, `name`, `age_group`, `gender`, `season`, coach count, `status`.
   This is read-only — it mutates nothing.

8. **archive** — retire a team at season's end. Set `status: archived`
   on `state/teams/<id>.md` (keep the record — archived teams are
   history, not deletions). Decide the coach back-references
   deliberately: archiving a *finished season's* team typically drops
   this team id from each coach's `teams[]` (the coach is no longer
   actively assigned) while leaving the team's own `coaches[]` intact as
   the historical record of who coached it. State what you did to the
   back-references in the report so the choice is auditable. Do not
   touch the roster contents — coordinate roster archival with
   `/roster` if both are retiring.

## When NOT to use

- **To manage roster contents (add/remove/lock players)** → that's
  `/roster`. `/team` owns `roster_id` (the pointer); `/roster` owns the
  `players[]`, `max_size`, and the forming/set/locked lifecycle, and
  keeps each member's `teams[]` consistent. Adding a player from inside
  `/team` would bypass the roster's integrity and the member back-refs.

- **To create or edit a coach as a person** → that's `/staff`. `/team`
  references existing `staff` coaches by id and maintains their
  `teams[]` back-reference; it never creates a staff record, edits a
  coach's certifications, or changes their `role`. If the coach isn't in
  staff, run `/staff` first.

- **To run a try-out that fills the team** → that's `/tryout`. A
  try-out evaluates candidates and produces roster assignments;
  `/tryout` feeds `/roster`, which feeds the team's `roster_id`. `/team`
  consumes the result; it does not run the evaluation.

- **To actually coach, train, or build a practice plan** → that's the
  role domains (`rasa.domain.soccer.coach`,
  `rasa.domain.soccer.team`, and the live
  `rasa.domain.soccer.goalkeeping`), each giving the *person* their role
  toolkit. The orchestrator manages the unit and its assignments; it
  never does the coaching work. These references are soft — graceful if
  the domain Element is absent.

- **To model a complex, league, or tournament as its own entity** →
  those are phase-2 skills (`/complex`, `/league`, `/tournament`), not
  authored in this build. `/team` records the *reference* (a
  `home_complex` id, a `leagues[]` / `tournaments[]` entry) and degrades
  gracefully; it does not stand up the entity record.

## What "done" looks like

The team is a managed club unit: `state/teams/<id>.md` is a complete,
human-readable record whose machine zone carries exactly the canonical
fields — `id`, `name`, `age_group`, `gender`, `season`, `coaches[]`,
`roster_id`, `home_complex`, `leagues[]`, `tournaments[]`, `status` —
with `age_group` + `gender` drawn from `state/club.md` dimensions. Every
id in `coaches[]` resolves to a real `state/staff/<id>.md` coach, and
each of those staff records lists this team in its `teams[]`: the
assignment agrees on both sides, no orphaned back-reference. `roster_id`
points at a roster `/roster` owns; phase-2 references (`home_complex`,
`leagues[]`, `tournaments[]`) are recorded as intent and degrade
gracefully against entities not yet modeled. Listing and filtering by
`age_group` / `gender` / `season` returns the team. Archiving flips
`status` to `archived`, preserves the record as history, and reconciles
the coach back-references as reported. Re-running an operation on the
same `id` updates that one record in place — never a duplicate team.
From here, `/roster`, `/tryout`, `/staff`, and the phase-2 schedule and
league skills all see the team when they iterate `state/teams/` — which
is the point: `/team` is the act that makes a team part of the club.
