---
name: staff
description: Register and manage the club's STAFF — coaches, directors, admins, and board members — as operational records the club owns. Use when adding a new coach/director/admin/board member, assigning a coach to a team (or teams), recording certifications (licenses, background checks, SafeSport, CPR), updating contact or role, looking up a staff member, listing or filtering staff by role/status/team, or deactivating someone who has left. Triggered by e.g. "/staff register coach Maria Reyes", "/staff assign coach STF-014 to TEAM-U12-boys", "/staff add cert to STF-009", "/staff list coaches", "/staff lookup STF-003", "/staff deactivate STF-021". The orchestrator owns the staff record + team assignments; the role's coaching/directing/governance playbook lives in the matching domain.soccer.* Element.
---

# /staff — register + manage club staff

The club's people-of-record on the operations side: coaches, directors,
admins, and board members. This skill is the **single front door** for
creating and maintaining a Staff record and for keeping the
coach↔team relationship consistent on both sides of the link.

Essence: **the orchestrator owns the staff record and the assignments;
the role-domain owns the playbook.** A coach's coaching toolkit is
`rasa.domain.soccer.coach`, a director's is `rasa.domain.soccer.director`,
a board member's is `rasa.domain.soccer.board`. This skill does not coach,
direct, or govern — it registers *who* the person is, *what role* they
hold, *which teams* they run, and *which certifications* they carry, and
it points at the right domain for the actual work. Those domain references
are **soft**: degrade gracefully if the Element is absent (only
`rasa.domain.soccer.goalkeeping` is live so far).

