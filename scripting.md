# Zephyr Scripting API — Lua

Scripting reference for Zephyr bots. Zephyr supports two script APIs:
the **Zephyr Official API** (native, full fleet control) and the
**Lucifer API** (scripts written for the Lucifer client run as-is).
Both live in the same Lua runtime and can be mixed in one script.

---

## Getting started

- Scripts are `.lua` files placed in the **`scripts/` folder beside `Zephyr.exe`**.
- Open the **Executor** page to browse, run, stop, and edit scripts. Hot reload picks up saved changes while a script runs.
- A script is bound to a bot (the one selected when it starts, or the bot you pass). `getBots()` reaches the whole fleet.
- A script error is printed to the console and the script stops.

```lua
-- scripts/hello.lua
print("Zephyr " .. zephyrVersion())
local bot = getBot()
print(bot.name, bot.status)          -- worker1   online
```

## Script model

- One cooperative body per script: `sleep(ms)` **yields** the script, so the window keeps drawing. Never busy-wait.
- Timers: `after(ms, fn)` fires once; `every(ms, fn)` repeats. Both return a handle for `stopTimer(handle)`.
- Long walks yield too: `bot:findPath(x, y)` suspends the script until the bot arrives.
- Event handlers run serialized with the script body (per-script execution lock).

```lua
every(5000, function()
    if bot.status == BotStatus.online then
        bot:say("heartbeat")
    end
end)
```

---

## Getting bots

| Call | Result |
| --- | --- |
| `getBot([nameOrId])` | A bot object. No argument = the script's own bot |
| `getBots()` | Array of every bot |
| `getBotId()` | Id of the bound bot, or `0` |
| `addBot{ name=, ltoken= }` | Create a bot from a token. `ltoken` may be spelled `token` or `secret`. Returns the bot object, or `nil, error` |
| `addBot{ name=, password= }` | Create a bot that signs in with GrowID + password |
| `addBot("name")`, `addBot("name", "token")` | String spellings |
| `removeBot([name])` | Remove a bot (the bound one when unnamed). Returns boolean |
| `getUsername()` | Bound bot's name, or `""` |

`addBot` accepts full token lines (`token|mac|rid|wk|platform` shapes and a
leading `@name` segment are understood). A created bot is **not** logged in —
call `bot:connect()` or press Start.

```lua
local bot = addBot{ name = "worker1", secret = "token|mac|rid|wk" }
if bot then bot:connect() end
```

---

## Events

Subscribe with `listen(name, fn)`, or define a global handler (`function onChat(player, message)`) — no subscription needed.

```lua
listen("chat", function(e) print(e.name .. ": " .. e.text) end)
unlisten("chat")     -- drop one event's listeners
unlisten()           -- drop all of this script's listeners
```

| Event | Aliases | Global handler | Payload |
| --- | --- | --- | --- |
| `packet` | — | `onPacket` | `direction`, `net_id`, `type_name`, `size`, `summary` |
| `world_enter` | `onWorldJoin`, `world_join` | `onWorldJoin` | `name`, `width`, `height` |
| `world_leave` | `onWorldLeave` | `onWorldLeave` | `name` |
| `bot_connect` | `onConnect`, `connect` | `onConnect` | `bot`, `server` |
| `bot_disconnect` | `onDisconnect`, `disconnect` | `onDisconnect` | `bot`, `reason`, `detail` |
| `console` | `onConsole` | `onConsole` | `severity`, `message` |
| `player_join` | `onPlayerJoin` | `onPlayerJoin` | `name`, `netId`, `x`, `y` |
| `player_leave` | `onPlayerLeave` | `onPlayerLeave` | `name`, `netId` |
| `chat` | `game_message` | `onChat(player, message)` | `name`, `text` |
| `tile_change` | `onTileChange` | `onTileChange` | `x`, `y`, `foreground`, `background` |
| `dialog` | `onDialog` | `onDialog` | `name`, `text`, `buttons` (`id:label, ...`) |
| `tick` | — | none | `now` (uptime ms) |

