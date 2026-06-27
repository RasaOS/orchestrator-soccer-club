<!--
  TRYOUT RECORD TEMPLATE — rasa.orchestrator.soccer.club
  ======================================================

  This file is the SHAPE every `state/tryouts/<id>.md` follows. It is
  Element-owned: it ships in `content/state/tryouts/_template.md`, installs to
  `state/tryouts/_template.md` in the club's umbrella repo, and is refreshed on
  upgrade. Do NOT schedule a tryout by editing this file — copy it to
  `state/tryouts/<id>.md` (that's what `/tryout` does) and fill the copy.

  ONE FILE PER TRYOUT. The filename stem IS the tryout `id` and primary key
  (`ty-u12-boys-2026-spring.md` → id `ty-u12-boys-2026-spring`). The set of
  these files is the club's tryout log; /tryout and /roster iterate it.

  A TRYOUT is the club PROCESS that evaluates candidate players for an
  age-group/gender and PRODUCES roster assignments. It links staff (evaluators)
  and members (candidates) at a location on a date, and its `outcomes` become
  the membership of one or more `state/rosters/` records.

  THE LOAD-BEARING RULE
  ---------------------
  The orchestrator OWNS THE CLUB'S OPERATIONAL DATA (state/). This record is the
  source of truth for the evaluation EVENT and its results. It is NOT where the
  evaluation criteria or a coach's player-assessment method live — those are the
  role-DOMAINS (`domain.soccer.coach`, live `domain.soccer.goalkeeping` for
  keeper assessment), referenced SOFTLY.

  RELATIONAL INTEGRITY (a change lands in every surface it touches, or none)
  -------------------------------------------------------------------------
  - `evaluators[]` are `state/staff/<id>.md` ids (typically role: coach /
    director); /tryout reads their certifications[] for eligibility.
  - `candidates[]` are `state/members/<id>.md` ids (type: player). A candidate
    is often a `status: prospect` member not yet enrolled.
  - `outcomes` are the EVALUATION RESULTS that map candidates → roster
    assignments. When /tryout finalizes (status → complete), it WRITES those
    assignments into the target roster(s): adds the player to
    state/rosters/<roster_id>.md#players[], sets that roster's from_tryout to
    this id, AND updates each placed player's teams[] + flips prospect → active.
    The whole assignment lands across all surfaces, or none.
  - `location` references a PHASE-2 Complex/Field (state/complexes/,
    state/fields/ — not authored this build). Record a soft id/string now.

  TWO ZONES, BELOW
  ----------------
  1. FRONTMATTER (YAML) — machine-readable fields the skills parse. `/tryout`
     owns and overwrites these.
  2. PROSE SECTIONS — [orchestrator-owned] rewritten from facts;
     [human-authored — preserved] carried forward untouched.

  Replace every <PLACEHOLDER>. Delete this comment block in real records.
-->

---
# ─── machine-readable: parsed by the org/process skills (/tryout, /roster, /team) ───
id: <ty-agegroup-gender-season>       # PRIMARY KEY = filename stem. lowercase, hyphenated, unique. never reused.
age_group: "<U6..U19|Adult>"          # taxonomy value from state/club.md#age_groups. who this tryout is for.
gender: "<boys|girls|coed>"           # taxonomy value from state/club.md#genders.
date: "<YYYY-MM-DD>"                   # the day of the tryout (or first session if multi-day — note extras in prose).
location: <c-complex-id / f-field-id / "Field name">   # PHASE 2: where it's held (state/complexes/ or state/fields/). soft id/string.
evaluators: [<s-staff-id>, ...]       # staff ids running the evaluation. typically role: coach/director. /tryout checks certifications[].
candidates: [<m-player-id>, ...]      # member ids (type: player) trying out. often status: prospect (not yet enrolled).
outcomes:                             # the RESULTS: candidate → disposition. drives roster assignments when status: complete.
  - candidate: <m-player-id>
    result: <offered|waitlisted|cut|no-show>   # disposition for this candidate.
    roster: <r-roster-id|>            # the roster assigned to (when result: offered). blank otherwise. /roster realizes this.
status: <scheduled|complete>          # scheduled = upcoming/in-progress; complete = evaluated, outcomes finalized + roster(s) written.

# stamps — written by /tryout, read for staleness
registered_at: "<YYYY-MM-DD>"         # first time this tryout was scheduled. set once, never overwritten.
updated_at: "<YYYY-MM-DD>"            # last /tryout run that touched this file. bumps every update.
---

# <id> — <age_group> <gender> tryout

> `<age_group>` `<gender>` tryout · `<date>` · `<location>` · status **<status>**
> <N> candidate(s) · <N> evaluator(s)

One-line essence: the evaluation that selects players for this group's
roster(s). *(orchestrator-owned)*

---

## Logistics *(orchestrator-owned — rewritten by `/tryout`)*

When, where, and who runs it.

| Field | Value | Resolves to |
|---|---|---|
| **Date** | `<date>` | — |
| **Location** | `<location>` | **PHASE 2** `state/complexes/<id>.md` / `state/fields/<id>.md` |
| **Age group · gender** | `<age_group>` · `<gender>` | `state/club.md` taxonomy |

- Location is a **phase-2** soft reference — the Complex/Field entities aren't
  authored this build. A plain field name is fine until they exist.

---

## Evaluators *(orchestrator-owned — rewritten by `/tryout`)*

The staff running the evaluation. Each id is a `state/staff/<id>.md`; `/tryout`
checks `certifications[]` for eligibility (e.g. SafeSport on file).

| Evaluator | Role | Certs checked |
|---|---|---|
| `<s-staff-id>` | <coach | director | GK evaluator> | <e.g. SafeSport ✓> |

---

## Candidates & outcomes *(orchestrator-owned — `/tryout` writes, `/roster` realizes)*

The players evaluated and what happened to each. **This is the load-bearing
table** — when `status: complete`, each `offered` row's `roster` assignment is
written into `state/rosters/<roster>.md` and the placed player's `teams[]` +
member `status` are updated in the same pass.

| Candidate | Result | → Roster | Member status after |
|---|---|---|---|
| `<m-player-id>` | <offered\|waitlisted\|cut\|no-show> | `<r-roster-id>` / `—` | <active\|prospect (unchanged)> |

- `offered` + a `roster` → `/roster` adds the player and flips a `prospect`
  member to `active`.
- `waitlisted` / `cut` / `no-show` → no roster write; member `status` unchanged.
- While `status: scheduled`, outcomes are provisional and NO roster is written.
  Only finalizing (`scheduled` → `complete`) commits assignments.

---

## Role domain *(orchestrator-owned — SOFT reference)*

Evaluation method + criteria are the role-DOMAIN's job, not this record's:
`rasa.domain.soccer.coach` for field-player assessment, live
`rasa.domain.soccer.goalkeeping` for keeper assessment (**the only live
role-domain today**). The orchestrator runs the EVENT and records WHO was
assigned where; the domain informs HOW players are judged.

- **Degrade gracefully.** If the named domain Element is absent, this record
  still stands alone.

---

## Notes *(human-authored — preserved across updates)*

Free-form prose the director / coach owns. `/tryout` **never** overwrites this —
it carries it forward verbatim. Put here multi-day session schedule, weather
reschedule history, scouting context on specific candidates, why a strong
candidate was waitlisted, call-up intentions — anything not captured by a
structured field above.

- <e.g. "Two-session format: technical Sat, small-sided Sun. m-sam-cole
   impressed as a keeper — flagged to the GK coach. Held three offers pending
   registration completion before writing to the roster.">

---

<!--
  CONSISTENCY CONTRACT (for /tryout and /roster)
  ----------------------------------------------
  After writing this file:
  - Every id in evaluators[] is a state/staff/<id>.md; every id in candidates[]
    is a state/members/<id>.md (type: player). Every outcomes[].candidate is in
    candidates[].
  - When status: complete, for each outcomes row with result: offered and a
    `roster` set: that state/rosters/<roster>.md exists, its players[] contains
    the candidate, its from_tryout == this id, and the candidate member's
    teams[] contains that roster's team (with member status flipped prospect →
    active). All those surfaces land together or the tryout is NOT complete.
  - While status: scheduled, NO roster has been written — outcomes are
    provisional. Realizing assignments is exactly what completing the tryout does.
  - location is a PHASE-2 soft ref; it may name a complex/field with no record
    yet. Expected — do not fabricate phase-2 records to satisfy it.
-->