Records live at `state/staff/<id>.md`, created from
`state/staff/_template.md`. The governing rules are
`.claude/club-ops-rules.md` (entity model + relationship-integrity
invariants). Sibling skills this one links to: `/member` (the players +
parents side of People), `/team` (the unit a coach is assigned to),
`/roster`, and `/tryout` (a tryout's `evaluators[]` are staff ids).

## Behavior contract

- **One Staff record per person.** A Staff `id` (`STF-NNN`) is the
  permanent handle. Never mint a second record for someone who already
  has one — if a coach is promoted to director, you *update the role*
  on the existing record, you do not create a new one. Before
  registering, search `state/staff/` for the name to avoid a duplicate.
- **The orchestrator owns operational data; the domain owns the
  playbook.** This skill records the person and their assignments. It
  does NOT author practice plans (that's `rasa.domain.soccer.coach`),
  board governance docs (`rasa.domain.soccer.board`), or program strategy
  (`rasa.domain.soccer.director`). When a request is about *doing the
  role's work*, hand off to the domain — see **When NOT to use**.
- **`role` is one of exactly four values:** `coach | director | admin |
  board`. No free-text roles. A person who genuinely wears two hats
  (a board member who also coaches) gets the role that drives their
  primary operational footprint, with the second noted in the record's
  human-authored section — the relational model keys assignments off the
  single `role` field. If dual-role assignment becomes common, that's a
  rules change (`club-ops-rules.md`), not a per-record improvisation.
- **`teams[]` is meaningful for coaches only.** It holds Team ids
  (`TEAM-...`) the coach runs. For directors/admins/board it stays empty
  (`[]`). The inverse link is the Team's `coaches[]` (staff ids). These
  two MUST agree — see the integrity rule below.
- **Coach↔team integrity is two-sided and atomic.** Assigning coach
  `STF-014` to `TEAM-U12-boys` writes the team id into the coach's
  `teams[]` AND the coach's id into that team's `coaches[]`. Both
  surfaces change or neither does. A half-applied assignment (one side
  references the other, the other doesn't) is corruption — this is the
  `state/repos`-style integrity the club ops depend on. The same holds
  for un-assignment.
- **Certifications are facts with dates, not flags.** `certifications[]`
  entries carry a `type`, optional `level`, `issued`, and `expires`.
  Recording a cert never silently overwrites another — append or update
  the matching `type`. An expired or missing cert is surfaced, never
  hidden; eligibility decisions (can this coach be assigned?) read the
  cert list but this skill does not block on them unless
  `club-ops-rules.md` says to.
- **Deactivate, don't delete.** Leaving the club sets `status: inactive`
  and unwinds active assignments (remove the staff id from each team's
  `coaches[]`, clear the coach's `teams[]`). The record stays on disk as
  history. Hard deletion is out of scope for this skill.
- **Every write lands in all the surfaces it touches or none.** If an
  operation can't complete every side of a relationship (e.g. a team id
  in an assignment doesn't resolve to a `state/teams/` record), stop,
  report the missing surface, and write nothing. Partial writes are the
  one failure mode this skill is built to prevent.

## The Staff record — canonical fields

Every `state/staff/<id>.md` carries this frontmatter (the machine zone,
orchestrator-owned). These names are fixed across every skill, template,
and rule in this Element — do not rename or add ad-hoc keys:

| Field | Meaning |
|---|---|
| `id` | `STF-NNN`, permanent. Assigned at register; never reused. |
| `name` | Full name, display form. |
| `role` | `coach` \| `director` \| `admin` \| `board`. Exactly one. |
| `teams[]` | Team ids this person runs. **Coaches only**; `[]` otherwise. Mirrors each team's `coaches[]`. |
| `certifications[]` | List of `{ type, level?, issued, expires? }` — licenses, background checks, SafeSport, CPR, etc. |
| `contact` | `{ email, phone }` (and optional address). |
| `status` | `active` \| `inactive`. |

The body below the frontmatter is the human-authored zone (bio, notes,
dual-role context, references to the person's role-domain). The skill
owns the frontmatter; humans own the prose.

## Operations

### 1. Register a staff member

`/staff register <role> <name> [contact]`

1. Search `state/staff/` for an existing record matching `name`. If a
   plausible match exists, surface it and ask before minting a new id —
   one record per person.
2. Mint the next `id` (`STF-NNN`, highest existing + 1).
3. Copy `state/staff/_template.md` → `state/staff/<id>.md`.
4. Fill frontmatter: `id`, `name`, `role` (must be one of the four),
   `teams: []`, `certifications: []` (or seed any provided), `contact`,
   `status: active`.
5. In the human zone, note which role-domain is this person's playbook —
   coach → `rasa.domain.soccer.coach`, director →
   `rasa.domain.soccer.director`, board → `rasa.domain.soccer.board`,
   admin → none (operations-only). State it as a soft reference: "playbook:
   <domain> (install to activate)".
6. Report the new id + role + the domain pointer.

### 2. Assign a coach to a team (or teams)

`/staff assign <staff-id> to <team-id> [<team-id> ...]`

1. Resolve the staff record. **Require `role: coach`** — directors,
   admins, and board members are not assigned to teams via `teams[]`.
   If the role isn't `coach`, stop and explain (a director "over" a team
   is a different relationship — record it in prose, not `teams[]`).
2. Resolve each `<team-id>` against `state/teams/`. If any does not
   resolve, stop and write nothing — report the missing team.
3. **Two-sided write, atomic:** add each team id to the coach's
   `teams[]`, AND add the coach's id to each team's `coaches[]`. Skip
   ids already present (idempotent — re-running is a no-op). If you can't
   complete both sides for every team, revert and report.
4. Report the resulting `teams[]` for the coach and the `coaches[]` for
   each team touched.

To **un-assign**, run the inverse: remove the team id from `teams[]` and
the staff id from that team's `coaches[]`, both sides or neither.

### 3. Record a certification

`/staff cert <staff-id> <type> [level=<L>] [issued=<date>] [expires=<date>]`

1. Resolve the staff record.
2. Build the cert object: `{ type, level?, issued, expires? }`. Common
   `type` values: `USSF-license`, `background-check`, `SafeSport`, `CPR`,
   `first-aid`, `concussion-protocol`. Keep `type` consistent so the list
   is filterable.
3. **Append or update by `type`:** if an entry of that `type` exists,
   update it (a renewed license, a refreshed background check); otherwise
   append. Never silently drop an existing cert.
4. Report the updated `certifications[]` and flag any entry whose
   `expires` is in the past or within the club's warning window (per
   `club-ops-rules.md` if defined).

### 4. Update a staff member

`/staff update <staff-id> <field>=<value> [...]`

1. Resolve the staff record.
2. Apply the change. Allowed fields: `name`, `role`, `contact`,
   individual `certifications[]` entries (prefer Op 3 for certs).
3. **Role change is the integrity-sensitive case.** Promoting a coach to
   director means their `teams[]` no longer applies — before clearing it,
   unwind each assignment (remove the staff id from each team's
   `coaches[]`), then set `teams: []`, then set the new role. Both sides,
   atomic. Note the role history in the human zone. Update the role-domain
   pointer to match the new role.
4. Report what changed and any assignments that were unwound.

### 5. Look up a staff member

`/staff lookup <staff-id | name>`

1. Resolve by id, or by name search across `state/staff/`.
2. Print the record: id, name, role, teams (resolved to team names where
   possible), certifications with expiry status, contact, status, and the
   role-domain pointer.
3. Read-only — no writes.

### 6. List / filter staff

`/staff list [role=<coach|director|admin|board>] [status=<active|inactive>] [team=<team-id>]`

1. Scan `state/staff/`.
2. Apply filters: by `role`, by `status` (default to `active` if
   unspecified), by `team` (membership in `teams[]`).
3. Render a table: `id | name | role | teams | status | certs (count, any expired?)`.
4. Read-only.

### 7. Deactivate a staff member

`/staff deactivate <staff-id>`

1. Resolve the staff record.
2. **Unwind active assignments first:** for each team in the coach's
   `teams[]`, remove this staff id from that team's `coaches[]`; then set
   `teams: []`. Both sides, atomic.
3. Set `status: inactive`. Leave the record on disk (history).
4. Report the teams left without this coach so `/team` / `/roster` can
   backfill coverage if needed.

## When NOT to use

- **Doing the role's actual work.** Writing a practice plan, a session
  curriculum, or player development notes → `rasa.domain.soccer.coach`.
  Board governance, bylaws, meeting minutes → `rasa.domain.soccer.board`.
  Program strategy, hiring philosophy, club-wide standards →
  `rasa.domain.soccer.director`. This skill records the *person and the
  assignment*, not the craft. If the domain Element isn't installed, say
  so and stop — don't improvise the role's playbook here.
- **Players and parents.** Those are `Member` records, not `Staff`. Use
  `/member`. Coaches/directors/admins/board are staff; players/parents are
  members. The wall between the two entity types is deliberate.
- **Creating or editing a team itself** (name, age_group, season,
  home_complex) → `/team`. This skill only edits a team's `coaches[]`,
  and only as the mirror side of a coach assignment.
- **Picking who's on a team's roster** → `/roster`. Staff run teams;
  rosters are players. Different relationship, different skill.
- **A tryout's evaluators.** `/tryout` reads staff ids into its
  `evaluators[]`; it does not register them. Register the evaluator here
  first if they're new, then reference the id from `/tryout`.
- **Phase-2 surfaces** (a coach's certification gating a `League` or
  `Tournament` registration; field/complex assignment for a session) —
  reference them as "phase 2" and degrade gracefully. Those skills are
  not authored in this build.

## What "done" looks like

- **Register** → a new `state/staff/<id>.md` exists with complete,
  valid frontmatter (`role` one of four, `status: active`), a role-domain
  pointer in the human zone, and the new id reported.
- **Assign / un-assign** → the coach's `teams[]` and every touched team's
  `coaches[]` agree exactly — no half-applied links. The report names both
  sides.
- **Cert** → `certifications[]` reflects the new/renewed cert by `type`,
  nothing dropped, expiry status surfaced.
- **Update** → the field changed; if the role changed, all prior
  assignments were unwound atomically and the domain pointer updated.
- **Lookup / list** → an accurate read with team names and cert/expiry
  status, no writes.
- **Deactivate** → `status: inactive`, `teams[]` cleared, every former
  team's `coaches[]` no longer references this person, record retained as
  history.

In every case: the relationship is consistent on **all** surfaces it
touches, or the operation made no change at all. That invariant is the
whole point of routing staff through this skill instead of editing
`state/staff/` by hand.
