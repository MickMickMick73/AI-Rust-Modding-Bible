# The AI Rust-Modding Bible
## Part C — Grok Agent Ruleset

*This document is written for an AI coding agent, not a human. It is **Grok's
half** of the technical bible (xAI), from live Oxide/Carbon work: MixMods AU,
MixMenuKit under real marketplace review, Salvo, MixCore, MixSprint, MixGovern,
MixSignboard, MixRaidBases, OSAutoTurrets, and the rest of that pack.*

*Part B (Claude) and Part C (Grok) are the same *kind* of document — verified
rules, complete examples, evidence attached, uncertainty marked. They are not
the same *discoveries*. Two agents hit different walls on the same project.
Load **both** before you write or edit a Rust plugin. Do not treat this file
as a replacement for Part B, and do not treat Part B as covering the items
below.*

*Attribution: Grok (xAI), 2026, working with Mick (MiX Apps). MIT, same as
the rest of this repo.*

*Living document. Section 22 is how to add to it. Same honesty bar as Part B
§10: verified or say you don't know.*

---

## 0. How to use this with Part B

Part B already owns, and you must still follow:

- Plugin skeleton, file-scoped `namespace Oxide.Plugins;`, version only in `[Info]`
- The base Ricky config read/save (no `.bak`, no silent in-memory preserve)
- Oxide CUI as the default, not Carbon-native
- Input-field stale-echo debounce
- Self-healing pagination
- No server flag for "own inventory is open"
- Reload ≠ guaranteed disk reread
- Native+plugin keybind desync
- `PeekState` on global hooks
- Hygiene, security baseline, submission screenshots / no config-in-zip

Part C owns what Grok actually broke, shipped wrong, or proved on a live
server after Part B's floor was already in play. If a rule below collides
with Part B, **do not silently overwrite Part B.** Append, and say what
changed — same as Part B §10.4.

---

## 1. Config — leftover fields, not a second migrate

Part B §2 is the happy path. Part B §3 is *when* migration is legal (only if
a real old format is in the wild). This section is *how* Grok was told to
migrate a **shipped** shape break, after an extra JObject / `.bak` stack was
explicitly rejected.

**Evidence:** MixMenuKit went from a v1 `Buttons` list to a v2 `Menus` list.
A marketplace reviewer (Ricky) supplied the leftover-field pattern and said
to implement it **exactly** — no `BackupCorruptConfig`, no second migrate,
no extra clamp/save in the catch. Agent-added extras were stripped. Token
compare against the reviewer's snippet was True (indent only, because nested
in the plugin class). The house auditor was flipped: `.bak` in `LoadConfig`
is now `CONFIG_BAK_CLUTTER`; **missing** `.bak` is not a defect.

```csharp
private sealed class PluginConfig
{
    [JsonProperty("NextId")] public int NextId = 1;
    [JsonProperty("Menus")] public List<MenuDef> Menus = new();

    // v1 leftover — only because this plugin already shipped the old shape.
    // Greenfield plugins: do not add this field at all.
    [JsonProperty("Buttons", NullValueHandling = NullValueHandling.Ignore)]
    public List<MenuButton> Buttons;

    public static PluginConfig DefaultConfig() => new()
    {
        NextId = 2,
        Menus =
        {
            new MenuDef { Id = 1, Name = "MAIN MENU", Trigger = "main", Default = true, Public = true }
        }
    };
}

protected override void LoadConfig()
{
    base.LoadConfig();
    try
    {
        _config = Config.ReadObject<PluginConfig>();
        if (_config == null) LoadDefaultConfig();
        if (_config.Menus == null) _config.Menus = new List<MenuDef>();

        if (_config.Menus.Count == 0 && _config.Buttons != null && _config.Buttons.Count > 0)
        {
            _config.Menus.Add(new MenuDef
            {
                Id = 1,
                Name = "MAIN MENU",
                Trigger = "main",
                Default = true,
                Public = true,
                Buttons = _config.Buttons
            });
            _config.Buttons = null;
            PrintWarning("Migrated v1 Buttons into Menus.");
        }

        if (_config.Menus.Count == 0) LoadDefaultConfig();
        SaveConfig();
    }
    catch (Exception ex)
    {
        Debug.LogException(ex);
        PrintWarning("Creating new configuration file.");
        LoadDefaultConfig();
    }
}

protected override void LoadDefaultConfig() => _config = PluginConfig.DefaultConfig();
protected override void SaveConfig() => Config.WriteObject(_config, true);
```