Registry verbs: `listeners()`, `addEvent(name)`, `removeEvent(name)`,
`removeEvents()` (clear all), `listenEvents(secs)` (returns the interval in ms),
`unlistenEvents()`.

`onLogin` is not an event — there is nothing to fire it from, and subscribing
to it is rejected loudly.

---

## Utility globals

| Call | Result |
| --- | --- |
| `log(level, message)` | Log at `info` / `warn` / `error`, … into the console |
| `print(...)` | Tab-joined arguments at `info` |
| `sleep(ms)` | Yield the script body |
| `zephyrVersion()` | Version string |
| `now()` | Monotonic milliseconds (not Unix time) |
| `removeColor(text)` | Strip Growtopia colour codes |
| `runThread(fn, ...)` | Runs `fn(...)` inline, in this script |
| `getProxyManager()` | `{ mode = "socks5" \| "none", label = <connection path> }` |
| `clearConsole()` | Empty the bound bot's log ring |

Enum tables (name → string): `Platform`, `BotStatus`, `Proxy`, `VariantType`,
`Event`, `StorageType` (`seed = 1`, `pack = 2`).

---

# Zephyr Official API

## The bot object

A bot answers **methods** (`bot:say(...)`) and **properties** (`bot.name`).
Method names win over fields. You may store your own fields on the object.

### Properties

| Property | Meaning |
| --- | --- |
| `name`, `growid` / `username` | Labels and the account's GrowID |
| `status` | `offline` \| `connecting` \| `online` \| `error` \| `suspended` (compare with `BotStatus.online`) |
| `world` | Current world name, `""` when none |
| `netid` / `netId` | Local player net id |
| `x`, `y` | Tile position |
| `ping` | Milliseconds (0 = not measured) |
| `gem_count` | Gem balance as reported by the server |
| `auto_farm` | Read: is the farm task queued? |
| `auto_collect` | Read/write: the farm's AutoCollect filter |
| `collect_range` | Read/write: collect radius, `1..5` |
| `auto_reconnect` | Read/write: automatic reconnect |
| `pnb_punch_ms`, `pnb_place_ms` | Read/write: PnB pacing, clamped `10..60000` |
| `level` | Always `nil` (not tracked) |

`bot:state()` returns the exact engine state (`connected`, `in world`,
`entering world`, …) when the five-word `status` is not enough.

### Session

```lua
bot:connect()        -- start / log in (respects the login gate)
bot:disconnect()     -- stop the session
```

### Movement & pathfinding

```lua
bot:move(dx, dy)               -- one tile by offset
bot:moveTile(x, y)             -- one tile, absolute
bot:moveTo(x, y)               -- offset alias (Lucifer spelling)
bot:moveLeft()  bot:moveRight()  bot:moveUp()  bot:moveDown()
bot:walkTo(x, y)               -- pathfind (8-direction search)
bot:findPath(x, y)             -- pathfind AND walk; yields until arrival.
                               -- returns true, or false + message
bot:getPath(x, y)              -- read-only: the route as {x, y} steps
bot:setDirection(dir)  bot:face(dir)
bot:isInTile(x, y)  bot:isInWorld(name)
```

`findPath` **walks**; `getPath` only computes. They are not aliases.

### Worlds

```lua
bot:warp("WORLD")       -- enter a world (paced, cancels automation)
bot:join("WORLD")       -- raw join request
bot:leaveWorld()        -- exit to the world-select menu
bot:respawn()           -- action|respawn
bot:getWorld()          -- the world object, or nil
```

### Actions

