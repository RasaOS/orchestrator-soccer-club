# Club-Ops Rules — the entity-model spine

These rules apply to **every** skill this orchestrator runs. They are
the club-operations coordination contract — the load-bearing spine the
skills all reference. Read this file before running any club-ops skill.

This file installs to `.claude/club-ops-rules.md` in the club's
umbrella repo. The skills install to `.claude/skills/`; the club's
operational **state** lives in `state/`. The two are deliberately
separate: the toolkit (these rules + the skills) is replaceable; the
state (the records) is the club's own, forever.

> **What this orchestrator is.** `rasa.orchestrator.soccer.club` is a
> **club-operations platform** — Pattern 3 (Hybrid / CTO-style) per
> `orchestrator-core/content/SHAPE.md`. It sits ABOVE the whole
> organization and manages its **operational data**: the people, the
> units, the events, the taxonomy. It is the first sub-namespaced
> orchestrator in the substrate (`rasa.orchestrator.soccer.club`, not
> `rasa.orchestrator.<word>`).
>
> It does **not** coach, train, scout, or referee. The *doing* of each
> role lives in a role-domain (`rasa.domain.soccer.{player, coach,
> parent, director, board, team, league}` plus the live
> `rasa.domain.soccer.goalkeeping`). This orchestrator manages the
> club's records of **who, what, and when**; the domains give each
> person the toolkit for their role. The boundary is load-bearing —
> see §3.

> **How to read this file.** §1 is the entity model — the thirteen
> concerns, their exact field sets, and the relationships and
> invariants that bind them. Everything else in the Element (every
> skill, every template) is downstream of §1; if a field name appears
> in a skill, it is defined here first. §2 is the cross-cutting
> taxonomy (the dimensions). §3 is the load-bearing rule. §4 is the
> `state/` layout. §5 is the foundation-vs-phase-2 wave split. §6 is
> the integrity contract every CRUD skill honors. When a skill and
> this file disagree, **this file wins** — fix the skill.

---

## 1. The entity model

The club is a **relational model of thirteen operational concerns**.
Five are *people-and-unit records* with full CRUD skills in this
foundation build; eight more are *defined here* and given a `state/`
home, but their skills land in a later wave (§5). The model covers all
thirteen now so the relationships are coherent from day one — the
skills just arrive in waves.

Every record is one `.md` file under `state/<entity>/<id>.md`, created
from `state/<entity>/_template.md` (§4). The field sets below are
**canonical**: every skill and every template MUST use these exact
field names. They are the column definitions of the club's database;
drift in a field name is drift in the schema.

### 1.1 The thirteen concerns

| # | Entity | Kind | Wave | The record answers |
|---|---|---|---|---|
| 1 | **Member** | person | foundation | a player or a parent in the club |
| 2 | **Staff** | person | foundation | a coach, director, admin, or board member |
| 3 | **Team** | unit | foundation | a competitive unit (one age_group + gender) |
| 4 | **Roster** | unit | foundation | the player list bound to one team |
| 5 | **Tryout** | event | foundation | an evaluation event that produces roster assignments |
| 6 | **League** | external | phase 2 | a competition a team plays in |
| 7 | **Tournament** | external | phase 2 | a bracketed event a team enters |
| 8 | **Field** | facility | phase 2 | a playing surface, belongs to a complex |
| 9 | **Complex** | facility | phase 2 | a venue (fields, parking, vending, address) |
| 10 | **Schedule** | event | phase 2 | a practice or game: team × field × datetime |
| 11 | **Registration** | record | phase 2 | a member or team enrolled in a system for a season |
| (dim) | **age_groups** | taxonomy | foundation | the club's age bands (U6..U19, Adult) |
| (dim) | **genders** | taxonomy | foundation | boys \| girls \| coed |
| (dim) | **registration_systems** | taxonomy | foundation | the club's own + state/regional/national orgs |

The three **dimensions** (age_groups, genders, registration_systems)
are cross-cutting taxonomy, not per-record entities — they live in
`state/club.md`, not in a per-record directory (§2). They are still
part of the model: a Member's `age_group` is only valid if it is one
of the age_groups defined in `state/club.md`.

### 1.2 The canonical field sets

These are the exact frontmatter fields. A template's YAML and a skill's
writes MUST match them name-for-name.

