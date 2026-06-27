---
name: member
description: Register and manage MEMBERS of the club — the players and the parents. Create a player (name, age_group, gender, linked parent(s)), create a parent (linked to player(s)), update an existing member, look one up, list/filter the roster of members (by age_group, gender, team, status), link or unlink a player↔parent relationship, and deactivate a member. Writes the human record at state/members/<id>.md from the template and keeps every relationship it touches consistent on both sides. The orchestrator owns the member RECORD; a player's actual training/development toolkit is the role-domain (domain.soccer.player / .goalkeeping), not this skill. Triggered when the user wants to add or manage a person who is a player or parent — e.g. "/member register player", "add a new U10 girls player", "register Maria's mom as a parent", "link this player to that parent", "list all active U12 boys", "look up member m-0042", "deactivate this player".
---

# /member — register and manage players and parents

Creates and maintains the club's **member** records — the people who
are **players** and the **parents** linked to them. A member is the
canonical record of a person in the club's operational data; this
skill is the only sanctioned way to create, update, relate, and retire
those records. After it runs, the person exists in one authoritative
place every other club-ops skill reads from: a human-facing record at
`state/members/<id>.md` carrying the machine fields in its YAML
frontmatter. **If a person is not a member record, they do not exist
to the orchestrator** — they can't be put on a roster, evaluated at a
try-out, or rolled into a registration. `/member` is how they come to
exist.

Per the load-bearing Pattern-3 rule of this platform: **the
orchestrator owns the club's operational DATA; the role-domains give
each person their role TOOLKIT.** This skill writes and relates member
records in the club's umbrella repo. It does **not** coach, train,
evaluate, or develop anyone. A player's development work lives in
`rasa.domain.soccer.player` (and `rasa.domain.soccer.goalkeeping` for
keepers); a parent's role surface lives in `rasa.domain.soccer.parent`.
Those references are **soft** — the member record names the person and
their facts; the domain, if installed for that person, is where their
actual soccer work happens. Of the role-domains, only
`rasa.domain.soccer.goalkeeping` is live today; the rest are referenced
by name and degrade gracefully when absent. `/member` never reaches
into a role-domain and a role-domain never writes a member record.

This skill governs the **player** and **parent** member types only.
Coaches, directors, admins, and board members are **staff**, a
separate entity with its own record shape and lifecycle — use
[/staff](../staff/SKILL.md) for those. Putting a player on a team is
[/roster](../roster/SKILL.md)'s job; evaluating a player is
[/tryout](../tryout/SKILL.md)'s. `/member` owns the person; those
skills own the events that reference the person.

## Behavior contract

- **The member record is the source of truth for a person.** A player
  or parent is in the club if and only if they have a
  `state/members/<id>.md` record. This skill is the sole writer of that
  record. Every other skill that names a member — `/roster`, `/tryout`,
  `/team`, and the phase-2 registration/schedule skills — reads it;
  none of them invent members. They reference an `id` that this skill
  minted.

