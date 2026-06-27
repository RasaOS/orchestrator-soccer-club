<!--
  This README is ELEMENT-OWNED. It ships at content/state/README.md, installs
  to state/README.md in the club's umbrella repo, and is REFRESHED ON UPGRADE.
  It documents the data model; it is not a place to record club-specific facts
  (those go in the records themselves and in state/club.md).
-->

# `state/` — the club's operational data

This directory is **the club's operational data**, and the orchestrator
(`rasa.orchestrator.soccer.club`) owns it. Everything the platform knows
about the club as an organization — every player, parent, coach, team,
roster, and try-out — lives here as a plain-text record on disk. The
skills installed under `.claude/` (`/member`, `/staff`, `/team`,
`/roster`, `/tryout`, `/status`) read and write these files.

The split is load-bearing:

- **`state/` is the orchestrator's** — the operational *data* (who's in
  the club, which teams exist, who's rostered where).
- **`.claude/` is the toolkit's** — the installed *skills + rules* that
  CRUD that data.

A record that isn't in `state/` does not exist to the club. Creating,
changing, and retiring records — and keeping their cross-references
consistent — is the whole job of the skills.

> **Not in this directory:** a player's training plan, a coach's
> methodology, a parent's playbook. Those belong to the **role-domains**
> (`rasa.domain.soccer.{player,coach,parent,director,board,team,league}`
> plus the live `rasa.domain.soccer.goalkeeping`), referenced **softly**.
> The orchestrator manages *people, units, and events*; it does not do a
> coach's coaching or a player's training. If a role-domain Element is
> absent, the records here still stand on their own — the references are
> pointers, not dependencies.

---

## The data model: one dir per entity, one `.md` per record

The shape is uniform and deliberately repos-style — flat directories of
human-readable Markdown, each file a record, the filename stem the
primary key.

```
state/
├── club.md                  # the club PROFILE + cross-cutting DIMENSIONS (one file, not a dir)
├── members/
│   ├── _template.md         # the SHAPE every member record follows (Element-owned)
│   └── m-jordan-rivera.md   # one record per member (id = filename stem)
├── staff/
│   ├── _template.md
│   └── s-alex-mendez.md     # one record per staff member
├── teams/
│   ├── _template.md
│   └── u12-boys-fall-2026.md
├── rosters/
│   ├── _template.md
│   └── roster-u12-boys-fall-2026.md
├── tryouts/
│   ├── _template.md
│   └── tryout-u12-boys-2026-spring.md
└── …                        # phase-2 dirs (see below)
```

**The rules of the model:**

1. **One directory per entity type.** `members/`, `staff/`, `teams/`,
   `rosters/`, `tryouts/` — each holds all records of that kind. A skill
   that operates on an entity iterates its directory.

2. **One `.md` file per record.** Each file is a single member, staff
   member, team, roster, or try-out. There is no database, no index file
   — the *set of files in a directory* is the collection.

3. **The filename stem IS the `id` and primary key.**
   `members/m-jordan-rivera.md` has `id: m-jordan-rivera`. Ids are
   lowercase, hyphenated, stable, and never reused. Facts inside a record
   change; the id does not. (A team whose age_group/gender/season identity
   genuinely changed is a *new* record, not a renamed one.)

4. **Every record is created from its directory's `_template.md`.** The
   template is the canonical shape — YAML frontmatter (the machine fields
   the skills parse) plus prose sections. A skill registers a record by
   *copying the template* to `<id>.md` and filling it; you never register
   a record by editing the template itself.

5. **`club.md` is the odd one out** — a single config file, not a
   directory of records. It holds the club profile and the dimensions
   (`age_groups`, `genders`, `registration_systems`, `seasons`) every
   other record validates against.

### Two zones inside every record

Each record (and each `_template.md`) has two zones, marked in the file:

- **Machine zone — YAML frontmatter.** The structured fields the skills
  parse and own. Re-running a skill overwrites changed machine fields.
  The canonical field set per entity is fixed and consistent across the
  templates, the skills, and `.claude/club-ops-rules.md` — don't invent
  fields; if a club needs something the model doesn't carry, it goes in
  the human zone (and gets flagged for a possible rules update), not as a
  new frontmatter key.

- **Prose zones — readable sections.** Marked either
  *[orchestrator-owned]* (rewritten from the facts on each run) or
  *[human-authored — preserved]* (the club admin owns it; skills carry it
  forward verbatim, never overwriting it).

### Element-owned vs project-owned files

Two kinds of file live under `state/`, and they behave differently on
**upgrade**:

| File | Owner | On upgrade |
|---|---|---|
| `state/<entity>/_template.md` | **Element** | refreshed (the record shape can evolve) |
| `state/README.md` (this file) | **Element** | refreshed |
| `state/club.md` | **project** (the director) | **never overwritten** (seeded once via `seed/club.md.template`, policy `skip-if-exists`) |
| `state/<entity>/<id>.md` (real records) | **project** (the club's data) | **never touched** by upgrade — they're your data |

So: the *shapes* (`_template.md`, this README) come from the Element and
stay current with it; the *content* (`club.md` and every real record) is
the club's, written by the skills and the director, and upgrade-safe.

---

## Relational integrity

The entities form a graph, and the skills keep it consistent. The
governing convention (repos-style): **a change lands in every surface it
touches, or none.** Some load-bearing links:

- **`/roster`** adds a player to a team's roster AND appends that team to
  the member's `teams[]` — both sides or neither. It also keeps the
  team's `roster_id` pointing back.
- **`/member`** keeps `parents[] ⇄ players[]` symmetric: a parent listing
  a child means that child lists the parent.
- **`/team`** keeps coach assignment two-sided: a team's `coaches[]` and
  each coach's `teams[]` agree.
- **`/tryout`** produces roster assignments and hands them to `/roster`,
  which commits them — the tryout *proposes*, `/roster` *writes*, so the
  cross-entity write happens in one place.
- **`/status`** is read-only: it iterates every directory, rolls up
  counts, and reports the **gaps** (empty age_group×gender cells,
  unrostered players, coachless teams, under-staffed try-outs, broken
  references) — but repairs nothing. It routes each gap to the skill that
  owns the fix.

A link present on one side but not the other is **drift**; the owning
skill heals it on its next run.

The canonical field sets, the integrity rules, and the orchestrator ⇄
domain boundary are all specified in `.claude/club-ops-rules.md` — the
record templates here conform to it, and every skill reads it.

---

## Foundation vs phase 2

This build ships the club-operations **spine** — People and Org/Process —
and leaves a second tier of entities *defined but not operated*.

**Foundation (built this build):**

| Entity | Directory | Skill | What it is |
|---|---|---|---|
| Dimensions / profile | `club.md` | (config) | club identity + `age_groups` / `genders` / `registration_systems` / `seasons` |
| Member | `members/` | `/member` | a person — `player` or `parent` |
| Staff | `staff/` | `/staff` | a club worker — `coach` / `director` / `admin` / `board` |
| Team | `teams/` | `/team` | a fielded unit — age_group × gender × season + coaches + roster |
| Roster | `rosters/` | `/roster` | the players assigned to a team |
| Tryout | `tryouts/` | `/tryout` | a selection event that produces roster assignments |
| (roll-up) | — | `/status` | read-only dashboard + gap report across all of the above |

**Phase 2 (defined in the rules + given a `state/` directory, but no
skills authored this build):**

| Entity | Directory (phase 2) | Why it's deferred |
|---|---|---|
| League | `leagues/` | competition structure (level: club\|state\|regional\|national) |
| Tournament | `tournaments/` | discrete competitive events |
| Complex | `complexes/` | a venue — `fields[]`, parking, vending, address |
| Field | `fields/` | a playable field belonging to a Complex |
| Schedule | `schedules/` | practice / game fixtures (team × field × datetime) |
| Registration | `registrations/` | `member\|team × system × season` enrollment record |

**How foundation skills treat phase-2 entities:** they *reference* them
and **degrade gracefully**. A team records a `home_complex` id, a tryout
records a `location`, a member carries `registrations[]` strings — these
are stored as **intent** and noted as "(phase 2)". The foundation skill
does **not** try to validate them against a record that may not exist yet,
and never blocks an operation because a phase-2 entity isn't modeled. When
the phase-2 skills ship, they reconcile against what the foundation
recorded. Phase-2 directories may be absent or empty; nothing in the
foundation errors on that.

---

## How a record comes to life

The flow is the same for every entity:

1. A skill is invoked (e.g. `/member create a U12 player`).
2. It reads `state/club.md` to validate dimensions (`age_group`,
   `gender`, etc.) and reads any related records it must cross-reference.
3. It copies `state/<entity>/_template.md` → `state/<entity>/<id>.md`.
4. It fills the machine zone and the orchestrator-owned prose, leaving the
   human-authored zones for the admin.
5. It writes every other surface the change touches (back-references,
   mirrors) so integrity holds.
6. It stages the change (`git add`) for the human to review and commit —
   the skills don't auto-commit.

That's the model: a flat, legible, version-controlled tree of records the
club director can read directly, and the skills keep honest.

---

**See also:** `state/club.md` (the dimension catalog every record
validates against), `state/<entity>/_template.md` (the per-entity shapes),
`.claude/club-ops-rules.md` (the canonical entity model + integrity
convention), and the skills under `.claude/skills/` that CRUD these
records.