```lua
bot:punch(x, y)  bot:hit(x, y)         -- fist / held item
bot:place(x, y, itemId)                -- place a block
bot:wrench(x, y)  bot:wrenchPlayer(netId)
bot:drop(itemId, count)  bot:fastDrop(itemId)
bot:trash(itemId)  bot:fastTrash(itemId)
bot:wear(itemId[, slot])  bot:use(itemId)  bot:unwear(itemId)
bot:buy("item_pack")                   -- buy from a store pack
bot:collectObject(x, y, uid)  bot:collect(x, y, uid)
bot:activate(...)  bot:active(...)     -- object activation spellings
bot:say("text")                        -- chat (max 200 chars, paced)
bot:dialogReturn(name[, fields])       -- answer an OnDialogRequest
```

Dialog example:

```lua
function onDialog(e)
    if e.name == "popup" and string.find(e.buttons, "acceptlock", 1, true) then
        bot:dialogReturn("popup", "netID|12|\nbuttonClicked|acceptlock")
    end
end
```

Nothing answers a dialog automatically — a reply the bot did not choose would
be a guess. A dialog name containing `|` or control characters is refused.

### Automation settings

```lua
bot:autofarm(true)      -- START the farm task (requires the bot in a world)
bot:autofarm(false)     -- stop it (success even if nothing ran)
bot:autocollect(true)   -- AutoCollect filter; a running farm re-reads it
bot.auto_collect = true -- same as the line above (property form)
bot.collect_range = 3   -- 1..5, refused outside
bot.auto_reconnect = true
bot.pnb_punch_ms = 150  -- PnB pacing floors, 10..60000, persisted
bot.pnb_place_ms = 150
```

### Per-bot pacing

```lua
bot:walkms(500)   -- 0..3000
bot:tilems(110)   -- 0..2000
bot:chatms(900)   -- 0..5000
bot:dropms(300)   -- 0..2000 (drop/wear clock)
bot:warpms(3000)  -- 0..30000
bot:jitterms(250) -- 0..2000
bot:pacing()      -- read back: { walk_ms=, tile_ms=, chat_ms=, inventory_ms=, warp_ms=, jitter_ms= }
```

Out-of-range values return `false, message` (refused, not clamped). All
settings persist with the bot.

### Raw packets

```lua
bot:sendPacket(type, text)     -- generic text frame on a net id
bot:sendPacketText(netId, text)
bot:sendRaw(packetTable)       -- a GameUpdatePacket.new() table
```

### Information methods

`getInventory()`, `getLog()`, `getConsole()`, `getLogin()`, `getProxy()`,
`getPlaytime()`, `getAge()`, `getActiveTime()`, `getSignal()`, `getCaptcha()`,
`getPing()`, `getX()`, `getY()`, `isWearing(itemId)`, `setSkin(1..6)`,
`setCountry()`, `setMac()`, `setRid()`, `updateBot()`, `runScript()`,
`stopScript()`.

Behaviour notes: `getSignal`/`getCaptcha` return `nil`; `runScript` returns
`false` (one body per script — use a separate script file); `stopScript` does
nothing (a script stops by returning); `getLogin` reports empty `mac`/`rid`;
`setMac`/`setRid`/`setCountry`/`updateBot` are accepted and change nothing;
`setSkin` is validated and applied on the next reconnect.

---

## World & tiles

| Call | Result |
| --- | --- |
| `getWorld()` | The bound bot's world object, or `nil` |
| `getTile(x, y)` | Tile table, or `nil` |
| `getTiles()` / `getTilesSafe()` | Every loaded tile |
| `getObjects()` / `getObject(uid)` | Dropped items |
| `getPlayers()` | Players in the world |
| `getNPCs()` / `getNPC(id)` | NPCs |
| `isWalkable(x, y)` | Boolean |
| `findItem(name)` | Item id, or `nil` |
| `findPath(sx, sy, tx, ty)` | Route as steps, or `nil` |