- **`id` is the primary key and it is immutable.** Each member gets a
  stable `id` (e.g. `m-0042`) at creation and keeps it for life. Names
  change, age groups roll over, status flips — the `id` never does.
  Other records (rosters, tryouts, teams, a parent's `players[]`, a
  player's `parents[]`) point at members by `id`, so re-keying a member
  would orphan every reference. Generate the `id` once; never reuse a
  retired one.

- **The member is one of exactly two types: `player` or `parent`.**
  The `type` field decides which sub-fields are load-bearing:
  - A **`player`** carries `age_group` and `gender` (the player's own
    facts), a `parents[]` list of linked parent member ids, a `teams[]`
    list of team ids they're rostered to, and a `registrations[]` list
    (phase 2). `age_group` + `gender` are **required** for a player —
    they drive every downstream operation (which try-out, which team,
    which league). A player with no age_group can't be rostered.
  - A **`parent`** carries a `players[]` list of linked player member
    ids and contact info. A parent does **not** carry `age_group`,
    `gender`, or `teams[]` — those are player facts. A parent's reason
    to exist is the link to one or more players; a parent linked to no
    players is allowed (e.g. a guardian registered ahead of their child)
    but flagged.

  Never put player-only fields on a parent or parent-only fields on a
  player. The type gates the shape.

- **Relationships are bidirectional and kept consistent on both sides
  — always.** A player↔parent link lives in two places: the player's
  `parents[]` and the parent's `players[]`. This skill **never** writes
  one side without the other. Linking player `m-0042` to parent `m-0101`
  adds `m-0101` to the player's `parents[]` **and** `m-0042` to the
  parent's `players[]`, in the same operation. Unlinking removes both.
  A one-sided link is drift; if the skill detects it on entry (a player
  names a parent who doesn't name them back, or vice versa), it
  reconciles rather than erroring and reports the heal. This is the
  club-ops integrity discipline: a relationship change lands in every
  surface it touches, or none of them does.

- **Gather, don't guess.** The fields below are required to write a
  complete record. For each fact you cannot derive, **ask** — one round
  trip beats a roster full of `TODO`. Never invent a name, an
  age_group, a gender, a contact, or a parent link. In particular,
  never guess `age_group` from a stated age without confirming the
  club's mapping — age groups are a club-defined dimension (see below),
  and the cutoffs are the club's to set, not yours to assume.

- **`age_group` and `gender` are club-defined dimensions, not free
  text.** The club's valid `age_groups` (e.g. `U6`…`U19`, `Adult`) and
  `genders` (`boys` | `girls` | `coed`) are the taxonomy declared in
  `state/club.md`. A player's `age_group` and `gender` MUST be drawn
  from those declared values. If the user gives a value that isn't in
  the club's taxonomy, stop and reconcile — either the value is wrong
  or the taxonomy needs extending (extending it is a `state/club.md`
  edit, out of this skill's scope; surface it, don't silently widen).

- **`status` tracks lifecycle, and deactivation is a status flip, not
  a delete.** A member is `prospect` (in the pipeline, e.g. a try-out
  candidate not yet placed), `active` (a current club member), or
  `inactive` (left, aged out, or withdrawn). Deactivating a member sets
  `status: inactive` — it never deletes the record, because rosters,
  try-outs, and registrations still reference that `id` historically.
  Hard-deleting a member orphans every reference; this skill does not
  do it.

- **The record is the orchestrator's; the toolkit is the domain's.**
  Member records live in `state/members/` in the umbrella repo. The
  rules this skill obeys live in `.claude/club-ops-rules.md`. The
  person's actual soccer work — a player's training plan, a keeper's
  positioning drills — lives in the role-domain installed for that
  person, never here. Keep the two straight: `state/` is operational
  data the orchestrator mutates; the role-domain is toolkit the person
  runs.

## The fields a member record carries

Every member record carries these in its YAML frontmatter. They are
the exact fields `state/members/_template.md` defines and the columns
the list/filter operation reads. Keep these names identical across
every skill and template in this Element.

| Field | Applies to | What it is | Example |
|---|---|---|---|
| **id** | both | Primary key = record filename. Immutable. | `m-0042` |
| **name** | both | The person's full name | `Maria Alvarez` |
| **type** | both | `player` \| `parent` | `player` |
| **age_group** | player | Club age dimension (from `state/club.md`) | `U10` |
| **gender** | player | Club gender dimension: `boys` \| `girls` \| `coed` | `girls` |
| **parents[]** | player | Linked parent member ids | `[m-0101]` |
| **players[]** | parent | Linked player member ids | `[m-0042, m-0043]` |
| **contact** | both | Email / phone / preferred channel | `maria.a@…, 555-0100` |
| **status** | both | `active` \| `inactive` \| `prospect` | `active` |
| **registrations[]** | both | Registration ids (phase 2 — see note) | `[]` |
| **teams[]** | player | Team ids the player is rostered to | `[t-u10-girls-a]` |

**Phase-2 note — `registrations[]`.** A member can be registered into
the club's own system and into external state / regional / national
systems; each is a **Registration** record. Registration is a
**phase-2** entity — its skill is **not** authored in this build. Carry
`registrations[]` as a (typically empty) field on every member record
so the shape is forward-compatible, but do not attempt to create or
manage registration records here. When `/registration` lands, it will
write those records and back-reference the member by `id`; until then,
leave the list empty and degrade gracefully.

**Cross-field rule.** `age_group`, `gender`, `teams[]`, and
`parents[]` are **player-only**. `players[]` is **parent-only**.
`id`, `name`, `type`, `contact`, `status`, `registrations[]` are
shared. The `type` field is what decides which set applies; honor it.

## Operations

### 1. Register a player

Create a new `player` member. Required up front: `name`, `age_group`
(from the club's declared `age_groups`), `gender` (from the club's
declared `genders`). Optionally link one or more existing parents.

1. **Gather + validate.** Collect `name`, `age_group`, `gender`,
   `contact` (if applicable for the player directly), and any
   parent(s) to link. Validate `age_group` and `gender` against
   `state/club.md`'s declared dimensions — reject (and surface)
   anything outside the taxonomy rather than silently accepting it.
2. **Mint the `id`.** Allocate the next free member id (e.g. `m-0042`).
   Never reuse a retired id.
3. **Write the record.** Copy `state/members/_template.md` to
   `state/members/<id>.md`, set `type: player`, and substitute the
   gathered facts. Set `status` — `prospect` if this player is entering
   via the try-out pipeline and not yet placed, `active` if they're a
   confirmed current member. Leave `teams[]` and `registrations[]`
   empty (those are populated by [/roster](../roster/SKILL.md) and the
   phase-2 registration skill respectively).
4. **Link parents, both sides.** For each parent to link, run the
   link operation (op 6): add the parent id to this player's `parents[]`
   **and** this player's id to that parent's `players[]`. If a named
   parent doesn't exist yet, register them first (op 2) — don't write a
   dangling link.
5. **Note the domain seam (soft).** The record names the player; their
   training toolkit is `rasa.domain.soccer.player` (or `.goalkeeping`).
   Do not install or invoke it here — just record the person. Mention
   in the report that the player's development surface is the
   role-domain, available if/when installed for them.

### 2. Register a parent

Create a new `parent` member, linked to one or more players.

1. **Gather.** Collect `name`, `contact`, and the player(s) this parent
   is responsible for. A parent's defining fact is the link to a
   player; if no player is known yet, the parent may be registered
   ahead of their child but the report flags the empty `players[]`.
2. **Mint the `id`** and **write the record** — copy the template, set
   `type: parent`, substitute facts, set `status` (`active` for a
   current guardian, `prospect` if entering with a not-yet-placed
   player). Do **not** set `age_group`, `gender`, `teams[]`, or
   `registrations[]`-as-player on a parent; `players[]` is the parent's
   relationship field.
3. **Link players, both sides.** For each player, run the link
   operation (op 6): add the player id to this parent's `players[]`
   **and** this parent's id to that player's `parents[]`. Register the
   player first (op 1) if they don't exist.

### 3. Update a member

Patch an existing member's facts. Resolve the member by `id` (or by an
unambiguous name lookup — see op 4). Overwrite only the fields the user
changed; carry every other field forward unchanged. Updating is a
patch, not a reset.

- `name`, `contact`, `status` are freely updatable on either type.
- `age_group`, `gender` are updatable on a **player** only, and only
  to values in the club's taxonomy. An age-group roll-over (e.g.
  `U10` → `U11` at season turn) is an update, not a new record — the
  `id` stays.
