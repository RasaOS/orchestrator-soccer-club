<!--
  TEAM RECORD TEMPLATE — rasa.orchestrator.soccer.club
  ====================================================

  This file is the SHAPE every `state/teams/<id>.md` follows. It is
  Element-owned: it ships in `content/state/teams/_template.md`, installs to
  `state/teams/_template.md` in the club's umbrella repo, and is refreshed on
  upgrade. Do NOT create a team by editing this file — copy it to
  `state/teams/<id>.md` (that's what `/team` does) and fill the copy.

  ONE FILE PER TEAM. The filename stem IS the team `id` and primary key
  (`t-u12-boys-navy.md` → id `t-u12-boys-navy`). The set of these files is the
  club's team directory; every org/process skill iterates it.

  A TEAM is the club's core competitive UNIT: an age-group + gender squad for a
  season, run by one or more coaches, holding exactly one roster, and entered in
  leagues / tournaments. It is the hub the other entities point at — coaches
  (staff), players (members via the roster), and (phase 2) a home complex,
  schedules, and registrations all hang off the team.

  THE LOAD-BEARING RULE
  ---------------------
  The orchestrator OWNS THE CLUB'S OPERATIONAL DATA (state/). This record is the
  source of truth for the team as a UNIT — its identity, who coaches it, which
  roster it holds, what it competes in. It is NOT where the coaching method or
  match tactics live — those are the role-DOMAINS, referenced SOFTLY.

  RELATIONAL INTEGRITY (a change lands in every surface it touches, or none)
  -------------------------------------------------------------------------
  - `coaches[]` here ⇄ each staff member's `teams[]`. `/team` writes both sides.
  - `roster_id` here points at exactly ONE `state/rosters/<roster_id>.md`, whose
    `team` field MUST point back at THIS id. `/roster` keeps the pointer paired
    and propagates each rostered player's `teams[]`. One team, one roster.
  - `home_complex`, `leagues[]`, `tournaments[]` reference PHASE-2 entities
    (`state/complexes/`, `state/leagues/`, `state/tournaments/` — not authored
    this build). Record them as soft id strings; degrade gracefully until those
    entities + skills exist.

  TWO ZONES, BELOW
  ----------------
  1. FRONTMATTER (YAML) — machine-readable fields the skills parse. `/team`
     owns and overwrites these.
  2. PROSE SECTIONS — [orchestrator-owned] rewritten from facts;
     [human-authored — preserved] carried forward untouched.

  Replace every <PLACEHOLDER>. Delete this comment block in real records.
-->

---
# ─── machine-readable: parsed by the org/process skills (/team, /roster, /tryout) ───
id: <t-agegroup-gender-name>          # PRIMARY KEY = filename stem. lowercase, hyphenated, unique. never reused.
name: "<Display Team Name>"           # human display name, e.g. "U12 Boys Navy".
age_group: "<U6..U19|Adult>"          # taxonomy value from state/club.md#age_groups.
gender: "<boys|girls|coed>"           # taxonomy value from state/club.md#genders.
season: "<2026-spring|2026-fall|...>" # the season this team instance is for. a new season = a new team record.
coaches: [<s-staff-id>, ...]          # staff ids (role: coach) running this team. mirrors each coach's teams[].
roster_id: <r-roster-id>              # the ONE roster this team holds (state/rosters/<roster_id>.md). that roster's team == this id.
home_complex: <c-complex-id|>         # PHASE 2: the complex this team trains/plays at (state/complexes/). soft id, blank ok.
leagues: [<l-league-id>, ...]         # PHASE 2: leagues this team is entered in (state/leagues/). soft ids, [] ok.
tournaments: [<tn-tournament-id>, ...]# PHASE 2: tournaments this team is entered in (state/tournaments/). soft ids, [] ok.
status: <forming|active|inactive>     # forming = squad being assembled; active = competing this season; inactive = disbanded/off-season.

# stamps — written by /team, read for staleness
registered_at: "<YYYY-MM-DD>"         # first time this team was created. set once, never overwritten.
updated_at: "<YYYY-MM-DD>"            # last /team run that touched this file. bumps every update.
---

# <name>

> **<Display Team Name>** · `<age_group>` `<gender>` · `<season>` · status **<status>**
> Coaches: <coaches[]> · Roster: `<roster_id>`

One-line essence of this team as a competitive unit. *(orchestrator-owned)*

---

## Coaching staff *(orchestrator-owned — rewritten by `/team`)*

Who runs this team. Every id is a `state/staff/<id>.md` with `role: coach`.

| Coach | Role on team | Mirror surface |
|---|---|---|
| `<s-staff-id>` | <head | assistant | GK coach> | that staff member's `teams[]` |

- Symmetry is load-bearing: every coach here MUST list this `id` in their
  `teams[]`. `/team` writes both sides. Empty staff is a flag, not a valid
  state — write `unassigned` and let `/team` surface it.

---

## Roster *(orchestrator-owned — single pointer, owned by `/roster`)*

This team holds exactly **one** roster.

- **Roster:** `<roster_id>` → `state/rosters/<roster_id>.md`
- **Back-pointer invariant:** that roster's `team` field == `<id>` (this team).
- **Players:** live in the roster record, not here. `/roster` is the only skill
  that mutates membership, and it propagates each player's `teams[]` in lockstep.
  Read the roster for the squad; don't duplicate the player list into this file.

---

## Competition *(orchestrator-owned — PHASE 2, soft references)*

Where this team plays. **All PHASE-2** — the League, Tournament, and Complex
entities (and their skills) are not authored this build. Record soft id strings
now; the dedicated skills will resolve them later. Degrade gracefully: a missing
phase-2 entity is expected, not an error.

| Concern | Value | Resolves to (phase 2) |
|---|---|---|
| **Home complex** | `<home_complex>` | `state/complexes/<id>.md` |
| **Leagues** | `<leagues[]>` | `state/leagues/<id>.md` (level: club\|state\|regional\|national) |
| **Tournaments** | `<tournaments[]>` | `state/tournaments/<id>.md` |

- Schedules (practice/game × field × datetime) and Registrations (team × system
  × season) are likewise **phase 2** — referenced from the dimensions in
  `state/club.md`, not stored on the team yet.

---

## Role domain *(orchestrator-owned — SOFT reference)*

The toolkit the team's on-field work pulls from: `rasa.domain.soccer.team` for
team-level tactics/management, plus `rasa.domain.soccer.goalkeeping` for the
keeper unit (**the only live role-domain today**). The orchestrator manages the
team as a UNIT; the coaching itself is the domain's job.

- **Degrade gracefully.** If the named domain Element is absent, this record
  still stands alone.

---

## Notes *(human-authored — preserved across updates)*

Free-form prose the club admin / coach owns. `/team` **never** overwrites this —
it carries it forward verbatim. Put here season goals, playing philosophy,
promotion/relegation context, why the squad is `forming`, history across
seasons — anything not captured by a structured field above.

- <e.g. "Carried up from last year's U11 group; targeting the state league
   first division. GK rotation between two keepers — see goalkeeping domain.">

---

<!--
  CONSISTENCY CONTRACT (for /team and /roster)
  --------------------------------------------
  After writing this file:
  - roster_id points at exactly ONE state/rosters/<roster_id>.md whose `team`
    field == this team's id. The pointer is paired both ways or it's drift.
  - Every id in coaches[] is a state/staff/<id>.md (role: coach) whose teams[]
    contains this id. Both sides or neither.
  - home_complex / leagues[] / tournaments[] are PHASE-2 soft refs — they may
    name ids that have no record yet. That's expected; a phase-2 skill resolves
    them when it ships. Do NOT fabricate phase-2 records to satisfy a ref.
  - A team's player list is NEVER stored here — it lives in the roster, and
    /roster is the sole writer of membership + each player's teams[].
-->
