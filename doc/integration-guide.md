# AlifePlus Integration Guide

The public API for foreign mods is `ap_api`. It covers ownership coordination, reading scripted-squad state, releasing squads, registering causes and consequences, and resolving localized text. Anything outside `ap_api` is internal and not part of the contract.

Requires: AlifePlus loaded. See `architecture.md` for internals.

---

## API Reference

### Ownership

```lua
ap_api.register_owner(name, fn)
-- Register a squad ownership filter. Squads where fn(squad) returns true
-- are excluded from AP routing. Replaces existing filter on name match.
-- @param name      string  Owner name (e.g. "warfare")
-- @param fn        function(squad) -> boolean

ap_api.unregister_owner(name)
-- Remove a squad ownership filter by name. No-op if not registered.
-- @return boolean  true if removed, false if not found

ap_api.get_owner(squad)
-- Query squad ownership.
-- @param squad     userdata  Squad server object
-- @return string|nil          Owner name, or nil if unowned
```

### Scripted-squad state

```lua
ap_api.get_scripted_squads()
-- Returns a shallow copy of the AP-scripted squads table, keyed by squad id.
-- Outer table is fresh per call; entry tables are shared with broker state
-- (treat fields as read-only). Each entry carries:
--   consequence       string   identity of what the squad is doing
--   scripted_target   string   smart name, or "actor" for actor-pursuit
--   target_smart_id   number   smart id (nil for actor-pursuit)
--   tracked_at        number   game-time the script started
--   pre_release_gulag number   seconds the broker holds the squad after arrival
--   on_arrive         string   internal arrival-handler dispatch key
--   on_arrive_args    table    args passed to the arrival handler
--   arrived           boolean  set true after arrival (optional)
--   release_at        number   game-time of pending release (optional)
-- @return table

ap_api.release_squad(squad_id)
-- Release a squad from AP scripted control. Stops the broker's reassert loop
-- and clears scripted_target.
-- @param squad_id  number
-- @return boolean  false if AP wasn't currently scripting the squad

ap_api.script_squad(squad, smart, opts)
-- Script a squad to a smart terrain with full AP lifecycle (PDA marker, TTL,
-- pre-release hold, target reassertion).
-- @param squad     userdata
-- @param smart     userdata
-- @param opts      table { consequence, rush?, on_arrive?, on_arrive_args?, pre_release_gulag? }
--   consequence (required) - CONSEQUENCE.X identity
--   rush                   - run + danger anim when online
--   on_arrive              - dispatch key, only set when a consequence with
--                            this name registered an arrival handler
--   on_arrive_args         - passed to the arrival handler
--   pre_release_gulag      - seconds held after arrival (default from const)
-- @return table { code, id, dst, dst_id }

ap_api.script_actor_target(squad, opts)
-- Script a squad to pursue the player. No arrival detection (player moves).
-- @param squad     userdata
-- @param opts      table|nil { consequence }
-- @return table { code, id, dst, dst_id }
```

### Display text

```lua
ap_api.get_text(consequence)
-- Resolve a consequence key to its localized action phrase
-- ("Investigating a Massacre Site", "Heading to a Campfire to Rest", ...).
-- @param consequence  string   CONSEQUENCE.X enum value
-- @return string|nil           localized text, or nil for unknown / unresolved keys
```

### Cause and consequence registration

```lua
ap_api.get_cause(consequence)
-- The cause string a registered consequence subscribes to (1:1 mapping).
-- @param consequence  string   consequence registration name
-- @return string|nil           cause string ("cause:massacre"), or nil

ap_api.register_radiant_cause(category, predicate)
-- Register a radiant cause predicate. Auto-iterates RADIANT_CALLBACKS.
-- Predicate fires per radiant tick with one squad and returns either a
-- publishing payload, a rejection table, or nil.
-- @param category  string  Must be one of ap_core_const.CAUSE_CATEGORY values
-- @param predicate function(squad) -> table|nil
-- @return boolean  false if category invalid

ap_api.register_reactive_cause(callback, category, predicate)
-- Register a reactive cause predicate for one engine callback.
-- Predicate signature matches the engine callback's args.
-- @param callback   string  CALLBACK enum value
-- @param category   string  Must be one of ap_core_const.CAUSE_CATEGORY values
-- @param predicate  function(...) -> table|nil
-- @return boolean  false if category invalid

ap_api.register_consequence(name, opts, handler)
-- Register a consequence handler.
-- @param name     string  Consequence registration name (also the arrival dispatch key)
-- @param opts     table { cause, condition?, on_arrive_fn? }
--   cause         (required) - cause string the handler subscribes to
--   condition     (optional) - function() -> bool; false to skip dispatch entirely
--   on_arrive_fn  (optional) - function(squad, on_arrive_args) called when a squad
--                              scripted by this consequence arrives at its smart
-- @param handler  function(event_data) -> { code = RESULT.X, ... }
-- @return boolean
```

### Predicate payload contract

A predicate that publishes a cause must return a table with at least:

```lua
{
    cause     = "cause:<modname>:<event>",  -- routing key, must be unique
    squad_id  = squad.id,                   -- for AP debug logging
    level_id  = squad.level_id,             -- for AP debug logging
    -- ...any other fields your consequence needs
}
```

A predicate that rejects returns either nil, or a structured rejection:

```lua
{ code = RESULT.FAILED_RULES, reason = REASON.WRONG_ALIGNMENT }
```

