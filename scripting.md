# Panduan Lua API — Sylphie

Dokumen ini menjelaskan **cara memakai Lua API Sylphie** dari nol sampai bisa:
di mana script disimpan, cara menjalankannya, aturan eksekusi, daftar event,
referensi lengkap API (bot, world, player, inventory, packet), contoh script
yang bisa langsung dipakai, dan daftar hal yang *sengaja* tidak benar-benar
bekerja supaya kamu tidak membuang waktu mendebugnya.

Referensi teknis internal ada di [`docs/scripting.md`](docs/scripting.md).
Halaman web versi dokumen ini: [`index.html`](index.html).

---

## 1. Apa itu scripting di Sylphie

Sylphie menanam **Lua 5.4** di dalam aplikasi. Setiap script mendapat:

- `lua_State` sendiri — dua script tidak bisa saling merusak global,
- satu coroutine kooperatif sendiri,
- salinan API sendiri.

Artinya: satu script bisa crash/loop tanpa membekukan UI, dan script lain tetap
jalan. Script tidak punya thread OS — semua berbagi frame loop aplikasi.

---

## 2. Mulai cepat

### 2.1 Di mana script disimpan

```
<folder exe>/scripts
```

Folder ini dibuat otomatis saat aplikasi start, dan letaknya **di samping
`Sylphie.exe`**, bukan di `%LOCALAPPDATA%`. Jadi kalau kamu punya folder
portable (`Sylphie.exe` + `items.dat` + `scripts/` + `config.json`), cukup drop
file `.lua` ke `scripts/`.

Folder lama `%LOCALAPPDATA%\Sylphie\scripts` **tidak dipindahkan otomatis** —
pindahkan sendiri.

### 2.2 Cara menjalankan

Command line sudah dihapus (tidak ada `--headless` / `--exec`). Jalankan GUI,
lalu pakai **halaman Scripts** atau ketik di halaman **Console**:

```
script list
script run <nama>
script stop <nama>
script reload <nama>
```

Sebuah script **berhenti sendiri saat selesai**: begitu body-nya selesai dan
tidak ada lagi `sleep()` yang menggantung, tidak ada `listen()` / handler
`onXxx`, dan tidak ada timer `after()`/`every()` yang hidup, tombol kembali ke
*Run*. Script yang isinya `while true do ... end` tetap `running` sampai kamu
tekan Stop.

### 2.3 Script pertama

Simpan sebagai `scripts/hello.lua`, lalu **Run**:

```lua
local bot = getBot()              -- tanpa argumen: bot yang dipasangkan ke script ini
if not bot then
    log("error", "script ini tidak punya bot")
    return
end

log("info", "bot " .. bot.name .. " status: " .. bot.status)

bot:say("halo dari script!")
```

### 2.4 Konvensi yang perlu dihafal

**Hampir semua method bot mengembalikan `(ok, error)`.** Periksa hasilnya:

```lua
local ok, err = bot:say("halo")
if not ok then
    log("warn", "gagal kirim chat: " .. tostring(err))
end
```

`ok` = boolean, `err` = string kosong saat sukses dan pesan error (mis.
`BOT_ACTION_RATE_LIMITED`) saat gagal.

**Pacing itu fitur, bukan gangguan.** Mesin membatasi sendiri tiap aksi
(gerak, tile, chat, inventory, warp). Itu yang membuat bot tidak dianggap
spam oleh server. Karena itu **jangan hapus `sleep()`** di loop — script yang
mencoba bergerak secepat mungkin tidak menambah apa pun selain tekanan rate
limit di sisi server.

---

## 3. Aturan eksekusi (scheduler)

- Body script adalah **coroutine** yang di-resume per potong. `while true do ... end`
  akan *yield*, tidak nge-hang.
- `sleep(ms)` hanya bekerja **di body script**. Di dalam `listen()` callback atau
  handler global tidak ada coroutine untuk di-yield, jadi delay **dilewati** dan
  sebuah warning dicatat. Jangan sleep di callback.
- Setiap resume menjalankan paling banyak `scripts.instruction_budget` instruksi
  Lua (default `2000000`; `0` mematikan hook), supaya loop ketat tidak membuat
  UI/network kelaparan. Script yang terus melewati budget dilaporkan
  `LUA_TIMEOUT`.