**Member** — a player or a parent.
```
id            # PRIMARY KEY = filename stem. e.g. member-0412. unique, stable, never reused.
name          # display name.
type          # player | parent. determines which type-specific fields apply.
status        # active | inactive | prospect.
contact       # email / phone / preferred channel.
# --- if type: player ---
age_group     # one of state/club.md age_groups. EXACTLY ONE.
gender        # one of state/club.md genders. EXACTLY ONE.
parents[]     # member ids of this player's parent records (type: parent). may be empty.
teams[]       # team ids this player is rostered on (derived from rosters; kept in sync — §6).
registrations[]  # registration ids this player holds (phase 2; [] until registrations land).
# --- if type: parent ---
players[]     # member ids of this parent's player records (type: player).
```

**Staff** — a coach, director, admin, or board member.
```
id            # PRIMARY KEY = filename stem. e.g. staff-018. unique, stable.
name          # display name.
role          # coach | director | admin | board.
status        # active | inactive | prospect.
contact       # email / phone / preferred channel.
certifications[]  # licenses / clearances (e.g. "USSF D", "SafeSport 2026", "background-check").
teams[]       # team ids this staff member is assigned to (coaches only; [] for non-coaching roles).
```

**Team** — a competitive unit. One age_group, one gender.
```
id            # PRIMARY KEY = filename stem. e.g. team-2014b-red. unique, stable.
name          # display name (e.g. "2014 Boys Red").
age_group     # one of state/club.md age_groups. EXACTLY ONE.
gender        # one of state/club.md genders. EXACTLY ONE.
season        # the season this team instance belongs to (e.g. "2026-fall").
coaches[]     # staff ids (role: coach) assigned to this team.
roster_id     # the one roster bound to this team (one team ↔ one active roster). may be null pre-roster.
home_complex  # complex id this team plays/practices out of (phase 2; may be null until complexes land).
leagues[]     # league ids this team competes in (phase 2; [] until leagues land).
tournaments[] # tournament ids this team has entered (phase 2; [] until tournaments land).
status        # active | inactive | prospect (a forming team is prospect; a folded team inactive).
```

**Roster** — the player list bound to one team.
```
id            # PRIMARY KEY = filename stem. e.g. roster-2014b-red-fall26. unique, stable.
team          # the ONE team id this roster belongs to. a roster serves exactly one team.
players[]     # member ids (type: player) on this roster.
from_tryout   # tryout id this roster was assembled from, if any (else null).
max_size      # roster cap (integer; the club's per-age_group rule, or null for uncapped).
status        # forming | set | locked. forming = in progress; set = complete; locked = no changes.
```

**Tryout** — an evaluation event that produces roster assignments.
```
id            # PRIMARY KEY = filename stem. e.g. tryout-2014b-fall26. unique, stable.
age_group     # one of state/club.md age_groups. the band being evaluated. EXACTLY ONE.
gender        # one of state/club.md genders. EXACTLY ONE.
date          # the tryout date (YYYY-MM-DD).
location      # complex id and/or field id where it's held (phase-2 entities; free-text fallback until they land).
evaluators[]  # staff ids running the evaluation.
candidates[]  # member ids (type: player, often status: prospect) being evaluated.
outcomes      # the assignment result: per-candidate → which roster (member id → roster id, or "cut"/"hold").
status        # scheduled | complete. complete means outcomes are final and roster assignments have been applied.
```

**League** *(phase 2 — defined, no skill yet)* — a competition a team plays in.
```
id            # PRIMARY KEY. e.g. league-state-d1.
name          # display name.
level         # club | state | regional | national.
season        # season this league instance covers.
teams[]       # team ids participating (mirror of each team's leagues[]).
status        # active | inactive.
```

**Tournament** *(phase 2 — defined, no skill yet)* — a bracketed event a team enters.
```
id            # PRIMARY KEY. e.g. tournament-spring-cup-26.
name          # display name.
level         # club | state | regional | national.
date          # event date or date range.
location      # complex id / venue.
teams[]       # team ids entered (mirror of each team's tournaments[]).
status        # scheduled | complete.
```

**Field** *(phase 2 — defined, no skill yet)* — a playing surface, belongs to a complex.
```
id            # PRIMARY KEY. e.g. field-northpark-3.
name          # display name (e.g. "North Park Field 3").
complex       # the ONE complex id this field belongs to. a field belongs to exactly one complex.
surface       # grass | turf | indoor | other.
size          # full | 11v11 | 9v9 | 7v7 | 4v4 (the formats the field supports).
status        # available | maintenance | retired.
```