- **Relationship fields (`parents[]` / `players[]`) are NOT edited
  directly here.** Changing who a player's parents are goes through the
  link/unlink operation (op 6) so both sides stay consistent. Hand-
  editing one side's list is exactly the drift this skill exists to
  prevent — route relationship changes through op 6.

### 4. Look up a member

Find and read one member's record. Resolve by `id` (exact) or by name
(may match more than one — if ambiguous, list the candidates with their
`id`, `type`, `age_group`/`gender`, and `status`, and ask which). Return
the record's facts plus its relationships resolved to names: for a
player, the linked parents by name; for a parent, the linked players by
name + their age_group/gender. Read-only — a lookup never mutates.

### 5. List / filter members

Iterate `state/members/` and return the members matching the requested
filter. Supported filters, combinable:

- **by `age_group`** — e.g. all `U12` (players only; parents have no
  age_group and are excluded from an age_group filter unless explicitly
  asked).
- **by `gender`** — `boys` | `girls` | `coed` (players only).
- **by `team`** — all players whose `teams[]` contains a given team id
  (players only).
- **by `status`** — `active` | `inactive` | `prospect` (either type).
- **by `type`** — `player` | `parent`.

Render a scannable table: `id`, `name`, `type`, `age_group`/`gender`
(for players), `status`, and a relationship count (parents linked /
players linked). Default to excluding `inactive` members unless the
filter explicitly asks for them — a roster scan usually wants the
current club, not its alumni. State the active filter and the count in
the output so the result is unambiguous.

