<!--
  ROSTER RECORD TEMPLATE — rasa.orchestrator.soccer.club
  ======================================================

  This file is the SHAPE every `state/rosters/<id>.md` follows. It is
  Element-owned: it ships in `content/state/rosters/_template.md`, installs to
  `state/rosters/_template.md` in the club's umbrella repo, and is refreshed on
  upgrade. Do NOT build a roster by editing this file — copy it to
  `state/rosters/<id>.md` (that's what `/roster` does) and fill the copy.

  ONE FILE PER ROSTER. The filename stem IS the roster `id` and primary key
  (`r-u12-boys-navy-2026.md` → id `r-u12-boys-navy-2026`). The set of these
  files is the club's roster directory; /team, /roster, /tryout iterate it.

  A ROSTER is the membership LIST that binds players (members) to exactly one
  team. It is its own entity — separate from the Team — because membership has
  its own lifecycle (forming → set → locked) and is often produced by a tryout.
  Exactly one roster per team; the team's `roster_id` points here and this
  record's `team` field points back.

  THE LOAD-BEARING RULE
  ---------------------
  The orchestrator OWNS THE CLUB'S OPERATIONAL DATA (state/). This record is the
  source of truth for WHO is on the team. `/roster` is the SOLE writer of
  membership: every add/remove here MUST also update the affected member's
  `teams[]` in `state/members/<id>.md`. Membership integrity is the whole point
  of this entity — both surfaces move together or neither does.

  RELATIONAL INTEGRITY (a change lands in every surface it touches, or none)
  -------------------------------------------------------------------------
  - `team` ⇄ that team's `roster_id`. The pointer is paired both ways.
  - `players[]` here ⇄ each member's `teams[]` (each member is type: player).
    /roster propagates every add/remove to the member side in the same pass.
  - `from_tryout` (optional) points at the `state/tryouts/<id>.md` whose
    `outcomes` produced these assignments. /tryout sets it when it builds a
    roster from evaluation results.
  - `max_size` is a club rule (often by age group); /roster enforces it —
    `len(players) <= max_size`, and refuses to over-fill a `locked` roster.

  TWO ZONES, BELOW
  ----------------
  1. FRONTMATTER (YAML) — machine-readable fields the skills parse. `/roster`
     owns and overwrites these.
  2. PROSE SECTIONS — [orchestrator-owned] rewritten from facts;
     [human-authored — preserved] carried forward untouched.

  Replace every <PLACEHOLDER>. Delete this comment block in real records.
-->

---
# ─── machine-readable: parsed by the org/process skills (/roster, /team, /tryout) ───
id: <r-team-season>                   # PRIMARY KEY = filename stem. lowercase, hyphenated, unique. never reused.
team: <t-team-id>                     # the ONE team this roster belongs to. that team's roster_id == this id (paired both ways).
players: [<m-player-id>, ...]         # member ids (type: player) on this roster. mirrors each member's teams[]. /roster is sole writer.
from_tryout: <ty-tryout-id|>          # OPTIONAL: the tryout whose outcomes produced these assignments (state/tryouts/). blank if hand-built.
max_size: <18|22|...>                 # club cap on squad size (often by age group). /roster enforces len(players) <= max_size.
status: <forming|set|locked>          # forming = being assembled; set = squad chosen, still mutable; locked = frozen for the season.

# stamps — written by /roster, read for staleness
registered_at: "<YYYY-MM-DD>"         # first time this roster was created. set once, never overwritten.
updated_at: "<YYYY-MM-DD>"            # last /roster run that touched this file. bumps every update.
---

# <id> — roster for <team>

> Roster for team `<team>` · `<len(players)>` / `<max_size>` players · status **<status>**
> <from_tryout: built from tryout `<from_tryout>`> / <hand-built>

One-line essence: the squad bound to this team for the season. *(orchestrator-owned)*

---

## Membership *(orchestrator-owned — sole writer is `/roster`)*

The players on this roster. Each id is a `state/members/<id>.md` with
`type: player`. **This is the source of truth for the team's player list** — the
Team record does NOT duplicate it.

| # | Player | From tryout? | Mirror surface |
|---|---|---|---|
| 1 | `<m-player-id>` | <yes / no> | that member's `teams[]` contains `<team>` |
| 2 | `<m-player-id>` | <yes / no> | that member's `teams[]` contains `<team>` |

- **Count vs cap:** `<len(players)>` of `<max_size>`. `/roster` refuses to
  exceed `max_size` and refuses to mutate a `locked` roster.
- **Symmetry is load-bearing:** every player here MUST list `<team>` in their
  `teams[]`. `/roster` writes both sides in one pass; a player on the roster but
  not carrying the team (or vice versa) is drift `/roster` heals.

---

## Provenance *(orchestrator-owned)*

How this roster was assembled.

- **From tryout:** `<from_tryout>` → `state/tryouts/<from_tryout>.md` (or
  `none — hand-built`). When set, this roster's membership is the realized
  `outcomes` of that tryout; `/tryout` wrote the assignments here.
- **Status path:** `forming` → `set` → `locked`. `/roster` advances it; once
  `locked`, membership is frozen for the season and further changes require an
  explicit unlock.

---

## Notes *(human-authored — preserved across updates)*

Free-form prose the coach / club admin owns. `/roster` **never** overwrites this
— it carries it forward verbatim. Put here call-up policy, guest-player rules,
why the squad is short of `max_size`, injury holds, a waitlist of prospects not
yet added — anything not captured by a structured field above.

- <e.g. "Two slots held open for late tryout call-ups. Waitlist: m-sam-cole
   (pending registration). Locked after the first league fixture per club rule.">

---

<!--
  CONSISTENCY CONTRACT (for /roster, /team, /tryout)
  --------------------------------------------------
  After writing this file:
  - `team` MUST point at a state/teams/<team>.md whose roster_id == this id.
    Exactly one team ⇄ one roster, paired both ways.
  - Every id in players[] is a state/members/<id>.md (type: player) whose
    teams[] contains `team`. /roster propagates both surfaces every run; a
    one-sided membership is drift to heal.
  - len(players) <= max_size always. A status of `locked` means /roster rejects
    membership edits until an explicit unlock.
  - If from_tryout is set, that tryout exists in state/tryouts/ and its status is
    `complete` (a roster realizes a finished evaluation, not a scheduled one).
-->