**Complex** *(phase 2 — defined, no skill yet)* — a venue.
```
id            # PRIMARY KEY. e.g. complex-northpark.
name          # display name.
address       # street address.
fields[]      # field ids belonging to this complex (mirror of each field's complex).
parking       # parking notes / capacity.
vending       # vending / concessions notes.
status        # active | inactive.
```

**Schedule** *(phase 2 — defined, no skill yet)* — a practice or game. The join row: team × field × datetime.
```
id            # PRIMARY KEY. e.g. schedule-2014b-red-2026-09-12-1800.
type          # practice | game.
team          # the team id this slot is for.
field         # the field id booked (phase-2 facility).
datetime      # start datetime (ISO 8601).
opponent      # for type: game — opposing team name/id (free-text or external).
status        # scheduled | played | cancelled.
```

**Registration** *(phase 2 — defined, no skill yet)* — a member or team enrolled in a system for a season.
```
id            # PRIMARY KEY. e.g. reg-member-0412-state-2026.
subject_type  # member | team. what is being registered.
subject       # the member id or team id being registered.
system        # one of state/club.md registration_systems (the club's own, or state/regional/national org).
season        # the season this registration covers.
status        # pending | submitted | approved | expired.
```

### 1.3 The relationships

The entities are not islands — they are a graph. These are the edges.
Every skill that touches one end of an edge MUST keep the other end
consistent (§6).

- **member (parent) ↔ member (player).** A bidirectional link inside
  the Member entity: a parent's `players[]` lists the player member
  ids; each of those players' `parents[]` lists the parent's member
  id. Both ends move together — adding a player to a parent's
  `players[]` means adding that parent to the player's `parents[]`.

- **member (player) → team, *via roster*.** A player is **not** placed
  directly on a team. The roster is the join: the roster's `players[]`
  is the authoritative list of who is on the team. A player's
  `teams[]` is a **derived mirror** of "which rosters list me, and
  which team does each roster serve." Change membership by changing the
  roster; the player's `teams[]` and the team's roster follow.

- **roster ← tryout.** A roster is **produced by** a tryout. The
  tryout's `outcomes` (member id → roster id) is what assigns
  candidates onto a roster; the resulting roster carries
  `from_tryout: <tryout id>`. A roster can also exist without a tryout
  (`from_tryout: null`) — a tryout is one way to fill a roster, not the
  only way.

- **roster → team.** A roster belongs to **exactly one** team
  (`roster.team`), and a team points back at its roster
  (`team.roster_id`). One team ↔ one active roster. The two ids are
  each other's inverse and must always agree (§6).

- **team → staff (coaches).** A team's `coaches[]` holds staff ids
  (role: coach); each of those staff records lists the team in its own
  `teams[]`. Bidirectional — assigning a coach to a team writes both
  ends.

- **team → league / tournament** *(phase 2).* A team's `leagues[]` and
  `tournaments[]` mirror the participating-teams lists on the league
  and tournament records. Bidirectional when those entities are live;
  until phase 2 these stay `[]` and the team's competition is recorded
  as free-text or left empty.

- **team → home_complex** *(phase 2).* A team points at one complex
  (`team.home_complex`) it plays/practices out of. Soft until complexes
  land — `null` is valid and degrades gracefully (§5).

- **schedule = team × field × datetime** *(phase 2).* A schedule row is
  the join of a team, a field (within a complex), and a datetime. It is
  the booking. No two `type: game`/`practice` rows should double-book
  the same `field` at the same `datetime` (a phase-2 integrity check).

- **registration = (member | team) × system × season** *(phase 2).* A
  registration ties a subject (a member or a team) to a registration
  system for a season. A member's / team's `registrations[]` mirrors
  the registration records whose `subject` is that member/team.

### 1.4 The invariants

These hold at all times. A skill that would violate one must refuse,
not proceed.

1. **A player has exactly one age_group and exactly one gender.** Both
   drawn from the dimensions in `state/club.md`. A player who "could
   play up" is still recorded with one true age_group; playing up is a
   roster decision, not a second age_group on the member.

2. **A team has exactly one age_group and exactly one gender.** A team
   is the unit *of* an age_group and gender. "2014 Boys" is one team
   identity; a different age_group or gender is a different team.

3. **A roster belongs to exactly one team.** `roster.team` is single-
   valued and immutable for the life of the roster. A team's players
   for a different season is a *different roster* (a new record), not a
   re-pointed one.

