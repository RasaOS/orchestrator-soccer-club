# CHANGELOG — `rasa.orchestrator.soccer.club`

Reverse-chronological. Each entry is a version bump.

---

## 0.1.0 — 2026-06-25 — INITIAL (foundation)

**The first sub-namespaced orchestrator** and a **Pattern 3 (Hybrid /
CTO-style) club-operations platform** — the soccer analog of
`orchestrator.code`. Manages a soccer club's whole operation across 13
concerns. Forked from `orchestrator-core` v1.1.0.

### The model

- Installs into a **club's umbrella repo**; toolkit (rules + skills) →
  `.claude/`, the **operational data** (a relational model of 13 concerns) →
  `state/`.
- **Two truths:** a fact about the club's *organization* → the orchestrator
  (`state/`); the work of a *role* → a `domain.soccer.*` toolkit. The
  orchestrator manages people/units/events; the role domains are the
  playbooks. Soft references (no `requires.elements[]`).

### The 13 concerns

People (members: players/parents · staff: coaches/directors/admins/board) ·
Org units (teams, leagues, tournaments) · Processes (try-outs→rosters,
registrations) · Facilities (schedules, fields, complexes) · Dimensions
(age-groups, genders, registration-systems).

### v0.1.0 = foundation (People + Org + Process)

- **`content/club-ops-rules.md`** — the entity-model spine: the canonical
  field sets for every entity, the relationships (parent↔player, member→team
  via roster, roster←tryout, team→league/tournament/complex, schedule =
  team×field×time, registration = entity×system×season), the invariants, and
  the foundation-vs-phase-2 split.
- **6 skills:** `/member`, `/staff`, `/team`, `/roster`, `/tryout`, `/status`.
- **5 entity templates** (`state/{members,staff,teams,rosters,tryouts}/_template.md`,
  Element-owned) + the `state/README.md` data-model doc.
- **`state/club.md`** (project-owned seed) — the club profile + the dimensions
  (age-groups, genders, registration-systems) every skill resolves.
- **Phase-2 concerns DEFINED in the model + `state/` dirs scaffolded**
  (`leagues/`, `tournaments/`, `fields/`, `complexes/`, `schedules/`,
  `registrations/`); their skills + templates land in v0.2.0.

### Shape

`rasa.shape_pattern: hybrid`; capabilities `club.{member,staff,team,roster,tryout}-management`
+ `club.operations-rollup`. Install posture YES. Removed the fork's
`content/SHAPE.md` (→ `content/README.md`) + the empty `rules/agents` scaffold.

### Verification

`bin/check-manifest` GREEN; `schema/bin/validate` OK; cross-refs resolve;
`bin/init` smoke-tested → installs the toolkit to `.claude/`, the entity
templates + data-model doc + the 13 entity dirs to `state/`.

### Build method

`bin/new-element` scaffold (dotted name hand-set) → 9-agent parallel authoring
workflow grounded in the entity model + a shared format → orchestrator-authored
README/CLAUDE/manifest + integration gate.

### Role family + sibling

Sits over `domain.soccer.{player,coach,parent,director,board,team,league}`
(scaffolded v0.1.0 shells this session) + the content-bearing
`domain.soccer.goalkeeping`. Engineering Pattern-3 sibling: `orchestrator.code`.
