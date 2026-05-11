# AlifePlus Integration Guide

The public API for foreign mods is `ap_api`. It covers ownership coordination, reading scripted-squad state, the activity record, releasing squads, registering causes and consequences, and resolving localized text. Anything outside `ap_api` is internal and not part of the contract.

Requires: AlifePlus loaded. See `architecture.md` for internals.

> [!IMPORTANT]
> Always feature-detect: check `ap_api` is non-nil and the specific function you want exists. The API surface evolves between AP versions; a missing function is non-fatal if the integration is graceful.

---

## API Reference

### Ownership

| Function | Purpose |
|---|---|
| `ap_api.register_owner(name, fn)` | Register a squad ownership filter. Squads where `fn(squad)` returns true are excluded from AP routing. Replaces existing filter on name match. Returns `true` on success, `false` if args invalid. |
| `ap_api.unregister_owner(name)` | Remove a filter by name. Returns `true` if removed, `false` if not registered. |
| `ap_api.get_owner(squad)` | Returns owner name (`string`) or `nil` if unowned. |

<details>
<summary>Parameter details</summary>

- `name` — string, e.g. `"warfare"`
- `fn` — `function(squad) -> boolean`
- `squad` — squad server object (userdata)

</details>

### Scripted-squad state

Scripted-squad entries hold control state only (target, on_arrive, gulag). Activity attribution (consequence, cause, display facts) lives in the activity record — see [Activity record](#activity-record).

| Function | Purpose |
|---|---|
| `ap_api.get_scripted_squads()` | Shallow-copied snapshot of the AP-scripted squads table, keyed by squad id. Outer table and each entry are fresh per call. Prefer `get_scripted_squad(id)` for single-entry lookups. |
| `ap_api.get_scripted_squad(squad_id)` | Shallow copy of the entry for one squad id, or `nil`. |
| `ap_api.release_squad(squad_id)` | Release from AP scripted control. Stops the broker reassert loop and clears `scripted_target`. Returns `false` if AP wasn't currently scripting the squad. |
| `ap_api.script_squad(squad, smart, opts)` | Script a squad to a smart with full lifecycle. Returns `{ code, id, dst, dst_id }`. |
| `ap_api.script_actor_target(squad)` | Script a squad to pursue the player (no arrival detection). Returns `{ code, id, dst, dst_id }`. |

<details>
<summary>Entry fields (get_scripted_squad / get_scripted_squads)</summary>

- `scripted_target` — smart name, or `"actor"` for actor-pursuit
- `target_smart_id` — smart id (nil for actor-pursuit)
- `tracked_at` — game-time the script started
- `pre_release_gulag` — seconds the broker holds the squad after arrival
- `on_arrive` — internal arrival-handler dispatch key
- `on_arrive_args` — args passed to the arrival handler
- `arrived` — `true` after arrival (optional)
- `release_at` — game-time of pending release (optional)

</details>

<details>
<summary>script_squad opts</summary>

- `rush` — run + danger anim when online
- `on_arrive` — dispatch key (typically the consequence registration name); invoked when a consequence with this name registered an arrival handler
- `on_arrive_args` — table passed to the arrival handler
- `pre_release_gulag` — seconds held after arrival (default from const)

</details>

> [!NOTE]
> `script_squad` / `script_actor_target` populate the broker control state only. Call `ap_api.add_record` after a SUCCESS result to drive HUD markers and news compose.

### Activity record

The activity record is a 256-slot FIFO of dispatched consequences with denormalized display facts. It drives HUD markers, news compose, and external tooltip queries.

| Function | Purpose |
|---|---|
| `ap_api.add_record(subject_squad, cause_key, consequence_key, opts)` | Capture translation keys and write one entry. Call after `script_squad` / `script_actor_target` SUCCESS. |
| `ap_api.get_record(opts)` | Most-recent entry matching `opts` (highest `cons_id`). Required arg. |
| `ap_api.get_records(opts)` | All entries matching `opts`. `assigned=true` reads the live index. |

<details>
<summary>add_record opts</summary>

- `action_key` — `ACTION.X` enum value (player-facing label, localization id)
- `other_squad` — optional second-party squad (e.g. revenge target)
- `smart` — destination smart server object (id stored, resolved lazily at render)
- `level_id` — level the event happened on
- `cause_id` — optional correlator if your cause publishes sibling consequences

</details>

<details>
<summary>Record entry fields</summary>

Every translatable field stores a localization key (suffix `_key`); consumers translate via `ap_api.get_text` at the render site. Numeric/string ids use `_id`. Names are engine-resolved strings (no translation).

- `cons_id`, `squad_id`, `cause_id`, `other_squad_id`, `smart_id`, `level_id` — ids
- `cause_key`, `consequence_key`, `action_key` — translation keys for the cause/consequence/action labels
- `subject_faction_key`, `subject_species_key`, `other_faction_key`, `other_species_key` — translation keys for faction and species labels (mutant indirection resolved at write time, so these are directly translatable)
- `subject_name`, `other_name` — commander name strings (engine, no translation)
- `assigned` (bool, live = true), `game_hours` (number, hour-of-write)

</details>

> [!TIP]
> `get_record({ squad_id = X, assigned = true })` is O(1) (live index hit). Other queries are full FIFO scans.

### Display text

| Function | Purpose |
|---|---|
| `ap_api.get_text(id)` | Resolve a `CONSEQUENCE.X` or `ACTION.X` enum value to its localized phrase. Both enum values are themselves localization ids. Returns `nil` for unknown / unresolved ids. |

The PDA marker for an AP-scripted squad reads from the activity record. The label is the action phrase (resolved via `game.translate_string(record.action_key)`), a Subject line, and an optional Other line when the record carries a second-party squad:

```
Investigating a Massacre Site
Subject: Loner Squad (Loners)
Other: Bandit Squad (Bandits)
```

### Cause and consequence registration

| Function | Purpose |
|---|---|
| `ap_api.get_cause(consequence)` | The cause string a registered consequence subscribes to (1:1 mapping). |
| `ap_api.register_radiant_cause(category, predicate)` | Register a radiant predicate. Auto-iterates `RADIANT_CALLBACKS`. |
| `ap_api.register_reactive_cause(callback, category, predicate)` | Register a reactive predicate for one engine callback. |
| `ap_api.register_consequence(name, opts, handler)` | Register a consequence handler. Strict: refuses duplicates. |
| `ap_api.unregister_consequence(name)` | Remove a consequence handler by name. |

<details>
<summary>register_consequence opts</summary>

- `cause` (required) — cause string the handler subscribes to
- `condition` (optional) — `function() -> bool`; false to skip dispatch entirely
- `on_arrive_fn` (optional) — `function(squad, on_arrive_args)`, called when a squad scripted by this consequence arrives at its smart

</details>

<details>
<summary>Predicate signatures and return shapes</summary>

Radiant predicate: `function(squad) -> table | nil`
Reactive predicate: `function(...) -> table | nil` (signature matches the engine callback's args)

A predicate that publishes a cause must return:

```lua
{
    cause     = "cause:<modname>:<event>",  -- routing key, must be unique
    squad_id  = squad.id,                   -- for AP debug logging
    level_id  = squad.level_id,             -- for AP debug logging
    -- ...any other fields your consequence needs
}
```

A predicate that rejects returns either `nil`, or a structured rejection:

```lua
{ code = RESULT.FAILED_RULES, reason = REASON.WRONG_ALIGNMENT }
```

</details>

### Categories

`category` must be one of:

```lua
ap_core_const.CAUSE_CATEGORY.REACTIONS      -- triggered by world events (NPC died, item used)
ap_core_const.CAUSE_CATEGORY.NEEDS          -- triggered by squad internal state (eat, sleep)
ap_core_const.CAUSE_CATEGORY.INSTINCTS      -- triggered by stimulus (scent, sense of danger)
ap_core_const.CAUSE_CATEGORY.OPPORTUNITIES  -- triggered by environmental state (empty smart, abandoned stash)
```

Foreign causes share AP's per-category MCM caps and stats grouping. `register_radiant_cause` and `register_reactive_cause` reject unknown categories.

### Consequences and actions

`script_squad` / `script_actor_target` opts hold control state only (rush, on_arrive, gulag). Activity attribution (cause, consequence, action, display keys) goes into the activity record via `ap_api.add_record` after a SUCCESS result.

Two enum families participate in label rendering:

- `CONSEQUENCE.X` — routing key for arrival dispatch (`on_arrive`), stored as `consequence_key` on the record. Example: `"consequence:massacre_investigate"`.
- `ACTION.X` — player-facing label. Stored as `action_key` on the record; HUD/news/external translate via `game.translate_string` (or `ap_api.get_text`) at the render site. Example: `"action:massacre_investigate"` -> "Investigating a Massacre Site".

Both enum values are themselves localization ids. Strings live in `gamedata/configs/text/eng/ui_st_ap_consequence.xml` and `gamedata/configs/text/rus/ui_st_ap_consequence.xml`.

> [!WARNING]
> The Russian XML is windows-1251. Edit it via `xmlstarlet` only — Edit/Write/sed corrupt Cyrillic to U+FFFD.

`ap_api.get_text(id)` is `game.translate_string(id)` and works for any localization key on the record (action, consequence, faction, species).

For foreign mods adding new consequences:

1. Pick keys, define your enum constants:
   ```lua
   local CONSEQUENCE_AMBUSH_SETUP = "consequence:my_mod_ambush_setup"
   local ACTION_AMBUSH_SETUP      = "action:my_mod_ambush_setup"
   ```
2. Add matching `<string id="...">` entries in your mod's own `ui_st_<modname>.xml` (eng + rus) for both ids.
3. Register the consequence with `ap_api.register_consequence(CONSEQUENCE_AMBUSH_SETUP, ...)`.
4. After a successful `script_squad`, call `ap_api.add_record(squad, CAUSE_X, CONSEQUENCE_AMBUSH_SETUP, { action_key = ACTION_AMBUSH_SETUP, ... })`. The record carries the keys; HUD and news translate at render via `get_text`.

---

## Integration Scenarios

### Detect AlifePlus

```lua
local function has_alifeplus()
    return ap_api ~= nil
end
```

`ap_api` is the public namespace. If AP is loaded, `ap_api` is non-nil.

> [!IMPORTANT]
> Presence of `ap_api` does not guarantee every function exists across versions. Wherever you call a specific function, also check it directly: `if ap_api and ap_api.add_record then ... end`. The patterns below follow this convention.

### Coordinate squad ownership

So AP doesn't route to squads your mod owns. Warfare's pattern:

```lua
function on_game_start()
    if not (ap_api and ap_api.register_owner) then return end
    ap_api.register_owner("warfare", function(squad)
        if not squad then return false end
        return squad.registered_with_warfare == true
    end)
end
```

`register_owner` replaces the existing filter on name match, so calling it again with the same name updates the filter. AP excludes owned squads at multiple layers (producer gate, cause predicate, find_squads, scripting).

To stop owning squads (e.g. on MCM toggle):

```lua
if ap_api and ap_api.unregister_owner then
    ap_api.unregister_owner("warfare")
end
```

### Display AP activity in your UI

Tooltip showing what AP is doing with a squad:

```lua
local function ap_action_for(squad_id)
    if not (ap_api and ap_api.get_record and ap_api.get_text) then return nil end
    local rec = ap_api.get_record({ squad_id = squad_id, assigned = true })
    if not rec or not rec.action_key then return nil end
    return ap_api.get_text(rec.action_key)
end

-- Call site:
local text = ap_action_for(squad.id)
if text then
    tooltip = tooltip .. "\n" .. text
end
```

Returns localized strings: "Investigating a Massacre Site", "Heading to a Campfire to Rest", etc. `nil` when AP has no live record for the squad. The record carries translation keys for both subject and other party (`subject_faction_key`, `subject_species_key`, `subject_name`, plus `other_*`) so callers can build richer tooltips with one `get_text` call per key:

```lua
local faction = ap_api.get_text(rec.subject_faction_key)
local species = rec.subject_species_key and ap_api.get_text(rec.subject_species_key)
local name    = rec.subject_name  -- engine string, no translation
```

### Take a squad back from AP

When your mod needs a specific squad immediately:

```lua
if ap_api and ap_api.release_squad and ap_api.release_squad(squad.id) then
    -- AP was scripting it; broker's reassert loop is now stopped
end
xsquad.acquire_squad(squad, your_smart, true)
```

`release_squad` returns `false` if AP wasn't scripting the squad. Safe to call unconditionally.

### Add a new cause and consequence

Foreign mod adds its own cause + consequence. Goes through AP's full pipeline (gates, rate limits, marker lifecycle, news compose).

<details>
<summary>Worked example (cause + consequence + arrival)</summary>

```lua
local CAUSE_AMBUSH             = "cause:my_mod:ambush"
local CONSEQUENCE_AMBUSH_SETUP = "consequence:my_mod_ambush_setup"
local ACTION_AMBUSH_SETUP      = "action:my_mod_ambush_setup"
local CATEGORY = ap_core_const.CAUSE_CATEGORY.OPPORTUNITIES
local cfg = my_mod_mcm.cfg

-- Cause: outlaws notice an empty smart, want to ambush
local function _predicate(squad)
    if not cfg.ambush_enabled then return nil end
    if not is_outlaw(squad) then return nil end
    if not ap_core_limiter.check_cause_rate_limit(CATEGORY, cfg.cause_max_opportunities) then
        return { code = ap_core_const.RESULT.FAILED_RULES,
                 reason = ap_core_const.REASON.BUDGET_EXHAUSTED }
    end
    local smart = find_empty_smart_near(squad)
    if not smart then return nil end
    ap_core_limiter.increment_cause_counter(CATEGORY)
    return {
        cause     = CAUSE_AMBUSH,
        squad_id  = squad.id,
        level_id  = squad.level_id,
        smart_id  = smart.id,
        community = squad.player_id,
    }
end

-- Consequence: send nearby outlaws to the smart
local function _on_arrive(squad, args)
    log("squad %d arrived to ambush at smart %d", squad.id, args.smart_id)
end

local function _handler(event_data)
    local smart = xobject.se(event_data.smart_id)
    if not smart then return { code = ap_core_const.RESULT.FAILED_SCAN } end
    local squads = find_outlaw_squads_near(event_data)
    for _, squad in ipairs(squads) do
        local res = ap_api.script_squad(squad, smart, {
            on_arrive         = CONSEQUENCE_AMBUSH_SETUP,
            on_arrive_args    = { smart_id = event_data.smart_id },
            rush              = true,
            pre_release_gulag = 1800,
        })
        if res.code == ap_core_const.RESULT.SUCCESS and ap_api.add_record then
            ap_api.add_record(squad, CAUSE_AMBUSH, CONSEQUENCE_AMBUSH_SETUP, {
                action_key = ACTION_AMBUSH_SETUP,
                smart      = smart,
                level_id   = event_data.level_id,
            })
        end
    end
    return { code = ap_core_const.RESULT.SUCCESS }
end

function on_game_start()
    if not (ap_api and ap_api.register_radiant_cause and ap_api.register_consequence) then return end
    ap_api.register_radiant_cause(CATEGORY, _predicate)
    ap_api.register_consequence(CONSEQUENCE_AMBUSH_SETUP, {
        cause        = CAUSE_AMBUSH,
        condition    = function() return cfg.ambush_setup_enabled end,
        on_arrive_fn = _on_arrive,
    }, _handler)
end
```

</details>

> [!TIP]
> Conventions:
> - Cause strings: namespace as `"cause:<modname>:<event>"` to avoid collision with AP's standard causes
> - Consequence names: namespace as `"consequence:<modname>_<event>"` so the consequence enum value is itself a valid localization id
> - Pass the consequence registration name as `on_arrive` if you want the consequence to handle its own arrivals
> - Always call `ap_api.add_record` after a SUCCESS dispatch — without a record, no marker is shown and news cannot narrate the event
> - Self-gate against the per-category cause budget with `ap_core_limiter.check_cause_rate_limit` and `increment_cause_counter`; the framework does not gate this for you
> - Predicate payload must include `squad_id` and `level_id` (AP debug log reads them)

### Resolve localized text outside the tooltip pattern

For any consequence or action enum value you have:

```lua
if ap_api and ap_api.get_text then
    local action_text     = ap_api.get_text(ap_core_const.ACTION.MASSACRE_INVESTIGATE)
    -- "Investigating a Massacre Site"

    local consequence_text = ap_api.get_text(ap_core_const.CONSEQUENCE.MASSACRE_INVESTIGATE)
    -- "Massacre Investigate"
end
```

Both enum values are themselves localization ids (`"action:massacre_investigate"`, `"consequence:massacre_investigate"`). Strings live in `gamedata/configs/text/<lang>/ui_st_ap_consequence.xml`. AlifePlus includes English and Russian; locale switches via `game.translate_string`.

---

## Notes

- `ap_api` is the contract. `ap_core_*` modules are internal; using them directly is possible but unsupported across versions.
- For an alternative integration that doesn't use `ap_api`, you can subscribe directly to AP's xbus events. Cause names are listed in `ap_core_const.CAUSE`. Listener-only mods (telemetry, journal, sound) typically use this path.

---

## Contact

Questions, integration requests, API feedback:

- Discord: **damian_sirbu**
- ModDB: **damian_sirbu**
- Email: **dami.sirbu@gmail.com**
- GitHub: **github.com/damiansirbu-stalker**