4. **A roster's players should match the team's age_group + gender.** A
   player on a team's roster is expected to share that team's age_group
   and gender (modulo a deliberate, recorded "playing up" exception).
   A roster skill flags a mismatch rather than silently accepting it.

5. **A record change must update every surface it touches, or none.**
   This is the integrity invariant (§6). Roster ↔ team ↔ player are
   three surfaces of one fact; a change lands in all of them
   atomically or the change is refused. Two surfaces disagreeing is the
   failure mode the whole spine guards against — the "two truths"
   problem (cf. `orchestrator.code`'s registry rule).

6. **Every foreign-key id resolves, or is explicitly null.** A
   `coaches[]` entry is a real `staff-*` id; a `roster_id` is a real
   `roster-*` id; a `home_complex` is a real `complex-*` id **or**
   `null`. A dangling id (points at a record that doesn't exist) is
   drift an audit must catch.

7. **A status is one of its entity's enumerated values.** No
   free-text statuses. Member/Staff: `active | inactive | prospect`.
   Roster: `forming | set | locked`. Tryout: `scheduled | complete`.
   Team: `active | inactive | prospect`. (Phase-2 entities carry their
   own enumerations, listed in §1.2.)

---

## 2. The dimensions — taxonomy in `state/club.md`

Three cross-cutting taxonomies parameterize the whole model. They are
**not** per-record entities; they are the club's configuration, and
they live in **one file**: `state/club.md`. Every entity that carries
an `age_group`, `gender`, or `system` validates against this file.

- **age_groups.** The club's age bands — e.g. `U6, U7, … U19, Adult`.
  A real club sets its own (some run `U8..U18`; some add `Adult` /
  `Senior`). The list is authored once in `state/club.md`; every
  Member (player), Team, and Tryout draws its `age_group` from it.

- **genders.** `boys | girls | coed`. (`coed` exists for clubs that run
  mixed teams at young ages or recreational divisions.) Every Member
  (player), Team, and Tryout draws its `gender` from this list.

- **registration_systems.** The systems a member or team can be
  registered with: the **club's own** system, plus the **state**,
  **regional**, and **national** governing bodies the club affiliates
  with (the actual org names are club-specific — a US club might list
  its state association, a regional league, and a national federation).
  Every Registration (phase 2) draws its `system` from this list.

**Why one file, not three directories.** Dimensions are small, stable,
and read by nearly every skill on nearly every operation. A team-create
skill reads age_groups and genders to validate; a registration skill
reads registration_systems. Keeping them in `state/club.md` makes them
a single, cheap, always-loaded lookup — and makes "what age groups does
this club run?" a one-file answer. A new club configures its model by
editing `state/club.md` **before** creating any records.

`state/club.md` also carries the club's own identity header (club name,
current season, the season list) so a skill has a single home for
"facts about this club as a whole" that aren't about any one record.

---

## 3. The load-bearing rule

This is the **load-bearing rule** of the whole Element. Read it twice.

**The orchestrator OWNS the club's operational data (`state/`). The
role-domains OWN each person's toolkit.** The boundary is absolute:

- **The orchestrator manages people, units, and events — the
  *records*.** Who is a member, which team they're on, who coaches it,
  when the tryout is, which players got assigned to which roster. This
  is **operational data**: facts about the club's organization. It all
  lives in `state/`, and these skills are the only thing that writes
  it.

- **The orchestrator does NOT do the role's work.** It does **not**
  coach a session, write a training plan, scout a player, run a
  parent's logistics, or make a director's strategic call. The *doing*
  of each role is a **role-domain**:
  - `rasa.domain.soccer.player` — a player's own training/development
    toolkit.
  - `rasa.domain.soccer.coach` — a coach's session-planning and team-
    management toolkit.
  - `rasa.domain.soccer.parent` — a parent's logistics toolkit.
  - `rasa.domain.soccer.director` — a director of coaching's toolkit.
  - `rasa.domain.soccer.board` — a board member's governance toolkit.
  - `rasa.domain.soccer.team` — a team-level operations toolkit.
  - `rasa.domain.soccer.league` — a league-operations toolkit.
  - `rasa.domain.soccer.goalkeeping` — **the one live role-domain
    today** (`rasa.domain.soccer.goalkeeping`, v0.1.0) — a goalkeeper-
    development toolkit.

  The orchestrator records that `member-0412` is a coach assigned to
  `team-2014b-red`; `rasa.domain.soccer.coach` is what that coach uses
  to actually coach. The orchestrator knows the *who/what/when*; the
  domain owns the *how*.

- **References to the role-domains are SOFT.** The orchestrator's
  `rasa.json` does **not** list any `rasa.domain.soccer.*` in
  `requires.elements[]`. A skill may *mention* the relevant domain
  ("a coach's session work lives in `rasa.domain.soccer.coach`") and
  *point* a person at it, but it never depends on it being present.
  **Only `rasa.domain.soccer.goalkeeping` is live so far** — the rest
  are referenced as future role-domains and degrade gracefully when
  absent. A skill that references a not-yet-existing domain names it as
  the future home of that work and does the records part regardless.

- **The one-line test:** *is this a fact about the club's organization,
  or is this the work of a role?* If it is "who is on what, with whom,
  when" — it is **operational data**, and it belongs here, in `state/`.
  If it is "how this person does their job" — it belongs in that
  person's **role-domain**, not here. Records here; craft there.

**Why the boundary is hard.** Two truths is worse than one wrong
truth. If the orchestrator both holds the club's roster *and* starts
authoring coaching content, the operational model and the role toolkits
drift, and nobody knows which is canonical. Keep the orchestrator's
hands on the records and the records honest; let each domain own its
craft.

---

## 4. The `state/` layout

The club's operational data is a small file-database under `state/` in
the umbrella repo. The shape:

```
state/
├── club.md                  # the dimensions (§2) + club identity/seasons. ONE file.
├── member/
│   ├── _template.md         # the record shape; copy it to create a record.
│   ├── member-0412.md       # one .md per member. filename stem = id.
│   └── …
├── staff/
│   ├── _template.md
│   ├── staff-018.md
│   └── …
├── team/
│   ├── _template.md
│   ├── team-2014b-red.md
│   └── …
├── roster/
│   ├── _template.md
│   └── …
├── tryout/
│   ├── _template.md
│   └── …
│   # ── phase 2: directories exist, _template.md present, skills land later (§5) ──
├── league/        { _template.md }
├── tournament/    { _template.md }
├── field/         { _template.md }
├── complex/       { _template.md }
├── schedule/      { _template.md }
└── registration/  { _template.md }
```

The rules of the layout:

- **One directory per entity.** `state/<entity>/` holds all records of
  that entity. The directory name is the entity name, singular,
  lowercase (`member`, `team`, `roster`, …).

- **One `.md` per record.** `state/<entity>/<id>.md`. The **filename
  stem is the record's `id` and primary key** (`member-0412.md` → id
  `member-0412`). Ids are unique, stable, and never reused.