- `after()` / `every()` ditembakkan dari frame loop yang sama — bukan thread
  kedua.
- Kalau body script `return`, script selesai — kecuali masih ada handler/timer
  hidup, yang membuatnya tetap `running`.

Infrastruktur peringatan di Console: `script list` menampilkan script beserta
statusnya.

---

## 4. Event

Ada **dua cara** menerima event, dan keduanya memberi payload yang sama:

1. **Handler global** dengan nama referensi — `function onChat(player, message)`.
   Tidak perlu registrasi apa pun. Ini cara script gaya GTBot-JS ditulis.
2. **`listen(name, fn)`** — spelling milik Sylphie.

```lua
listen("chat", function(event)
    log("info", event.name .. ": " .. event.text)
end)

-- Event yang sama sebagai handler global:
function onChat(player, message)
    log("info", player.name .. ": " .. message)
end
```

`listen(name, fn)` tidak mengembalikan apa pun. `listeners()` mengembalikan
tabel nama-event → **jumlah** listener (bukan handle). Batalkan per nama:

```lua
unlisten("chat")   -- buang listener untuk satu event
unlisten()         -- buang semua listener script ini
```

Nama alternatif yang juga diterima: `addEvent`, `removeEvent`, `removeEvents`,
`listenEvents`, `unlistenEvents`.

### 4.1 Daftar event

| Event | Alias | Handler global | Payload |
| --- | --- | --- | --- |
| `packet` | `onPacket` | `onPacket` | `direction`, `net_id`, `type_name`, `size`, `summary` |
| `world_enter` | `onWorldJoin`, `world_join` | `onWorldJoin` | `name`, `width`, `height` (width/height hanya saat enter) |
| `world_leave` | `onWorldLeave` | `onWorldLeave` | `name` |
| `bot_connect` | `onConnect`, `connect` | `onConnect` | `bot`, `server` |
| `bot_disconnect` | `onDisconnect`, `disconnect` | `onDisconnect` | `bot`, `reason` (token mesin, mis. `ServerClosed`), `detail` (bisa tidak ada) |
| `console` | `onConsole` | `onConsole` | `severity`, `message` |
| `player_join` | `onPlayerJoin` | `onPlayerJoin` | `name`, `netId`, `x`, `y` |
| `player_leave` | `onPlayerLeave` | `onPlayerLeave` | `name`, `netId` |
| `chat` | `onChat`, `game_message` | `onChat(player, message)` | `name`, `text`; teks juga jadi argumen kedua |
| `tile_change` | `onTileChange` | `onTileChange` | `x`, `y`, `foreground`, `background` |
| `tick` | — | **tidak ada** | `now` (uptime ms) |

Catatan:

- Semua nilai payload adalah **string**, termasuk angka. Nama event juga ada di
  tabel sebagai field `type`.
- Handler berjalan di thread yang mem-publish event, diserialisasi terhadap
  scheduler script oleh lock. Dipanggil lewat `pcall`: callback `listen()` yang
  throw dilaporkan lalu **dimatikan**; handler global yang throw dilaporkan tapi
  dibiarkan (karena script yang mendefinisikannya).
- `onLogin` **bukan** event. Tidak ada event terautentikasi untuk menembakkannya,
  jadi `listen("onLogin")` ditolak, bukan berlangganan ke sesuatu yang tak pernah
  jalan.
- `tick` hanya lewat `listen()`; per-frame callback harus murah untuk layak.

---

## 5. Referensi API

### 5.1 Global utilitas

