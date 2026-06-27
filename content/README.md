# `rasa.orchestrator.soccer.club` — content

The **first sub-namespaced orchestrator** and a **Pattern 3 (Hybrid / CTO-style)
club-operations platform** — the soccer analog of `orchestrator.code`. It
installs into a **club's umbrella repo** and runs the whole organization:
people, teams, events, facilities, registrations.

> **Two truths.** *Is this a fact about the club's organization* (a member, a
> team, a roster, a tryout)? → the orchestrator owns it, in `state/`. *Or is
> it the work of a role* (how a coach coaches, how a keeper trains)? → that's
> a `domain.soccer.*` role toolkit. The orchestrator manages people and units;
> the domains are their playbooks.

## What it manages (the 13 concerns, as one relational model)

| Group | Concerns | `state/` |
|---|---|---|
| **People** | Members (players, parents) · Staff (coaches, directors, admins, board) | `members/`, `staff/` |
| **Org units** | Teams · Leagues · Tournaments | `teams/`, `leagues/`, `tournaments/` |
| **Processes** | Try-outs → Rosters · Registrations (club/state/regional/national) | `tryouts/`, `rosters/`, `registrations/` |
| **Facilities** | Practice/game schedules · Fields · Complexes (parking, vending) | `schedules/`, `fields/`, `complexes/` |
| **Dimensions** | Age groups · Genders · Registration systems | `club.md` |

The full entity model — every field set, relationship, and invariant — is in
[`club-ops-rules.md`](club-ops-rules.md) and [`state/README.md`](state/README.md).

## v0.1.0 = the foundation (People + Org + Process)

Foundation skills + the entity model. Phase-2 concerns are **defined in the
model and have `state/` dirs**, but their skills land later.

| Skill | Does |
|---|---|
| **`/member`** | Register + manage players and parents (age-group, gender, parent↔player links). |
| **`/staff`** | Register + manage coaches, directors, admins, board; assign coaches to teams. |
| **`/team`** | Create + manage teams (age-group × gender × season; coaches; home complex). |
| **`/roster`** | Manage a team's roster (eligibility, max-size; keeps team + member surfaces consistent). |
| **`/tryout`** | Run a try-out end-to-end; outcomes feed `/roster`. |
| **`/status`** | The club-ops dashboard — counts by age-group × gender, gaps, what needs attention. |
| *phase 2* | `/field` · `/complex` · `/schedule` · `/tournament` · `/league` · `/registration` |

## What installs where

| Source | Installs to | Policy | Owner |
|---|---|---|---|
| `content/club-ops-rules.md` | `.claude/club-ops-rules.md` | file-replace | Element |
| `content/skills/` | `.claude/skills/` | directory-mirror | Element |
| `content/state/README.md` | `state/README.md` | file-replace | Element |
| `content/state/<entity>/_template.md` | `state/<entity>/_template.md` | file-replace | Element |
| `seed/club.md.template` | `state/club.md` | skip-if-exists | **Project** (the dimensions) |
| `seed/state/<phase2>/.gitkeep` | `state/<phase2>/` | skip-if-exists | Project |

**Records are created by the skills, not seeded** — `state/<entity>/<id>.md`
from the entity's `_template.md`. The dirs exist from install so the full
13-concern model is present day one.

## The role family it sits over (soft references)

`domain.soccer.{player, coach, parent, director, board, team, league}` (+ the
content-bearing `goalkeeping`). A coach managed here as **staff** uses
`domain.soccer.coach`; a player managed as a **member** uses
`domain.soccer.player` / `.goalkeeping`. All **soft** — no `requires.elements[]`;
the club platform stands alone, the domains fill in per role.

## See also

- `~/rAI/rasa-os/elements/orchestrator-code/` — the engineering Pattern-3 sibling (same shape, different vertical).
- `~/rAI/rasa-os/elements/orchestrator-core/content/SHAPE.md` — Pattern 3.