- **Every record is created from a `_template.md`.** Each
  `state/<entity>/_template.md` is the canonical shape for that
  entity — the YAML frontmatter (the §1.2 field set) plus readable
  sections. A create-skill copies the template to `<id>.md` and fills
  it; it never authors a record from scratch. The template ships in the
  Element (`content/state/<entity>/_template.md`) and installs into the
  umbrella repo's `state/`, where it is refreshed on upgrade. The
  template is the schema-by-example; if §1.2 changes, the template
  changes with it.

- **All eleven per-record directories exist from install** — including
  the six phase-2 entities. Their `_template.md` is present so the
  model is complete and a human *could* hand-author a phase-2 record;
  what's deferred is the *skill* that CRUDs them (§5), not the *shape*.

- **`state/` is the club's, not the Element's.** Like every Pattern 3
  orchestrator: the toolkit installs to `.claude/` and is replaceable;
  the records in `state/` are the club's own and survive a toolkit
  upgrade. The Element ships templates and skills; it never ships a
  club's actual member records.

---

## 5. Foundation vs phase 2 — the waves

The **model** (§1) covers all thirteen concerns now. The **skills**
land in waves. This build authors the foundation; phase 2 is the next
wave.

### Foundation (this build) — the spine + People + Org/Process

| Concern | Skill (this build) | Why foundation |
|---|---|---|
| **Member** | `/member` | the club is people first. |
| **Staff** | `/staff` | coaches/directors/admins/board — the people who run it. |
| **Team** | `/team` | the core competitive unit. |
| **Roster** | `/roster` | binds players to a team; the load-bearing join. |
| **Tryout** | `/tryout` | the process that produces rosters. |
| **dimensions** | (in `state/club.md`) | the taxonomy every record validates against. |