| Fungsi | Hasil |
| --- | --- |
| `log(level, message)` | Log pada level eksplisit (`info`, `warn`, `error`, …) + publish ke console |
| `print(...)` | Argumen digabung tab, di-log pada level `info` |
| `sleep(ms)` | Yield body script; di tempat lain diabaikan + warning |
| `sylphieVersion()` | String versi Sylphie yang berjalan |
| `now()` | Jam monotonik **milidetik** (bukan Unix seconds) |
| `getBotId()` | Id bot script ini, atau `0` |
| `addBot{ name=, ltoken=\|token=\|secret= }` | Buat bot dari ltoken. Kembalikan objek bot, atau `nil, error` |
| `removeColor(text)` | Buang kode warna Growtopia |
| `runThread(fn, ...)` | Memanggil `fn(...)` inline (tidak ada thread kedua) |
| `getProxyManager()` | `{ mode = "socks5"\|"none", label = <jalur koneksi nyata> }` |
| `after(ms, fn)` / `every(ms, fn)` | Timer sekali / berulang. Kembalikan handle |
| `stopTimer(handle)` | Batalkan timer |
| `clearConsole()` | Kosongkan ring log bot yang terikat |

Contoh `addBot` (importer token):

```lua
local bot = addBot({ name = "worker1", secret = "token|mac|rid" })
if bot then bot:connect() end
```

`addBot` menerima ltoken dengan tiga ejaan — `ltoken`, `token`, atau `secret` —
karena daftar token yang beredar memakai ketiganya untuk field yang sama.
`details.token` boleh berupa satu baris token penuh bergaya
SylphieJS/Tempo (`public_key|token|mac|rid|hash|platform`), dan segmen `@name`
di depan dilewati. `name` dipakai sebagai kunci penyimpanan token; kalau kosong,
segmen token sendiri yang dipakai.

Bot **langsung masuk ke fleet** (muncul di halaman Bots dan tersimpan saat keluar),
tapi **belum login**. Panggil `bot:connect()` untuk membelanjakan satu login.

#### Enum

Tabel enum adalah `nama → string`, jadi `Platform.macos == "macos"` bernilai true:
`Platform`, `BotStatus`, `Proxy`, `VariantType`, `Event`.

### 5.2 Bot

```lua
local bot = getBot()          -- bot milik script ini
local bot = getBot("worker1") -- atau berdasarkan nama/id
local all = getBots()         -- semua bot yang dimiliki manager
```

Objek bot menjawab method (`bot:say(...)`) **dan** property (`bot.name`). Nama
di-resolve sebagai method dulu, lalu sebagai field hidup. Script juga boleh
menaruh field sendiri di objek bot (disimpan di side table, tidak error).

**Property:** `name`, `growid`/`username`, `status`, `world`, `netid`/`netId`,
`x`, `y`, `ping`, `gem_count`, `auto_collect`, `collect_range`, dan `level`
(selalu `nil` — tidak dilacak).

**Method:** `connect`, `disconnect`, `move`, `moveTile`, `moveTo`,
`moveLeft`/`moveRight`/`moveUp`/`moveDown`, `isInWorld`, `walkTo`, `punch`,
`hit`, `wrench`, `wrenchPlayer`, `place`, `drop`, `fastDrop`, `trash`,
`fastTrash`, `wear`, `use`, `unwear`, `buy`, `activate`, `active`, `enter`,
`retrieve`, `respawn`, `setDirection`, `face`, `isInTile`, `getPath`,
`findPath`, `findWorldPath`, `getWorld`, `getInventory`, `getLog`, `getConsole`,
`getLogin`, `getProxy`, `getPlaytime`, `getAge`, `getActiveTime`, `getSignal`,
`getCaptcha`, `setSkin`, `setCountry`, `setMac`, `setRid`, `updateBot`,
`runScript`, `stopScript`, `getPing`, `isWearing`, `collectObject`, `collect`,
`clearConsole`, `sendPacket`, `sendPacketText`, `sendRaw`, `say`, `join`,
`warp`, `leaveWorld`, `state`, `x`, `y`, `name`, `ping`, `getX`, `getY`.

Signature yang paling sering dipakai:

| Panggilan | Arti |
| --- | --- |
| `bot:connect()` / `bot:disconnect()` | Mulai / hentikan sesi (login lewat login gate) |
| `bot:say(text)` | Kirim chat (kena pacer Chat) |
| `bot:join(nama)` | Warp ke world — lewat `warp`, jadi dapat cek state + Warp pacer |
| `bot:warp(nama [, remember])` | Sama, dengan flag mengingat world untuk reconnect |
| `bot:leaveWorld()` | Keluar dari world sekarang |
| `bot:punch(x, y)` / `bot:hit(x, y)` | Pukul tile |
| `bot:place(x, y, itemId)` | Taruh item di tile |
| `bot:drop(itemId)` / `bot:fastDrop(id)` | Buang item dari tas |
| `bot:collectObject(uid)` / `bot:collect(uid)` | Ambil item jatuh berdasarkan uid |
| `bot:walkTo(x, y [, avoidItemId])` | Pathfinding A* ke tile |
| `bot:moveTile(x, y)` | Jalan lewat path bergaya Mori (sama dengan klik peta) |
| `bot:move(x, y)` | Bergerak menuju tile absolut |
| `bot:moveTo(dx, dy)` | Melangkah sebesar offset dari posisi sekarang |
| `bot:moveLeft(n)` … `moveDown(n)` | Langkah 1 tile (atau `n`) ke arah itu |
| `bot:getPath(x, y)` / `bot:findPath(x, y)` | Array langkah, atau `nil` |
| `bot:isInWorld([nama])` | Boolean |
| `bot:isInTile(x, y)` | Boolean |
| `bot:collectObject(uid)` | Butuh world + uid valid, kalau tidak `ITEM_NOT_FOUND` |
| `bot:sendPacket([type,] text)` | Kirim frame teks mentah |
| `bot:sendRaw(packetTable)` | Kirim GameUpdatePacket dari tabel Lua |

**Nama Lucifer/SylphieJS adalah alias, bukan implementasi kedua:** `hit` = `punch`,
`use` = `wear`, `active` = `activate`, `face` = `setDirection`, `findPath` =
`getPath`, `collect` = `collectObject`.

### 5.3 World

| Fungsi | Hasil |
| --- | --- |
| `getWorld()` | World bot yang terikat, atau `nil` |
| `getTile(x, y)` | Tabel tile, atau `nil` |
| `getTiles()` / `getTilesSafe()` | Array semua tile yang termuat |
| `getObjects()` / `getObject(uid)` | Item jatuh |
| `getPlayers()` | Pemain di world |
| `getNPCs()` / `getNPC(id)` | NPC |
| `isWalkable(x, y)` | Boolean |
| `findItem(nama)` | Id item, atau `nil` |
| `findPath(sx, sy, tx, ty)` | Array langkah, atau `nil` |

Objek world — field: `name`, `x`/`width`, `y`/`height`, `tile_count`, `version`,
`objects`, `players`, `npcs`, `tiles`, `public`.
Method: `getTile`, `getTiles`, `getTilesSafe`, `getObject`, `getObjects`,
`getPlayer(key)`, `getPlayers`, `getNPC(key)`, `getNPCs`, `getLocal`,
`getTileParent`, `isValidPosition`, `hasAccess`, `hasAdventure`, `getAdventures`,
plus alias `.name()`, `.width()`, `.height()`, `.version()`, `.tileCount()`.

**Field tile:** `fg`/`foreground`, `bg`/`background`, `flags`, `parent` (sebuah
*index* tile, bukan id item), `object` (jenis extra-data), `foreground_name`.

**Method tile:** `hasExtra()`, `getExtra()` (`{ kind = n }` atau `nil`),
`canHarvest()` (true bila extra-data-nya data seed), `hasFlag(flag)`,
`getParent()` (index mentah).

**Field item jatuh:** `id`, `count`, `flags`, `oid` (uid), `x`, `y` — dalam
**pixel world**, bukan tile.

**Field NPC:** `type`, `id`, `x`, `y`, `destx`, `desty`, `var`, `unk`.

### 5.4 Player

| Fungsi | Hasil |
| --- | --- |
| `getLocal()` | Tabel pemain lokal |
| `getPlayer(idOrName)` | Tabel pemain, atau `nil` |
| `getPlayers()` | Semua pemain di world |
| `getPlayerCount()` | Jumlahnya |
| `findPlayer(nama)` | Tabel pemain, atau `nil` |
| `itemName(id)` | Nama item, atau `item#<id>` |