**World object** — fields `name`, `x`/`width`, `y`/`height`, `tile_count`,
`version`, `objects`, `players`, `npcs`, `tiles`, `public`; methods `getTile`,
`getTiles`, `getTilesSafe`, `getObject`, `getObjects`, `getPlayer(key)`,
`getPlayers`, `getNPC(key)`, `getNPCs`, `getLocal`, `getTileParent`,
`isValidPosition`, `hasAccess`, `hasAdventure()` (always `false`),
`getAdventures()` (empty).

**Tile** — fields `fg`/`foreground`, `bg`/`background`, `flags`, `parent`
(a tile index), `object` (extra-data kind), `foreground_name`; methods
`hasExtra()`, `getExtra()`, `canHarvest()` (true for seed data), `hasFlag(f)`,
`getParent()`.

**Dropped object** — `id`, `count`, `flags`, `oid`, `x`, `y` (world pixels).
**NPC** — `type`, `id`, `x`, `y`, `destx`, `desty`, `var`, `unk`.

## Players

| Call | Result |
| --- | --- |
| `getLocal()` | The local player table |
| `getPlayer(idOrName)` | A player, or `nil` |
| `getPlayers()` | Every player in the world |
| `getPlayerCount()` | How many |
| `findPlayer(name)` | A player, or `nil` |
| `itemName(id)` | Item name, or `item#<id>` |

Player fields: `id`, `netid`, `name`, `country`, `userid`, `x`, `y`, `posx`,
`posy`, `direction`, `isLocalPlayer`, `state`, `local`.

## Inventory & items

| Call | Result |
| --- | --- |
| `getInventory()` | Inventory handle, or `nil` |
| `getItem(id)` | `{ id, name, category, slot, count }`, or `nil` |
| `getItemCount(id)` | Total stack count across slots |
| `getItems()` | Array of `{ slot, id, name }` |
| `selectItem(id)` | Put an item in the active slot |
| `searchItems(query)` | Up to 64 matching item ids |
| `getInfo(idOrName)` | An item record, or `nil` |
| `getInfos()` | Every known item record |

Item ids are **numbers**. `getInfo("242")` and `getInfo(242)` agree — a numeric
string is an id, not a name. `getInfo`/`getInfos` work even without a bound bot.

Inventory handle: `used()`, `free()`, `getItem(id)`, `getItems()`,
`getItemCount(id)`, `findItem(id)`, `canCollect(id)`; fields `itemcount`,
`slotcount`, `version`, `items`.

## Packets (global)

| Call | Result |
| --- | --- |
| `sendPacket(netId, fields)` | Send a key→value table as a variant list |
| `sendAction(kind, x, y[, itemId])` | `kind` = `punch` \| `place` \| `walk` \| `drop` |
| `packetStats()` | `(sent, received)` |
| `packetNames()` | Array of `{ id, name }` — read net ids from here, never guess |

## Protocol classes

```lua
local v  = Vector2.new(x, y) ; local v3 = Vector3.new(x, y, z)
local va = variant(...)            -- set/reset/getType/getString/getInt/getFloat/
                                   -- getVector2/getVector3/print/toString
local vl = variantlist(...)        -- get/append/print
local pk = GameUpdatePacket.new()  -- mutable packet table for bot:sendRaw
local rv = RTVAR.new([str])        -- params/append/contains/size/get/getInt/
                                   -- getLong/erase/dump/parse
```

---

# Lucifer API

Scripts written for the Lucifer client run without changes. These names are
supported alongside the official API and can be mixed freely.

| Call | Result |
| --- | --- |
| `addBot(details)` | Table form `{ name=, token=/ltoken=/secret= }`, or the string forms. Returns the bot object |
| `read(path)` / `write(path, content)` | File access inside the scripts folder (same rules as `file.read`/`file.write`) |
| `getUsername()` | Bound bot's name |
| `removeBot([name])` | Remove a bot |
| `getWorldManager()` | Rotation manager object (below) |
| `Webhook.new(url)` | Discord webhook sender (below) |
| `StorageType` | `{ seed = 1, pack = 2 }` |
| `hit`, `use`, `active`, `face`, `collect` | `punch`, `wear`, `activate`, `setDirection`, `collectObject` |
| `bot.auto_farm.*` | Read-only view; use `bot:autofarm(...)` / `bot:autocollect(...)` to change state |
| `bot:findPath(x, y)` | Walking pathfind (yields until arrival) |
| `bot.status` / `BotStatus` | The five-word status vocabulary |
| `getInfo(idOrName)` / `getInfos()` | Item catalogue, no bot required |