**Deliberately not here:** `File.Copy` to `*.bak`, a `MigrateV1` method that
re-reads `JObject`, a `hadConfig` guard, a second save in the catch. Those
were written, reviewed, and removed. If you re-add them "to be safe," you
are undoing a reviewer's call.

**Greenfield:** no leftover fields, no migrate block. `ReadObject<T>` already
ignores unknown keys and fills missing ones from C# defaults. That is enough
for additive changes.

---

## 2. Do not run two copies of the same plugin

**Evidence:** MixMenuKit 2.x and 3.0 must not load together. Same class name,
same hook table, same CUI names, same data files. Carbon/Oxide will compile
both if they sit in `plugins/` under different filenames. The live symptom is
not always a clean "duplicate type" — it is split UI, split config, and
"which one is actually handling `/menu`?"

```
# Wrong
oxide/plugins/MixMenuKit.cs
oxide/plugins/MixMenuKit-3.0.cs

# Right
one file, one version, [Info] version is the only version string
```

**Also:** do not "improve" a frozen marketplace Submission folder in place.
Copy out, edit the copy, leave the submitted zip's source as the review
artifact. That is process, not style — a reviewer diffing against what they
already saw will otherwise chase ghosts.

---

## 3. Carbon compile is a ship gate, not a suggestion

**Evidence:** a MixCore patch adding `API_GetHubRoute` hung Carbon compile.
The plugin dropped out of `c.plugins`. The live pack was restored from
`MixCore.cs.bak-hubapi-…` and the Carbon service was restarted. Players were
kicked. The API was not needed — existing MixCore hub APIs already did the
job.

**Rules:**

- After any Core/shared-plugin edit on Carbon: wait until `c.plugins` lists
  it again, with **0 compile failures**, before you call the deploy done.
- Do not add unused public `API_*` surface "for later" on a plugin that
  every other plugin in the pack already depends on. A compile hang there
  takes the hub down, not just your experiment.
- A hung compile is not fixed by `c.reload`. Restart the host process
  (Part B §5.2 is the cousin of this: in-memory state lies).

```bash
# Carbon example — confirm YOUR unit name, do not copy blindly
systemctl restart rust-mixmods-au-carbon
# then
c.plugins
# the edited plugin must be present; compile-fail count must be 0
```

Uncertain on purpose: Oxide `oxide.reload` after a failed compile is a
different host. Do not assume the hang reproduces the same way on Oxide.
The rule that *does* travel: **never declare a shared-plugin deploy green
from the file copy alone.**

---

## 4. Object-return hooks cancel on non-null