**Field pemain:** `id`, `netid`, `name`, `country`, `userid`, `x`, `y`, `posx`,
`posy` (sama dengan `x`/`y` — model menyimpan tile), `direction`,
`isLocalPlayer`. `altName` selalu string kosong (tidak ada di wire). `state` dan
`local` ada pada pemain dari `getLocal`/`getPlayer`/`getPlayers`/`findPlayer`.

### 5.5 Inventory

| Fungsi | Hasil |
| --- | --- |
| `getInventory()` | Handle inventory, atau `nil` |
| `getItem(id)` | `{ id, name, category, slot, count }`, atau `nil` |
| `getItemCount(id)` | Total count untuk sebuah id item |
| `getItems()` | Array `{ slot, id, name }` untuk slot terisi |
| `selectItem(id)` | Taruh item ke slot aktif |
| `searchItems(query)` | Sampai 64 id item yang namanya cocok |
| `getInfo(idOrName)` | Record item, atau `nil` |
| `getInfos()` | Semua record item yang dikenal |

> **Id item adalah number, bukan nama.** `getItem` dan `getItemCount` menerima id.

Handle inventory: `used()`, `free()`, `getItem(id)`, `getItems()`,
`getItemCount(id)`, `findItem(id)`, `canCollect(id)`; field `itemcount`,
`slotcount`, `version`, `items`. `getItemCount` menjumlah lintas slot, jadi item
yang terpecah dua slot dilaporkan sebagai total gabungan.

### 5.6 Packet

| Fungsi | Hasil |
| --- | --- |
| `sendPacket(netId, fields)` | Kirim `fields` (tabel key → value) sebagai variant list pada frame `netId`. `(ok)` atau `(false, error)` |
| `sendAction(kind, x, y [, itemId])` | `kind` = `punch`, `place`, `walk`, atau `drop`. `itemId` opsional (default `0`) |
| `packetStats()` | `(packetsSent, packetsReceived)` |
| `packetNames()` | Array `{ id, name }` untuk setiap net id yang dikenal |

Perhatikan arti argumen per `kind`: untuk `walk`, `x`/`y` adalah tile tujuan;
untuk `drop`, `itemId` adalah item yang dibuang dan `x`/`y` tidak dipakai; untuk
`punch`/`place`, `x`/`y` adalah tile target.

Net id dan nama CallFunction adalah bagian protokol wire. **Jangan menebak** —
baca daftarnya dari `packetNames()`.

### 5.7 Kelas protokol

- `Vector2.new(x, y)`, `Vector3.new(x, y, z)`
- `variant(...)` dengan `set`/`reset`/`getType`/`getString`/`getInt`/`getFloat`/
  `getVector2`/`getVector3`/`print`/`toString`
- `variantlist(...)` dengan `get`/`append`/`print`
- `GameUpdatePacket.new()` — tabel packet mutable yang diterima `bot:sendRaw`
- `RTVAR` — `new([str])`, `params`, `append`, `contains`, `size`, `get`,
  `getInt`/`getLong`, `erase`, `dump`, `parse`

### 5.8 HTTP & file

| Fungsi | Hasil |
| --- | --- |
| `http.get(url)` | `(ok, statusCode, body)` |
| `http.post(url, body [, contentType])` | `(ok, statusCode, body)` |
| `file.read(path)` | Isi file, atau `nil, error` |
| `file.write(path, contents)` | `true`, atau `false, error` |

Dua batas yang penting dan jujur:

- `file.read` / `file.write` **terkunci di dalam script root** (`scripts/`).
  Path apa pun yang keluar dari folder itu ditolak (`path escapes the script
  directory`).
- `http.get` / `http.post` bersifat *fire and forget*: request diserahkan ke task
  pool, dan yang dikembalikan hanyalah hasil **pengiriman**. `statusCode` masih
  `0` dan `body` masih `""` — respons server belum disalurkan ke script. Jangan
  bangun logika di atas body-nya.

---

## 6. Contoh siap pakai

### 6.1 Auto-reconnect

```lua
local bot = getBot()
if not bot then
    log("error", "script ini tidak punya bot")
    return
end

while true do
    sleep(200)
    if bot.status ~= "online" then
        log("warn", "status " .. bot.status .. " — coba connect")
        bot:connect()
        sleep(5000)   -- beri waktu login; login gate tetap mengatur
    end
end
```