### 6. Link / unlink player↔parent

The relationship operation, and the **only** sanctioned way to change a
player↔parent link. Always writes **both** sides.

- **Link.** Given a player id and a parent id (creating either via op 1
  / op 2 first if missing): add the parent id to the player's
  `parents[]` and the player id to the parent's `players[]`. Idempotent
  — if the link already exists on both sides, it's a no-op; if it exists
  on one side only, reconcile to both.
- **Unlink.** Remove the parent id from the player's `parents[]` and the
  player id from the parent's `players[]`. Removing the last parent from
  a player, or the last player from a parent, is allowed but flagged in
  the report (a player with no linked parent / a parent linked to no
  player is a real but notable state).

After any link/unlink, re-read both records and confirm the two lists
agree. Report the change as a pair ("linked `m-0042` ↔ `m-0101` on both
records").

### 7. Deactivate a member

Retire a member without deleting them. Set `status: inactive` on the
record. **Do not delete the file** — historical rosters, try-outs, and
registrations still reference the `id`. Leave existing relationships and
`teams[]` in place as historical record unless the user explicitly asks
to unlink; deactivation records that the person left, it doesn't rewrite
history. Reactivation is the reverse: an update setting `status` back to
`active`. Report what the member was referenced by at deactivation time
(which teams, which registrations) so the user sees the historical
footprint they're leaving behind.

## When NOT to use

- **To coach, train, evaluate, or develop a player** → that's the
  role-domain (`rasa.domain.soccer.player` / `.goalkeeping`), not this
  skill. `/member` records that the person exists and their facts; it
  never does their soccer work. Reaching into a role-domain to "improve
  a player" violates the owns-the-data / owns-the-toolkit split.
- **To create a coach, director, admin, or board member** → those are
  **staff**, a different entity with a different record shape. Use
  [/staff](../staff/SKILL.md). `/member` is players and parents only.
- **To put a player on a team** → that's [/roster](../roster/SKILL.md),
  which updates the team's `roster_id` and the player's `teams[]`
  together. `/member` owns the person; `/roster` owns the placement.
  (`/member` only ever reads `teams[]`; it's written by `/roster`.)
- **To evaluate a player at a try-out** → that's
  [/tryout](../tryout/SKILL.md), which produces roster assignments.
  `/member` registers a candidate (often as a `prospect`); `/tryout`
  runs the evaluation.
- **To register a member into a club/state/regional/national system**
  → that's the phase-2 **registration** entity; its skill is not in
  this build. Leave `registrations[]` empty for now.
- **To hard-delete a person** → deactivation (op 7) is the supported
  retirement. Deleting a record orphans every roster/tryout/registration
  reference to its `id`; this skill does not do it.
- **To define or extend the club's age_groups / genders** → those
  dimensions live in `state/club.md`. Changing them is a club-config
  edit, not a member operation. `/member` consumes the taxonomy; it
  doesn't author it.

## What "done" looks like

The person is a member: `state/members/<id>.md` exists with a complete,
type-correct record (a player carries `age_group` + `gender`; a parent
carries `players[]`; neither carries the other's fields), and every
player↔parent link it touched is present and identical on **both**
records. The `id` is stable and unique. List/filter sees the member when
it iterates `state/members/`. Re-running an operation on the same member
updates that one record in place and never produces a duplicate.
Deactivation leaves the record intact with `status: inactive`. From this
point on, [/roster](../roster/SKILL.md), [/tryout](../tryout/SKILL.md),
[/team](../team/SKILL.md), and the phase-2 registration/schedule skills
can reference the member by `id` — which is the whole point: the member
record is the act that makes a person part of the club's operational
data, and the player's actual soccer development lives, separately and
softly, in their role-domain.