### Categories

`category` must be one of:

```lua
ap_core_const.CAUSE_CATEGORY.REACTIONS      -- triggered by world events (NPC died, item used)
ap_core_const.CAUSE_CATEGORY.NEEDS          -- triggered by squad internal state (eat, sleep)
ap_core_const.CAUSE_CATEGORY.INSTINCTS      -- triggered by stimulus (scent, sense of danger)
ap_core_const.CAUSE_CATEGORY.OPPORTUNITIES  -- triggered by environmental state (empty smart, abandoned stash)
```

Foreign causes share AP's per-category MCM caps and stats grouping. `register_radiant_cause` and `register_reactive_cause` reject unknown categories.

---

## Integration Scenarios

### Detect AlifePlus

```lua
local function has_alifeplus()
    return ap_api ~= nil
end
```

`ap_api` is the public namespace. If AP is loaded, `ap_api` is non-nil and every documented function exists.

### Coordinate squad ownership

So AP doesn't route to squads your mod owns. Warfare's pattern:

```lua
function on_game_start()
    ap_api.register_owner("warfare", function(squad)
        if not squad then return false end
        return squad.registered_with_warfare == true
    end)
end
```

`register_owner` replaces the existing filter on name match, so calling it again with the same name updates the filter. AP excludes owned squads at multiple layers (producer gate, cause predicate, find_squads, scripting).

To stop owning squads (e.g. on MCM toggle):

```lua
ap_api.unregister_owner("warfare")
```

### Display AP activity in your UI

Tooltip showing what AP is doing with a squad:

```lua
local function ap_action_for(squad_id)
    if not ap_api then return nil end
    local entry = ap_api.get_scripted_squads()[squad_id]
    if not entry or not entry.consequence then return nil end
    return ap_api.get_text(entry.consequence)
end

-- Call site:
local action = ap_action_for(squad.id)
if action then
    tooltip = tooltip .. "\n" .. action
end
```

Returns localized strings: "Investigating a Massacre Site", "Heading to a Campfire to Rest", etc. nil when the squad isn't currently scripted by AP. Call fresh on every render tick — entries change as squads move between consequences.

### Take a squad back from AP

When your mod needs a specific squad immediately:

```lua
if ap_api.release_squad(squad.id) then
    -- AP was scripting it; broker's reassert loop is now stopped
end
xsquad.acquire_squad(squad, your_smart, true)
```

`release_squad` returns `false` if AP wasn't scripting the squad — safe to call unconditionally.

### Add a new cause and consequence

Foreign mod adds its own cause + consequence. Goes through AP's full pipeline (gates, rate limits, PDA news, marker lifecycle).

```lua
local CAUSE_AMBUSH = "cause:my_mod:ambush"
local CONSEQUENCE_AMBUSH_SETUP = "MY_MOD_AMBUSH_SETUP"
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
        ap_api.script_squad(squad, smart, {
            consequence       = CONSEQUENCE_AMBUSH_SETUP,
            on_arrive         = CONSEQUENCE_AMBUSH_SETUP,
            on_arrive_args    = { smart_id = event_data.smart_id },
            rush              = true,
            pre_release_gulag = 1800,
        })
    end
    return { code = ap_core_const.RESULT.SUCCESS }
end

function on_game_start()
    ap_api.register_radiant_cause(CATEGORY, _predicate)
    ap_api.register_consequence(CONSEQUENCE_AMBUSH_SETUP, {
        cause        = CAUSE_AMBUSH,
        condition    = function() return cfg.ambush_setup_enabled end,
        on_arrive_fn = _on_arrive,
    }, _handler)
end
```

Conventions:

- Cause strings: namespace as `"cause:<modname>:<event>"` to avoid collision with AP's standard causes
- Consequence names: namespace similarly (`"MY_MOD_X"` or your own pattern)
- Pass the same string for `consequence` and `on_arrive` if you want the consequence to handle its own arrivals
- Self-gate against the per-category cause budget with `ap_core_limiter.check_cause_rate_limit` and `increment_cause_counter` — the framework does not gate this for you
- Predicate payload must include `squad_id` and `level_id` (AP debug log reads them)

### Resolve localized text outside the tooltip pattern

For any consequence enum string you have:

```lua
local text = ap_api.get_text(ap_core_const.CONSEQUENCE.MASSACRE_INVESTIGATE)
-- "Investigating a Massacre Site"
```

Returns nil if the key isn't in `CONSEQUENCE_INFO` or localization didn't resolve. English and Russian ship with AP; locale switches automatically via `game.translate_string`.

---

## Notes

- `ap_api` is the contract. `ap_core_*` modules are internal — using them directly is possible but unsupported across versions.
- Backward-compatibility shims for pre-`ap_api` integrations live in `ap_core_compat.script` (e.g. `ap_core_broker.get_scripted_ids` aliased, `get_record` synthesized). New integrations should not rely on them.
- For an alternative integration that doesn't use `ap_api`, you can subscribe directly to AP's xbus events. Cause names are listed in `ap_core_const.CAUSE`. Listener-only mods (telemetry, journal, sound) typically use this path.

---

## Contact

Questions, integration requests, API feedback:

- Discord: **damian_sirbu**
- ModDB: **damian_sirbu**
- Email: **dami.sirbu@gmail.com**
- GitHub: **github.com/damiansirbu-stalker**