### 6.2 Farm sederhana (petik + buang hasil)

```lua
local bot = getBot()
if not bot then return end

-- Id item stabil di items.dat; beri nama di atas supaya mudah diganti.
local DIRT  = 16
local CHEST = 415

local SIDE    = 4     -- tile per sisi
local HARVEST = 100   -- ms antar pukulan
local SETTLE  = 150   -- ms jeda setelah pukul

local function step(dx, dy)
    -- walkTo memakai pathfinding; teleport jauh ditolak server.
    bot:walkTo(bot:x() + dx, bot:y() + dy)
    sleep(HARVEST)
end

for pass = 1, 2 do
    local dir = pass == 1 and 1 or -1
    for _ = 1, SIDE do
        step(dir, 0)
        bot:punch(bot:x(), bot:y())   -- berdiri di atasnya = drop terambil
        sleep(SETTLE)
    end
    for _ = 1, SIDE do
        step(0, dir)
        bot:punch(bot:x(), bot:y())
        sleep(SETTLE)
    end
end

if getItemCount(CHEST) > 0 then
    bot:drop(CHEST)
    log("info", "hasil dibuang ke chest")
end
```

### 6.3 Bereaksi ke dunia (event, tanpa polling)

```lua
local bot = getBot()
if not bot then return end

function onWorldJoin(world)
    log("info", "masuk " .. world.name .. " (" .. world.width .. "x" .. world.height .. ")")
end

function onChat(player, message)
    log("info", "<" .. player.name .. "> " .. message)
end

function onDisconnect(event)
    log("warn", "terputus: " .. tostring(event.reason))
end

local seen = 0
listen("packet", function(event)
    seen = seen + 1
    lastKind = event.type_name
end)

-- Body utama harus tetap hidup, kalau tidak handler ikut hilang.
while true do
    sleep(5000)
    if seen > 0 then
        log("info", "packet: " .. seen .. " (terakhir " .. tostring(lastKind) .. ")")
        seen = 0
    end
end
```

### 6.4 Rotasi world

```lua
local WORLDS  = { "FARM", "BLAZE", "PENJORY" }
local DWELL_MS = 30000   -- lama tinggal sebelum pindah

local bot = getBot()
if not bot then return end

local index = 0
while true do
    index = index % #WORLDS + 1
    local target = WORLDS[index]
    log("info", "warp ke " .. target)

    -- join hanya mengirim request; world datang beberapa saat kemudian,
    -- jadi sleep di bawah satu-satunya jaminan urutan.
    bot:join(target)
    sleep(DWELL_MS)
end
```

### 6.5 Importer token massal

```lua
local TOKENS = {
    { name = "worker1", token = "token|mac|rid" },
    { name = "worker2", token = "token|mac|rid" },
}

for _, entry in ipairs(TOKENS) do
    local bot = addBot(entry)
    if bot then
        log("info", "menambah " .. bot.name)
        bot:connect()
        sleep(8000)   -- login gate yang mengatur; beri napas
    end
end
```

### 6.6 Timer

```lua
local handle = every(60000, function()
    log("info", "satu menit berlalu")
end)

-- Nanti, kalau mau berhenti:
-- stopTimer(handle)
```

---

## 7. Sandbox & batas

Setiap state dibuat dengan membuang bagian standard library yang tidak ada
urusannya dengan script bot:

```
io  os  package  loadfile  dofile  load  loadstring  require  collectgarbage
debug  rawequal  rawget  rawset  rawlen  setmetatable  getmetatable  module
string.dump  next
```

`coroutine` tetap ada. `print` dan `log` selalu tersedia. **Tidak ada tabel
`console`** — `console.log` tidak ada, pakai `log`/`print`.

Memori dibatasi allocator kustom: default **64 MB**, diatur di kode
(`LuaRuntime.hpp`), bukan config key. Penolakan diterjemahkan Lua menjadi error
out-of-memory.

| Hal | Batas |
| --- | --- |
| `file.read` / `file.write` | Path harus tetap di bawah script root |
| `http.get` / `http.post` | Hanya hasil submit yang dikembalikan; respons tidak sampai ke script |

