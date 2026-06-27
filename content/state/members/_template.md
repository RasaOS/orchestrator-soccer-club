<!--
  MEMBER RECORD TEMPLATE — rasa.orchestrator.soccer.club
  ======================================================

  This file is the SHAPE every `state/members/<id>.md` follows. It is
  Element-owned: it ships in `content/state/members/_template.md`, installs to
  `state/members/_template.md` in the club's umbrella repo, and is refreshed on
  upgrade. Do NOT register a member by editing this file — copy it to
  `state/members/<id>.md` (that's what `/member` does) and fill the copy.

  ONE FILE PER MEMBER. The filename stem IS the member's `id` and primary key
  (`m-jordan-rivera.md` → id `m-jordan-rivera`). The roster of all members is
  the set of these files; every people skill iterates this directory.

  A MEMBER is a PERSON in the club's orbit: a `player` or a `parent`. The two
  link to each other — a player record lists its `parents[]`, a parent record
  lists its `players[]`. The orchestrator keeps that link symmetric.

  THE LOAD-BEARING RULE
  ---------------------
  The orchestrator OWNS THE CLUB'S OPERATIONAL DATA (state/). This record is
  the source of truth for WHO this person is and HOW they relate to teams,
  rosters, try-outs, and registrations. It is NOT the place a player's TRAINING
  or a coach's COACHING lives — those are the role-DOMAINS
  (`domain.soccer.{player,parent,...}` + live `domain.soccer.goalkeeping`),
  referenced SOFTLY: name the domain if the person has that role, degrade
  gracefully if the domain Element is absent (only goalkeeping is live so far).

  RELATIONAL INTEGRITY (a change lands in every surface it touches, or none)
  -------------------------------------------------------------------------
  - `teams[]` here MUST match each named Team's roster membership. `/roster`
    is the skill that writes both sides — it adds a player to a `state/rosters/`
    record AND appends that team to this member's `teams[]`. Never edit one
    side alone.
  - `parents[] ⇄ players[]` is symmetric. Adding parent P to player C's
    `parents[]` MUST add C to P's `players[]`. `/member` writes both.
  - `registrations[]` here are references; the Registration entities themselves
    are a PHASE-2 concern (`state/registrations/`, not authored this build).
    Until then this is a soft list of system+season strings.

  TWO ZONES, BELOW
  ----------------
  1. FRONTMATTER (YAML) — machine-readable fields the skills parse. `/member`
     owns and overwrites these.
  2. PROSE SECTIONS — human-readable. [orchestrator-owned] sections are
     rewritten from the facts; [human-authored — preserved] sections the club
     admin owns and skills carry forward untouched.

  Replace every <PLACEHOLDER>. Delete this comment block in real records.
-->

---
# ─── machine-readable: parsed by the people skills (/member, /roster, /tryout) ───
id: <m-firstname-lastname>            # PRIMARY KEY = filename stem. lowercase, hyphenated, unique. never reused.
name: "<Full Name>"                   # human display name.
type: <player|parent>                 # the kind of member. drives which player-only / parent-only fields apply.

# ── player-only fields (present when type: player; omit/blank for parents) ──
age_group: "<U6..U19|Adult|>"         # taxonomy value from state/club.md#age_groups. blank for parents.
gender: "<boys|girls|coed|>"          # taxonomy value from state/club.md#genders. blank for parents.
parents: [<m-parent-id>, ...]         # member ids of this player's parents. symmetric with each parent's players[].
teams: [<t-team-id>, ...]             # team ids this player is rostered on. mirrors each team's roster (via /roster).

# ── parent-only field (present when type: parent; omit/blank for players) ──
players: [<m-player-id>, ...]         # member ids of this parent's children. symmetric with each player's parents[].

# ── shared fields (both types) ──
contact:                              # how to reach this person.
  email: "<name@example.com|>"
  phone: "<+1-555-0100|>"
status: <active|inactive|prospect>    # active = current member; inactive = lapsed; prospect = not yet enrolled (e.g. a tryout candidate).
registrations: [<club-2026|state-2026|...>, ...]   # SOFT refs: <system>-<season>. Registration entity is PHASE 2.

# stamps — written by /member, read for staleness
registered_at: "<YYYY-MM-DD>"         # first time this member was added. set once, never overwritten.
updated_at: "<YYYY-MM-DD>"            # last /member run that touched this file. bumps every update.
---

# <name> — <type>

> **<Full Name>** · `<type>` · status **<status>**
> <player: <age_group> <gender> · rostered on <N> team(s)> / <parent: <N> player(s)>

One-line essence of who this person is in the club. *(orchestrator-owned)*

---

## Relationships *(orchestrator-owned — rewritten by `/member` / `/roster`)*

The family + team graph this member sits in. Every line is a **live link** the
orchestrator keeps symmetric; the skills read it to traverse the club.

**If `type: player`:**

| Link | Value | Mirror surface |
|---|---|---|
| **Parents** | `<parents[]>` | each parent's `players[]` |
| **Teams** | `<teams[]>` | each team's roster (`state/rosters/<roster_id>.md`) |
| **Age group / gender** | `<age_group>` / `<gender>` | `state/club.md` taxonomy |

**If `type: parent`:**

| Link | Value | Mirror surface |
|---|---|---|
| **Players** | `<players[]>` | each player's `parents[]` |

- Symmetry is load-bearing: if this section names a counterpart, that
  counterpart's record MUST name this member back. `/member` writes both sides.

---

## Registrations *(orchestrator-owned — soft, PHASE 2)*

Which systems this member is registered in, per season. **SOFT references** —
the Registration entity (`member|team × system × season`) is a PHASE-2 concern
not authored this build. Until then, record `<system>-<season>` strings drawn
from `state/club.md#registration_systems` (the club's own + state / regional /
national orgs).

- `<club-2026>` — <status / notes, or `—`>
- `<state-2026>` — <status / notes, or `—`>

Write `none` rather than deleting the section.

---

## Role domain *(orchestrator-owned — SOFT reference)*

The toolkit this person's ROLE pulls from. The orchestrator manages the
person; it does NOT do their training or coaching — that's the role-DOMAIN.

| If type | Soft domain reference |
|---|---|
| `player` | `rasa.domain.soccer.player` · plus `rasa.domain.soccer.goalkeeping` if a keeper (**the only live role-domain today**) |
| `parent` | `rasa.domain.soccer.parent` |

- **Degrade gracefully.** If the named domain Element is absent, this record
  still stands on its own — the reference is a pointer, not a dependency.

---

## Notes *(human-authored — preserved across updates)*

Free-form prose the club admin owns. `/member` **never** overwrites this — it
carries it forward verbatim on every update. Put here anything not captured by
a structured field above: medical flags, scholarship status, sibling context,
communication preferences, history with the club, why a prospect hasn't
enrolled yet.

- <e.g. "Returning U12 player; sat out '25 for injury. Mother (m-dana-rivera)
   is the primary contact and a team-parent volunteer.">

---

<!--
  CONSISTENCY CONTRACT (for /member and /roster)
  ----------------------------------------------
  After writing this file:
  - Every id in `parents[]` MUST be a parent member whose `players[]` contains
    THIS id. Every id in `players[]` MUST be a player member whose `parents[]`
    contains THIS id. Symmetry both ways or neither.
  - Every id in a player's `teams[]` MUST appear in that team's roster
    (`state/rosters/<team.roster_id>.md#players[]`). /roster owns both surfaces.
  - `status: prospect` is the expected state for an unenrolled tryout
    candidate; /tryout flips it toward `active` on a roster assignment.
  A link present on one side but not the other is drift — /member heals it.
-->