**Evidence:** Oxide/Carbon object hooks (`object OnPlayerAttack`,
`object OnEntityTakeDamage`, …) treat **any non-null return as cancel**.
`return false` and `return true` both cancel. To let vanilla continue,
return `null` or use a `void` hook. Verified against Salvo's attack/damage
hooks (they already returned correctly; the bug class is what *other*
plugins do when they `return false` thinking that means "I did not handle
this").

```csharp
private object OnEntityTakeDamage(BaseCombatEntity entity, HitInfo info)
{
    if (entity == null || info == null) return null;
    if (!IsOurThing(entity)) return null;   // not us — do not cancel
    ApplyOurThing(entity, info);
    return true;                            // cancel vanilla — we handled it
}
```

**Multi-plugin:** if two loaded plugins both return non-null on the same
hook, the host logs a hook-conflict and **one result wins**. That is not
fixable from inside one plugin. If a customer says "your rocket plugin
fights my other rocket plugin," this is the first question, not a rewrite
of your math.

---

## 5. CUI lifecycle — destroy, cursor, connect timing

Part B §4 is how to *draw*. This is how CUI *leaks*.

### 5.1 Unload / disconnect must actually destroy

**Evidence:** an external report flagged MixHud, MixSignboard, and ServerLogo
as "Unload does not DestroyUi." All three were **false positives** — they
route through `KillUi` / `CloseAdminPanel` / `DestroyUI`. MixRaidBases was
**real**: `Unload()` never called its own `ClosePanel`. Anyone with the admin
panel open during a live reload kept a `CursorEnabled = true` dim overlay.
Mouse-look stayed locked until reconnect.

```csharp
private const string UiRoot = "PluginName.Root";

private void CloseUi(BasePlayer player)
{
    if (player == null) return;
    CuiHelper.DestroyUi(player, UiRoot);
}

private void Unload()
{
    foreach (var player in BasePlayer.activePlayerList)
        CloseUi(player);
    // kill timers here too
}

private void OnPlayerDisconnected(BasePlayer player, string reason) => CloseUi(player);
```

When you audit Unload, **trace the helper**. A grep of `Unload()`'s body for
the literal `CuiHelper.DestroyUi` will lie.

### 5.2 `CursorEnabled = true` is a reconnect-level bug if it leaks

Grep every `CursorEnabled = true`. Each one needs a destroy path that runs
on close, death, disconnect, and Unload. There is no "the next AddUi will
clear it."

### 5.3 Destroy before add; fresh container per draw

Repeated `AddUi` of the same root name **without** a prior `DestroyUi` stacks
elements client-side. Root name must be a **const string**, identical on add
and destroy. Do not keep a plugin-level `CuiElementContainer` that code paths
keep `.Add()`-ing to — it grows forever. `new CuiElementContainer()` per
draw.

### 5.4 Do not draw CUI in `OnPlayerConnected`

The client is still eating its world snapshot. CUI can silently not appear.
**Evidence:** MixHud (2s) and MixCore (2.5s) already delay and re-validate.
Make that explicit so the next plugin does not skip it.

```csharp
private void OnPlayerConnected(BasePlayer player)
{
    timer.Once(2f, () =>
    {
        if (player == null || !player.IsConnected) return;
        DrawHud(player);
    });
}
```

A tighter version polls `player.IsReceivingSnapshot` and `NextTick`s until
clear. The flat delay is the one that is already in production and known
good. Do not draw from the connect hook itself "because it's simpler."

### 5.5 `DestroyUi` is an unconditional network RPC

It does **not** check whether the element exists. Do not call it from a
server-wide hot hook (every hotbar scroll, every item move) for a panel you
are not even drawing. That is not cleanup. That is spam.

---

## 6. Hub CUI vs native loot

Part B §5.1: the server cannot see "own inventory open." Part B §5.3: binds
that mix `inventory.toggle` and `chat.say /menu` desync on loot.

**Grok's extra, live:** MixPlaytest against MixCore hub. Shop / kit / balance
/ home **passed**. Opening the **backpack** while the hub overlay was up
**failed** — a stash spawned ~50 m below the player, loot RPC fought the
hub CUI. `API_IsGod` stayed false for a separate reason (admin perm), but
the backpack miss was specifically overlay vs loot container.

**Rule:** if your plugin owns a full-screen hub, **close it before** you
open a loot panel, stash, or any `player.inventory.loot` session. Do not
stack native loot UI under a CursorEnabled hub and call the click "tested"
because a chat command returned ok.

```csharp
private void OpenBackpack(BasePlayer player)
{
    CloseHub(player);                       // first
    NextTick(() =>
    {
        if (player == null || !player.IsConnected) return;
        // then start the loot session
    });
}
```

---

## 7. `Init` vs `OnServerInitialized` vs `Unload`

| Hook | Put here | Do not put here |
|------|----------|-----------------|
| `Init` | `RegisterPermission`, local dicts, `Unsubscribe` of *optional* hooks | `Plugins.Find`, `[PluginReference]` use, monument scans |
| `OnServerInitialized(bool initial)` | Cross-plugin refs, world caches | Assuming `initial` is always true — it is **false** on hot reload |
| `Unload` | Destroy UI, kill timers, `SaveData` | Starting new timers, talking to other plugins that may already be gone |

**Evidence:** MixCore vs MixGovern both registering `mixpack.*` looked fine
across many `c.reload`s (the loser had already claimed the permission in the
long-running process). A real `systemctl restart` produced a clean,
repeatable "already used by another plugin" warning. **A hot reload is not
a load-order test.** Part B §8 asks for three reloads. This section adds:
if the change touches permission registration or `[PluginReference]` timing,
**restart the host** once as well.

`Init` is also where optional combat hooks should `Unsubscribe` if the
feature is config-off, so they do not pay dispatch forever. Do not
Unsubscribe a hook that *is* the plugin's job.

---

## 8. `entity.net` is not `entity`

**Evidence:** MixSignboard `OnEntityKill` did `if (entity == null) return;`
then `entity.net.ID.Value`. Not every `BaseNetworkable` is networked.
Universal entity hooks (`OnEntityKill`, `OnEntitySpawned`, `OnEntityDeath`)
will eventually hand you one with `net == null`. That is a
NullReferenceException on a hot path, not a rare edge.

```csharp
private void OnEntityKill(BaseNetworkable entity)
{
    if (entity?.net == null) return;
    var id = entity.net.ID.Value;
    // ...
}
```

Grep `.net.ID` / `.net.` inside every `OnEntity*` and confirm the guard is
`entity?.net`, not only `entity`.

---

## 9. Live admin toggles must save in the same click

**Evidence, twice in one session:**

- MixSprint had **no** master disable at all.
- Salvo's admin ON/OFF (`ApplyMasterToggle`) flipped `_config.Enabled`,
  applied it live, and **never called `SaveConfig()`**. A separate "Save"
  control a few lines away did. Admin toggles off, confirms it works, later
  `c.reload` from an unrelated deploy silently turns it back on (or off).
  Fixed in Salvo 2.9.3 by putting `SaveConfig()` in `ApplyMasterToggle`.

```csharp
private void ApplyMasterToggle(BasePlayer admin)
{
    if (!IsAdmin(admin)) return;
    _config.Enabled = !_config.Enabled;
    ApplyLive();
    SaveConfig();                           // same action, not a second button
}
```

Grep every `_config.X = !_config.X` (and every admin setter that mutates
`_config`). If `SaveConfig()` is not in that method, it is this bug.

---

## 10. Dangerous bypasses default false — including live JSON

**Evidence:** `DevServerOpenPanelToAll` (and anything shaped like it) skipped
the admin perm check. MixSprint and OSAutoTurrets shipped the **C# field
default as `true`**. MixCore / MixSignboard / Salvo / MixEntityScale /
FreeBuild correctly defaulted `false`. The live server's **saved JSON** also
had `true`, so fixing the class default alone did **nothing** until the
config file was patched and the plugin reloaded.

```csharp
[JsonProperty("DevServerOpenPanelToAll")]
public bool DevServerOpenPanelToAll = false;   // never true
```

**Ship rule:** a bool that bypasses a permission check defaults `false` in
the class, in `DefaultConfig()`, and you grep live configs after the fix.
A "dev convenience" that ships on is an open admin panel for every player
who loads the plugin once.

---

## 11. Permissions — register, grep the string, deny for real

### 11.1 Register in `Init`

Every string you pass to `UserHasPermission` is registered with
`permission.RegisterPermission(..., this)` in `Init`. Unregistered perms
fail silent or throw depending on host version. Not a compile error.

### 11.2 Grep the **string**, not the call shape

**Evidence:** `mixpack.admin` / `mixpack.use` looked unregistered if you
grepped `RegisterPermission(PermAdmin`. MixGovern registers a 30-entry
`ManagedPerms` table in a loop. Acting on the false positive (registering
again from MixCore) created the load-order warning in §7. If a perm looks
missing, grep the **literal string** across every plugin in the pack first.

### 11.3 Checks use `UserIDString`

```csharp
permission.UserHasPermission(player.UserIDString, PermUse)
```

Admin-power commands (god, noclip, vanish, entity wipe) need a **second**
gate (`PermAdmin` / vanilla auth), not "has plugin.use." Part B §7 already
says this. Grok's live miss: MixPlaytest `API_IsGod` stayed false until
`mixpack.admin.god` was in play. **Uncertain:** a steam-id vs `player.userID`
mismatch was *suspected* on that fail, not proven. Do not copy that guess
as fact. Do copy: god is an admin power, test it with the admin perm
**absent** as well as present.

### 11.4 Deny-path actually denies

Category "perm is registered" ≠ "the console command is gated." Call the
underlying `ConsoleCommand` without the perm. The action must not happen.
Hiding a button in CUI is not a security boundary.

---

## 12. Hot paths

Hooks that fire per shot / per tick / per item-move:

1. Cheapest exclusion first (null, wrong item type) **before** permission
   lookups, rank resolves, LINQ.
2. Resolve expensive things **once** per invocation, reuse the local.
3. No `new List<>` / `new Dictionary<>` / LINQ in those hooks.
   `Pool.Get<List<T>>()` / `Pool.FreeList()` if you truly need a scratch
   list. Verified: Salvo's `new List<string>` sites were all cold (startup,
   disconnect cache, equip edge) — none in per-shot hooks. Keep it that way.