The foundation is: the club-ops spine (this file) + People (member,
staff) + Org/Process (team, roster, tryout) + status. That is enough to
stand up a club's organizational core — enroll people, form teams, run
tryouts, set rosters — end to end.

### Phase 2 (next wave) — Facilities, Scheduling, Competition, Registration

| Concern | Skill (phase 2) | Depends on |
|---|---|---|
| **Complex** | `/complex` (future) | — |
| **Field** | `/field` (future) | complex (a field belongs to one). |
| **Schedule** | `/schedule` (future) | team + field. |
| **League** | `/league` (future) | team. |
| **Tournament** | `/tournament` (future) | team. |
| **Registration** | `/registration` (future) | member/team + registration_systems. |

**Where a foundation skill touches a phase-2 entity, it degrades
gracefully.** Concretely:

- A **Team** carries `home_complex`, `leagues[]`, `tournaments[]` —
  all **phase 2**. The `/team` skill accepts them as `null` / `[]`,
  records a value if the human supplies a free-text one, and notes
  "complex/league/tournament are phase-2 entities — full linking lands
  with those skills." It never blocks team creation on a missing
  complex.

- A **Tryout** carries `location` (a complex/field reference) — **phase
  2**. The `/tryout` skill accepts a free-text location now and notes
  that structured complex/field linking arrives in phase 2.

- A **Member (player)** carries `registrations[]` — **phase 2**. The
  `/member` skill leaves it `[]` and notes registrations land later.

The rule: **a foundation skill never hard-depends on a phase-2 entity.**
It models the relationship (the field is there, in the canonical set),
references the phase-2 entity by name, and degrades — exactly the soft-
reference posture §3 takes toward the role-domains. **Do not author
phase-2 skills in this build.** Their *shape* is defined (§1.2) and
their `state/` home exists (§4); their *skills* are the next wave.

---

## 6. The integrity contract

Every CRUD skill in this Element honors one rule above all others:

**A change lands in every surface it touches, or in none of them.**

The model is relational, so a single human intent ("put this player on
this team") is several record writes. They are one transaction. The
skill computes the full set of surfaces first, writes them all, and —
if it cannot complete the set — writes nothing and reports what's
missing. Half-applied is forbidden: it is exactly the two-truths drift
§1.4(5) and §3 guard against.

The canonical multi-surface operations:

- **`/roster` adding a player to a team** touches **three** surfaces:
  1. the **roster** record — add the member id to `roster.players[]`;
  2. the **player** record — add the team id to the member's `teams[]`;
  3. the **team** record — ensure `team.roster_id` points at this
     roster.
  All three or none. (Removing a player reverses all three.)

- **`/tryout` completing an evaluation** writes the tryout's `outcomes`
  (member id → roster id / cut / hold) **and then** applies each
  assignment through the roster operation above — so a completed tryout
  lands the tryout record, the affected roster(s), each assigned
  player's `teams[]`, and each team's `roster_id`, together.

- **`/staff` assigning a coach to a team** touches **two** surfaces:
  the staff record's `teams[]` and the team record's `coaches[]`. Both
  or neither.

- **`/member` linking a parent and a player** touches **two** surfaces:
  the parent's `players[]` and the player's `parents[]`. Both or
  neither.

- **Status changes cascade-check, not cascade-write.** Setting a roster
  to `locked` doesn't rewrite the players, but a skill should refuse a
  change that would strand an invariant (e.g. deleting a team that
  still has a live roster — resolve the roster first, or refuse).

**The integrity discipline, stated once:** before writing, a skill
enumerates every record the change touches and every inverse edge it
must keep consistent (§1.3). It writes the complete set or it writes
nothing. After writing, the graph satisfies every invariant in §1.4.
An audit-style sweep (a future skill) re-checks the whole graph for
dangling ids, missing inverses, and orphaned records — but the live
skills are responsible for never *creating* drift in the first place.

---

## The spine in one breath

The club is a **relational model of thirteen concerns** (§1) over three
**dimensions** in `state/club.md` (§2). The orchestrator owns the
**operational data** in `state/` — one directory per entity, one `.md`
per record from a `_template` (§4); the **role-domains** own each
person's craft, referenced **softly**, only `goalkeeping` live so far
(§3). Five concerns get skills now (member, staff, team, roster,
tryout); the other six are **modeled now, skilled later** (§5). Every
write keeps the **whole graph consistent** — all surfaces of a change,
or none (§6). When in doubt: **is this a fact about the club's
organization, or the work of a role?** Records here; craft in the
domains.