### World manager (rotation)

```lua
local wm = getWorldManager()
wm.addFarm("FARM1")            -- add a farm world (optional block id)
wm.addFarm("FARM2", 2)
wm.removeFarm("FARM1")
wm.addStorage("STORAGE", StorageType.seed, 3)   -- seed storage, seed id 3
wm.removeStorage("STORAGE", StorageType.seed)
wm.selectAll()
wm.apply()
```

`wm.worlds` lists the rotation. Farm entries carry the world and block id; the
storage entry is the seed-storage world.

### Webhooks

```lua
local hook = Webhook.new("https://discord.com/api/webhooks/...")
hook.username = "Zephyr"
hook.avatar_url = "https://example.com/avatar.png"
hook.content   = "farm finished"
hook:send()                    -- posted asynchronously; the script never blocks
```

Only `username`, `avatar_url` and `content` are sent. Embeds, files and
threads are ignored.

---

## HTTP & files

```lua
http.get(url)                  -- background request; returns true when dispatched
http.post(url, body)           -- body is sent as JSON
file.read("data/notes.txt")    -- contents, or nil + reason
file.write("out/log.txt", "hi") -- true, or false + reason
read("x.txt")  write("x.txt", "hi")   -- same doors, shorter names
```

File access is confined to the **scripts folder**: a path that escapes it
(`../`) is refused, not trimmed. Requests run in the background — the script
continues while the network works.

## Timers

```lua
local h = after(10000, function() print("ten seconds") end)
local r = every(1000, function() print("tick") end)
stopTimer(r)                    -- cancel
sleep(500)                      -- yield half a second
```

## Errors

A script error is reported to the console and log, and the script stops. Codes:
`LUA_RUNTIME_ERROR`, `LUA_SCRIPT_NOT_FOUND`, `LUA_SCRIPT_EXISTS`,
`LUA_SYNTAX_ERROR`, `LUA_TIMEOUT`, `LUA_SANDBOX_VIOLATION`, `LUA_MODULE_ERROR`,
`LUA_SCRIPT_NOT_RUNNING`.

## Full bot method index

`connect`, `disconnect`, `move`, `moveTile`, `moveTo`, `moveLeft`,
`moveRight`, `moveUp`, `moveDown`, `isInWorld`, `walkTo`, `punch`, `hit`,
`wrench`, `wrenchPlayer`, `place`, `drop`, `fastDrop`, `trash`, `fastTrash`,
`wear`, `use`, `unwear`, `buy`, `activate`, `active`, `enter`, `retrieve`,
`respawn`, `setDirection`, `face`, `isInTile`, `getPath`, `findPath`,
`findWorldPath`, `getWorld`, `getInventory`, `getLog`, `getConsole`,
`getLogin`, `getProxy`, `getPlaytime`, `getAge`, `getActiveTime`, `getSignal`,
`getCaptcha`, `setSkin`, `setCountry`, `setMac`, `setRid`, `updateBot`,
`runScript`, `stopScript`, `getPing`, `isWearing`, `collectObject`, `collect`,
`clearConsole`, `sendPacket`, `sendPacketText`, `sendRaw`, `say`, `join`,
`warp`, `leaveWorld`, `state`, `x`, `y`, `name`, `ping`, `getX`, `getY`,
`walkms`, `tilems`, `chatms`, `dropms`, `warpms`, `jitterms`, `pacing`,
`autofarm`, `autocollect`, `dialogReturn`.