4. No `SaveData()` / `Config.WriteObject` from those hooks unless a value
   **actually changed**, and even then debounce. Unconditional disk I/O
   per shot is how a plugin dies in a 200-pop fight.
5. Optional feature hooks: `Unsubscribe(nameof(OnWhatever))` when config-off
   (see §7). Core-job hooks stay subscribed.

---

## 13. Per-player state and reload stress

Every `Dictionary<ulong, …>` / `HashSet<ulong>` keyed by user id:

- `.Remove` in `OnPlayerDisconnected` (or `OnUserDisconnected`)
- clear in `Unload`
- every per-player `Timer` / coroutine destroyed on both

A short test session will never show the leak. A long wipe will.

**Reload stress (Grok bar, on top of Part B's three reloads):**
`c.reload PluginName` **ten** times, then `c.plugins` — hook-exception count
and memory should be flat vs a single reload. If UI or timers duplicate per
reload, memory climbs. Ran against Salvo 2.9.2: 10× reload, asset count
stuck at 21, memory 3.0 mb, zero exceptions. That is the shape of a pass.

---

## 14. Commands, args, delayed callbacks

- Prefix console commands (`salvo.`, `mixmenu.`, `mixcore.`). Bare `yes` /
  `no` / `/kit` / `/home` silently overwrite whoever registered second. No
  error. Verified across 17 live MiX plugins: everything prefixed except
  two low-risk chat commands (`/tod`, `/welcome`). Do not add a third.
- Every handler checks `args.Length` / `arg.Args` before indexing.
- `NextTick` / `timer.Once` must re-check `player != null && player.IsConnected`
  **at fire time**, not only at capture. They may have disconnected.

```csharp
[ConsoleCommand("mixmenu.page")]
private void CPage(ConsoleSystem.Arg arg)
{
    var player = arg.Player();
    if (player == null) return;
    if (!permission.UserHasPermission(player.UserIDString, PermUse)) return;
    if (arg.Args == null || arg.Args.Length < 1) return;
    var dir = arg.GetInt(0, 0);
    // ...
}
```

---

## 15. Cross-plugin `Call` and "standalone"

A null-check on `[PluginReference] Plugin Foo;` only proves Foo is loaded.
It does **not** catch Foo throwing inside `Foo.Call(...)`.

**Evidence:** MixSignboard had 11 `MixImages.Call(...)` sites null-checked
and **none** wrapped. One of them was inside a CUI draw loop (`DrawThumb`).
One bad thumbnail would have broken the whole gallery, not one cell.

```csharp
object thumb = null;
try
{
    if (MixImages != null)
        thumb = MixImages.Call("GetImage", id);
}
catch (Exception ex)
{
    Debug.LogException(ex);                 // this cell fails; the panel still draws
}
```

If you **sell** the plugin as standalone, every `[PluginReference]` is either
absent or has a working fallback for a **core advertised feature**. Cosmetic
chrome may degrade. The feature on the store page may not. MixSignboard
slideshow images vs Salvo UI-kit chrome were the two reference verdicts —
same pattern, opposite call, because one was core.

---

## 16. Facepunch DLC / skins (July–August 2025)

**Evidence:** Facepunch *Community Server and Hosting Guidelines*, updated
15 July 2025, enforced in the wild from ~7 August 2025. Servers must not
grant Facepunch DLC (paid packs **or** approved Item Store / Marketplace
skins) to players who do not own it. Custom / **unaccepted** Steam Workshop
content is explicitly **not** DLC. Delist (or worse) on breach. Salvo 2.9.2
**removed** `LauncherSkinId` rather than half-build an ownership check for
a cosmetic. Player DLC API + SkinBox/Skinner in the community did the
filter-not-pirate version.

```
Facepunch DLC     = paid packs + accepted store/market skins
Not DLC           = unaccepted Workshop IDs, custom content Facepunch does not sell
```

**If your plugin assigns a skin ID to a player's item:**

- Default: do not apply a configurable store skin at all, **or** filter
  through a real ownership API (Player DLC API or equivalent).
- A "free Workshop pack" must be **rechecked**, not frozen at build time.
  Facepunch can accept a previously-free Workshop skin later. A stale ID
  list starts handing out paid DLC.
- Ship **Workshop IDs** (and/or a collection id). Do **not** zip the
  PNG/VTF files into a paid plugin download. The game already fetches
  Workshop art by id. Bundling the files is a worse Steam-subscription
  risk and you do not need them for the feature.
- Other people's Workshop skins need their permission. Yours, unaccepted,
  used as IDs inside a **paid plugin** (you are selling the software, not
  Facepunch DLC) is the shape that matches the 2025 text.

This is a **compliance** axis. A perfect, clean skin-apply function can
still get a server delisted. Check it separately from code quality.

---

## 17. Wipe: `OnNewSave` is a decision, not a default

**Evidence:** audit 2026-08-30, **zero of 17** live-source MiX plugins
implemented `OnNewSave`. Salvo ranks, MixCore economy, MixGovern homes,
MixSignboard / FreeBuild / MixEntityScale stored data all survived a map
wipe **by omission**. That is not a design. That is forgotten.

Before first release, each `DataFileSystem` store gets an explicit call:

- **Reset** — per-map state (signs on this seed, bases, monument caches)
- **Keep** — progression / wallet / ranks, if you mean it and you say so
  in the listing

```csharp
private void OnNewSave(string filename)
{
    _data = new PluginData();
    SaveData();
}
```

An unimplemented `OnNewSave` is allowed only when someone can point to the
sentence in the product that says "data survives wipe." Otherwise add the
hook.

---

## 18. Chat.say is not a function test

**Evidence:** MixPlaytest against the AU Carbon pack. **137** chat/open
checks **PASS**. Function suite then: shop / kit / balance / home **PASS**;
backpack loot panel **FAIL**; god toggle **FAIL**. A plugin can look green
from `chat.say "/menu"` and still be wrong when a human clicks.

**Grok bar, on top of Part B §8 item 9:**

1. Open the UI for real (not only the chat command returning).
2. Click the control, do not only fire the console command from RCON.
3. If the click is supposed to open loot / a vanilla panel, watch the
   panel, not the chat log.
4. Test the **deny** path (perm absent).
5. Connect mid-snapshot → open UI → die → respawn → disconnect with UI
   open → reconnect. Confirm no stuck cursor, no orphan timer, no UI
   ghost. That sequence found more than any reload.

Do not write "live verified" in a listing from a chat-command harness
alone.

---

## 19. Engine truth — ILSpy and live fields, not wiki memory

Part B §5.1 found "no inventory-open flag" by reflecting `BasePlayer` in
three live states. Keep doing that.

Grok's tool on this pack: decompile the **actual** `Assembly-CSharp` the
server is running (protocol moves; private field names move). Do not copy
a field name from a two-year-old plugin and assume it still exists after
a force wipe.

If you cannot see the field on the live binary, you do not get to claim
the server "must have" a flag. Design around the public hooks.

---

## 20. Grok verification checklist

Run this with real tool calls. Part B §8 still applies; this is the extra
pass from Grok's misses.

1. Brace/paren counts both `== 0` on every touched file (Part B §6).
2. `c.plugins` after deploy: plugin **present**, compile-fail **0**. Shared
   plugins (Core) get a process restart if compile hung once this week.
3. Three reloads **and**, if perms / PluginReference / load order moved, one
   host restart.
4. Ten-reload memory/exception flatness on anything with timers or CUI.
5. Live click, not only chat.say. Deny-path. Connect/die/disconnect cycle.
6. Every new `_config` mutation from admin UI calls `SaveConfig()` in that
   method.
7. Every `OnEntity*` `.net` access is `entity?.net`.
8. Every `CursorEnabled = true` has Unload + disconnect destroy (via helper
   is fine).
9. Object hooks: `null` = passthrough, non-null = cancel. You meant it.
10. Skin IDs: unaccepted Workshop or ownership-gated. No stale store IDs.
11. `OnNewSave` decided, not forgotten.
12. Standalone listing: every PluginReference has a fallback or is gone.
13. Version only in `[Info]`, every deployed copy matches.
14. No `.bak` writer in `LoadConfig`. No leftover migrate on a plugin that
    has never shipped the old shape.
15. Config file on a server with real data still has that data after the
    deploy, not a default/test dump.

---

## 21. Submission extras Grok actually got wrong or almost shipped

Part B §9 still stands (no config in the zip, real screenshots, no "we used
to be broken" changelog). Extra:

- **Do not describe a bypass that ships `true` as a "dev option"** in the
  listing. If it can open the admin panel to everyone, it is a security
  issue, not a feature.
- **Do not ship two versions of the plugin in one zip.**
- **Do not put the reviewer's rejected extras back in** "for safety" after
  they told you to delete them. That is how MixMenuKit almost grew a second
  migrate.
- Changelogs: user-visible only. No names of reviewers, no "Grok/Claude
  found X." The buyer was not in that room.

---

## 22. How to extend Part C

Same contract as Part B §10, because that contract is the point of this
repo:

1. State what you **did** to find out (live hook, ILSpy, three-state
   reflect, 10× reload, actual click).
2. Complete code, not pseudocode.
3. Reasoning, so the next agent can extend it to a case this file missed.
4. Mark uncertainty. The god steam-id suspicion in §11.3 is the template:
   say "suspected, not proven" or leave it out.
5. **Do not overwrite Part B** to make room for a Grok correction. Append
   here, and say what differs.

If a finding is really Claude's, it belongs in Part B. If it is really
Grok's, it belongs here. The value of having two parts is that the trail
stays honest about **who hit what**. The moment Part C becomes a rewrite of
Part B, users lose that.

---

*End of Part C. Load Part A (why), Part B (Claude), Part C (Grok), then
build. The floor is the union. Testing is still the ceiling.*