---

## 8. Yang sengaja no-op (jujur)

Mesin tidak memalsukan fitur yang tidak dimilikinya:

- `bot:getSignal()` dan `bot:getCaptcha()` mengembalikan `nil`.
- `bot:runScript(content)` mengembalikan `false` + alasan: satu body kooperatif
  per script, jadi pakai script terpisah.
- `bot:stopScript()` tidak melakukan apa-apa; script berhenti sendiri saat
  `return`.
- `bot:getLogin()` melaporkan `mac`/`rid` kosong.
- `bot:setMac`, `bot:setRid`, `bot:updateBot`, `bot:setCountry` mengembalikan
  `true` tanpa mengubah apa pun. `bot:setSkin(1..6)` divalidasi lalu diterima
  untuk reconnect berikutnya (skin sampai ke server lewat identitas login, bukan
  lewat packet).
- `runThread` memanggil callback secara inline; tidak ada thread kedua.
- `world:hasAdventure()` selalu `false`; `world:getAdventures()` kosong.
- `bot.level` selalu `nil`.
- `listenEvents(secs)` adalah no-op yang mengembalikan ms (listener memang
  mengirim setiap frame).

---

## 9. Error

Error script dilaporkan ke console dan log, lalu script **dihentikan**, bukan
dibiarkan rusak. Kode error:

```
LUA_RUNTIME_ERROR      LUA_SCRIPT_NOT_FOUND   LUA_SCRIPT_EXISTS
LUA_SYNTAX_ERROR       LUA_TIMEOUT            LUA_SANDBOX_VIOLATION
LUA_MODULE_ERROR       LUA_SCRIPT_NOT_RUNNING
```

Status aksi bot (dari `(ok, err)`) juga string mesin yang stabil, mis.
`BOT_ACTION_RATE_LIMITED`, `BOT_ALREADY_IN_WORLD`, `BOT_NOT_FOUND`,
`BOT_NO_WORLD`, `ITEM_NOT_FOUND`, `PLAYER_NOT_FOUND`, `INVALID_ARGUMENT`.

**`BOT_ACTION_RATE_LIMITED` bukan kegagalan.** Artinya aksi ditolak pacer
sebelum dikirim (tidak ada yang setengah jalan). Diam sebentar, lalu coba lagi.

---

## 10. Config keys

| Key | Default | Efek |
| --- | --- | --- |
| `scripts.instruction_budget` | `2000000` | Instruksi per resume; `0` mematikan hook |
| `scripts.script_root` | `scripts` | Root yang membatasi `file.read`/`file.write` |

Key berikut **dibaca tapi belum mengubah apa pun**: `scripts.autorun`,
`scripts.allow_file_io`, `scripts.allow_http`, `scripts.allow_console`. Jangan
mengandalkan namanya — tulis script yang membuktikan behavior-nya.

---

## 11. Tips & jebakan

- **Selalu periksa `(ok, err)`** untuk aksi yang mengirim paket.
- **Jangan `sleep()` di dalam callback.** Tidak ada coroutine untuk di-yield.
- **Jaga body utama tetap hidup** kalau memakai handler (`while true do sleep(...) end`),
  kalau tidak script dianggap selesai dan handler mati.
- **`tile_change` dan `packet` sangat ramai** (ribuan per menit saat farm) —
  hitung, jangan log per kejadian.
- **Id item adalah number.** Beri nama lewat `local DIRT = 16` di atas file.
- **Pacing adalah teman.** Biarkan `sleep()` tetap ada.
- **`join` sekarang lewat `warp`**, sehingga kena cek state, cek "sudah di world
  itu", dan Warp pacer. Loop `join()` tanpa jeda tidak akan bekerja seperti dulu.
- **Warp membatalkan automation**: task yang menyimpan daftar tile/posisi pemain
  dari world lama tidak lagi valid setelah pindah world.

---

## Lisensi & kontribusi

Dokumen ini mengikuti aturan gaya repo. Untuk perilaku protocol/wire dan detail
yang diverifikasi lewat capture, lihat `docs/` (khususnya `docs/scripting.md`,
`docs/protocol.md`, dan `docs/packet-capture.md`).
