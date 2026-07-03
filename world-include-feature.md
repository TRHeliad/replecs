# Feature: per-world "include" filter (world mask)

A server-level allowlist of members that gates **all** replication from a world, applied
**on top of** existing per-entity / per-component filters. When deciding whether a player
receives any replicated data, their membership in the world include set is now an additional
`AND` term.

- **`nil` / unset = allow-all** (default; zero overhead, behaviour identical to before).
- **A set = restrict** to those members. **An empty / emptied set = deny-all.**
- The filter is **orthogonal to readiness** (`mark_player_ready` / `is_player_ready`): a member
  must be both *ready* **and** *included* to receive anything.
- Naming uses **"include"** (consistent with the existing `create_include_generator` /
  `is_include_filter`), not "whitelist".

---

## How replication targeting works (context)

replecs (jecs-based Roblox replication) buckets every networked entity/component into a
`StorageGroup` keyed by the hash of its recipient bitmask. Players are bit indices
(`client_indexes`). The send loops (`get_full`, `collect_updates`, `collect_unreliable`,
`collect_entity` in `src/server.luau`) iterate `masking.storages` and ship each group's buffer
to `storage.mask.members` / gate on `storage.mask.bitmask`. There is **no per-send re-filtering** —
the storage mask *is* the final audience.

Every recipient mask is ultimately derived as `something AND <floor generator>`:

- no-filter entity → `follow(<floor>)`
- include filter → `filterBits AND <floor>`
- exclude filter → `<floor> AND NOT filterBits`
- component → `entityMask AND componentMask`

Previously the floor was `active_members_generator` (`= all_members AND NOT unactive`, i.e. ready
players). This feature inserts the world include mask into that floor.

## Design chosen: dedicated `replicable_members_generator`

A new derived generator sits **below** `active_members`:

```
all ─┐
     ├─► active_members  ───────────────┐   (readiness; member_is_active reads THIS, untouched)
unactive ─┘                             │
   include mask (replicable.bitmask) ───┴─► replicable_members   ◄── NEW
                                              = include_enabled ? (active AND include) : active
                                                       │
                          include / exclude / follow derivations band against THIS
```

The three derivation points were re-pointed from `active_members_generator` to
`replicable_members_generator`. Because the mask-generator graph already cascades
`compute()` through `tracked_by`, every existing `active_members:compute()` (register / activate /
unregister) automatically flows into `replicable_members` and onward — **no lifecycle edits were
needed**, and the send loops were **not touched** (storage masks are already narrowed).

When disabled, `replicable_members.result` is a transparent **alias** of `active_members.result`,
so behaviour and storage bucketing are byte-for-byte identical to before the change.

**Why this approach:** zero steady-state send cost (the include is baked into storage masks, so
send work scales with *included* players, not all registered players — important because every
world/server registers every player). Cost is paid only when the include set changes, which is the
same re-bucketing the engine already does on `mark_player_ready`. Keeping a separate generator
(rather than folding into `active_members`) preserves readiness/inclusion orthogonality.

---

## Changes

### `src/masking/init.luau`
- **Type** `MaskingControllerClass`: added `include_enabled: boolean` and
  `replicable_members_generator: MaskGeneratorWithBitmask`.
- **`create_replicable_generator`** (~`:1642`): new derived generator. Holds the include bits in
  its own `.bitmask`; compute returns `active.result` when disabled, else
  `active.result:band(self.bitmask)`. (`band` reads out-of-range words as 0, so a narrower include
  mask correctly excludes higher indices — no `expand` needed, unlike the exclude `bnot` path.)
- **`set_world_include(masking, given_filter)`** (~`:1662`): `nil` → `include_enabled=false` +
  clear bits; a set → `include_enabled=true` + rebuild the include bitmask from the (normalized)
  filter; then `replicable:compute()` to cascade storage moves. Normalizes via the existing
  file-local `get_usable_filter`.
- **`add_world_include(masking, member)`** (~`:1687`) / **`remove_world_include(masking, member)`**
  (~`:1696`): incremental set/clear of one member's bit + `compute()`. `add` auto-registers a valid
  member (via `resolve_member_index`); `remove` resolves without registering (no-op if unknown).
- **`resolve_member_index`** (~`:1408`): extracted the per-member alias/auto-register resolution
  from `bitmask_from_set` so the include methods reuse it (behaviour-preserving refactor).
- **Re-pointed derivations** to `replicable_members_generator`:
  `create_include_generator` (~`:1601`), `create_exclude_generator` (~`:1581`, incl. its
  `expand` entry-count guard), and the no-filter `follow(...)` in `update_entity_filter`.
- **`compact_members`**: remap `replicable.bitmask` and `replicable.result` alongside the other
  roots (when disabled, re-alias `replicable.result` to the freshly-remapped `active.result`).
  **Mandatory** — without it the include bits desync from `client_indexes` past the 100-member
  compaction threshold.
- **`unregister_client`** (~`:1736`): clear the leaving member's include bit (hygiene).
- **`create()`** (~`:1784`): init `include_enabled=false` and the new generator (after
  `active_members_generator`).

### `src/server.luau`
- **`ServerImp` type** (~`:72`): added `set_world_include`, `add_to_world_include`,
  `remove_from_world_include`.
- **Implementations** (~`:2150`, near `add_player_alias`): thin pass-throughs to the masking
  controller methods.

---

## Public API

```lua
local server = replecs.create_server(world)

-- initial participants at world creation
server:set_world_include({ [player1] = true, [player2] = true })

-- incremental (drive these off participant/spectator join & leave events)
server:add_to_world_include(spectator)
server:remove_from_world_include(player)

-- disable -> allow-all again
server:set_world_include(nil)
```

`filter` is the existing `AnyFilter` (`Player | { [Player]: boolean }`); aliases
(`add_player_alias`) resolve as in normal filters. Effective audience for any datum =
`entityFilter AND componentFilter AND active(ready) AND include`.

---

## Edge cases handled
- **Player joins while a set is active** → not auto-included (correct allowlist semantics); call
  `add_to_world_include` on their join event.
- **Compaction (>100 members)** → include bitmask remapped in `compact_members`.
- **Player leaves** → include bit cleared in `unregister_client`.
- **Disabled** → `replicable` aliases `active`; identical to pre-change behaviour.
- **Orthogonality** → `member_is_active` / `is_player_ready` read `active_members`, unchanged.

---

## Tests (`tests/replecs.spec.luau`)
New `TEST("world include")` block — 10 unit cases: passthrough-when-disabled, restrict to set,
narrow an unfiltered entity, intersect with per-entity include filter, intersect with per-entity
exclude filter, incremental add/remove, re-open via `nil`, empty-set deny-all, readiness
orthogonality, and **compaction remap**. Plus one end-to-end case in
`TEST("server -> client replication")` asserting an excluded player's client receives nothing
through the real `get_full` send path.

## Verify
```bash
zune run test          # 74/74 pass (10 unit + 1 e2e added)
selene src             # 0 errors / 0 warnings
lune run ts-process    # roblox-ts build (compiles + darklua), exit 0
```
`luau-lsp analyze` adds **zero** new type errors vs. baseline (remaining ones are
pre-existing/environmental, mostly inside vendored `jecs.luau`).

> Note: running tests/build locally requires `jecs` in `luau_packages/` (gitignored). It was staged
> from a `wally install` (`Packages/_Index/ukendio_jecs@*/jecs/src/jecs.luau` → `luau_packages/jecs.luau`).
