<!--
  STAFF RECORD TEMPLATE — rasa.orchestrator.soccer.club
  =====================================================

  This file is the SHAPE every `state/staff/<id>.md` follows. It is
  Element-owned: it ships in `content/state/staff/_template.md`, installs to
  `state/staff/_template.md` in the club's umbrella repo, and is refreshed on
  upgrade. Do NOT register a staff member by editing this file — copy it to
  `state/staff/<id>.md` (that's what `/staff` does) and fill the copy.

  ONE FILE PER STAFF MEMBER. The filename stem IS the staff `id` and primary key
  (`s-alex-mendez.md` → id `s-alex-mendez`). The set of these files is the
  club's staff directory; every people/team skill iterates it.

  A STAFF member is a club WORKER, distinct from a Member (player/parent):
  a `coach`, `director`, `admin`, or `board` member. Coaches link to the Teams
  they run; that link is symmetric with each Team's `coaches[]`.

  THE LOAD-BEARING RULE
  ---------------------
  The orchestrator OWNS THE CLUB'S OPERATIONAL DATA (state/). This record is the
  source of truth for WHO this staff person is, what they're certified for, and
  which units they run. It is NOT where a coach's COACHING method lives — that's
  the role-DOMAIN (`domain.soccer.{coach,director,board}` + live
  `domain.soccer.goalkeeping` for a GK coach), referenced SOFTLY: name the
  domain, degrade gracefully if absent.

  RELATIONAL INTEGRITY (a change lands in every surface it touches, or none)
  -------------------------------------------------------------------------
  - `teams[]` here (coaches only) MUST match each named Team's `coaches[]`.
    `/team` and `/staff` write both sides — assigning a coach to a team adds the
    team to this `teams[]` AND adds this id to that team's `coaches[]`.
  - `certifications[]` is local to this record (no mirror), but `/tryout` reads
    it when picking `evaluators[]` and `/team` may gate coach assignment on it.

  TWO ZONES, BELOW
  ----------------
  1. FRONTMATTER (YAML) — machine-readable fields the skills parse. `/staff`
     owns and overwrites these.
  2. PROSE SECTIONS — [orchestrator-owned] rewritten from facts;
     [human-authored — preserved] carried forward untouched.

  Replace every <PLACEHOLDER>. Delete this comment block in real records.
-->

---
# ─── machine-readable: parsed by the people/team skills (/staff, /team, /tryout) ───
id: <s-firstname-lastname>            # PRIMARY KEY = filename stem. lowercase, hyphenated, unique. never reused.
name: "<Full Name>"                   # human display name.
role: <coach|director|admin|board>    # the staff function. coach = runs teams; director/admin/board = club operations/governance.
teams: [<t-team-id>, ...]             # team ids this person coaches. coaches only — mirrors each team's coaches[]. [] for non-coaches.
certifications: [<USSF-D|SafeSport|CPR|...>, ...]   # licenses/clearances this person holds. read by /tryout (evaluators) + /team (gating).
contact:                              # how to reach this person.
  email: "<name@example.com|>"
  phone: "<+1-555-0100|>"
status: <active|inactive>             # active = current staff; inactive = departed / on leave.

# stamps — written by /staff, read for staleness
registered_at: "<YYYY-MM-DD>"         # first time this staff member was added. set once, never overwritten.
updated_at: "<YYYY-MM-DD>"            # last /staff run that touched this file. bumps every update.
---

# <name> — <role>

> **<Full Name>** · `<role>` · status **<status>**
> <coach: runs <N> team(s)> / <director|admin|board: club operations>

One-line essence of this person's job in the club. *(orchestrator-owned)*

---

## Assignments *(orchestrator-owned — rewritten by `/staff` / `/team`)*

The units this staff person runs. **Coaches only**; for director / admin /
board this is empty (their scope is the whole club, not a team).

| Team | Age group · gender | Mirror surface |
|---|---|---|
| `<t-team-id>` | `<age_group>` · `<gender>` | that team's `coaches[]` (`state/teams/<id>.md`) |

- Symmetry is load-bearing: every team in this `teams[]` MUST list this `id` in
  its `coaches[]`. `/team` and `/staff` write both sides.
- Non-coaches: write `none — club-wide role` rather than deleting the section.

---

## Certifications *(orchestrator-owned — mirrors frontmatter)*

Licenses + clearances on file. `/tryout` reads this when assembling
`evaluators[]`; `/team` may gate a coach assignment on a required cert.

| Certification | Status / expiry |
|---|---|
| `<USSF-D>` | <valid | expires YYYY-MM-DD | `—`> |
| `<SafeSport>` | <valid | expires YYYY-MM-DD | `—`> |

- Record only certs actually held. A missing required cert is a data point
  `/team` and `/tryout` can act on — don't fabricate one to fill the row.

---

## Role domain *(orchestrator-owned — SOFT reference)*

The toolkit this staff member's ROLE pulls from. The orchestrator schedules and
assigns the person; it does NOT do their coaching or directing — that's the
role-DOMAIN.

| Role | Soft domain reference |
|---|---|
| `coach` | `rasa.domain.soccer.coach` · plus `rasa.domain.soccer.goalkeeping` for a GK coach (**the only live role-domain today**) |
| `director` | `rasa.domain.soccer.director` |
| `board` | `rasa.domain.soccer.board` |
| `admin` | (operational; no dedicated role-domain — served by this orchestrator directly) |

- **Degrade gracefully.** If the named domain Element is absent, this record
  still stands alone — the reference is a pointer, not a dependency.

---

## Notes *(human-authored — preserved across updates)*

Free-form prose the club admin owns. `/staff` **never** overwrites this — it
carries it forward verbatim. Put here background, tenure, licensing pathway in
progress, board term dates, conflicts of interest, why a coach is between teams
this season — anything not captured by a structured field above.

- <e.g. "Technical director since '23; holds USSF B, pursuing A. Also the
   goalkeeping lead — evaluates keepers across all age groups at tryouts.">

---

<!--
  CONSISTENCY CONTRACT (for /staff and /team)
  -------------------------------------------
  After writing this file:
  - Every id in a coach's `teams[]` MUST appear in that team's `coaches[]`
    (`state/teams/<id>.md`), and vice versa. /team and /staff own both surfaces;
    a coach on one side but not the other is drift to heal on next run.
  - Non-coach roles (director/admin/board) carry `teams: []` — a non-empty
    teams[] on a non-coach is a flag /staff surfaces.
  - certifications[] has no mirror but is READ by /tryout (evaluator eligibility)
    and /team (assignment gating); keep it current so those gates are honest.
-->
