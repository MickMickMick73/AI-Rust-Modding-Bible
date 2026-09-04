# MiX Rust Mods — Pre-Submission QA Checklist

**Status: LIVING DOCUMENT.** This grows every time a review (Ricky's, mine, or anyone else's)
surfaces a bug class not already covered here. Never treat it as finished — treat gaps found
during a real review as a signal to add a new category, not just fix the one instance.

**How to use it:** run every item against the plugin's *lean* source (no embedded Base64 — that's
unreadable at scale) before it's considered ready for review or submission. Each item lists what
to grep/read for, not just the principle, so it's actually repeatable and not dependent on
"reading carefully."

---

## 1. Dead / orphaned code
- [ ] Every config settings-class field has at least one **read** site outside of
  `ClampConfig`/tune/serialization. Grep each `_config.X.Y` reference; if every hit is a write
  (clamp, tune stepper, default assignment), it's dead — remove it, don't wire it up reactively
  right before a submission (new untested behavior is its own risk).
- [ ] Every hook method (`On[A-Z]\w+\(`) actually does something in every branch. A hook whose
  body is a no-op for all inputs still costs Oxide/Carbon dispatch overhead on every firing —
  delete the subscription, not just the dead logic inside it.
- [ ] After removing/gutting *any* feature: grep every call site of the removed method, not just
  the definition. Check whether arguments computed for that call (e.g. `GetPlayerMag(player,
  false)`) have side effects worth keeping or removing too.
- [ ] **Any `bool` config field that bypasses a permission/admin check (`DevServerOpenPanelToAll`
  and anything shaped like it) must default to `false`** — grep `= true;` on every such field, not
  just whether it's read. A "dev testing" bypass that ships defaulting to *on* means every server
  that loads the plugin without noticing this one field has its admin panel open to every player,
  by default, out of the box. Real, live, confirmed-exploitable finding: both `MixSprint.cs` and
  `OSAutoTurrets.cs` shipped with `DevServerOpenPanelToAll = true` as the class default (not just
  a stray runtime toggle — the actual C# default), and the live server's saved config genuinely
  had it `true`. Fixing the class default alone does **not** fix an already-running server — the
  saved config file already has the field with the old value, so it must also be patched directly
  (same as any other live-config fix) and the plugin reloaded. Compare against every other plugin
  with the same field name in the pack; a field this dangerous should never vary silently between
  plugins that otherwise share the exact same pattern (MixCore/MixSignboard/Salvo/MixEntityScale/
  FreeBuild all correctly default `false` — MixSprint/OSAutoTurrets were the only two outliers).

## 2. Hot-path hook discipline
- [ ] For every hook that fires on a **server-wide** event (damage, attack, item-container,
  active-item-changed, weapon-fired) — not scoped to one entity/item first — read the first few
  lines. Cheapest/most-exclusionary check (null, item-type, `IsLauncherItem`-style) must run
  before any expensive call (`ResolveRank`, permission lookups, LINQ scans, `CanUse`).
- [ ] No hook calls the same expensive resolver (`ResolveRank`, permission lookup) more than once
  per invocation. Resolve once, reuse the local.
- [ ] For a hook that's only meaningful while some feature/config toggle is on (not core to the
  plugin — e.g. an optional damage-tracking add-on), consider `Unsubscribe(nameof(HookName))` in
  `Init`/on toggle-off rather than an early-return every firing. Doesn't apply to a hook that's
  core to the plugin's one job (Salvo's damage/attack hooks always matter while the plugin is
  loaded) — this is for genuinely optional side-features layered onto a plugin.
- [ ] Object-return hooks (`object On...`) cancel on **non-null**, not on a specific boolean value
  — `return false;` cancels exactly the same as `return true;` because Oxide checks null vs
  not-null, not the boolean's value. Verified Salvo's `OnPlayerAttack`/`OnEntityTakeDamage` already
  follow this correctly. The real risk this creates is a **multi-plugin conflict**: if another
  installed plugin hooks the same event and also returns non-null, Oxide logs a hook-conflict
  warning and only one plugin's result wins — not fixable from one plugin's code, but worth knowing
  if a customer reports "my other rocket-launcher plugin and Salvo fight" during support.
- [ ] **Never `Resources.FindObjectsOfType`/`FindObjectsOfTypeAll` from inside any hook.** A full
  scene-wide object scan, not a targeted lookup — community measurements going back to the Oxide
  forums in 2015 (still true in current Unity) show multi-hundred-millisecond pauses on a call
  that reads like an ordinary API. Maintain a plugin-owned `HashSet<T>`/`Dictionary<T,...>` built
  incrementally via `OnEntitySpawned`/`OnEntityKill` (or the equivalent connect/disconnect pair)
  instead, and iterate that. Checked immediately rather than left open: zero matches for either
  call anywhere in Salvo (both submission and META builds) or MixMenuKit — clean.
- [ ] **Every admin-facing toggle/setter that changes `_config` must call `SaveConfig()` in the
  same action that applies it live** — not just apply the in-memory change and rely on the admin
  separately clicking a distinct "Save & Apply" control. An admin who toggles a switch, sees it
  take effect immediately, and reasonably assumes that's the whole action has no way to know it
  silently reverts on the next plugin reload. Real, found live twice in one session: MixSprint had
  no way at all to disable the plugin (no toggle existed anywhere); Salvo's master ON/OFF admin
  toggle (`ApplyMasterToggle`) flipped `_config.Enabled` and applied it live but never called
  `SaveConfig()`, while a separate "save" action a few lines away did — so toggling Salvo on,
  confirming it worked, then hitting any later plugin reload (a `c.reload` from an unrelated
  deploy, a server restart) silently reverted it to off with no error and no obvious cause from
  the admin's side. Fixed by adding `SaveConfig()` directly into `ApplyMasterToggle()` (Salvo
  v2.9.3). Grep every method that flips a `_config.X = !_config.X`-style bool and confirm
  `SaveConfig()` is in the same call, not assumed to happen elsewhere.

## 3. Unconditional I/O in hot paths
- [ ] Every `SaveData()`/`Config.WriteObject`/`DataFileSystem.WriteObject` call site: is it
  reachable from a hook that can fire many times per second (per-shot, per-tick, per-item-move)?
  If so, is it gated on something actually changing, or does it fire unconditionally every time?
- [ ] Every `CuiHelper.AddUi`/`DestroyUi` call site: same question. `DestroyUi` is an unconditional
  network RPC — it does not check whether the element exists client-side. Never call it from a
  hook that fires on unrelated server-wide events (e.g. every hotbar scroll) for a UI element the
  plugin doesn't even draw anymore.
- [ ] No `new List<>()`/`new Dictionary<>()`/LINQ allocation inside a hook that fires per-shot or
  per-tick (`OnEntityTakeDamage`, `OnPlayerAttack`, any `timer.Every` with a short interval).
  Garbage-collector pressure from combat hooks is a known perf complaint in Oxide/Carbon circles
  (`Pool.Get<List<T>>()`/`Pool.FreeList()` is the fix when a scratch collection is unavoidable).
  Verified: Salvo's three `new List<string>(...)` allocations are all in cold paths (startup,
  disconnect-cache rebuild, an equip-time reconciliation edge case) — none in the per-shot hooks.
  **Not just general principle — confirmed for the runtime this actually runs on** (Rust's
  dedicated server is Mono, not CoreCLR — see Category 12): closures/lambdas passed to `.Where`/
  `.FirstOrDefault`/etc. are heap-allocated delegate objects, and LINQ's enumerator allocations
  are real even for calls that look trivial (`.Any()` on a plain `IList` still allocates). This
  has genuinely improved over the years but hasn't disappeared. `Pool.Get<List<Conflict>>()`/
  `FreeUnmanaged` is exactly the pattern Carbon's own `HookCaller.CallStaticHook` uses internally
  for its own scratch collection — this isn't advice aimed only at plugin authors, it's what the
  framework does too. Sources: [JacksonDunstan — Just How Much Garbage Does LINQ Create?](https://www.jacksondunstan.com/articles/4840), [sebaslab.com — Zero allocation code in C# and Unity](https://www.sebaslab.com/zero-allocation-code-in-unity/).

## 4. Per-player state cleanup
- [ ] Every `Dictionary<ulong, ...>` / `HashSet<ulong>` keyed by userID has a matching
  `.Remove(userId)` in `OnUserDisconnected` (or equivalent) and in `Unload()`. Otherwise it grows
  unbounded over a long-running server's connect/disconnect churn — a slow memory leak that never
  shows up in a short test session.
- [ ] **Confirming the `.Remove()` call exists inside a method named `OnUserDisconnected` is not
  the same as confirming that method actually fires — live-test the disconnect specifically,
  don't just read the source and assume the hook name is enough.** Real, serious, found on this
  server: `OnUserDisconnected(BasePlayer player, string reason)` did **not** fire on a genuine
  client disconnect, verified directly (a live player disconnected, the server confirmed it via
  `global.status` showing 0 players, and a per-player dictionary keyed by that exact `BasePlayer`
  stayed non-empty with no reload in between to explain it away). `OnPlayerDisconnected(BasePlayer
  player, string reason)` is what actually fires — confirmed both by this same live test working
  once wired to it, and independently by grepping Carbon's own decompiled source, where
  `CorePlugin`/`AdminModule`/`HammerModule` all implement `OnPlayerDisconnected`, not
  `OnUserDisconnected`, for their own internal cleanup. `OnUserDisconnected` is real (it's the
  Covalence/universal-hook name) but evidently isn't reliably dispatched the same way on this
  Carbon build. **This invalidates prior "Cat 4 clean" verdicts that were reached by reading
  source rather than live-testing a disconnect** — every previous pass crediting a plugin's
  `OnUserDisconnected` handler with working cleanup (Salvo, every version audited) was verified by
  confirming the `.Remove()` calls exist in the right-named method, never by watching a real
  disconnect actually trigger them. Wire both hook names defensively when in doubt
  (`Dictionary.Remove` on an absent key is a harmless no-op, so having both cost nothing if only
  one ever fires) — don't trust the hook name alone, and don't trust a static review of Cat 4 as
  equivalent to a live disconnect test.
- [ ] **"Clean up on disconnect" is only correct for state that's supposed to reset — check which
  kind of state it actually is before applying this bullet, don't apply it uniformly.** A
  disconnecting `BasePlayer` is not destroyed: `OnDisconnected()` calls `StartSleeping()`, the
  object persists in the world, and an ordinary reconnect (`ServerMgr.SpawnPlayerSleeping` via
  `BasePlayer.FindSleeping(userid)`) reattaches that *same* object — confirmed directly against
  Facepunch's own decompiled source, not assumed. So for state whose whole point is to survive a
  disconnect (cooldown timers, rate limits, anything gating "don't let a player do X again too
  soon") — clearing it on disconnect isn't a smaller, safer version of this bullet, it's a
  **different bug**: it hands every player a way to bypass the cooldown by disconnecting and
  reconnecting. Only state that's genuinely session-scoped (what UI is currently showing, an
  in-progress interaction) should actually clear on disconnect; persistent-intent state needs a
  periodic sweep instead, removing an entry only once its player is confirmed truly gone (absent
  from both `BasePlayer.activePlayerList` *and* `BasePlayer.sleepingPlayerList`), not on the
  disconnect event itself. Real case, found in Carbon's own framework code, not a plugin: Carbon's
  `CuiHelper.ActivePanels` (session-scoped — correct to clear on disconnect, and does not) and
  `CarbonPlugin.CommandCooldownBuffer` (persistent-intent — wrong to clear on disconnect even
  though it currently doesn't) sit right next to each other as the same missing-cleanup class of
  bug, but would need *opposite* fixes. A plugin — or a mitigation for someone else's leak, as
  here — that "fixes" a cooldown buffer by wiring it into `OnUserDisconnected` has made a real
  regression, not a fix.
- [ ] Any `Timer`/`Coroutine` created per-player is tracked and destroyed on disconnect/unload,
  not left running against a player who's gone.
- [ ] **A hot `c.reload` is not the same test as a genuine service restart** for anything that
  depends on *load order across plugins* (permission-registration races, `[PluginReference]`
  resolution timing). Reloading one plugin repeatedly re-tests that plugin against an already-
  settled world; only a full restart re-creates every plugin loading fresh, simultaneously, in
  whatever order the framework picks. Real case: a permission-ownership conflict between MixCore
  and MixGovern was invisible across many hot reloads (the loser had already claimed the
  permission earlier in the long-running process and kept "working") and only surfaced as a
  clean, repeatable, load-order-dependent warning on an actual `systemctl restart`. Don't treat a
  clean hot-reload as proof a load-order-sensitive fix is correct — restart to confirm when the
  fix touches permission registration, `[PluginReference]` timing, or anything else order-dependent.
- [ ] **Reload stress test**: `c.reload <Plugin>` (or `oxide.reload`) ten times in a row on a live
  server, then check `c.plugins` for hook-exception count and memory — should be unchanged from a
  single reload. If a timer or UI duplicates per reload, hook-fire count or memory climbs
  noticeably faster than normal usage would explain. Ran against Salvo v2.9.2: 10x reload, asset
  count stayed at 21 every time (no duplicate installs), memory flat at 3.0mb, zero exceptions.

## 5. Permission registration hygiene
- [ ] Every permission string used in a `permission.UserHasPermission`/`GrantUserPermission` call
  is actually registered via `permission.RegisterPermission(...)` in `Init`/`Loaded`. An
  unregistered permission fails checks silently or throws, depending on framework version — easy
  to miss because it doesn't show up as a compile error.
- [ ] **When checking "is this registered," search for the permission STRING, not a grep for
  `RegisterPermission(PermXyz`** — a shared/cross-plugin permission is often registered via a
  loop over a table (`foreach (var perm in ManagedPerms) permission.RegisterPermission(perm,
  this);`), not a named constant passed directly. A grep for the direct-call pattern misses this
  entirely. Real, corrected-via-live-testing false positive: `mixpack.admin`/`mixpack.use`
  looked unregistered anywhere in the pack from a `RegisterPermission(PermAdmin`/`PermUse` grep
  across every plugin — they're actually registered correctly by MixGovern via a 30-entry
  `ManagedPerms` loop covering the entire shared "mixpack.*" namespace. Acting on the false
  positive (adding a redundant registration to MixCore) created a real, if harmless, symptom —
  an "already used by another plugin" warning on every reload, whichever plugin lost the race for
  load order — that only became obvious on an actual fresh service restart, not a hot reload. If
  a permission looks unregistered, grep for the literal string across every plugin file first,
  not just the direct-registration-call shape, and verify on a genuine restart before concluding
  it's actually missing.
- [ ] **A registered permission is not the same as a usable feature — every genuinely baseline
  player-tier permission (a plugin's "use" permission, not its VIP/admin one) needs a matching
  `permission.GrantGroupPermission("default", perm, this)` next to its `RegisterPermission` call,
  or a brand-new player has access to nothing across the entire pack.** `RegisterPermission` only
  makes Oxide/Carbon aware a permission string exists — it grants it to no one. Real, found live
  via a genuine first-time-player beta test, not a static review: every plugin in the pack
  registered its permissions correctly and every single one still blocked a new player from
  everything (shop, teleports, Salvo, all of it) because nothing had ever granted the "default"
  group anything. **Do not assume a `PermUse`-shaped name is safe to bulk-grant without reading
  what it actually does** — `osautoturrets.use` turned out to grant VIP1-tier turret status, and
  `MixAdminMove`'s `mixmove.use` is a trusted-tier building tool despite not being named `.admin`;
  a name-only pass would have wrongly handed both to every player. Check each one's actual gate
  logic (config bypass flags, whether it returns a VIP tier, what authLevel default it sits next
  to) before deciding it belongs in a bulk default-group grant.
- [ ] **RCON on this server does not echo any response for `oxide.grant`/`o.grant`-style commands
  — not even an "unknown command" error for a genuinely invalid one — so a live RCON command
  cannot be used to confirm whether a permission change landed.** The real, persisted proof is
  `carbon/data/oxide.groups.data` (binary/protobuf, not JSON — readable via a
  `[\x20-\x7e]{4,}`-printable-string extraction, not a text editor). That file also doesn't save
  synchronously on every grant — it only flushes on some autosave/trigger event, and `server.save`
  reliably forces one. A read of this file taken too soon after a batch of grants can show far
  fewer entries than actually landed and look like a real failure — force a save and re-check
  before concluding a grant didn't take.

## 6. CUI lifecycle safety
- [ ] Every panel `AddUi`'d has a `DestroyUi` reachable on every real exit path: explicit close,
  disconnect, plugin unload. (Distinct from #3 above — this is "does cleanup happen at all," not
  "does it happen too often.")
- [ ] **When checking "does `Unload` destroy the panel," trace through helper methods
  (`ClosePanel(player)`, `KillUi(player)`, `CloseAdminPanel(player)`) — don't just grep `Unload()`'s
  own body for a literal `CuiHelper.DestroyUi`.** A static scan that only looks inside the method
  itself produces false positives on any plugin that (correctly) routes cleanup through a shared
  helper — which is most of them. Real case: an external report flagged MixHud, MixSignboard, and
  ServerLogo as not destroying CUI on `Unload` — all three were false positives, each calling a
  helper (`KillUi`/`CloseAdminPanel`/`DestroyUI`) that does the real `CuiHelper.DestroyUi` work.
  But the same report also flagged MixRaidBases, and that one was real: `Unload()` never called
  its own `ClosePanel` helper at all, leaving the admin panel's `CursorEnabled=true` Dim overlay
  stuck for anyone who had it open during a live reload. Check each claim individually — the
  pattern that produces false positives on three plugins doesn't mean the fourth is also wrong.
- [ ] No `CuiPanel`/`CuiElement` sets `CursorEnabled = true` without a guaranteed matching
  destroy — a leaked cursor-lock panel freezes the player's mouse-look until they reconnect. Known
  bug class from earlier this project; grep every `CursorEnabled = true` and trace its destroy path.
- [ ] Every panel is destroyed *before* being re-added on refresh, not added again on top of the
  existing one — repeatedly `AddUi`-ing the same panel name without a prior `DestroyUi` stacks
  elements client-side (long-standing Oxide CUI footgun); the panel's root name should be a string
  constant, not something that could vary between the add and the destroy call for the same logical
  panel. Also: don't keep a single plugin-level `CuiElementContainer` that different code paths
  keep `.Add()`-ing to across calls — build a fresh container per draw, or the list grows forever.
- [ ] A plugin that draws CUI automatically on connect (welcome panel, HUD, admin greeting) delays
  and re-validates rather than drawing in `OnPlayerConnected` itself — the client is still
  receiving its initial world snapshot at that point and CUI can silently fail to render. A
  `timer.Once(1.5–2.5f, () => { if (player != null && player.IsConnected) Draw(...); })` guard
  is sufficient (the more precise version polls `player.IsReceivingSnapshot` and retries via
  `NextTick` instead of a flat delay). Verified: MixHud and MixCore already do this correctly
  (2s/2.5s delay + re-validate) — this item makes that existing, previously-implicit practice
  explicit so a future plugin doesn't skip it.
- [ ] **A full-screen/overlay panel (hub menus, admin panels) is never drawn while a native loot
  interface is open (a container, the player's own inventory-with-a-storage-box, a vending
  machine) — check for and close/skip-drawing over an active loot panel rather than stacking a
  custom overlay on top of it.** Native loot UI has its own `CursorEnabled`/close-button wiring
  that a plugin-drawn overlay on top of it doesn't coordinate with, so closing one can leave the
  other's cursor state stuck, or the two can fight over which one's Escape/click handling wins.
  (Flagged by Part C of the AI Rust-Modding Bible; not yet confirmed as a live bug in this pack —
  add as a check the next time a menu/hub-style plugin's open path is touched, same as Cat 12's
  struct-key item was added on principle before a live instance existed.)

## 7. Defensive bounds / null-safety
- [ ] Every chat/console command handler checks `args.Length` (or `arg.Args.Length`) before
  indexing into it.
- [ ] **`ConsoleSystem.Arg.Args` is `Facepunch.StringView[]`, not `string[]`** — never index it
  as a string or pass it where a `string[]` is expected; use `arg.GetString(i)` /
  `arg.GetString(i, fallback)` or `.Select(a => a.ToString())`. This is a hard compile error on
  the current Rust build, invisible to a read-through (MixThirdPerson 1.0.1, 2026-09-04) and
  greppable: `Args\[\d+\]` without a trailing `.ToString()`.
- [ ] **A global multiplier over `ItemDefinition.stackable` (or any per-item scale) must skip
  items whose vanilla value is 1** — weapons, tools, armour, deployables, anything with condition
  or a held entity — regardless of the multiplier or a per-item override. Rust tolerates a
  stacked non-stackable badly: two rifles in one slot render as a tiny "2" badge and the second
  is effectively gone. Real case 2026-09-04: MixWorldTune's `ApplyStacks` applied a 5x preset to
  *every* item, `rifle.ak` went to 5, and the shop's second AK merged into the first — reported
  as "bought it, never received it, nothing on the ground". Fix is `if (vanilla <= 1) keep 1`,
  plus a repair pass that splits already-merged non-stackables back out for online players
  (`item.SplitItem(1)` + `player.GiveItem`) when stack sizes come back down.
- [ ] Cached `BasePlayer`/entity references used inside a delayed callback (`NextTick`,
  `timer.Once`) are re-validated (`!= null`, `.IsConnected`) at the point of use, not just at the
  point of capture — the player may have disconnected in between.
- [ ] **`entity.net` is null-checked before `.net.ID`/`.net.Something`, not just `entity` itself**
  — especially in hooks that fire on every entity across the whole server (`OnEntityKill`,
  `OnEntitySpawned`, `OnEntityDeath`). Not every `BaseNetworkable` is networked. Real bug found in
  MixSignboard's `OnEntityKill`: `if (entity == null) return;` followed by unguarded
  `entity.net.ID.Value` — a hot, universal hook that could NullReferenceException on any
  non-networked entity kill. Grep every `.net.ID`/`.net.Something` access inside an `OnEntity*`
  hook and confirm the null-check covers `entity?.net`, not just `entity`.
- [ ] **Full-lifecycle live test**, not just a reload: connect → open the plugin's UI while the
  connection is still mid-snapshot (tests the connect-timing guard for real, not just in theory) →
  die → respawn → disconnect while UI is open → reconnect. Confirm via `c.plugins`/log that no
  stale UI reappears and no orphaned timer survives the disconnect. Broader than category 4's
  per-player cleanup check (which is a code-read) — this is the live equivalent, exercising the
  actual disconnect/death/reconnect sequence rather than just tracing that the cleanup code exists.
- [ ] **Deny path actually denies** — for every permission-gated action, test it with the
  permission *absent* (not just present), and confirm the action is actually blocked rather than
  just skipping a UI element client-side. Category 5 checks that the permission is registered;
  this checks that the gate around the actual action holds even if someone calls the underlying
  command/console command directly, bypassing whatever UI would normally hide it.
- [ ] Command names (`ChatCommand`/`ConsoleCommand`) use a distinctive prefix
  (`salvo.`, `mixgovern.`, etc.) rather than a generic word another plugin's author might also
  pick (`/home`, `/kit`, bare `yes`/`no`) — a same-name collision silently overwrites one
  plugin's command registration with the other's, no error, no warning. Verified across all 17
  live-source MiX plugins: every command is prefixed except two bare chat commands (`/tod`,
  `/welcome`) — low risk (chat commands, not console; collision just fails to register rather than
  misrouting data) but worth a prefix if either is ever revisited. `ImageLibrary.cs`'s generic
  names (`yes`/`no`/`cancelstorage`) are the upstream third-party plugin's own established API,
  not ours to rename.
- [ ] **Hardening, not a defect — `[ProtectedCommandAttribute]` (Carbon-only) for CUI-button
  console commands, evaluated and deliberately not applied to MixMenuKit.** Ground-truthed real
  via decompiled `Carbon.Core.ModLoader`: Carbon recognizes it as a distinct command-registration
  type alongside `ChatCommandAttribute`/`ConsoleCommandAttribute`. This is *not* the security
  boundary — every CUI console command in this pack is already gated by its own in-handler
  permission or state check, confirmed by Category 7's existing "deny path actually denies"
  testing — it's a second, obscurity layer nothing in the pack currently uses. **The real cost is
  higher than "add an attribute"**: checked against Carbon's own usage (`Carbon.Modules.ModalModule`)
  — a `[ProtectedCommand]` handler's button must be built through Carbon's separate **native** CUI
  builder (`Carbon.Components.CUI`, `cui.CreateProtectedButton(...)`), not the standard Oxide
  `CuiElementContainer`/`CuiButton`/`CuiHelper.AddUi` pattern this entire pack is built on. For a
  dual-target plugin (ships both Carbon and Oxide builds) that means a genuine `#if CARBON`/`#else`
  split per protected button, not a one-line addition. Evaluated for MixMenuKit's
  `mixmenukit.showbindinfo`/`hidebindinfo` (the two commands with no permission gate, only a state
  check, and the ones the real v2.15.1 bug hit) — owner decision: skip it, the existing state-gate
  is already correct and live-verified, not worth the dual-target maintenance cost for this pair.
  Revisit if a future plugin is Carbon-only from the start.

## 8. Exception containment
- [ ] **Anything that moves currency or items logs its outcome server-side (`Puts`), not just
  to the player** — `player.ChatMessage()` never reaches the console or `server.log`, so a
  "charged but never received" dispute is unverifiable after the fact. The purchase log must
  carry the balance delta *and where the item actually landed* (`item.parent` → main/belt/wear
  slot, `item.GetWorldEntity()` → dropped, `!item.IsValid()` → merged into a stack or destroyed).
  A give helper that returns `true` the moment `ItemManager.Create` succeeds, without looking at
  what `BasePlayer.GiveItem` did with the item, is the pattern to hunt for — MixCommerce's did
  exactly that until 2026-09-04. Pair it with an RCON-usable server-side inventory dump
  (`mixcommerce.inv <steamid>` prints amount/max-stack per slot); the client view lags and lies,
  and that one command turned a two-hour "the item vanished" hunt into a one-line answer
  (`rifle.ak x2/5`).
- [ ] File I/O, cross-plugin `Call()`s, and anything touching external state are wrapped in
  try/catch where a failure shouldn't take down the plugin's other hooks. A single unhandled
  exception in one hook can disable that hook (or worse) for the rest of the session.
- [ ] This applies to a **genuine, disclosed dependency** just as much as an optional one — grep
  every `SomePlugin.Call(` site individually, don't stop at "the null-check is there so it's
  safe." A null-check only guards against the plugin being unloaded; it does nothing once the
  call actually happens and something inside the other plugin throws. Real finding: MixSignboard
  had all 11 of its `MixImages.Call(...)` sites null-checked but completely unwrapped — including
  one inside a CUI draw loop (`DrawThumb`) where a single bad thumbnail would have broken the
  entire gallery panel render, not just that cell.

## 9. Cross-plugin dependency honesty
- [ ] **On Carbon, every `public` method that another plugin reaches via `Call("Name", ...)` MUST
  carry `[HookMethod("Name")]` — without it the call silently returns `null`.** Not a style
  preference: Carbon's `BaseHookable.BuildHookCache` (read in the decompiled Carbon.Common)
  indexes *non-public* methods by name, but only indexes a *public* method if it has the
  attribute. Oxide indexes everything, which is why code ported from Oxide passes every reload
  and every "did the panel open" test while a whole API surface is dead. No exception, no log
  line, no warning — `Call()` just hands back null and the caller's `is bool b && b` / null-guard
  quietly takes the failure path. Found 2026-09-04 as the root cause of a shop bug three layers
  removed from it (see the log entry for that date): 23 dead cross-plugin calls across 7 plugins,
  including `API_GetStatusLines` (status HUD lines never rendered), `API_OpenAdmin`,
  `API_RegisterModule` (6 callers), `API_Get/SetWorldTune`, and OSAutoTurrets'
  `API_ConfigureRaidTurret`/`API_PowerAndStartTurret` (raid-base turrets never configured via
  API). **Verification is a scripted scan + a data probe, never a reload**: regex every
  `Call("X"` site across the pack, resolve each `X` to its definition, flag `public` + no
  attribute (the one-screen Python that did this is trivial to rewrite — collect names from
  `\bCall(?:<[^>]+>)?\(\s*"(\w+)"`, then match `^(\s*)public\s+[^=;(]*\bNAME\s*\(` with the
  preceding line checked for `HookMethod`). Then prove it at runtime with state the dead call
  should have written: MixCore.json's `ModuleVersions` listed only MixCore itself the whole time
  `API_RegisterModule` was dead, and filled in within seconds of the fix.
- [ ] **Compile the plugin WITHOUT Carbon's compat shims (`NO_PROXY=1 harness/compile.sh`).**
  Carbon ships `Carbon.Proxy.dll` — static shims (`RustProxies.*`) that keep old code building
  after Facepunch removes an API, each marked obsolete and removable in any Carbon release, and
  none of them exist on Oxide. A plugin that only builds with the shim is already broken on a
  timer nobody controls. Real case 2026-09-04: `BaseEntity.SetFlag` was removed from Rust that
  month and **14 of 47 pack plugins (78 call sites)** compiled only through the shim — asked
  about as "what happens to BoatHelm when things change", found to be the whole pack. Fix
  with the replacement API (`StartSetFlags(mode)` scope → `Set(f, b, recursive)`), or with a
  per-plugin extension method in `Oxide.Plugins` that shadows the shim (what
  `harness/add_setflag_compat.py` does — zero call-site churn). Decode what the shim actually
  does (Cecil IL) before writing the replacement, so behaviour stays identical.
- [ ] If a plugin is listed/sold as "standalone," verify every `[PluginReference]` is either (a)
  genuinely absent, or (b) truly optional with a working fallback path when the referenced plugin
  isn't loaded — not a dependency for a *core* advertised feature. (MixSignboard's slideshow
  images vs. Salvo's UI kit are the two reference cases — same pattern, different verdict, because
  one is core functionality and the other was cosmetic chrome.)
- [ ] **No two loaded plugins independently do the same job off the same underlying state** — same
  clock/timer, same world system, same command name — even if each one is individually correct in
  isolation. Check the *pack's* plugin list for functional overlap, not just each plugin's own
  code. Real case: `MixDayNight` and `TimeOfDay` were both loaded on AU simultaneously, both
  managing day/night length and the skip-night vote off the same `TOD_Sky` clock and registering
  the same `skipnight` chat command — the resulting "Command 'skipnight' already exists" warning
  had been live all session and was dismissed as benign Carbon reload noise until traced to its
  real cause. Fix was removing the redundant plugin entirely (`MixDayNight`), not a code change to
  either — this class of bug doesn't show up in Cat 4's reload-stress test or Cat 6's CUI checks,
  it only shows up by literally cross-referencing `c.plugins`' full list for two entries that
  plausibly overlap in purpose. (Independently flagged as a general rule by Part C of the AI
  Rust-Modding Bible — "avoid running duplicate plugin versions simultaneously" — before either
  side knew the other had already hit the concrete case.)

## 10. Platform compliance (Facepunch TOS)
- [ ] Any feature that assigns a skin ID to a player's item (cosmetic rewards, kits, admin-set
  server skins) is checked against Facepunch's skin-ownership policy (effective July 16 2025,
  enforced from August 7 2025): servers may not give players access to paid DLC/workshop skins
  they don't own — violation risks delisting from the server browser, or worse for repeated/
  serious breaches. If the feature can't verify per-player ownership, the safest default is not
  to apply a configurable skin ID at all (this is what Salvo v2.9.2 did — removed the feature
  rather than build ownership verification for a cosmetic).
- [ ] This is a distinct axis from code quality — a feature can be perfectly well-written and
  still be a compliance risk. Check it separately, not as a substitute for categories 1-9.
- [ ] If any MiX product ever ships a "free Workshop skin" list (giveaway pack, imported-ID list
  for a skin plugin), that list needs periodic rechecking, not a one-time check at build time —
  Facepunch can accept a previously-free Workshop skin into the paid store at any point, and a
  list that isn't rechecked keeps handing out what's now a paid skin. **Now applicable**: this is
  exactly MixSkinsLight's situation (see audit log) — noted so it's not missed on the next pass.
- [ ] **A skin-ownership check that exists in code is not automatically safe — check what it does
  when it *can't* verify, not just what it does when it can.** A verification path that fails
  *open* (allows the skin through when the check itself is unavailable) gives the false
  appearance of compliance while doing nothing on any server where the checking dependency isn't
  installed — which, for an uncommon plugin like PlayerDLCAPI, is most servers. Real, serious
  finding: MixSkinsLight's `EnforceSkinOwnership` (on by default) called into `PlayerDLCAPI` for
  ownership checks, but every branch where that plugin was missing/not-yet-loaded/not-initialized
  returned `true` (allow) instead of `false` (deny) — and the shipped `LISTING.md` documented
  this as an acceptable feature ("still works fine... doesn't error"), not a known gap. Fixed to
  fail closed: v1.3.0 denies when ownership can't be verified, with a clear player-facing message
  distinguishing "can't verify" from "confirmed you don't own this." **Check the actual behavior
  of every branch of a compliance check, not just that a check exists — and check whether existing
  docs have enshrined the buggy behavior as intentional, since that's a sign nobody caught it.**

## 11. Lifecycle hook correctness
- [ ] Nothing in `Init()` assumes other plugins exist yet (`Plugins.Find`/`[PluginReference]`
  resolution belongs in `OnServerInitialized`/`Loaded`, not `Init` — plugin load order isn't
  guaranteed at `Init` time). `Init` is perms/`Unsubscribe`/local-state only.
  `OnServerInitialized(bool initial)` is where cross-plugin references, monument scans, and
  world-dependent caches belong — note the `initial` param is `false` on a hot reload, so don't
  redo one-time-only setup work if the plugin also needs reload-safe behavior.
  `Unload()` is destroy-UI/kill-timers/save-data — nothing else.
- [ ] **Every plugin with persisted data (`DataFileSystem`) has an explicit, documented decision
  about map-wipe survival** — either it hooks `OnNewSave` and clears/resets what should reset, or
  someone deliberately decided the data should carry across a wipe (progression/economy/ranks are
  plausible carry-over candidates; per-map state usually isn't). Grep `class.*Data` +
  `OnNewSave` per plugin — an unimplemented `OnNewSave` isn't automatically wrong, but it should
  be a decision, not an oversight. **Current state (2026-08-30 audit): zero of the 17 live-source
  MiX plugins implement `OnNewSave`** — every plugin's data (Salvo ranks, MixCore economy/stats,
  MixGovern homes/teleports, MixSignboard/FreeBuild/MixEntityScale stored data) currently survives
  a wipe unconditionally, by default rather than by decision. Not yet resolved — needs a call per
  plugin before first release, not a blanket fix.
- [ ] Config schema changes (renaming/removing a field, changing a type) are safe by default
  because `Config.ReadObject<T>()` is tolerant — unknown keys in an existing user's `config.json`
  are ignored, missing keys get the C# default. That's sufficient for additive/rename changes on a
  plugin with no live installs yet (e.g. Salvo's `Skin`→`Kit` rename this pass — nobody has a
  config.json to migrate). Once a plugin actually ships to real servers, a genuinely *breaking*
  structural change (e.g. a field changing from a single value to a list) still needs an explicit
  `ConfigVersion` field + a migration step in `LoadConfig()` — silently reverting a live user's
  tuned settings to defaults is a real support-ticket generator. Not yet needed for any MiX plugin
  (all pre-first-release), but the policy should exist before the first post-release config change.
- [ ] `LoadConfig()`'s catch-and-overwrite path (invalid/corrupt `config.json` → load defaults)
  backs the broken file up (e.g. `File.Copy` to `<name>.json.bak`) before overwriting it, rather
  than just printing a warning telling the admin they should have backed it up themselves. Cheap
  insurance against a real support ticket ("your plugin ate my tuned config"). Currently a shared
  gap across every MiX plugin including Salvo — not urgent (no live installs to lose data yet),
  but worth fixing before first release rather than after the first complaint.
- [ ] **Any `List<T>`/`Dictionary<K,V>` config field with a non-empty C# field-initializer default
  needs `[JsonProperty(ObjectCreationHandling = ObjectCreationHandling.Replace)]`, or edits to that
  field silently, permanently revert.** Newtonsoft's default behavior for a non-null collection
  property is to REUSE the field initializer's own instance and ADD the deserialized JSON items to
  it, not replace it — so whatever's hardcoded as the "default" survives every future load
  regardless of what's actually saved in `config.json`, merging back in on every single boot
  forever. Real, serious, genuinely hard to diagnose: `MixCore.cs`'s `CorePackPlugins`/
  `ProtectedPlugins`/`RetiredPlugins` fields all had this — a real fix (correcting which plugins
  count as "core pack") kept silently reverting after every attempt, including a `c.reload`, an
  explicit `c.unload` → edit file → `c.load` sequence, AND a genuine full `systemctl restart` of
  the service — the last of which should have ruled out every caching theory and initially looked
  like an unexplainable, unfixable bug. Root cause confirmed by diffing the corrupted result
  against the known-clean edit: the reappearing entries were exactly the stale field-initializer
  default's unique items, in their original order, appended after the correct new list — proving
  Newtonsoft was merging, not the OS/Carbon caching anything. **A clean reload or even a full
  service restart is not proof a config fix actually landed** if the field carries a non-empty
  default — check the field declaration itself, not just that the file on disk looks right
  immediately after editing it. Fixed 2026-09-04 by adding the attribute to all four `ModManager`
  list fields and correcting their now-actually-effective defaults to match.

## 12. Static-field memory footprint (CLR/GC, not hook lifecycle)
- [ ] **Any `static` (or `static readonly`) field holding a large blob of one-time-bootstrap data
  (embedded assets, a big lookup table only needed once at first install) is not left permanently
  rooted for the plugin's entire lifetime.** A `static` field is a GC root — the garbage collector
  will never reclaim it or anything it references while the plugin stays loaded, regardless of
  whether the data is ever actually used again after the first read. Check: does the data get
  built (string literals decoded, a dictionary populated) unconditionally at class-load time, even
  on the 999th load where the actual work it exists for (e.g. `File.Exists` finding the asset
  already installed) will always skip using it? If so, the memory cost is paid on *every* load for
  work that's only ever needed on the *first*.
- [ ] **Any single string/array/object ≥8KB lands in Mono SGen's Large Object Space (LOS), not
  the normal nursery/major-heap path — and that 8KB figure is specific to *this* runtime, not the
  85KB figure widely quoted for modern CoreCLR (.NET Core/5+).** Rust's dedicated server runs on
  Mono (confirmed directly against the live server: `RustDedicated_Data/Mono/x86_64/` — the
  classic Unity-Mono directory layout, not an IL2CPP AOT build), and Mono's default GC since
  Unity's move off the old Boehm collector is SGen, whose large-object threshold is
  `MAX_SMALL_OBJ_SIZE` = 8KB per the Mono project's own SGen documentation. That's roughly 10x
  lower than CoreCLR's 85,000-byte LOH threshold — meaning a 4,096-character UTF-16 string
  literal already qualifies here, not 42,500. **Citing the CoreCLR 85KB figure for a Rust
  plugin is citing the wrong runtime's number** — an easy mistake since almost all public .NET
  GC writing targets CoreCLR, not Mono/SGen, and the two share enough vocabulary (generations,
  a separate large-object area) to look interchangeable when they aren't.
  SGen's heap shape also differs from CoreCLR's Gen0/1/2 + LOH: SGen has two generations (the
  **nursery** — young objects, default 4MB, bump-pointer allocated — and the **major heap** —
  survivors, mark-and-sweep managed in 16KB blocks) plus the separate **Large Object Space**.
  One place SGen is actually *less* bad than the CoreCLR framing implies: large objects are
  allocated on OS-managed pages, and per Mono's own docs those pages *do* get released back to
  the OS once nothing references them — so once a static field holding large objects actually
  becomes collectible (e.g. the old assembly unloading on a genuine plugin reload), the memory
  isn't stuck permanently fragmented, it can genuinely go back to the process/OS. That doesn't
  change the core finding — a `static` field still roots the data for as long as the *current*
  load stays alive, still paying the cost on every load for work only the first load needed —
  it just means "not compacted" was the wrong CoreCLR-flavored word for what SGen does here.
  Sources: [Mono docs — Working With SGen](https://www.mono-project.com/docs/advanced/garbage-collector/sgen/working-with-sgen/), [Mono docs — Generational GC](https://www.mono-project.com/docs/advanced/garbage-collector/sgen/).
- [ ] **This class of bug does not show up in a reload-stress test.** `c.reload` on Oxide/Carbon
  recompiles the plugin into a fresh assembly, so old static-field memory becomes collectible once
  the old assembly unloads — ten reloads in a row will show flat memory even with this bug present.
  The cost is the fixed amount paid once per load and held for that load's entire uptime, not
  something that accumulates *across* reloads. Don't treat a clean stress test as clearing this
  category — it's testing a different signal (leaks across reloads) than this one (dead weight
  within a single load).
  - Fix pattern: build the data as a local inside the one method that needs it (so it falls out of
  scope and becomes collectible right after first use), or — better — check `File.Exists()` per
  asset *before* decoding/allocating that asset's data at all, so a server past its first install
  never allocates any of it in the first place.
  - **Real case: Salvo's embedded UI-kit assets.** `MixPackAssetsEmbedded.Chunks` is a
  `private static readonly Dictionary<string, string[]>` holding 75 Base64 chunk strings, ~8MB of
  UTF-16 text total, largest single chunk 120KB. Ricky's original review cited these against
  CoreCLR's 85KB LOH line (67 of 75 chunks crossing it) — re-verified against the runtime this
  plugin actually runs on (Mono/SGen, 8KB line, not CoreCLR's 85KB): the real bar is roughly 10x
  lower, so very likely closer to all 75 chunks land in the Large Object Space, not just 67 —
  exact per-chunk sizes weren't available to confirm the precise count, but every plausible even
  split of ~8MB across 75 chunks puts each one well over 4,096 characters. `EnsureInstalled()`
  only ever reads a given chunk if that file isn't already on disk, but the field itself is built
  unconditionally at class-load, every load, for the plugin's entire life — meaning every server
  restart after the very first pays the full ~8MB cost for work it will never do. Found by an
  external reviewer (Ricky), not caught by any check or live test run against this plugin before
  or since — see the 2026-09-01 log entry below for why: none of categories 1-11 ask this
  question, and the standard reload-stress test specifically can't surface it (see the bullet
  above). Not yet fixed as of this writing.
- [ ] **A `struct` used as a `Dictionary<TKey,TValue>` key (or anywhere else genuinely compared/
  hashed as `TKey`) implements `IEquatable<TKey>` and overrides `GetHashCode()`.** Without both,
  key comparison falls back to `ValueType.Equals`/`GetHashCode()` — which boxes the struct on
  *every* hash lookup and equality check, not once. This is real and measured, not CoreCLR-vs-Mono
  folklore in either direction: one cited source measured implementing `IEquatable<T>` as a
  2.3-4.3x speedup depending on JIT/platform. Not currently live anywhere in this pack — every
  dictionary key checked so far (across MixMenuKit, Salvo, and their META builds) is either a
  reference type or a primitive that already implements `IEquatable` natively (`ulong`, `int`,
  ordinary `string`) — but it's exactly the kind of pattern nobody thinks to check until a future
  plugin keys something by a `Vector3`, a coordinate pair, or a custom lightweight struct, and
  this category exists precisely for "looks like ordinary C#, this runtime punishes it
  specifically." Sources: [SomaSim — C# performance tips for Unity: structs and enums](https://somasim.com/blog/2015/08/07/c-performance-tips-for-unity-part-2-structs-and-enums/), [Tedd's blog — Speeding up Unity's Vector in lists/dictionaries](https://blog.tedd.no/2018/01/13/speeding-up-unitys-vector-in-lists-dictionaries/).

---

## Log of additions
- 2026-08-30: Initial version, built from Ricky's Salvo review (categories 1–3) plus a
  proactive sweep for the same bug classes (found `OnWeaponFired` dead hook, `AutoKitOnConnect`
  dead field) and general Oxide/Carbon knowledge (categories 4–9, not yet exercised against real
  findings — first live test is Salvo v2.9.0, see audit log below).
- 2026-08-30: Added category 10 (platform compliance) after verifying an externally-sourced claim
  about Facepunch's July/Aug 2025 skin-ownership TOS enforcement was accurate, then checking it
  against Salvo's own `LauncherSkinId` feature and finding it was exactly the risk pattern
  described — applying an admin-configured skin to every player's item with no ownership check.
  Also ran the rest of that external content's suggestions (hot-path collection types, `OnServerSave`
  hook usage, LINQ in combat hooks, full timer-cleanup sweep) against Salvo — all clean, no new
  findings from those four, folded into categories 2–4's existing wording rather than new items.
- 2026-08-30: Second external dump (forum/Codefling/Carbon-team practitioner knowledge) evaluated
  critically rather than accepted wholesale — most of it was either already covered or didn't
  apply once actually checked against the code (object-hook return convention, allocation-in-hot-
  path, connect-time CUI timing were all *already correct* in our plugins, just not written down
  as explicit checklist items until now). Genuinely new and added: subscribe/unsubscribe discipline
  for optional hooks (cat 2), no-allocation-in-hot-hooks with `Pool.Get`/`Pool.Free` (cat 3), a
  live 10x-reload stress test as a standing verification step (cat 4), destroy-before-add/const-
  name/no-growing-container CUI rules (cat 6), connect-time CUI delay pattern (cat 6), and a new
  category 11 for lifecycle-hook placement (`Init` vs `Loaded` vs `OnServerInitialized`) and a
  config-versioning policy for once the plugins are actually live. Discarded as not-checklist-
  material: general debugging workflow tips (`oxide.unload`+`server.fps` to find a lagging
  plugin), entity-spawn batch-size folklore (not applicable — no MiX plugin does CopyPaste-scale
  entity work), and the Harmony-DLL-vs-Carbon warning (not applicable — no MiX plugin ships a
  standalone Harmony .dll).
- 2026-08-30: Reviewed a working reference implementation (`PluginSkeleton.cs`, in
  `C:\Users\a\Desktop\Rogue Depot Skins\Large Bwooden box\`) that wires up the patterns from both
  external dumps as actual compilable code rather than description — confirms they're real and
  gives us a template to copy from for the next new plugin, rather than reconstructing the
  patterns from scratch each time. Found one genuinely new, real gap while reading it closely:
  `LoadConfig()`'s corrupt-config recovery path overwrites the file without backing it up first
  (added as a new item under category 11) — a gap the skeleton shares with every MiX plugin
  including Salvo, not something the skeleton got wrong that we got right.
- 2026-08-30: Third external dump — mostly server-operations knowledge (wipe/patch procedure,
  Oxide/Carbon reinstall-after-game-update, box-health monitoring, Steam branch matching) that
  belongs in a server-ops runbook, not this code checklist — flagged to the user rather than
  folded in here. The corrupt-config-overwrite item added last pass got independently corroborated
  ("Broken JSON → plugin writes a fresh default over your live rates. Validate before reload.").
  Genuinely new and added: full live-lifecycle test (connect-mid-snapshot → die → respawn →
  disconnect-with-UI-open → reconnect, not just a reload) and deny-path verification (cat 7);
  explicit per-plugin `OnNewSave`/wipe-survival decision requirement (cat 11) — audited immediately
  and found **zero of 17 live-source plugins implement `OnNewSave`**, meaning every plugin's data
  survives a wipe by default rather than by decision; command-name collision risk (cat 7) — audited
  immediately, confirmed low risk (prefixed almost everywhere, two bare chat commands noted);
  periodic-recheck requirement for any future free-skin giveaway list, since Facepunch can accept a
  previously-free skin into the paid store later (cat 10, not currently applicable — no MiX plugin
  ships one). Discarded as not-checklist-material: wipe/patch server procedure, box-health
  monitoring thresholds, Steam branch/build-mismatch troubleshooting, `users.cfg`/ownerid identity
  notes (already learned empirically this session) — all operational, not code-quality.
- 2026-08-30: FreeBuild and MixSignboard full passes. FreeBuild yielded mostly the by-now-familiar
  categories (dead config, SaveData exception safety, CUI Unload gap). MixSignboard yielded two
  genuinely new, reusable checklist items promoted out of its audit-log entry into the numbered
  categories so future plugins get checked against them directly rather than only in hindsight:
  `entity.net` null-safety in `OnEntity*` hooks (cat 7) and "wrap cross-plugin calls even when the
  dependency is genuine/disclosed, not just when it's optional" (cat 8) — both found via a real
  bug (`OnEntityKill` NullReferenceException risk) and a real gap (11 unwrapped `MixImages.Call`
  sites), not theoretical.
- 2026-08-30: Third-party "independent assessment" report evaluated and mostly rejected — its own
  numbers gave it away (cited "MixCore v0.5.72", a version far older than the v0.9.26 actually
  live; listed a dozen+ plugin names — MixDayNight, MixPlaytest, MixAdminMove, MixAngler,
  MixExtract, MixGrid, MixHorde, MixPlace, MixQuarry, MixSky, MixTrophies, TwinBarrel,
  OSRocketTurrets — that don't exist anywhere in this pack's live-source or active roster). It was
  scanning a different, much broader/older archive, not our actual current codebase — confirmed
  by the user, who supplied it as an outside check. Two things worth keeping came out of checking
  it anyway rather than dismissing it wholesale:
  1. **Its MixCommerce/MixWorldTune "never RegisterPermission" findings are false positives**,
     independently reproducing the exact table/loop-registration blind spot this checklist's own
     category 5 item already documents — both plugins check `mixpack.*` permissions that
     MixGovern's `ManagedPerms` table already registers correctly. Confirms that item is worth
     having, since even an external tool tripped on the identical gap.
  2. **TimeOfDay's admin commands were genuinely gated on raw server `authLevel` only, no Oxide
     permission fallback** — real, and the one finding in the whole report that held up. Fixed
     (v2.4.1 → v2.4.2): added an `IsStaff()` helper checking `mixpack.admin` alongside the
     original authLevel threshold, applied to all 6 gate points (`tod.daylength`,
     `tod.nightlength`, `tod.freezetime`, `tod.skipday`, `tod.skipnight`, `/skipnight force`).
     Also fixed a hardcoded `"TimeOfDay 2.4.1"` boot-banner string to interpolate `{Version}` —
     same stale-string class as Salvo/FreeBuild's earlier fixes. Verified live: zero exceptions.
  MixUiFix was also flagged for zero permission checks — checked and left alone: its only
  commands (`/cursor`, `/closeui`) are public UI-fix utilities, not admin tools, so no gate is
  correct by design, not a defect. General lesson: don't accept or reject an external report
  wholesale on its source's credibility alone — the parts that are checkable against files we
  actually control are checkable regardless of what else in the same report is stale or
  out-of-scope, and this one repaid a few minutes of verification with a real fix.
- 2026-08-30: Fourth external report — this one accurate (versions matched exactly what's
  actually live, unlike the third). Verified every checkable claim individually rather than
  trusting the "this one's the real one" framing:
  - **Real, high-severity, fixed**: `DevServerOpenPanelToAll = true` as the class default in both
    `MixSprint.cs` and `OSAutoTurrets.cs` (not just a stray config toggle — confirmed the actual
    live config files had it `true` too). New checklist item added under category 1. Both fixed
    (code default + live config file both patched, since fixing only the class default doesn't
    touch an already-saved config) — MixSprint v2.2.0→2.2.1, OSAuto-Turrets v1.6.9→1.7.0.
  - **Real, fixed**: MixRaidBases' `Unload()` never closed its admin panel — new checklist item
    added under category 6 (see there for the full false-positive/true-positive breakdown across
    the four plugins this same report flagged for the same claim). v1.7.0 → v1.7.1.
  - **Already known, already fixed earlier this session**: TimeOfDay's authLevel-only gating —
    this report re-surfaced the same finding the third report also caught, already resolved.
  - **Already known false positives, re-confirmed**: MixCommerce/MixWorldTune permission
    registration — same MixGovern `ManagedPerms` explanation as before.
  - **Noted, then resolved next turn** — all three follow-up items:
    1. `MixCore.API_GrantVanillaOwner` hardened: signature changed from `(ulong userId, string
       displayName)` to `(BasePlayer requestingAdmin, ulong userId, string displayName)`, and the
       method now independently re-verifies the requester against MixCore's own `IsAdmin()`
       before running `ownerid` — no longer trusts that whichever plugin calls it already gated
       correctly. Only real caller (`MixGovern.GrantAllMixAndOwner`) updated to match. Both
       deployed together (breaking signature change, had to ship as one unit) — MixCore
       v0.9.26→0.9.27, MixGovern v0.9.2→0.9.3. Verified live, zero exceptions. Actual grant flow
       not live-tested (would mean genuinely granting real server ownership) — verified by
       signature/logic review instead; flagged to the user rather than silently assumed correct.
    2. **RogueDepot's `oxide.load` concern was a false alarm** — the code already handles this
       correctly via `#if CARBON ... c.load ... #else ... oxide.load ... #endif`, a compile-time
       framework switch. Confirmed empirically rather than trusted: wrote a throwaway diagnostic
       plugin with the same `#if CARBON` guard, loaded it live, confirmed Carbon actually defines
       the `CARBON` symbol (`CARBON_SYMBOL_DEFINED: yes`) — so RogueDepot's Carbon branch is what
       actually compiles here, never the Oxide one. Deleted the diagnostic plugin after.
    3. **ImageLibrary**: clarified a real mix-up first — MixCore has zero references to the
       third-party `ImageLibrary` at all (it depends on `MixImages`, already deployed); the user's
       assumption it needed supplying was based on a name mix-up. OSAutoTurrets was the one with a
       genuine (cosmetic-only, gracefully-degrading) dependency on it. Deployed `ImageLibrary.cs`
       (Absolut & K1lly0u v2.0.64) at the user's choice over the alternative of converting
       OSAutoTurrets to the self-install pattern. One real deploy hiccup along the way: the same
       root-vs-rustserver ownership race from earlier in the session recurred (scp + chown +
       immediate c.load in tight succession beat the ownership change) — retried and it landed
       clean. OSAutoTurrets reloaded afterward to pick up the now-live reference. Verified: 22
       plugins loaded, zero exceptions, zero failed.
- 2026-09-01: Category 12 added (static-field memory footprint / LOH) after re-surfacing a
  reviewer comment (Ricky, 2026-08-30, re: Salvo's embedded UI-kit assets) that had never actually
  been fixed — the owner asked directly why none of our own checks or live tests had caught it.
  Honest answer, checked mechanism by mechanism rather than guessed: categories 1-11 genuinely
  don't ask this question (Cat 1's dead-code check passes because the field *is* read, just rarely;
  Cat 3's unconditional-I/O check is about hook-firing frequency, not class-load-time allocation),
  and the standard 10x-reload stress test can't surface it structurally — Oxide/Carbon recompiles
  the plugin into a fresh assembly on each reload, so the old static data becomes collectible when
  the old assembly unloads; the bug is a fixed per-load cost held for that load's entire uptime,
  not something that accumulates *across* reloads the way a real leak would. This was a genuine
  coverage gap, not a missed application of an existing category — confirmed still present,
  unfixed, in both current Salvo builds (submission v2.9.5 and META v3.0.1) by direct grep before
  writing any of this up.
- 2026-09-02: Three items folded in from an extended runtime-internals research pass (see
  `rust-mods/_research/RUST-MODDING-RUNTIME-DEEP-DIVE.md` for the full trail — GitHub source
  reading across Facepunch's and Carbon's own repos, plus decompiling the live AU server's actual
  `Carbon.dll`/`Carbon.Common.dll` where public source ran out). The research pass's stated
  purpose was feeding this checklist, not standing alone — auditing it directly turned up that
  only one category (12) had actually been added despite several other checklist-worthy findings
  sitting unused in the research doc; these three close that gap:
  1. **Cat 4 — a real refinement to existing guidance, not just a new example.** Confirmed
     directly against Facepunch's own decompiled `BasePlayer.cs` that a disconnecting player's
     object isn't destroyed (`OnDisconnected` → `StartSleeping`, `FindSleeping` reattaches the
     same object on reconnect) — meaning "clean up on disconnect" is only correct for
     session-scoped state, and actively wrong (a real exploit, not a smaller bug) for state meant
     to survive a reconnect, like a cooldown buffer. Found live in Carbon's own framework code
     (`CuiHelper.ActivePanels` vs `CarbonPlugin.CommandCooldownBuffer`, right next to each other,
     needing opposite fixes) while investigating a leak report, not a plugin.
  2. **Cat 3 — the LINQ-in-hot-paths item upgraded from asserted principle to sourced fact for
     this specific runtime**, with real citations on closure/enumerator allocation and
     confirmation that Carbon's own `HookCaller` uses the same `Pool.Get`/`FreeUnmanaged` pattern
     internally, not just advice aimed at plugin authors.
  3. **Cat 12 — a new item**, struct-as-dictionary-key boxing without `IEquatable<T>`. Not
     currently live anywhere in this pack, added because it's exactly the "looks like ordinary
     C#, this runtime punishes it" shape the category exists for, not because a live instance was
     found.
- 2026-09-02: **Cat 4 addition — `OnUserDisconnected` genuinely does not fire on this Carbon
  build, live-confirmed, and this invalidates every prior "Cat 4 clean" verdict that rested on
  reading source rather than watching a real disconnect.** Found while live-testing a small
  mitigation plugin (`MixCarbonLeakGuard`, written for the Carbon-framework leaks noted above): a
  real player disconnected (server confirmed via `global.status`: 0 players), the plugin's
  `OnUserDisconnected` handler was live and had been for the whole session with no reload to
  explain a miss, and the per-player entry it should have removed stayed exactly where it was.
  Switching to `OnPlayerDisconnected(BasePlayer player, string reason)` and re-testing the same
  live disconnect fixed it immediately. Cross-checked against Carbon's own decompiled source:
  `CorePlugin`, `AdminModule`, and `HammerModule` all implement `OnPlayerDisconnected` for their
  own internal cleanup — zero matches anywhere for `OnUserDisconnected`. **Direct consequence for
  this pack**: grepped both plugins — MixMenuKit already uses `OnPlayerDisconnected` (correct,
  by what looks like luck rather than a verified decision) and was never at risk; **Salvo uses
  only `OnUserDisconnected`**, meaning every "Cat 4 clean, all N dictionaries verified cleared on
  `OnUserDisconnected`" line in every Salvo audit pass below (v2.9.1 through the current build)
  was true of the *source*, not of what a real disconnect on a live server actually does. Salvo's
  `_reloading`/`_firing`/`_magCache`/`_lastShotConsumeAt`/`_lastSalvoShotRealtime`/`_adminTab`
  have very likely been leaking on every real player disconnect this entire time, undetected
  because nothing before this pass live-tested a genuine disconnect against a real Carbon server —
  every prior verification was a source read confirming the `.Remove()` calls exist inside a
  correctly-named method, which turned out not to be the same claim as "this method runs." Fix
  applied directly to Salvo (both hook names wired, matching the `MixCarbonLeakGuard` pattern) —
  see the dedicated Salvo entry below rather than duplicating it here.
- 2026-09-02: Cat 2 addition — `Resources.FindObjectsOfType`/`FindObjectsOfTypeAll` from any hook,
  after a user-supplied external guide (`rust-modding-guide.md`, well-sourced, most claims cited
  to primary Oxide/Carbon docs) named it specifically. Checked immediately against this pack
  rather than left as an open item: zero matches anywhere in Salvo or MixMenuKit, both builds —
  clean. The same guide's other checkable claims were evaluated individually rather than trusted
  as a set: its 30-40%-faster-boot/99%-compatible skepticism and its "`oxide.perf`/`oxide.stats`
  aren't real, `oxide.plugins` hooktime is the actual tool" finding both independently corroborate
  this project's own prior findings. Its claim that Carbon publishes a specific "70-plugin box
  using a third of hooks" performance example was checked directly against `carbonmod.gg`'s
  homepage and hooks-reference page and could not be found anywhere — same unverifiable-marketing
  shape as the numbers already flagged, not accepted on the guide's citation alone. Its named
  Carbon profiling commands (`carbon.profile`/`carbon.track`/`carbon.export_profile`) don't exist
  under those names either — checked live against AU via `find profile`, which turned up the real,
  much larger `profile.*`/`memsnap.*` namespace instead (Unity Memory Profiler snapshots,
  allocation watching with stack traces, per-RPC/RCON/command lag-spike thresholds) — the
  underlying capability the guide described is real and more extensive than it stated, just under
  different names. See `rust-mods/_research/RUST-MODDING-RUNTIME-DEEP-DIVE.md` Section 10 for the
  full breakdown, including items not folded into this checklist (dated Oxide/Carbon version
  snapshot, an `OnEntityReskin` hook-signature-drift example, `c.migrate_perms_sql`).
- 2026-09-02: Cat 7 addition — `[ProtectedCommandAttribute]` (Carbon-only CUI command-name
  obfuscation), re-checked against the same `rust-modding-guide.md` after a later re-read (the
  guide is a work in progress on the user's end, edited between reads). This claim, unlike the
  `carbon.profile`/`carbon.track` one above, checked out: ground-truthed directly against the
  decompiled `Carbon.Core.ModLoader` in `Carbon.Common.dll` — `ProtectedCommandAttribute` is real,
  registered by Carbon as a distinct command type alongside `ChatCommandAttribute`/
  `ConsoleCommandAttribute`. Not a fix for anything currently broken (every CUI console command in
  the pack is already permission- or state-gated at the handler, which is the real boundary) —
  added as an available hardening option, not a requirement.
- 2026-09-02: Follow-up — evaluated actually applying it to MixMenuKit's
  `mixmenukit.showbindinfo`/`hidebindinfo` (the strongest candidate: no permission check, only a
  state check, the exact pair the real v2.15.1 bug hit). Before implementing, checked how Carbon's
  own first-party code actually wires a `[ProtectedCommand]` handler's button up
  (`Carbon.Modules.ModalModule`, `cui.CreateProtectedButton(...)`) rather than assume it stacks
  cleanly on the existing `[ConsoleCommand]`/`CuiElementContainer` pattern — it doesn't. It
  requires Carbon's separate native CUI builder (`Carbon.Components.CUI`), incompatible with the
  standard Oxide `CuiHelper.AddUi`/`CuiButton` pattern this entire pack uses, meaning a real
  `#if CARBON`/`#else` split per button on a dual-target plugin, not a one-line attribute add.
  Owner decision: skip it for MixMenuKit — not worth the dual-target cost for two buttons whose
  existing state-gate is already correct and live-verified. See the corrected Cat 7 item above and
  `RUST-MODDING-RUNTIME-DEEP-DIVE.md` Section 10 for the full mechanism.
- **Salvo v2.9.1** — full pass, categories 1–9, all clean or fixed:
  - Cat 1–3: covered by v2.6.0–v2.9.0 fixes (dead config removed, hook ordering fixed, dead
    `UpdateHud`/`OnWeaponFired` removed, per-shot `SaveData()` gated).
  - Cat 4 (per-player cleanup): 7 dictionaries audited — all correctly cleaned on
    `OnUserDisconnected`; found `_lastSalvoShotRealtime` missing from `Unload()`'s clear list
    (inconsistency, not a real leak since Unload discards the instance) — fixed for consistency.
  - Cat 5 (permission registration): clean — all 4 fixed perms + dynamic custom-rank perms
    registered.
  - Cat 6 (CUI lifecycle): 2 `CursorEnabled=true` panels found — one (`MixUiV2.Card`) is unused
    dead code in the shared cross-plugin utility class (zero runtime cost, left alone —
    consistency with other plugins' copies of the same shared class matters more than removing an
    uncalled helper), the other (admin panel Dim overlay) has confirmed destroy paths on every
    exit. Clean.
  - Cat 7 (bounds/null-safety): clean — every command handler bounds-checks `args.Length` before
    indexing.
  - Cat 8 (exception containment): found `SaveData()` completely unwrapped (asymmetric with
    `LoadData()`, which was already wrapped) — an unhandled disk-write failure would throw out of
    whichever hook called it. Fixed with try/catch + warning log.
  - Cat 9 (dependency honesty): fully verified live — standalone test with zero other plugins
    (not even MixImages) loaded, confirmed working.
  - Net new fixes this pass: 2 (Unload cleanup consistency, SaveData exception safety). Both
    low-severity compared to the Cat 1–3 findings, which is itself a signal the checklist is
    converging rather than surfacing endless new problems.

- **Salvo v2.9.2** — category 10 pass (platform compliance) plus a recheck of categories 2–4
  against externally-sourced suggestions:
  - Cat 10 (compliance): found `ApplyLauncherSkin`/`GetLauncherSkinId`/`SkinSettings` applied an
    admin-configured skin ID to every player's launcher with no ownership check — exactly the
    pattern Facepunch's TOS now prohibits if the configured skin were paid DLC. User decision:
    remove the mechanism entirely rather than build ownership verification for a cosmetic
    feature. Removed `SkinSettings` (renamed remaining `AutoKitRockets` field into a new
    `KitSettings` class, config key `Skin` → `Kit`), both methods, and all 4 call sites. Verified
    zero remaining references outside one explanatory comment; verified live reload (v2.9.2,
    zero hook exceptions).
  - Cat 2–4 recheck: `RocketAmmo` already a `HashSet<string>` (not `List`) — clean. No
    `OnServerSave` hook exists — not applicable. No LINQ in the combat hot-path hooks — clean.
    Both `timer.Once` calls are bounded one-shots (fixed reset, capped-attempt retry) — no
    untracked recurring timers. No new findings.
  - Cat 2 (return convention)/3 (allocations)/6 (CUI timing) recheck against the second external
    dump: `OnPlayerAttack`/`OnEntityTakeDamage` already return non-null correctly to cancel; all
    three `new List<>()` sites are cold-path (startup/disconnect/reconciliation), none in the
    per-shot hooks; Salvo has no auto-drawn connect-time UI so the connect-timing item doesn't
    apply to it (verified against MixHud/MixCore instead, both already correct). 10x reload stress
    test run live: asset count steady at 21, memory flat, zero exceptions across all 10 reloads.
    No new findings — second consecutive clean pass.

- **Salvo v2.9.3** — found live, not from a scheduled audit pass: a real admin (not a test
  account) toggled the mod on via the admin panel during actual live play, it worked, then reverted
  to off with no visible cause after a later unrelated plugin reload. Root cause: `ApplyMasterToggle()`
  flipped `_config.Enabled` and applied it live but never called `SaveConfig()` — a separate "save"
  action a few lines away did, with no indication the toggle button alone wasn't enough. Same bug
  class as the MixSprint finding from the same session (a toggle that visibly works but doesn't
  persist), now generalized into its own checklist item under category 2. Fixed by adding
  `SaveConfig()` into `ApplyMasterToggle()`. Verified live: v2.9.3 loaded clean, zero exceptions,
  config confirmed `Enabled: true` held through the redeploy. Not yet swept across the other 19
  live plugins for the same pattern — worth a dedicated pass, flagged rather than rushed while the
  server's live.

- **Salvo v2.9.3 → v2.9.4** — full 11-category re-run, owner-requested specifically to see what
  the checklist (matured considerably since v2.9.1–v2.9.3, most recently through the MixMenuKit
  passes) would catch on a second look. Reviewed against a Base64-stripped lean copy per this
  doc's own top-line rule (21 embedded GUI asset blobs, one line each, swapped for short
  placeholders — the surrounding code is untouched and line-for-line identical to the real file
  outside those 21 lines, confirmed before editing anything). Three real findings, all new since
  the last pass:
  - **Cat 1 (dead code) — a fully orphaned subsystem, not just a dead field.** `CommitMag`,
    `ResolveLauncherItem`, `ResolvePlayer`, and `ReconcileEntityToMag` (60 contiguous lines) had
    zero call sites anywhere in the file for any of the four — confirmed by grep before touching
    anything, not assumed. Actual magazine tracking is driven entirely by
    `ScheduleMagSync`/`RestoreMagOnEquip`/`GetPlayerMag`, wired to real hooks
    (`OnActiveItemChanged` etc.); this cluster was a separate, never-wired-up reconciliation path,
    likely superseded during an earlier rewrite and never removed. Fixed: removed all four
    methods, replaced with a one-paragraph comment explaining what was there and why (same
    "remove rather than speculatively wire up" call as this file's own dead-code history —
    Magazine/FireModes/Reload/Recoil settings, `AutoKitOnConnect`, `LauncherSkinId`,
    `OnWeaponFired`, `UpdateHud` — consistent with the established pattern, not a new policy).
    Second, smaller instance of the same category: `_panelOpen` (`HashSet<ulong>`) was written on
    every admin-panel open/close/unload but never read anywhere (no `.Contains`, no enumeration) —
    removed along with its four write sites.
  - **New finding class: stale hardcoded version strings, drifted from `[Info]`.** Two separate
    places displayed two separate *wrong* version numbers, neither matching the real `2.9.3`: the
    `OnServerInitialized` boot-log banner said `"Salvo v2.1.0"`, and the admin panel's own status
    badge said `"v2.5.1"`. This is exactly the pattern flagged (from a different plugin, citing
    Salvo's own drift as the cautionary example) earlier in this log — apparently that earlier
    fix interpolated the *other* plugin's banner from `[Info]` but never circled back to fix
    Salvo's own two instances. Fixed: both now interpolate the real `Version` property
    (`$"Salvo v{Version} — ..."` / `$"v{Version}"`) so they can't drift from `[Info]` again,
    matching the pattern already used elsewhere. Worth a dedicated sweep across the other live
    plugins for the same class, same as the v2.9.3 toggle-persistence finding above — flagged,
    not yet actioned.
  - **Cat 2–11 re-check, no new findings — genuinely converged, not just re-asserted.** Hot-path
    ordering on `OnPlayerAttack`/`OnEntityTakeDamage` still correctly leads with the cheapest
    exclusionary check before any `ResolveRank` call. `SaveData`/`LoadData`/asset-load I/O still
    fully exception-contained. All 7 real per-player dictionaries still fully cleared on
    `OnUserDisconnected` (the `_panelOpen` case was write-only, not a leak, since it's cleaned via
    `CloseAdminPanel` — real issue was Cat 1, not Cat 4). CUI destroy-before-add and full exit-path
    coverage confirmed unchanged. Every console-command branch still bounds-checks `arg.Args`
    before indexing. No `[PluginReference]`, `WarnPluginConflicts` is an honest overlap warning
    not a hidden dependency. `DevServerOpenPanelToAll` still correctly defaults `false`. TOS
    compliance (the v2.9.2 `LauncherSkinId` removal) unchanged, no new skin/DLC code since. The
    wipe-survival judgment call on `PlayerMags`/`Players` data (noted as ambiguous, not a bug, the
    first time this file was audited) remains the same open judgment call, not re-litigated here.
  - Verified live: v2.9.4 deployed to the live Carbon service, compiled clean, boot log confirms
    the banner now reads the real version (`Salvo v2.9.4 — ON · 4 ranks · /salvo admin`), 21 UI
    kit assets warmed correctly, 3x reload stress test — zero exceptions, 0 failed plugins
    server-wide each check.
  - Take-away for the owner's actual question ("what would the new checklist system turn up"):
    genuine findings, not just confirmations — same as the first MixEntityScale pass got compared
    against here. A plugin can pass several clean audit rounds and still be carrying dead code and
    version-string drift that a `grep`-first, trust-nothing pass catches on the next look; neither
    finding here was guessed at, both were confirmed by direct search before being called real.

- **Salvo v2.9.4 → v2.9.5** — full 11-category re-run against the current
  `Salvo-RogueDepot-Submission` copy (confirmed the newest version anywhere on disk before
  starting). Two small, real findings — first genuinely new ones since the "converged" v2.9.4
  pass, both low-severity:
  - **Cat 1 (dead code)**: `ApplyTune`'s `case "kit":` branch (adjusting
    `_config.Kit.AutoKitRockets`) was unreachable — `DrawSettingsTab` only ever sends
    `salvo.admin tune dmg ...`, no button or command path sends `tune kit`. The field itself
    isn't dead (still read as a rank fallback in `GiveTestKit`), just this one tuning path to it
    was. Removed the case and the now-unused `plus` local, same "remove rather than speculatively
    wire up" call as every other dead-code fix in this file's history.
  - **Cat 7 (delayed-callback re-validation)**: `OnMagazineReload`'s `NextTick` lambda called
    `player.ChatMessage(...)` in its "no rockets" branch without re-checking
    `player != null`/`player.IsConnected` first, unlike `ScheduleMagSync`'s equivalent callback
    which already does. Low real-world risk (one-tick delay, and the `HasInventoryRockets` call
    in the same lambda already null-guards internally) but doesn't follow the pattern this
    category exists to enforce. Added the same guard.
  - Both fixes applied to `Salvo.cs` (the real shipped file) and `Salvo-CodeReview-NoAssets.cs`
    (the lean copy this pass was actually read from) identically; `Salvo-Oxide-LATEST.zip`
    rebuilt from the updated `Salvo.cs`. v2.9.4 → v2.9.5. Not yet live-deployed or reload-tested
    (this pack copy isn't the one running on any test server) — verified by brace/paren balance
    and direct diff of both edit sites only.
  - Cat 2–6, 8–11 re-checked, no new findings — everything else matches what v2.9.4 already
    established (hot-path ordering, all 6 per-player dictionaries cleared on disconnect and
    `Unload`, CUI destroy-before-add and full exit coverage, all permissions registered, zero
    unwrapped file I/O, zero `[PluginReference]`s, no compliance surface).

- **Salvo v2.9.5 → v2.9.6 — real, live-confirmed Cat 4 bug, not a static-review finding.**
  `OnUserDisconnected` genuinely does not fire on a real client disconnect against this Carbon
  build — see the Cat 4 methodology note and the 2026-09-02 log entry above for the full live
  test (a real player disconnected against `MixCarbonLeakGuard` with no reload to explain a miss;
  server-confirmed via `global.status`; switching to `OnPlayerDisconnected` fixed it immediately).
  Every previous "Cat 4 clean, all 6 dictionaries cleared on `OnUserDisconnected`" verdict for
  Salvo (v2.9.1 through v2.9.5, above) was true of the source, never verified against an actual
  disconnect — this is the first time that specific claim has been live-tested, and it was wrong.
  Fixed: `OnUserDisconnected` and a new `OnPlayerDisconnected` both now call a shared
  `CleanupOnDisconnect(player)` — every cleanup action inside it (dictionary `.Remove()`, timer
  destruction in the META build) is idempotent, so both hooks firing for the same disconnect is
  safe, not a double-cleanup risk. Applied identically to `Salvo.cs`/`Salvo-CodeReview-NoAssets.cs`
  in both the submission (v2.9.5→v2.9.6) and META (v3.0.1→v3.0.2) builds. **Deployed and verified
  live on AU** (the only Salvo copy actually running anywhere) — v2.9.4 replaced with v2.9.6
  directly, 3x reload clean, zero exceptions each time, config confirmed intact (4 ranks, Enabled
  true) throughout. Not yet re-live-tested with an actual disconnect against *this* fix
  specifically from the checklist's own process — that verification happened via
  `MixCarbonLeakGuard`'s test, not a dedicated Salvo disconnect test; worth doing if the
  opportunity comes up, though the fix is the same pattern already proven to work.

- **MixEntityScale v1.3.0 → v1.4.0** — first full pass, categories 1–11. Smaller/newer plugin than
  Salvo (~680 lines lean vs ~2200) and it showed: genuine findings this time, not just
  confirmations.
  - Cat 1 (dead code): clean — all 8 config fields have real read sites beyond `ClampConfig`, no
    no-op hooks.
  - Cat 2 (hot-path discipline): `OnEntityKill`/`OnLootEntity` both fire server-wide; both already
    lead with the cheapest possible exclusionary check (`Dictionary.Remove`/`HashSet.Contains`,
    both O(1)) before any real work. Clean.
  - Cat 3 (unconditional I/O): `OnEntityKill`'s `SaveData()` is already debounced via a
    `_scaleSavePending` flag + 5s `timer.Once` coalesce — exactly the gating pattern this category
    asks for, already present. `DrawPanel` destroys before every add. Clean.
  - **Cat 4 (per-player cleanup) — real finding**: no `OnPlayerDisconnected`/`OnUserDisconnected`
    hook existed at all. `_ui` (Dictionary) and `_panelOpen` (HashSet) were never cleaned up on
    disconnect — a slow leak on any server with real connect/disconnect churn, exactly the kind
    this category exists to catch. Fixed: added `OnPlayerDisconnected` clearing both.
  - Cat 5 (permissions): clean — `mixscale.admin` registered; the `mixpack.admin` check in
    `CanUse` is intentionally not registered here since it's MixCore's own permission, checked not
    owned.
  - Cat 6 (CUI lifecycle): destroy-before-add confirmed in `DrawPanel`; both `CursorEnabled=true`
    panels (Dim overlay, root panel) have confirmed destroy paths on every exit *except* the
    disconnect gap from Cat 4, now fixed. No auto-drawn connect-time UI, so the connect-timing
    item doesn't apply. `MixUiV2.Card`'s `CursorEnabled=true` is the same unused shared-class dead
    code already noted in Salvo's audit — left alone for the same reason.
  - Cat 7 (bounds/deny-path): `ConsoleUi` bounds-checks `arg.Args.Length` before every indexed
    sub-action; `CanUse` gates once at the top of the single UI entry point, covering every
    sub-action uniformly — deny path actually denies.
  - **Cat 8 (exception containment) — real finding**: `SaveData()` was completely unwrapped, same
    class of gap as Salvo's original finding (asymmetric with `LoadData()`, which was already
    wrapped) — fixed with try/catch + warning log. Also wrapped the `MixCore?.Call(...)` module
    registration (minor, lower risk, but cheap to fix while in the area).
  - Cat 9 (dependency honesty): `MixCore` reference is null-safe and purely optional (hub-UI
    registration, not a feature dependency) — confirmed via the earlier live isolation test
    (v1.3.0 ran with zero other plugins, zero exceptions). Clean.
  - Cat 10 (compliance): not applicable — no skin/cosmetic-assignment feature exists in this
    plugin.
  - **Cat 11 (lifecycle) — real, clear-cut finding**: `OnNewSave` wasn't implemented, and unlike
    Salvo (where "should data survive a wipe" is a judgment call), this one has an unambiguous
    answer — `EntityScale` is keyed by net ID, which a map wipe invalidates entirely, so every
    entry would sit in the data file forever as permanently-orphaned dead weight rather than ever
    restoring anything. Fixed: added `OnNewSave` clearing `EntityScale`. `OnServerInitialized`'s
    placement (not `Init`) was already correct, and correctly runs its reapply-scale loop
    unconditionally on both cold boot and hot reload since that work isn't one-time.
  - Verified live: v1.4.0 loaded clean, zero hook exceptions. 10x reload stress test run:
    memory did not climb (actually dropped between passes, no leak), zero exceptions across all
    10 reloads.
  - Net new fixes this pass: 3 real (disconnect cleanup, SaveData exception safety, OnNewSave
    wipe-data clearing) plus 1 minor (MixCore call containment) — more than Salvo's last two
    passes combined, consistent with this being MixEntityScale's *first* full pass rather than a
    recheck.

- **FreeBuild v2.2.0 → v2.3.0** — MixImages removal (self-installing embedded assets, same
  pattern as Salvo/MixEntityScale) plus first full checklist pass, categories 1–11.
  - **Cat 1 (dead code) — real finding**: 4 dead config fields. `AutoPowerNearbyIo` and
    `AutoPowerIntervalSeconds` implied a periodic background auto-power feature that was never
    actually wired to a timer — only the manual `/freebuild power` command (using
    `AutoPowerRadius`, which *is* real) exists. `DevServerOpenPanelToAll` was force-reset to
    `false` on every `LoadConfig` with no read site anywhere — could never do anything even if an
    admin hand-edited it. `AdminGrantOnly`'s described behavior ("only owner/freebuild.admin may
    grant") is hardcoded directly into `IsPanelAdmin()` rather than actually gated by the field.
    All 4 removed.
  - Cat 2 (hot-path discipline): `OnEntityTakeDamage`, `OnEntityBuilt`, `OnEntitySpawned` are all
    server-wide hooks — checked ordering on all three. `OnEntitySpawned` in particular already
    excludes the highest-volume entity types (trees, ore, corpses, dropped items, NPCs,
    building blocks) via a cheap type-pattern match before touching `OwnerID` or calling the more
    expensive `PlayerHasFreeBuild` — already excellent discipline, no changes needed.
  - Cat 3 (unconditional I/O): `SaveData()` call sites are debounced (`TouchPlayer`'s 5s
    `timer.Once`) or human-rate-limited (explicit admin/self toggle actions) — clean. `DrawPanel`
    destroys before every add — clean.
  - Cat 4 (per-player cleanup): `_creativeOn` correctly cleared per-disconnect and in `Unload`.
    **Real, minor finding**: `_adminView` was cleared on `ClosePanel`/disconnect but not in
    `Unload` — same low-severity inconsistency class as Salvo's `_lastSalvoShotRealtime` finding.
    Fixed for consistency.
  - Cat 5 (permissions): clean — all 3 permissions registered and used.
  - **Cat 6 (CUI lifecycle) — real finding, more than cosmetic this time**: `Unload()` destroyed
    `UiRoot` for every online player but never `$"{UiRoot}.Dim"` — the `CursorEnabled=true` dim
    overlay. Unlike the MixEntityScale audit (where the equivalent gap was disconnect-only),
    this one is reachable through an ordinary live plugin reload while an admin has the panel
    open — exactly the "leaked cursor-lock panel freezes mouse-look" bug class this category
    exists to catch, not just a theoretical edge case. Fixed.
  - Cat 7 (bounds/deny-path): `CmdFreeBuild` and `ConsoleUi` both bounds-check args consistently
    across every sub-action; `ConsoleUi`'s single-entry-point permission gate covers every
    sub-action uniformly. Delayed callbacks inside `SetCreativeMode` re-validate connection state
    at point of use. Clean.
  - **Cat 8 (exception containment) — real finding**: `SaveData()` unwrapped, same recurring
    pattern as Salvo and MixEntityScale (asymmetric with an already-wrapped `LoadData()`). Fixed.
    Everything else (entity API calls, cross-plugin `MixToolBar?.Call`) was already
    defensively wrapped — this plugin's author was already exception-conscious elsewhere, just
    missed the one spot every other audit has also caught.
  - Cat 9 (dependency honesty): `MixToolBar` reference is genuinely optional (cosmetic toolbar
    refresh, already wrapped in try/catch, core features work without it) — confirmed correctly
    optional. MixImages dependency fully removed this pass.
  - Cat 10 (compliance): not applicable — no skin/cosmetic-assignment feature.
  - Cat 11 (lifecycle): `Init`/`OnServerInitialized`/`Unload` placement already correct.
    `OnNewSave` deliberately *not* implemented — audited and confirmed correct as-is: unlike
    MixEntityScale's net-ID-keyed data (invalidated by any wipe), FreeBuild's data is keyed by
    Steam ID (who's been granted access, last-seen timestamps) — genuinely should survive a wipe
    so admins don't have to re-grant access every time. Also fixed two hardcoded `"v2.2.0"`
    version-string literals (boot banner, panel badge) to `$"v{Version}"` — same stale-string
    pattern as Salvo's boot banner, now interpolated so it can't drift from `[Info]` again.
  - Verified live: v2.3.0 loaded clean, zero hook exceptions, all 8 UI kit assets self-installed
    independent of MixImages. 10x reload stress test: memory dropped between passes (no leak),
    zero exceptions across all 10 reloads.
  - Net new fixes this pass: 4 real (dead config ×4 as one finding, Unload CUI/dictionary
    cleanup, SaveData exception safety) plus a version-string consistency fix — first full pass,
    comparable in yield to MixEntityScale's first pass.

- **MixSignboard v1.7.0 → v1.8.0** — the "mixed case": MixImages stays as a genuine, disclosed
  dependency for the slideshow feature itself (image storage/import), only the 8-file UI-kit
  chrome gets the self-installing embedded-assets treatment. First full checklist pass,
  categories 1–11 — the richest yield of any plugin so far, several real bugs rather than mostly
  dead-code/consistency findings.
  - **Cat 1 (dead code) — 2 real findings**: `PermAdminLegacy` ("mixsign.admin") was registered
    in `Init` but never actually checked anywhere in `IsAdmin()` — a backward-compat permission
    that silently did nothing for anyone granted it. Wired into `IsAdmin()` rather than removed,
    since unlike a numeric tuning field this is a single well-understood permission-check
    addition to an existing OR chain, not new untested behavior. Separately, `ClosePanel(BasePlayer)`
    was dead code — defined, never called anywhere (the real, working close path is
    `CloseAdminPanel`) — and was also independently buggy (destroyed `UiRoot` but not the
    `CursorEnabled=true` Dim overlay). Deleted rather than fixed, since nothing reaches it.
  - **Cat 2 / Cat 7 — real bug, not just a code-quality nit**: `OnEntityKill(BaseNetworkable
    entity)` fires on every entity kill server-wide and accessed `entity.net.ID.Value` with only
    a null-check on `entity` itself, not `entity.net` — a real NullReferenceException risk on a
    hook that runs constantly, for any BaseNetworkable that isn't networked. Fixed:
    `entity?.net == null` guard before touching `.net`.
  - Cat 3 (unconditional I/O): all `SaveData()` sites are event-driven (explicit admin actions,
    entity-kill cleanup, one-time restore), and — notably different from every other plugin
    audited so far — `_data` is saved synchronously immediately after nearly every mutation
    rather than debounced, which turns out to be *correct* here (see Cat 11 below: nothing left
    unsaved at `Unload` time). `DrawPanel`/`CloseAdminPanel` destroy-before-add correctly.
  - **Cat 4 (per-player cleanup) — real, significant finding**: no `OnPlayerDisconnected` /
    `OnUserDisconnected` hook existed *at all* in this plugin — `_uiDraft` and `_panelOpen` were
    only ever cleared on full plugin `Unload`, never per-disconnect. Same bug class as
    MixEntityScale's original finding, but this plugin had zero disconnect handling rather than
    a partial one. Fixed: added the hook.
  - Cat 5 (permissions): `PermAdmin` correctly used; `PermAdminLegacy` fixed under Cat 1.
  - Cat 6 (CUI lifecycle): resolved by deleting dead `ClosePanel` under Cat 1 — the only
    `CursorEnabled=true` panels reachable in practice (`CloseAdminPanel`'s Dim overlay) already
    have a confirmed destroy path everywhere real code calls it. No connect-time auto-drawn UI,
    so the connect-timing item doesn't apply.
  - Cat 7 (bounds): `ConsoleUi` bounds-checks `arg.Args.Length` before every indexed sub-action
    across its ~20 branches — consistent throughout. `OnEntityKill`'s null-safety gap covered above.
  - **Cat 8 (exception containment) — real, and broader than usual**: `SaveData()` unwrapped
    (5th plugin this session with the identical gap — Salvo, MixEntityScale, FreeBuild, now this
    one). Fixed. More notably: **every one of the 11 `MixImages.Call(...)` sites in the file was
    completely unwrapped** — `QueueSignboardBanner`, `GetSignboardBannerPng`, `TryGetImageBytes`,
    `TryGetImageCrc`, both calls in `QueueImageImport`, `ImportUrlImage`, and `DrawThumb` (the
    last one particularly worth catching — a thrown exception there would have broken the entire
    gallery panel render over one bad thumbnail, not just that cell). Given MixImages is a
    genuine external dependency here (not optional/cosmetic like FreeBuild's `MixToolBar`), an
    unwrapped call is a real single-point-of-failure risk. All 11 wrapped.
  - Cat 9 (dependency honesty): the actual point of this pass — MixImages is kept as a real,
    disclosed dependency for the slideshow feature, while the UI kit chrome is now independent.
    Verified live, not just by reading the code: unloaded MixImages entirely and reloaded
    MixSignboard — UI kit still self-installed and warmed all 8 assets, panel chrome fully
    functional; the slideshow feature correctly logged "MixImages not loaded — slideshow images
    unavailable" and degraded gracefully rather than throwing. Zero hook exceptions throughout
    the test. This is the reference case for what a *correctly* mixed dependency looks like —
    contrast with Salvo/MixEntityScale/FreeBuild where the honest answer was "not needed at all."
  - Cat 10 (compliance): not applicable.
  - **Cat 11 (lifecycle) — real, nuanced finding**: no `OnNewSave` existed. Unlike a clean
    "clear it" (MixEntityScale) or "leave it" (FreeBuild) case, this plugin's persisted data is
    *mixed*: `Bindings` (Dictionary keyed by entity net ID → preset) is invalidated entirely by a
    wipe, same reasoning as MixEntityScale's `EntityScale`; but `CustomImages`/`CustomSlideshows`/
    `LibraryIntervals` (keyed by admin-chosen string identifiers, not map entities) are exactly
    the kind of data that should survive — an admin's custom-built slideshow presets and folder
    libraries shouldn't have to be rebuilt after every wipe. Fixed: added `OnNewSave` that clears
    *only* `Bindings`, leaving everything else intact — the first plugin this session where the
    correct wipe-handling isn't a blanket clear-or-keep, it's a partial one.
  - Verified live: v1.8.0 loaded clean, zero hook exceptions. 10x reload stress test: memory
    stable, zero exceptions. Separately verified the Cat 9 dependency-honesty claim live (see
    above) rather than just asserting it from the code.
  - Net new fixes this pass: 6 real (legacy-permission wire-up, dead/buggy `ClosePanel` removal,
    `OnEntityKill` null-safety, missing disconnect cleanup, `SaveData` + 11×`MixImages.Call`
    exception safety, partial `OnNewSave`) — the highest-yield full pass of any plugin so far,
    consistent with this being both a first pass *and* a larger, more complex plugin (2291 lines
    vs FreeBuild's 1593 / MixEntityScale's 678).

- **MixCore v0.9.24 → v0.9.26** — the hub plugin, 5668 lines, not a standalone product (no
  submission-ready packaging, internal only). Mapped via targeted greps rather than a full linear
  read given the size; first full pass, categories 1–11.
  - **Cat 3 — real, high-impact finding**: `OnEntityDeath` (fires on every entity death
    server-wide — every player, NPC, and animal kill) called `AddBalance(...)` for PvP/NPC/animal
    kills using the method's default `persist: true`, meaning a full `_stats`+`_balances`
    disk-write synchronously on every single qualifying kill whenever points are enabled — the
    exact "unconditional SaveData on a combat hot path" class Ricky's Salvo review caught,
    except in the hub plugin instead of one weapon, meaning it affects every kill server-wide,
    not just launcher shots. Worse in bursts (raiding a scientist-filled monument, a hunting run).
    Fixed with a genuine debounce (`QueueStatsSave()`, 5s coalesce), not just a relocated call —
    first attempt at this fix only moved the save around without reducing call frequency at all,
    caught and corrected before deploying. Also found and fixed the identical pattern in
    `OnPlayerDisconnected` (unconditional full-dataset save on every disconnect — a mass-
    disconnect event like a server restart would fire it once per player in the same tick).
  - **Cat 5 — real finding, but corrected after live testing (see the new checklist items in
    category 5 and the reload-vs-restart item in category 4)**: `mixpack.admin`/`mixpack.use`
    looked unregistered anywhere in the pack from a grep across every plugin for
    `RegisterPermission(PermAdmin`/`PermUse` — added registration to MixCore as the fix. A
    restart (done specifically to verify this fix, at the user's call) revealed MixGovern
    already registers the entire shared "mixpack.*" namespace correctly via a loop over a
    30-entry `ManagedPerms` table, which the direct-call grep pattern never would have caught.
    The added registration was reverted; net result for this plugin is a documentation-only
    correction, but the methodology fix (search for the permission string, not the call shape;
    verify load-order-sensitive fixes on a restart, not just a hot reload) is now generally
    applicable and was worth the false start.
  - Cat 4 (per-player cleanup): `_hubNav`, `_modsPage`, `_uiDebug`, `_uiDebugLast` were never
    cleared on disconnect (only `_hubShellActive` was) — same bug class as MixSignboard's finding.
    Fixed: added cleanup for all four to `OnPlayerDisconnected`, and separately added them to
    `Unload()` too (which was also missing them, plus missing the `{UiRoot}.Dim` overlay destroy
    — same CUI-leak class as FreeBuild's `Unload` finding).
  - Cat 6: `OpenHub` (the real, frequently-used panel-open path) already destroys `UiRoot` and
    `{UiRoot}.Dim` correctly before every add; `RefreshStatusHud` also destroys-before-add
    correctly. Only `Unload()` had the gap, covered under Cat 4 above.
  - **Cat 8 — real, found twice in one file**: `SavePersistedData()` (the main stats/economy
    save) was unwrapped, same recurring pattern as every other plugin audited this session.
    `SaveModManagerState()` — a completely separate persistence path in the same file for the
    mod-manager's own state — had the identical gap, independently. Both fixed; both had an
    already-wrapped `Load*` counterpart, so the asymmetry pattern held here too.
  - Cat 1/2/7/9/10/11: no further findings from the sections actually read in depth (config
    dead-field sweep was not exhaustive given the file's size — flagged as not fully covered,
    unlike the complete sweeps done on the four standalone plugins).
  - Verified live: v0.9.26 loaded clean on both a hot reload and a genuine full service restart
    (the latter specifically to validate the Cat 5 correction), zero hook exceptions either way,
    zero "already used by another plugin" warnings post-fix. 5x reload stress test: stable,
    memory flat.
  - Net new fixes this pass: 3 real (OnEntityDeath debounced save, OnPlayerDisconnected
    debounced save, disconnect/Unload cleanup for 4 dictionaries + Dim overlay) plus 2 exception-
    safety wraps, plus one corrected false-positive that improved the checklist's own methodology
    more than it improved this plugin.

- **MixCore v0.9.27 -> v0.9.28** -- full 11-category re-run, owner-requested as a direct follow-up
  to the Salvo re-audit, specifically to see whether the hub plugin (last covered at v0.9.26, with
  categories 1/2/7/9/10/11 explicitly flagged as *not fully swept* at the time given the file's
  size) would turn up anything new on a genuinely complete pass this time -- not the targeted-grep
  approach the first pass used. 5749 lines, no embedded Base64 (confirmed -- no line over 500
  chars), so reviewed directly rather than needing a lean copy.
  - Read the file in full (not sampled), then verified every non-trivial claim with a targeted
    grep before calling it a finding -- same discipline as the Salvo pass.
  - **Cat 1 (dead code) -- one real, minor finding**: `ConsoleCloseUi` was defined but never
    registered via `cmd.AddConsoleCommand` anywhere in `Init()`, and never called from anywhere
    else either (confirmed by grepping the whole file for the method name before removing it).
    Not a functional gap -- the actual close-UI behaviour (`CmdCloseUi`) is still fully reachable
    through `CmdMixPack`'s own subcommand routing, and `Init()`'s own comment already explains why
    MixCore deliberately doesn't register a standalone `closeui` command of its own (MixUiFix owns
    that namespace; registering a duplicate would just collide with it -- same class of
    self-inflicted collision as MixMenuKit's `/mixmenu`-trigger finding, avoided correctly here).
    Removed the orphaned method.
  - **Cat 2/3 (hot-path discipline) -- re-verified clean, all three real hot-path hooks read
    directly**: `OnLootEntity` (fires on every loot-open server-wide) leads with a single
    `HashSet.Contains` before any real work. `OnEntityDeath` (every kill server-wide) already
    debounces via `QueueStatsSave()` from the v0.9.26 fix -- confirmed still in place, not
    reverted. `OnDispenserGather` (the single most frequent of the three -- fires on every
    resource swing) does no I/O at all, just an in-memory stat increment gated by a cheap
    `Dictionary.ContainsKey` early-return in `EnsureStatsEntry`. `_statsSavePending` debounce
    flag confirmed still correctly gating `QueueStatsSave`.
  - **Cat 8 (exception containment) -- re-verified, one non-issue chased down explicitly**:
    `MixData.Read<T>` itself has no internal try/catch (unlike `Write<T>`), which looked like a
    gap at first glance -- but its only two call sites are both inside `LoadPersistedData()`,
    already wrapped in its own try/catch. Confirmed by reading the actual call sites, not assumed
    safe from the class name alone.
  - **Cat 9 (dependency honesty) -- every one of the 12 `[PluginReference]` fields checked**:
    reference-counted each by name across the whole file (all had 4+ real uses, none were
    zero-reference dead declarations) and grepped for any non-null-conditional `.Call(` on any of
    them -- zero found; every actual call site uses `?.`. `API_GrantVanillaOwner` (grants
    permanent vanilla server ownership -- the single most dangerous action in the whole pack)
    re-verified to independently re-check `IsAdmin()` on the requester rather than trusting the
    caller already did, per its own hardening comment from a prior pass -- still in place.
  - **Cat 5/6/7/10/11 -- re-verified, no new findings**: `mixpack.*` deliberately not
    double-registered here (MixGovern owns it via its `ManagedPerms` table -- the v0.9.26
    correction holds). `Unload()`/`OnPlayerDisconnected` cleanup for all tracked dictionaries plus
    the `Dim` overlay confirmed still present (not regressed since the last fix). Spot-checked
    every registered console/chat handler (`ConsoleMods`, `ConsoleStatusHud`, `ConsoleHub`,
    `ConsoleMixPackReload`, `CmdMixPack`) -- all null-check `player` and guard `arg.Args` before
    indexing. `DevServerOpenPanelToAll` still correctly defaults `false`. No `OnNewSave` -- correct
    by design, not an oversight: stats/economy balances are meant to persist across a wipe, same
    reasoning already established for this category of data elsewhere in the pack.
  - Verified live: v0.9.28 deployed to the live Carbon service, compiled clean (large file, ~87s
    compile -- normal for its size, not an error), zero exceptions, 0 failed plugins server-wide,
    boot summary banner unchanged and correct. 3x reload stress test, stable.
  - Take-away: genuinely converged this time -- one small, low-severity dead-code finding, nothing
    else new. Worth noting as the contrast case to Salvo's pass the same session: a plugin that's
    already been through one thorough audit *can* come back this clean on a second full pass --
    the checklist isn't guaranteed to keep finding things, and reporting "basically clean" honestly
    here is the same discipline as reporting real findings elsewhere, not a sign the pass was
    rushed.

- **MixGovern v0.9.3 -> v0.9.5** -- first full 11-category pass this plugin has actually had
  (earlier touches were single targeted fixes -- the authLevel/moderator lockout fix, the
  API_GrantVanillaOwner hardening -- never a systematic sweep). 3888 lines, no embedded Base64,
  read directly. Genuinely hot hooks here unlike MixCore/Salvo's re-audits: `OnPlayerInput` (fires
  many times per second per connected player -- about as hot as an Oxide hook gets),
  `OnEntityTakeDamage` (two overloads), `OnRunPlayerMetabolism`, `OnHammerHit`, `OnPlayerChat`.
  Five real findings -- more than either re-audit this session, consistent with this being a
  genuine first full pass rather than a recheck.
  - **Cat 2/3 -- the standout finding of this session's re-audit run, more severe than either
    MixCore's or Salvo's own hot-path bugs.** `TryBlockDamage` -- called from
    `OnEntityTakeDamage(BaseCombatEntity, HitInfo)`, which fires on every damage event
    server-wide: every hit landed by every player, every explosion, every NPC attack -- called
    `RefreshRules()` unconditionally as its very first line. `RefreshRules()` does a cross-plugin
    `MixCore?.Call("API_GetRulesJson")` (a full `JsonConvert.SerializeObject` on MixCore's own
    side) and then `JsonConvert.DeserializeObject` back into a fresh `RuleConfig` on this side --
    on every single combat hit on the entire server. The same "unconditional expensive work on a
    combat hot path" bug class already caught in Salvo and in MixCore's own `OnEntityDeath`, just
    markedly worse: a JSON round-trip plus cross-plugin dispatch, not a debounced disk write, on
    an even hotter hook. Fixed: removed the hot-path call. Checked every other `RefreshRules()`
    call site first (8 total) before removing this one -- the other 7 are all human-rate-bounded
    (boot, a MixCore reload, chat commands, UI panel renders) and `OnScheduleTick` (already
    running every 60s via an existing `timer.Every`) already calls `RefreshRules()` on its own
    cadence, so `_rules` stays reasonably fresh -- up to 60s of staleness on a live rule change
    reaching this one check, traded against a JSON round-trip on every hit.
  - **Cat 4 -- real finding, the `PersistVanish`/`PersistGod` asymmetry.** `PersistVanish=false`
    correctly removes a disconnecting player from `_vanished` in `OnPlayerDisconnected`.
    `PersistGod` (the parallel setting for godmode) was checked in exactly two places -- gating
    whether to *re-apply* godmode's visual/health-reset step on reconnect, and whether to *load*
    saved gods from disk at boot -- but nothing ever removed a disconnecting player from `_gods`.
    Since `OnEntityTakeDamage`/`CanBeWounded`/`OnRunPlayerMetabolism` all gate on `IsGod(player)`
    directly (a raw `_gods.Contains` check), a player who disconnected while in godmode kept full
    damage/wound/metabolism protection on reconnect within the same server session regardless of
    `PersistGod=false` -- only the *re-apply* step was skipped, not the actual protection, which
    was never actually removed in the first place. Fixed: added the same conditional removal
    `_vanished` already had, mirrored for `_gods`.
  - **Cat 8 -- three separate instances of the identical "Load wrapped, Save not" gap in one
    file.** `SaveTeleportData()`, `SaveModeration()`, and `SaveState()` were all completely
    unwrapped while their own `Load*` counterparts were already wrapped -- the exact pattern
    already found once each in Salvo (`SaveData`) and MixCore (`SavePersistedData`,
    `SaveModManagerState`) this session, turning up three times over in this one file. All three
    fixed with the same try/catch + warning-log pattern used everywhere else in the pack. Also
    fixed, lower severity: the one `MixWorldTune.Call(...)` site (`BuildRecyclerView`, an
    admin-panel render) was already null-guarded (`if (MixWorldTune != null)`) but not
    exception-wrapped against the *other* plugin's own handler throwing -- wrapped for
    consistency with the standard already applied to other genuine cross-plugin dependencies
    elsewhere in the pack.
  - **Cat 1/5/6/7/9/10 -- checked, no new findings.** All 18 per-player collections
    reference-counted -- none were zero-reference dead declarations; `_tprPending`/`_pendingTp`
    (pending teleport requests) looked like they might need disconnect cleanup too but turned out
    already safe on inspection -- both self-heal via an explicit `ExpiresAt` check on next access,
    and both consumers (`CmdTpa`/`CmdTpn`) already null/connection-check the requester before
    using it. `ManagedPerms` registration confirmed still the correct, singular owner of
    `mixpack.*` (matches the corrected understanding from the earlier session-external-report
    pass). Every registered command handler spot-checked for `player`/`args` null-safety. No
    `DevServerOpenPanelToAll`-class bypass flag exists in this plugin. No skin/DLC code.
  - **Cat 11 -- flagged as a real, unresolved judgment call, deliberately not decided
    unilaterally.** No `OnNewSave` existed, so `_homes` (player-set home points, tied to absolute
    world coordinates) survived a map wipe. Unlike Salvo's magazine contents or MixCore's economy
    balances (administratively/socially meaningful regardless of terrain), a home coordinate
    pointing at pre-wipe terrain is actively wrong after a regen -- could sit underwater, inside a
    rock, or off the new map entirely -- not just stale. Surfaced to the owner rather than guessed
    at either way (implementing the clear is irreversible on the next actual wipe); owner decision:
    clear homes on wipe. Fixed (v0.9.5): added `OnNewSave(string filename)`, clears `_homes` and
    calls `SaveTeleportData()` (now itself exception-safe from the fix above), logs how many
    homes/players were cleared. Deliberately scoped to homes only -- moderation/vanish/god/
    first-seen data is untouched, matching the same "administrative data survives, world-position
    data doesn't" reasoning. Confirmed the hook only fires on an actual map save event, not a
    plugin reload: `teleports.json` re-inspected after 4 separate reloads this pass, unchanged
    throughout.
  - Verified live: v0.9.5 deployed, compiled clean, zero exceptions, 0 failed plugins server-wide.
    3x reload stress test stable (plus the reload that shipped v0.9.4 first, then v0.9.5 on top).
    Moderation/teleport/admin-state data files re-inspected after each deploy -- all valid,
    well-formed JSON, existing data (one live `Gods` entry) intact and never spuriously cleared.
  - Take-away: the richest of this session's three re-audits, and the reason is structural, not
    incidental -- this was MixGovern's actual *first* full 11-category pass, not a recheck, and it
    owns the pack's hottest hooks (`OnPlayerInput`, `OnEntityTakeDamage`) by a wide margin. Same
    pattern as MixEntityScale's first full pass finding more than Salvo's second look earlier this
    session: a genuine first pass finds real things; a plugin that's already been swept once
    (MixCore) can legitimately come back clean.

- **MixSkinsLight v1.2.9 → v1.3.0** — found via a status check, not a scheduled pass: not
  currently deployed on the Carbon server at all, existing only as a held-back v1.2.4 copy on a
  separate, older Oxide server install on the same VPS (`plugins-hold-not-ship/`, a deliberate
  "don't ship" folder) plus a newer v1.2.9 in `submission-ready/`. First full checklist pass,
  categories 1–11 — genuinely the cleanest-built plugin audited this session on categories 1–9
  (per-player cleanup, permission registration, CUI lifecycle, destroy-before-add were all already
  correct without any fix needed) — but categories 7, 8, and especially 10 had real findings.
  - Cat 1–6, 9, 11: clean. Notably: `_ui` cleaned on both disconnect and `Unload`; all 4
    permissions (including the legacy aliases) both registered *and* checked, unlike
    MixSignboard's earlier finding; `DestroyUi` already reachable from disconnect, death, any
    loot-session-end (with a comment showing the author already understood the exact
    cursor-lock-leak risk category 6 exists to catch), and `Unload`; destroy-before-add confirmed
    in `OpenUi`. `OnNewSave` correctly not implemented — the skin whitelist is admin-curated and
    string-keyed, not tied to map entities, so it should survive a wipe.
  - Cat 7 (real, minor): `target.Entity?.net.ID.Value` only short-circuited on `Entity` being
    null, not on `Entity.net` — fixed to `?.net?.ID.Value`.
  - Cat 8 (real, the now-familiar pattern): `SaveConfig()` unwrapped — fixed. Also: the three
    `PlayerDlcApi.Call(...)` sites were unwrapped despite `PlayerDlcApi` being a genuine
    dependency (same category 8 item MixSignboard's `MixImages.Call` findings added) — wrapped,
    with an exception now correctly treated the same as "can't verify" (deny), consistent with
    the Cat 10 fix below rather than accidentally reopening the same fail-open hole via a
    different path.
  - **Cat 10 — the real finding, and the reason this pass happened at all**: `PlayerCanUseSkin`
    failed *open* — every branch where `PlayerDLCAPI` was missing, not loaded, or not yet
    initialized returned `true` (allow) instead of `false` (deny), meaning `EnforceSkinOwnership`
    defaulting to `true` gave the appearance of Facepunch TOS compliance while doing nothing on
    the vast majority of servers (PlayerDLCAPI is a separate, uncommon plugin). Worse: the
    shipped `LISTING.md` documented this as an intentional, acceptable feature ("still works
    fine... doesn't error"), meaning the gap had been seen and accepted as fine, not missed.
    Fixed to fail closed (new checklist item added — check what a compliance check does when it
    *can't* verify, not just that a check exists), with an honest player-facing message
    distinguishing "can't verify, blocked" from "confirmed you don't own this." `LISTING.md`
    rewritten to state the dependency is now effectively required, not optional, when enforcement
    is on (the default) — matches the actual behavior instead of describing around it.
  - Verified live: deployed to the Carbon server for the first time this session, v1.3.0 loaded
    clean, zero hook exceptions. 10x reload stress test: memory flat, zero exceptions throughout.
  - Net new fixes this pass: 1 high-severity (fail-open compliance bug, the actual point of the
    pass), 1 minor null-safety fix, 4 exception-safety wraps (SaveConfig + 3 cross-plugin calls).
    Fewer *categories* of finding than most plugins this session, but the one real finding was the
    most consequential of anything found tonight outside Salvo's original LauncherSkinId issue.

- **MixUiFix v1.2.5 → v1.3.0** — small (378 lines), single-purpose "emergency /closeui" utility
  that destroys other plugins' stuck CUI/input-lock state via `plugins.Find(...)?.Call(...)` and
  guarded reflection into private fields. First full pass. Mostly clean by design: no config (N/A
  cat 1/3), no permission gate (correct — it's a public self-rescue command, not an admin tool,
  matches the earlier per-report spot-check), no CUI of its own to leak (only destroys others'),
  no `Unload()` needed (nothing persisted, nothing drawn by itself, timers are one-shot and
  self-checking). Reentrancy guards (`_restoreGuard`/`_clearGuard`/`LootEndGuard`) are correctly
  scoped to a single synchronous call via try/finally, not a Cat 4 per-player leak risk.
  - **Cat 8 — the real finding, and it mattered more here than usual**: most
    `plugins.Find(...)?.Call(...)` sites were unwrapped. Ordinarily a moderate finding; here it's
    sharper, because this plugin's entire value proposition is "fix everything reliably" — one
    throw from a stale/broken plugin's API would have aborted every cleanup step after it in the
    same call, silently leaving the player still stuck. Fixed via a shared `SafeCall()` helper
    covering every site except one (MixSprint's `API_HubUi`, which has a deliberate nested
    `object[]` argument shape that a generic `params object[]` helper would have flattened
    differently — wrapped with its own explicit try/catch instead, to avoid quietly changing what
    gets passed).
  - Cat 7: reflection-based cross-plugin field access (`GetField(..., NonPublic)`) is inherently
    version-fragile, but already defensively coded — every site null-checks the field/cast and
    silently no-ops rather than throwing if another plugin's internals change shape. Noted as a
    design fragility, not a bug to fix.
  - Verified live: v1.3.0 loaded clean, zero hook exceptions, 5x reload stress test stable.

- **MixHud v4.1.3 → v4.1.4** — genuinely well-built: already has a fingerprint-based CUI skip
  (`_lastFp` — rebuild only when something actually changed), the full-entity-walk event scan
  is properly time-gated independent of the refresh timer (confirms the earlier external report's
  claim it exists was accurate, but the implied severity wasn't — already deliberately throttled,
  with a comment documenting exactly why an uncapped walk is intentional). Most cross-plugin calls
  were already wrapped in try/catch. Connect-time draw already uses the delay+re-validate pattern
  category 6 cites as the reference example. First full pass, 4 small real findings:
  - Cat 5: `mixhud.use` registered but never checked anywhere — no command or code path gated on
    it (`/hud` is unconditional for everyone, matching MixUiFix's precedent for self-service
    toggles). Removed rather than wired up — only `mixhud.hide` (grid-coordinate privacy) was ever
    actually wired to anything.
  - Cat 4: `_hidden`/`_menuOpen` missing from `Unload()`'s clear list (present in
    `OnPlayerDisconnected`, just not `Unload`) — same low-severity consistency gap as Salvo/MixCore.
  - Cat 8: `SaveConfig()` unwrapped (the recurring pattern); two `MixCore?.Call("API_PlaySfx", ...)`
    sites unwrapped — low-stakes (cosmetic SFX) but wrapped for consistency.
  - Stale hardcoded `"MixHud v4.1.0"` in the boot banner, actually 3 versions behind — fixed to
    interpolate `{Version}`.
  - Verified live: v4.1.4 loaded clean, zero hook exceptions, 5x reload stress test stable.

- **MixWorld v0.6.0 → v0.7.0** — furnace splitter, quicksort, backpacks, building upgrades,
  remover tool. First full pass, categories 1–11. Largest, most consequential finding of any
  plugin audited tonight outside the compliance fixes.
  - **Cat 4/8 — the real finding, high severity**: `Unload()` killed every player's backpack
    (`SafeKill` → raw `ent.Kill()`) with zero item preservation. Backpacks deliberately use
    `enableSaving = false` and are deliberately *not* cleared from `_backpacks` on disconnect
    (letting them persist across a player's session is the actual feature — confirmed this is
    correct-by-design, not a leak, before touching it) — which made `Unload()`'s unconditional
    kill a real, live data-loss bug on every plugin reload, not just a rare server restart. A
    routine deploy/update would have silently destroyed every online player's backpack contents.
    Fixed: `SafeKill` now drops contents on the ground at the entity's position before killing,
    mirroring the existing (already-correct) `OnPlayerDeath` drop-on-death pattern in the same
    file. Not live-tested with real items in play (no players online at the time) — verified by
    code review and mirroring an already-proven pattern in the same plugin, flagged rather than
    silently assumed perfect.
  - **Cat 1 — a real, deceptive finding**: the Remover Tool's refund was a confirmed no-op
    (`RefundResources` immediately `return`s, with its own comment explaining a 2026-08 Facepunch
    API change broke it), yet every successful remove told the player "(X% refund)" regardless —
    telling players they got resources back when they got nothing. Fixed the message to be
    honest about the broken state instead of quietly asserting success; removed the dead
    `RefundResources`/`GiveRefundItem` functions rather than leave non-functional scaffolding.
    The underlying refund feature still needs real work to restore (new `EntityBuildCost` API
    shape) — flagged as a decision point, not fixed blind.
  - **Cat 1 — a family of dead UI code**: `AddQoLCard`, `AddPlayerFeatureTile`, and
    `AddTuneColumn` were all defined but never called anywhere — an entire superseded layout
    generation, plus a tune-stepper UI that was built (working backend in `TuneFeature`, a ready
    helper) but never actually wired to a button in the current admin panel. Removed the three
    dead methods; left `TuneFeature`'s backend and config fields alone since their target fields
    (backpack rows, remover grace/refund) are genuinely used elsewhere and reachable via direct
    console command — adding the missing UI buttons would be new feature work, not a safety fix,
    so flagged rather than done unprompted.
  - Cat 1 (dead config): `PanelTitle` (never read — panel titles come from MixCore's hub chrome)
    and `MaxCookedStack` (tunable from a reachable code path, but never read by the furnace-split
    logic it's named for) both removed, along with `MaxCookedStack`'s now-orphaned tune case.
  - Cat 2: `OnHammerHit` already correctly Subscribe/Unsubscribe-gated behind the RemoverTool
    toggle via `SyncRemoverHooks()` — a working example of the category 2 optional-hook pattern,
    not a violation. `OnLootEntity`/`OnEntityBuilt`/`OnPlayerDeath` all lead with cheap
    config/null checks before expensive work. Clean.
  - Cat 8 (the recurring pattern, at unusual scale here): `SaveConfig()` unwrapped — fixed. This
    plugin is deeply MixCore-integrated (11 separate `MixCore?.Call(...)` sites across UI
    building, hub registration, staff checks, and SFX) and every single one was unwrapped — all
    11 wrapped, prioritizing the ones inside a panel-render path (where a throw would have broken
    the whole panel, not just degraded a cosmetic element) and the one inside a 5-card loop
    (where a throw would have aborted every card after the failing one).
  - Cat 9: `MixCore`/`MixImages` both genuinely optional with working local fallbacks throughout
    (the `UiWindow`/`UiHeader`/`UiBody` pattern in particular). Clean.
  - Verified live: v0.7.0 loaded clean, zero hook exceptions, 5x reload stress test stable.
  - Net new fixes this pass: 1 high-severity (backpack data loss), 1 deceptive-UX fix (fake
    refund message), 5 dead-code removals (2 config fields, 2 dead functions from the refund
    path, 3 dead UI-building methods), 11 exception-safety wraps.

- **MixCommerce v0.7.3 → v0.8.0** — the biggest plugin in the "not yet checklisted" backlog by
  call-surface (16 categories, 775 items, full CUI shop, `/mixshop`, `/balance`), and the plugin
  moving real player currency through MixCore's economy bridge. First full pass, categories 1–11.
  - **Cat 4 — a real, previously-unnoticed gap**: no `OnPlayerDisconnected` hook at all, and
    `Unload()` didn't clear any of the plugin's 4 per-player dictionaries (open-category state,
    cached balances, UI state, admin session state) — the exact per-player-leak pattern this
    category exists to catch, just never flagged for this plugin before because nobody had done a
    first pass on it. Added `OnPlayerDisconnected` clearing all 4 on disconnect, matching the
    pattern already established for MixHud/MixWorld/MixCore this session; `Unload()` now clears
    the same 4.
  - **Cat 8 — the real finding, and the highest-stakes instance of this pattern this session**:
    the entire 6-method MixCore economy bridge (`RegisterWithMixCore`, `DefaultCurrency`,
    `PointsCurrency`, `GetBalance`, `TrySpend`, `AddBalance`, `SetBalance`) was unwrapped —
    unlike the SFX/UI-cosmetic instances of this same finding in other plugins tonight, a throw
    here sits directly on the path that spends and credits real player currency, so an unhandled
    exception mid-transaction was a real risk, not just a degraded panel. All 6 wrapped. Also
    wrapped: `RegisterWithMixCore`, `RefreshCommerceHub`, both `API_UiGridCells` call sites (×5
    total across `BuildCategoryCards`/`BuildItemGrid`/`BuildAdminContent`) via a new
    `TryUiGridCells()` helper, both `API_UiActionCard` sites via a new `TryUiActionCard()` helper,
    and all 8 `API_PlaySfx` sites via a new `TryPlaySfx()` helper (same helper-consolidation
    approach as MixUiFix's `SafeCall`, chosen here because the call shape repeats identically at
    every site). Confirmed via grep this covered every `MixCore?.Call` site in the file before
    moving on — including two easy-to-miss ones inside the plugin's own access-gate functions:
    `HasCommerceAdmin`'s `API_IsPackStaff` check and `CanUseShop`'s `API_IsPackStaff`/
    `API_DevOpenToAll` checks, wrapped via new `TryIsPackStaff()`/`TryDevOpenToAll()` helpers so
    an API throw denies access (fails closed) rather than propagating out of a permission gate.
  - Cat 1 (dead code): `UiWindow`/`UiHeader`/`UiBody` confirmed via grep never called anywhere —
    same family as MixWorld's dead standalone-window helpers, leftover from before this plugin
    switched to rendering entirely through MixCore's hub (`API_RenderHub` invoked BY MixCore into
    a container it already owns). Removed.
  - Cat 5: the apparent "`mixpack.use`/`mixpack.shop.use`/`mixpack.shop.transfer`/`mixpack.admin`
    never registered by this plugin" was **not** re-flagged as a new finding — already confirmed
    a false positive earlier this session (MixGovern's `ManagedPerms` table registers all of
    these; the boot-log "already used by another plugin" lines are the expected, harmless result
    of two plugins agreeing on the same permission strings, not a real gap).
  - Cat 2/3/6/7/9/11: clean. Economy-bridge API surface (`API_GetBalance`/`API_TrySpend`/
    `API_AddBalance`/`API_SetBalance`/`API_GetDefaultCurrency`/`API_GetPointsCurrency`) confirmed
    via grep to genuinely exist as `[HookMethod(...)]` on MixCore before investing further audit
    time in it, preempting what could have been a wasted "broken bridge" investigation. No CUI
    leak paths found beyond what the existing per-player state (now correctly cleared) covers.
  - Verified live: safety-checked live risk before deploying (this plugin moves real currency;
    confirmed via the same reasoning applied to MixWorld's backpack fix) — v0.8.0 loaded clean per
    `server.log` (`Unloaded plugin MixCommerce v0.7.3` → `Loaded plugin MixCommerce v0.8.0` with no
    errors in between), zero hook exceptions in `c.plugins`, zero failed plugins. 5x reload stress
    test: clean unload/load cycle every time, zero exceptions throughout.
  - Net new fixes this pass: 1 high-severity (unhandled-exception risk across the entire currency
    bridge), 1 real per-player-leak gap (missing disconnect cleanup), 2 access-gate fail-closed
    fixes, 1 dead-code removal (3 methods), ~19 exception-safety wraps total (6 economy-bridge
    methods, 5 grid-cell sites, 2 action-card sites, 8 SFX sites via helpers, 2 hub/register sites,
    2 access-gate sites).

- **MixWorldTune v0.5.3 → v0.6.0** — world-rate tuning (stacks, cook/craft/gather speed, loot
  multiplier, recycler, presets) plus a legacy day/night sub-panel. First full pass, categories
  1–11.
  - **Cat 1 — the real finding, and the headline of this pass**: the entire "Day / night" admin
    panel (a toggle button plus three +/- steppers for day hours, day length, night length) was a
    fully live-looking, fully clickable control surface that did nothing. `ApplyDayNight()` —
    called after every one of those four actions — unconditionally forces `_tune.DayNight.Enabled`
    back to `false` and `return`s before any of the actual clock-driving code runs (a comment at
    the top explains the *reason*: "MixDayNight plugin owns the clock on this server," a separate
    plugin now genuinely drives day/night). So every click changed a number, played a sound, and
    persisted a value to MixCore's shared tune store, then was silently discarded a moment later —
    worse than a no-op, because the toggle's green/red color and the steppers' displayed values
    kept updating as if something had happened. Distinct from MixWorld's fake-refund-message
    finding only in shape (a whole panel vs. one string), same class of deceptive-UX bug. Fixed:
    `HandleAction`'s four `daynight.*` cases now no-op with a clear "controlled by MixDayNight,
    these controls are inactive" message instead of mutating and silently discarding state;
    `BuildDayNightView` rewritten to a plain delegation notice plus a still-useful current
    day/night readout, instead of rendering a toggle and three steppers wired to commands that do
    nothing; the main menu's day/night card line changed from a fake "Xh day · Yh night · on/off"
    status to "Controlled by MixDayNight."
  - **Cat 1 (dead code), a side effect of the fix above**: removing `BuildDayNightView`'s call to
    `ComputeDayNightBoundaries()` (the only place that ever reachably called it) left
    `ComputeDayNightBoundaries()`/`IsDaylight()`/the `_dayStartHours`/`_nightStartHours` fields with
    zero remaining callers anywhere in the file — genuinely dead, not merely superseded — so
    removed them. In the process, confirmed `IsSunUp()`'s exception fallback depended on those same
    fields, which (even before this pass) were only ever populated by opening the old day/night
    panel first — meaning the fallback silently always reported "night" on a fresh boot before any
    admin had opened that panel. Fixed to a small self-contained fallback (fixed 06:00–18:00 window
    off `ConVar.Env.time`) instead of restoring the removed fields just to patch the compile.
    `UiWindow`/`UiHeader`/`UiBody` also confirmed via grep never called (same family as
    MixWorld/MixCommerce's dead standalone-window helpers) — removed.
  - **Cat 4 — a real, previously-unnoticed gap**: no `OnPlayerDisconnected` hook at all, and
    `Unload()` didn't clear `_adminState` (per-player UI navigation state: view/category/page/
    filter) — same pattern as MixCommerce/MixHud/MixWorld's disconnect-cleanup gaps. Added
    `OnPlayerDisconnected` to remove the entry on disconnect; `Unload()` now clears the whole
    dictionary too.
  - Cat 8: `SaveTuneToCore()` — the single call site that persists every admin change (stacks,
    cook, craft, gather, loot, recycler, day/night, presets) to MixCore — was unwrapped despite
    being invoked from essentially every action in `HandleAction`. Also unwrapped: the boot-time
    `API_RegisterModule` call, `HasAdmin`'s `API_IsPackStaff` check (an access gate — wrapped via a
    new `TryIsPackStaff()` helper so a throw denies rather than propagates, same fail-closed pattern
    as MixCommerce's access-gate fix), `RefreshTuneHub`'s `API_RefreshHub` call, both
    `API_PlaySfx` sites (via a new `TryPlaySfx()` helper), `BuildMainView`'s `API_UiGridCells`/
    `API_UiActionCard` calls (via new `TryUiGridCells()`/`TryUiActionCard()` helpers, same naming
    convention as MixCommerce's for consistency across the pack), and `MixWorldTuneMixUiKit.Get()`
    (called from several hot CUI-building paths — a throw here would have broken tile/button
    rendering, not just degraded it).
  - Cat 5: `mixpack.admin` never registered by this plugin directly — **not** re-flagged as a new
    finding, already an established false positive this session (MixGovern's `ManagedPerms` table
    registers it for the whole pack).
  - Cat 2/3/6/7/9/11: clean. Gather/craft/cook/loot hooks (`OnDispenserGather`,
    `OnDispenserBonus`, `OnCollectiblePickup`, `OnItemCraft`, `OnLootSpawn`/`OnCorpsePopulate`) are
    always-subscribed by design (a global rate-tuning plugin needs to see every relevant hit) but
    lead with a cheap `Mathf.Abs(mult - 1f) < 0.001f` early-out, matching the established
    cheap-early-exit pattern rather than a Cat 2 violation. No standalone CUI of its own to leak
    (fully hub-delegated to MixCore's `API_RenderHub`, same architecture as MixCommerce/MixWorld —
    the `CuiHelper.DestroyUi(player, UiRoot)` calls in `Unload()`/the close handler are harmless
    vestigial safety code against a panel name nothing ever adds). Config values (stack/speed
    multipliers, day/night hours) all clamped on every write path.
  - Deliberately **not** touched: the larger retired day/night driving scaffold (`TryInitDayNight`,
    `PollDayNightPhase`, `CleanupDayNight`, `StartManualTimeDrive`, TOD event hooks, `EnvSync`
    wiring) — all pre-existing, already-unreachable-or-orphaned code guarded by explicit
    "disabled/retired — MixDayNight owns clock" comments from a prior pass. `CleanupDayNight()`
    stays genuinely reachable via `Unload()` and remains a correct, harmless no-op given the
    fields it touches are never set. Ripping out the full scaffold would be a bigger architectural
    change than this QA pass's scope — flagged as a decision point (a good candidate for actual
    removal in a future pass) rather than fixed blind, same precedent as MixWorld's broken
    remover-tool refund path.
  - Verified live: v0.6.0 loaded clean per `server.log`, zero hook exceptions, zero failed plugins.
    5x reload stress test: memory flat (~1.4-1.5mb), zero exceptions throughout.
  - Net new fixes this pass: 1 high-severity deceptive-UX fix (the entire fake day/night control
    panel), 1 latent fallback bug fixed as a side effect (IsSunUp's always-wrong exception path),
    1 real per-player-leak gap (missing disconnect cleanup), 4 dead-code removals (3 UI-window
    helpers + the boundary-computation method/fields), ~9 exception-safety wraps (SaveTuneToCore,
    RegisterModule, access gate, hub refresh, 2 SFX sites via helper, grid-cells, action-card,
    UI-kit image lookup).

- **MixInstantBases v0.9.0 → v0.10.0** — admin-granted instant base placement (4 tiers), a full
  capture/paste engine with electrical IO capture, and a credit-based grant system. Largest
  plugin audited tonight by mechanical complexity (paste engine directly spawns/grounds/wires
  entities). First full pass, categories 1–11.
  - Cat 1 (dead code): two entire static classes confirmed via grep never referenced anywhere in
    the file — `MixInstantBasesMixPackAssets` (a CDN/local-file image-import utility, ~40 lines)
    and `MixUiKit` (a PNG-menu-chrome shim whose own doc comment already said "disabled — always
    delegates to solid MixInstantBasesMixCui panels/buttons," ~27 lines), both superseded by the
    `UiKitAssets`/`RegisterUiKit` pattern actually wired up. Removed both — same family of finding
    as the dead standalone-window helpers found in MixWorld/MixCommerce/MixWorldTune this session,
    just a different kind of leftover (unused utility classes rather than unused methods).
  - **Cat 4 — a real, previously-unnoticed gap**: `_adminGrantTarget` (which admin-selected player
    is currently the grant target in the hub) was never cleared on disconnect or in `Unload()` —
    same per-player-leak pattern as every other plugin's Cat 4 finding this session. Added to both.
  - **Cat 6 — a real, minor CUI leak**: a player mid-placement when the plugin reloads had their
    preview timer/ddraw correctly stopped by `Unload()`, but the small "Left-click: place · Right-
    click cancel" HUD strip (`ShowPlaceHint`, parented directly to `"Hud"`, not tied to session
    state) was never destroyed — it would sit stuck on their screen until they relogged. Fixed:
    `Unload()` now also destroys it for every active player, alongside the existing Hub-UI
    destroy call.
  - Cat 8, the widest-scope finding of this pass: `SaveConfig()` and `SaveData()` were both
    unwrapped raw `WriteObject` calls — `SaveData()` in particular persists real player credit
    balances (spent on every placement, granted/revoked by admins), the same severity class as
    MixCommerce's currency-bridge finding. Worse for `SaveConfig()`: its only call site sits
    *outside* the try/catch that already exists around the rest of `LoadConfig()`, so a write
    failure during plugin init would have gone fully unhandled. Also wrapped: `UiKitAssets.Get()`
    (called from every painted-button/tile render), `IsAdmin`'s `API_IsPackStaff` check (via a new
    `TryIsPackStaff()` fail-closed helper, same pattern as MixCommerce/MixWorldTune's access-gate
    fixes), `OpenHub`/`RefreshHub`'s `API_OpenHub`/`API_RefreshHub` calls, 4 `API_PlaySfx` sites in
    `ConfirmPlace`/`CancelPlace` (via a new `TryPlaySfx()` helper), and `BuildHubContent`'s 3
    `API_UiGridCells` + 2 `API_UiActionCard` sites (via new `TryUiGridCells()`/`TryUiActionCard()`
    helpers, same naming convention as MixCommerce/MixWorldTune's for consistency across the pack).
    One more, specific to this plugin's paste engine: the `timer.Once(0.5f, ...)` callback in
    `PasteLevel` that restores electrical IO connections was unwrapped, and a throw there (e.g.
    from a corrupted/malformed captured blueprint) would have skipped `FinalizeSpawnedIo` — which
    finalizes *every* spawned entity, not just the IO ones — leaving freshly-placed electrical
    components looking broken (never `MarkDirty`/`UpdateOutputs`'d) even though the paste itself
    succeeded. Wrapped just the IO-restore portion so entity finalization still runs regardless.
  - Cat 5: `mixinstantbases.admin`/`mixinstantbases.use` are both genuinely registered in `Init()`
    *and* checked (`IsAdmin`/`CanUse`) — a correctly-implemented example, not a finding. The
    `mixpack.admin` fallback check in `IsAdmin` is the same already-established false positive
    (MixGovern's `ManagedPerms` table) as MixCommerce/MixWorldTune's.
  - Cat 2/3/7/9/11: clean. `OnPlayerInput` (fires on every player's input tick) leads with a cheap
    dictionary lookup before any real work — matches the established cheap-early-exit pattern, not
    a Cat 2 violation. Paste-engine entity work is well null/destroyed-guarded throughout
    (`SpawnEntity`, `RemoveActiveBase`, `FinalizeSpawnedIo`, `LockDoors`/`InstallKeyLock` already
    has its own try/catch). `MixImages` is genuinely optional with working flat-color fallbacks
    everywhere via `UiKitAssets.Get()`'s null-check; `MixCore` is a hard, by-design dependency
    (fully hub-delegated architecture, same as MixWorld/MixCommerce/MixWorldTune) rather than a
    Cat 9 violation. Config values are server-owner-edited (not exposed through the in-game admin
    UI), matching the lower-priority treatment other config-file-only settings got this session.
  - Verified live: safety-checked before deploying given this plugin manages real player-owned
    base entities and credit balances. `server.log` shows two clean load cycles (old-version
    reload, then a full stress pass) with no errors between unload/load in either. Zero hook
    exceptions and zero failed plugins in `c.plugins`. 5x reload stress test: memory flat
    (~2.5-2.6mb), zero exceptions throughout.
  - Net new fixes this pass: 2 dead-class removals (~67 lines), 1 real per-player-leak gap
    (missing disconnect/Unload cleanup for the admin grant-target dictionary), 1 real CUI-leak fix
    (stale placement HUD strip surviving a reload), 1 access-gate fail-closed fix, ~13 exception-
    safety wraps (SaveConfig, SaveData, UI-kit image lookup, hub open/refresh, 4 SFX sites via
    helper, 3 grid-cells sites, 2 action-card sites, the IO-restore paste callback).

- **MixApartmentHome v1.7.2 → v1.8.0** — apartment room home base (place/pickup/move furniture,
  leasing, skeleton-key monument pickup, electrical auto-power, blueprint import/export). Largest
  plugin audited this session (8460 lines) and the one an earlier external report specifically
  flagged for "the most entity-walk overhead." First full pass, categories 1–11.
  - Cat 1 (dead code): `MixApartmentHomeMixPackAssets` confirmed via grep never referenced
    anywhere in the file — the same leftover CDN/local-file image-import utility class already
    found and removed from MixInstantBases this session, here superseded by the working
    `UiKitAssets` pattern. Removed (~43 lines). Note: unlike MixInstantBases' `MixUiKit`, this
    plugin's own `MixApartmentHomeMixUiKit` shim genuinely *is* called once (a close-button
    delegate) — checked before assuming it was the same dead pattern, and left alone.
  - **Cat 4 — a real, previously-unnoticed gap, on this plugin's largest set of tracked state**:
    despite `Unload()` already thoroughly clearing 8 of its per-player/per-room dictionaries (this
    plugin was otherwise unusually disciplined about Cat 4 already), there was no
    `OnPlayerDisconnected` hook anywhere in the file. Six player-keyed mode/target collections
    (`_pickupMode`, `_skeletonKeyOff`, `_moveMode`, `_moveTarget`, `_wireSource`, `_uiTab`) were
    only ever cleared by an explicit toggle-off command or a full plugin `Unload()` — a player who
    disconnected mid-pickup/move/wire mode without exiting it first left a permanent entry behind
    until the next reload. Added `OnPlayerDisconnected` clearing all 6. (`_pendingResetConfirm` and
    `_uiRestoreGuard` were checked too but are genuinely self-cleaning via their own 30s/0.15s
    timers regardless of connection state — not the same leak class, left alone.)
  - **Investigated the external report's "entity-walk overhead" claim directly, rather than taking
    it at face value**: grepped every raw `foreach (var net in BaseNetworkable.serverEntities)` in
    the file (10 sites) and checked each call site's actual frequency. Most are already properly
    mitigated — admin-only diagnostic/survey commands (`/apt survey`, `/apt assets`, `/apt strip`),
    one-shot boot timers, or already TTL-cached (`BuildSurvey`'s 30s cache). `BuildContext` — the
    room-resolution function on the actual hot path (called on nearly every player interaction) —
    already has explicit comments ("Never full-scan wings in hot path — survey/cache only," "Skip
    GetComplexLabelForRoom (can walk world) unless admin/survey path needs it") showing the author
    had already engineered around this exact risk, most likely in response to that same report.
    **One genuine instance survived**: `StripLeasedRoom` (fires on every actual rent/vacate
    transition) did its own raw full-`serverEntities` scan matching by `GetInstanceID()`, duplicating
    a lookup `ApartmentRoomBridge` already maintains as a cached `instanceId → component` dictionary
    (`TryGetCachedRoomByKey`, same `MatchesRoomType`/`IsProtectedRoomEntity` predicate). Fixed to use
    the existing cache instead of re-walking every entity on the server on every lease transition.
  - Cat 8: `SaveConfig()`/`SaveData()` both unwrapped (`SaveData` persists real room/furniture
    placement records — same severity class as MixInstantBases' credit-persistence finding).
    `RegisterWithMixCore` was two unwrapped calls with no try/catch at all (not even the recurring
    "wrapped elsewhere, one call slipped through" pattern — genuinely bare). Also wrapped: 6
    `API_PlaySfx` sites (via a new `TryPlaySfx()` helper), `MixSkinsLight.Call("API_ApplySkinToEntity"...)`
    and 3 `MixRackKit.Call(...)` sites (all already correctly null/IsLoaded-checked before the call,
    just not exception-wrapped — a throw mid-command left the player with no reply at all instead
    of a graceful error message), `RefreshHub`/`API_UiSubTabBar` (hub render path), the deferred
    `API_RestoreControls` call to MixUiFix, and `UiKitAssets.Get()` (same recurring miss as every
    other plugin's painted-UI-kit registry this session).
  - Cat 2/3/5/6/7/9/11: clean. `PermUse`/`PermMove`/`PermAdmin` all genuinely registered *and*
    checked. `_hubEmbed`/`_uiCmdPrefix` are plugin-instance (not per-player) fields toggled during
    hub-embedded rendering — looked like a shared-mutable-state risk at first glance, but they're
    correctly scoped in a `try/finally` around the render switch and Rust/Carbon plugin hooks run
    single-threaded to completion, so this is safe as written, not a bug. `MixRackKit`/`MixSkinsLight`/
    `MixUiFix`/`MixImages` are all genuinely optional with real `!= null && IsLoaded` gates before
    every call. No standalone-CUI leak beyond what's already covered — `Unload()` already destroys
    `UiRoot` for every active player.
  - Verified live: v1.8.0 loaded clean per `server.log` (ApartmentRoom bridge resolved its
    reflection targets correctly, UI kit warmed, zero errors), zero hook exceptions, zero failed
    plugins. 5x reload stress test: memory flat (~7.4-7.5mb), zero exceptions throughout. Room
    cache reporting `rooms=0` on this map is pre-existing — confirmed via `server.log` that the
    prior v1.7.2 boot showed the identical `rooms=0`, so not a regression from this pass.
  - Net new fixes this pass: 1 dead-class removal, 1 real per-player-leak gap (6 dictionaries via
    a missing `OnPlayerDisconnected`), 1 genuine performance fix (redundant full-server entity scan
    on every lease transition replaced with an existing cached lookup — the one real instance of
    the external report's flagged concern, found after verifying the other 9 candidate sites were
    already properly mitigated), ~12 exception-safety wraps.

This closes out the "work down the list" pass — MixCommerce, MixWorldTune, MixInstantBases, and
MixApartmentHome (plus MixSignboard/MixSkinsLight earlier the same session) have all now had a
full first-pass audit against every category in this checklist, deployed, and verified live.

- **MixSiloRaid v0.5.0 → v0.6.0** — standalone raid monument overlay (paste-engine capture/
  restore, hostile scientist NPCs, admin CUI panel, CyberKnight HQ loot furnishing). Not deployed
  at the start of this pass (present on disk with leftover config/data from an 2026-08-27 audit,
  but not loaded on the live Carbon server) — deployed for the first time this session as part of
  resolving its flagged "genuine dependency question." First full pass under the current 11-
  category checklist (this plugin predates the checklist's Cat 10 addition).
  - **The dependency question, confirmed and fixed**: [line 42](E:\Projects\mix-apps\_rogue-depot-packaging\live-source\MixSiloRaid.cs:42)
    declared a bare `[PluginReference] private Plugin OSAutoTurrets, MixRackKit;`. The live turret
    plugin's registered `[Info(...)]` name is `"OSAuto-Turrets"` (hyphenated) — not `"OSAutoTurrets"`
    (the C# class name the bare reference binds against). Same root-cause class as the
    MixSkinsLight/PlayerDLCAPI fix earlier this session, confirmed by direct precedent:
    MixRaidBases — a sibling plugin doing the identical OSAutoTurrets integration — already had to
    work around this exact mismatch with an explicit `[PluginReference("OSAuto-Turrets")]`.
    MixSiloRaid never got that fix, so its `OSAutoTurrets != null && OSAutoTurrets.IsLoaded` gate
    almost certainly always evaluated false, silently disabling the "last resort: OSAutoTurrets
    API" fallback in `ArmTurret` — not a crash, a quietly dead integration. Fixed to
    `[PluginReference("OSAuto-Turrets")]`. Both actual `OSAutoTurrets.Call(...)` sites were already
    correctly wrapped in try/catch (one inside `ArmTurret`'s single enclosing try, one with its own
    inline try/catch) — the reference binding was the only real defect, confirmed by reading every
    line of `ArmTurret` before concluding no further fix was needed there. `MixRackKit`'s `[Info]`
    name matches its class name exactly (no mismatch risk), and its two call sites were already
    correctly null/IsLoaded-checked and try/catch-wrapped — genuinely clean, not deployed on this
    server right now by choice (plugin already degrades gracefully without it, confirmed via code
    read rather than assumed).
  - Cat 8: `SaveConfig()` unwrapped — the recurring pattern, fixed. Everything else touching disk
    (`SaveState`, template capture/load, room-diff logging) was already wrapped in try/catch
    throughout — this plugin was already unusually defensive (dozens of try/catch blocks across
    the paste engine, turret-arming, and NPC-spawn code) from its 2026-08-27 audit pass.
  - **Cat 4 — a real, previously-unnoticed gap**: no `OnPlayerDisconnected` hook at all;
    `_playerTypeLock` (an admin's locked monument-type selection from `/silo use #`) was only ever
    cleared by an explicit `/silo use clear` or a full plugin `Unload()`. Same per-player-leak
    pattern as every other plugin this session, here bounded to admin-only usage (low severity,
    still real). Added `OnPlayerDisconnected` clearing it; `Unload()` now clears it too.
  - Cat 1 (stale strings): two hardcoded `"MixSiloRaid v0.5.0"` / `"MixSiloRaid v0.5"` boot/help
    strings — fixed to interpolate `{Version}`.
  - Cat 2/5/6/7/9/11: clean. The four always-subscribed combat hooks (`OnTurretTarget`,
    `CanBeTargeted`, `OnTrapTrigger`, `OnEntityTakeDamage`) each lead with a cheap O(1) HashSet
    lookup before any real work — necessary (the plugin must see every combat event to protect its
    own NPCs from friendly-fire), matches the established cheap-early-exit pattern, not a
    violation. `mixsiloraid.admin` genuinely registered and checked (`IsAdmin`) on every command.
    `UnloadUi` already destroy-before-add on every panel open, plus destroyed for all active
    players in `Unload()`. No unbounded entity-walk hot paths — all `Vis.Entities` scans are
    radius-bounded around a monument/player, not raw `serverEntities` sweeps, and only run from
    admin-triggered commands, not per-tick hooks.
  - Verified live: had to `c.load` rather than `c.reload` since it wasn't previously loaded this
    session — confirmed a clean-slate deploy first (existing `state.json` had zero tracked pastes,
    `templates/` empty, so nothing at risk). Loaded clean, v0.6.0, zero hook exceptions, zero
    failed plugins. 5x reload stress test: memory flat (~124-288kb), zero exceptions throughout.
    One transient "Access to the path... is denied" line appeared in `server.log` from Carbon's own
    file-watcher racing the `scp` write with the explicit `c.load` — self-resolved on Carbon's
    retry (the very next line loaded successfully); not a code defect, just a scp/hot-reload race,
    confirmed harmless by the clean state immediately after.
  - Note for awareness, not a code defect: this plugin's HQ-furnishing commands use hardcoded
    world coordinates (`-1412.5, -19.3, 1095.0`) for a specific "CyberKnight HQ" bunker room — a
    bespoke, single-server tool rather than a general-purpose product, matching its "standalone
    silo raid monument" scope. Whether that location and the monument types this plugin targets
    (`silo`/`missile`/`bunker`/`launch`/`cave`) actually exist on this server's specific map/seed
    was not verified as part of this pass — same category of caveat as MixApartmentHome's
    map-dependent `rooms=0` finding, worth an in-game `/silo scan` check before relying on it.
  - Net new fixes this pass: 1 high-value binding fix (a silently-dead cross-plugin fallback,
    resolving the session's original flagged question), 1 real per-player-leak gap (missing
    disconnect cleanup), 2 stale-version-string fixes, 1 exception-safety wrap (SaveConfig).

- **MixSprint v2.2.1 → v2.3.0** — live bug report, not a routine audit pass: the user disabled
  the plugin at some point (`"Enabled": false` in config), then found `/mixsprint` — the only
  entry point to the admin panel — refused to do anything but print "disabled," with no way back
  in-game.
  - **The real bug**: `CmdMixSprint`'s very first line unconditionally gated on `_config.Enabled`,
    before even checking for the no-args → `OpenPanel` case. Confirmed via full-file grep that
    `_config.Enabled` is never written anywhere at runtime — no chat subcommand, no console
    command, no panel button. Once set to `false` (config edit, a future default change, anything),
    the plugin locks itself out completely — the only recovery path was SSH + manual config edit +
    reload. Immediate live unblock: flipped `Enabled` back to `true` directly in
    `carbon/configs/MixSprint.json` and reloaded so the user could keep testing while the real fix
    was written.
  - **Fix**: `CmdMixSprint`'s gate now only blocks non-admins (`!_config.Enabled && !IsAdmin(player)`)
    — admins pass through to `OpenPanel` and the command switch even while disabled. Added
    `/mixsprint enable` / `disable` subcommands and a `SetEnabled()` helper (writes config, saves,
    confirms, refreshes the panel). Added a new "Plugin — sprint system" row as the *first* row of
    the panel's admin section — deliberately above the existing drain/unlimited controls — with an
    Enabled/Disabled toggle button wired to a new `mixsprint.ui enabled` console action, mirroring
    the existing "Unlimited ON/OFF" toggle's exact pattern. This is now the actual recovery control
    for the lockout that used to exist here.
  - Cat 1 (stale strings), the second thing the user asked to fix: two hardcoded `"v2.2.0"` self-
    version strings (the `OnServerInitialized` boot line and the panel's version badge) — both were
    already one version behind live (v2.2.1) before this pass even started. Fixed both to
    interpolate `{Version}`.
  - Verified live: user was online testing at the time — deployed, reloaded once, confirmed zero
    hook exceptions and zero failed plugins via `c.plugins`, boot line now correctly prints
    `v2.3.0`. Skipped the usual repeated reload-stress-test since a live player was actively using
    the plugin; a single clean reload + health check is the appropriate bar for a live fix
    mid-session, not an unattended audit pass.

## Enabled-lockout sweep (found via MixSprint, fixed across the pack)

After the live MixSprint fix above, swept every plugin in `live-source/` for the same anti-pattern
— an `if (!_config.Enabled)` gate on a command handler with no in-game way to ever set `Enabled`
back to `true` — since finding one instance is a strong signal there are more of the same
copy-pasted scaffold. Grepped all `if (!_config.Enabled)` sites (10 hits across 6 files), read each
one's surrounding command/hook context, and confirmed via a second grep (`_config.Enabled\s*=`)
whether that plugin ever writes the flag anywhere. Two plugins were false positives — checked and
confirmed clean, not fixed:
- **MixHud** — `Enabled` only gates the internal `Tick()` refresh loop and the boot-time timer
  start in `OnServerInitialized`; the `/hud` command itself never checks it.
- **OSAutoTurrets** — the one hit is inside the `OnSwitchToggle` hook (a feature gate), not a
  command entry point.

Four plugins had the real bug, all fixed and redeployed together, same session as the live
MixSprint fix:

- **MixInstantBases v0.10.0 → v0.11.0** — `CmdMixBases`'s gate blocked everyone, including
  admins, before ever reaching `OpenHub` or any subcommand — identical shape to MixSprint's bug.
  Fixed: gate now only blocks non-admins; added `enable`/`disable` chat subcommands, a
  `SetEnabled()` helper, an `enabled` action in `API_HubUi`, and an Enabled/Disabled toggle button
  in the hub's admin section (`BuildHubContent`) routed through `mixcore.hub instantbases ui
  enabled` — confirmed the exact prefix requirement by reading MixCore's `AddUiActionCard`/
  `ForwardPackHub`, since this plugin renders inside MixCore's hub rather than its own window and
  a raw `AddButton` command needs the `mixcore.hub ` prefix manually (the action-card API adds it
  automatically; this hand-built button doesn't go through that API).
- **MixSiloRaid v0.6.0 → v0.7.0** — `CmdSilo` was already fully admin-gated first, but *still*
  fully blocked admins once disabled, with no toggle anywhere. Fixed: the disabled-check now
  allows `ui`/`menu`/`panel`/`enable`/`disable`/`help` through even when disabled (everything that
  actually captures/pastes/wipes/spawns still correctly refuses to run); added `enable`/`disable`
  subcommands, a `SetEnabled()` helper, and an ENABLED/DISABLED button woven into the existing
  admin panel's button grid (`DrawAdminUi`) — the grid's layout math is already fully dynamic
  (`rows = ceil(count/cols)`), so adding an 11th button needed no layout changes.
- **MixApartmentHome v1.8.0 → v1.9.0** — two affected entry points. `CmdApt`'s gate blocked
  admins from ever reaching the panel or any subcommand; fixed the same way as MixInstantBases
  (no-args/gui/menu/panel/enable/disable stay reachable for admins while disabled). `CmdRestoreKits`
  was also unconditionally gated — left it blocked while disabled (it's a destructive admin action,
  not the recovery path), just reordered so the redundant admin-check couldn't itself become a
  second point of confusion. Added a "Plugin" toggle card to `DrawControlsTab`'s existing admin
  card list (same `AptCmd(...)`/`HandleUiAction` pattern already used by the skeleton-key and
  master-key-sales toggles right next to it) and a `SetEnabled()` helper matching
  `ToggleBasementMasterKeySale`'s existing convention.
- **MixRaidBases v1.7.1 → v1.8.0** — the mildest case: `OpenPanel` was already reachable for
  admins regardless of `Enabled` (checked *before* `HandleCommand`, where the gate actually lived),
  so this was never a full lockout — but every subcommand was still blocked with no toggle
  anywhere, including via the admin-only `ConMixRaid` console/RCON command. Fixed: `HandleCommand`'s
  gate now lets `enable`/`disable` through; added those two cases plus a `SetEnabled()` helper
  (takes the existing `Action<string> reply` delegate so it works from both the chat command and
  the RCON-capable console command); added a "Plugin" toggle row as the first row of `DrawPanel`,
  above the difficulty-spawn list, reusing the same `PlateRowWithButtons`/`MixUiV2.Good`/`Bad`
  primitives already used elsewhere in the same panel.
- Verified live: a real player was online and had just finished testing the MixSprint fix — same
  standard as before, single clean reload per plugin (not a repeated stress test) plus a full
  `c.plugins` sweep afterward: 25 scripts + Carbon Core, zero hook exceptions, zero failed plugins
  across the entire server, not just the four touched plugins.

## MixMenuKit (separate project — `C:\Users\a\Desktop\MixMenuKit`, not part of the Rogue Depot pack)

New plugin, reviewed on request, then rewritten v1.2.0 → v2.0.0 to add multi-menu support (admin
can create any number of menus, each with its own trigger command, look, and set of plates) per
owner request. Deployed to BinaryLane for live testing alongside the pack.

- **`/menu` command collision** — v1.2.0 dropped the `/menu` alias; it collided with MixCore's own
  hub-opening command and would have silently broken the pack's main hub for every player (MixCore
  loads first alphabetically and would have lost the command to this plugin). `/mixmenu` is now the
  only player-facing command.
- **Builder input fields never actually saved (Name/Trigger/Label/Command)** — the real fault,
  root-caused only after live diagnostic logging, not guesswork: this Carbon build's
  `ConsoleSystem.Arg.Args`/`arg.FullString` are declared as `string[]`/`string` but are actually
  backed by `Facepunch.StringView` structs under the hood. Touching `arg.Args` directly — even a
  single clean-looking read, even just `(string[])arg.Args.Clone()` — could throw
  `InvalidCastException` outright or hand back a different/garbage array than a moment before;
  `arg.FullString` needs an explicit `.ToString()`, not the implicit conversion C# normally allows
  for a real string. `arg.GetInt`/`arg.GetString`, by contrast, never misbehaved anywhere in the
  file — they clearly go through Carbon's safe conversion path instead of touching the raw
  StringView buffer. Fixed (v2.0.8) by never touching `.Args` again: index-style values (menu id,
  plate index) use `arg.GetInt`/`arg.GetString` as normal; free text is parsed from
  `arg.FullString.ToString()`, skipping the leading space-separated index token(s) and keeping the
  remainder verbatim so embedded spaces survive. Five earlier fix attempts (CUI-redraw-timing,
  button-click-race deferral, a corrupted-text content filter, a stale-blur-echo check, removing
  per-keystroke `SaveConfig()`) were all chasing symptoms of this one cause and correctly diagnosed
  none of it — the content filter did stop the visible "Facepunch.StringView[]" garbage text from
  ever being saved, which is what made the underlying non-persistence bug visible on its own.
  Confirmed fixed by the owner testing live on BinaryLane.
- Worth remembering for any future Carbon console-command work in this pack: **never read
  `ConsoleSystem.Arg.Args` directly** — prefer `arg.GetInt`/`arg.GetString`/`arg.HasArgs`, and for
  free-text tails use `arg.FullString.ToString()` with manual token-skipping rather than joining
  from `.Args`.

### Full 11-category checklist pass (v2.4.0 → v2.4.1), after the multi-menu rewrite settled

Run against the file in full for the first time since the input-save saga — covers the style-
picker rewrite (v2.1.0–2.3.0: independent Button/Border pickers spanning all 3 families, NONE
border option, cursor-jump fix via a Root/Content UI split) and the new per-menu Public/Private
visibility flag (v2.4.0) together, since both landed in the same review pass.

- **Cat 1 (dead code)**: clean. Every `MenuDef`/`PluginConfig` field has a real read site outside
  clamp/migration code. The one bypass-shaped bool this plugin has — `MenuDef.Public`, which lets
  a menu skip the `mixmenukit.use` permission check entirely — defaults `false` (confirmed live:
  every migrated existing menu came through `"Public": false`), matching the exact pattern this
  category exists to catch (see MixSprint/OSAutoTurrets' `DevServerOpenPanelToAll = true` finding
  further up this log) — this one was built correctly from the start, not just checked after.
- **Cat 2 (found + fixed)**: `CName`/`CTrigger`/`CLabel`/`CCmd` apply typed edits live in memory
  per keystroke but deliberately never call `SaveConfig()` themselves (that per-keystroke disk
  write was an earlier, since-reverted theory during the input-save saga above) — only the
  explicit SAVE button flushed to disk. Gap: an admin who types an edit, then leaves via CLOSE or
  ← MENUS, or simply disconnects, without ever pressing SAVE, had that edit silently discarded on
  the next plugin reload or server restart — the in-memory `_config` instance is replaced by a
  fresh `LoadConfig()` read from whatever's actually on disk. Fixed (v2.4.1): `CClose` and
  `OnPlayerDisconnected` now call `SaveConfig()` when the player `CanEdit` (skipped for a plain
  player just closing/leaving a menu they were using — no wasted write); `CMenuList` always does,
  since it's already fully `CanEdit`-gated. Typing itself still never hits disk per keystroke —
  this only auto-flushes on definitive "done with this editor" actions.
- **Cat 3, 4, 5, 9, 10**: clean, mostly by not applying — no hot per-tick/per-shot hooks exist in
  this plugin at all (menu commands only fire from explicit player clicks), so no allocation or
  I/O-frequency concerns; `_edit` (the only per-player dictionary) is removed in
  `OnPlayerDisconnected` and cleared in `Unload()`; `mixmenukit.use`/`.edit` are both registered in
  `Init()`; no `[PluginReference]` exists to misrepresent; no skin/DLC feature exists to violate
  platform policy.
- **Cat 6 (CUI lifecycle)**: clean. Every `CursorEnabled = true` panel (built once per session in
  `BeginUi`, see the cursor-jump fix) is reachable from `CClose`'s `CuiHelper.DestroyUi(p, UiRoot)`
  — destroying the root destroys every child including the cursor shell — and `Unload()`
  unconditionally sweeps `UiRoot` for every currently active player, not just those known to have
  it open (harmless no-op for anyone who doesn't). No auto-drawn connect-time UI exists, so the
  "mid-snapshot CUI" risk this category also covers doesn't apply — every draw is triggered by an
  explicit player action.
- **Cat 7 (verified, not re-fixed)**: every one of the 20 console commands starts with a
  `CanEdit`/`CanUse` (or, for `CRun`, the new per-menu `Public`) check before touching `_config` —
  confirmed by re-reading every handler individually, not just grepping for the shape. This is the
  actual enforcement for "deny path denies": a non-privileged player calling `mixmenukit.save
  <id>`/`mixmenukit.name <id> x` etc. directly, bypassing the UI entirely, is blocked at the same
  gate the button click would have hit. All commands use the `mixmenukit.` prefix; the one chat
  command (`/mixmenu`) is distinctive.
- **Cat 8 (exception containment)**: the two real I/O boundaries — `StoreFile` (loading plate PNGs
  into FileStorage) and `SaveConfig` (`Config.WriteObject`) — are both try/caught. No cross-plugin
  `Call()`s exist (fully standalone). The CUI-construction code itself (`DrawMenu`/
  `DrawEditorDock`/`DrawMenuList`) isn't wrapped — assessed as low-risk (no external I/O or
  cross-plugin calls inside it, all indexing already bounds-checked by its own loop conditions)
  rather than a must-fix; noted for awareness, not actioned.
- **Cat 11 (lifecycle)**: `Init()` only registers permissions + local triggers, no cross-plugin
  assumptions; `OnServerInitialized` (not `Init`) is correctly where `LoadPlates` waits for
  `CommunityEntity.ServerInstance` to exist. No `OnNewSave` — a deliberate call, not an oversight:
  a menu's layout/plates/trigger is admin-authored UI configuration, not per-map game state, so it
  should carry across a wipe (same reasoning category, different verdict, from the "should this
  survive a wipe" question the rest of the pack still has open). The two additive schema changes
  this project made (v2.2.0's Button/Frame → ButtonSet+Button/FrameSet+Frame split, v2.4.0's added
  `Public` field) both went through `Config.ReadObject`'s natural tolerance plus an explicit
  `ClampMenu` backfill for the one genuinely ambiguous case (old `Set` → new `ButtonSet`/`FrameSet`)
  — confirmed correct against the live server's actual saved menus both times, no data lost.
  `LoadConfig()`'s corrupt-file path still doesn't back up the broken file before overwriting it —
  same known, not-yet-fixed gap already logged against the rest of the pack above, not unique to
  this plugin. **Fixed in v2.11.2 — see the entry near the end of this log.**
- **Live verification**: v2.4.1 deployed, 10x `c.reload MixMenuKit` in a row — 88 plates loaded
  identically every time (no duplicate-install growth), zero hook exceptions, zero failed plugins
  across the whole server each check, memory unchanged across the run. Config re-inspected after
  the stress test: both live menus intact byte-for-byte in structure, including the owner's own
  in-progress `"Public": true` test flag on "Standalone Mods" — confirms the reload cycle itself
  doesn't disturb saved state.

### Second pass (v2.4.1 → v2.5.0) — independent review, two real findings acted on

A second, independently-run review (owner-supplied, five numbered points) confirmed almost
everything above (unconditional resave-on-load is intentional for the migration backfill, no
`.bak` on corrupt read is the already-logged pack-wide gap, FileStorage's PNG re-`Store()` on
every reload is Facepunch's own CRC-dedup behaviour and not a leak — confirmed separately by the
10x-reload memory-flat result just above, the 12-plate/8-menu/48-char limits are exactly what the
code says). Two of the five pointed at something real that the first pass missed:

- **Residual save gap the v2.4.1 fix didn't close.** That fix flushed on CLOSE/← MENUS/disconnect
  — every path where the *player* deliberately leaves the editor. It missed the case where nothing
  the player does triggers it at all: a raw `c.reload MixMenuKit` (or a server restart) fired while
  an admin has an unsaved Name/Trigger/Label/Command edit sitting in memory and the UI still open.
  `Unload()` runs on that exact path — every route that discards the plugin instance and its
  `_config`, reload included — but didn't call `SaveConfig()`. Fixed (v2.5.0): added it there,
  which closes the gap completely rather than just narrowing it further. Also fixed the dock's own
  "Typing already saves as you go" comment, which was accurate for an earlier per-keystroke-save
  design that got reverted during the input-save saga above and was never updated to describe what
  actually ships now.
- **Real bug, not just a comment issue: renaming or deleting a Trigger never unregisters the old
  chat command.** `RegisterTriggers()` only ever called `cmd.AddChatCommand` — confirmed by reading
  it, not just trusting the finding. Renaming `/s` → `/outpost` left `/s` still bound to
  `CmdMenuTrigger`, which finds no matching menu for it and silently no-ops — harmless to the
  player who types it, but MixMenuKit keeps sitting on that word with no error or warning until a
  full reload, which would block any other plugin (or a later MixMenuKit menu) from ever claiming
  that exact command. Fixed (v2.5.0): `RegisterTriggers()` now tracks what it last actually
  registered (`_registeredTriggers`) and calls `cmd.RemoveChatCommand` on anything no longer
  claimed by any menu before re-registering the current set.
- Verified live: v2.5.0 compiled clean on first deploy (confirms `cmd.RemoveChatCommand` is the
  correct Carbon API shape), zero exceptions, config re-inspected after a further reload — both
  live menus and the owner's `"Public": true` test flag still intact.

### v2.6.0's PLATES preset: real data loss on a live menu, caused and then fixed same session

`mixmenukit.plates` (the preset-count cycler added in v2.6.0) wrapped past the top preset (36)
back down to the smallest (2) with no confirmation, and — critically — each individual click just
looked like normal cycling (the count climbing: 24, 36), so a few clicks in a row could wrap
around and silently discard every plate past the new, much smaller count before the admin
registered what was happening. That's exactly what happened on the owner's live "Standalone Mods"
menu during testing: ten of twelve custom plates (labels + commands) were lost, and the auto-save
added two rounds ago (v2.4.1/v2.5.0, specifically built to *stop* silent data loss) then persisted
that empty state to disk on exit — the fix built to close one data-loss gap became the vector for
a different one.

- **Root cause, `CPlates`**: forward-cycled through `PlatePresets` with wraparound and shrank
  destructively (trimmed from the end) whenever the wrap or a downward jump landed below the
  current count. Fixed (v2.6.1): grow-only. Jumps to the next preset above the current count;
  does nothing once at or past the top. Never shrinks, never wraps. `+BTN`/`-BTN` remain the only
  way to shrink a menu — one plate at a time, deliberately, with the change visible immediately —
  which was always the safer, already-accepted pattern for removal in this plugin.
- **Recovery**: the exact original 12 plates were recoverable because this session had captured
  the full live config verbatim multiple times earlier while investigating unrelated things (the
  border-asset check, the close-button-style fix) — reconstructed locally, verified the two
  surviving plates (SALVO, FREEBUILD) matched byte-for-byte before touching anything, then pushed
  the restored file to the server. This is bespoke, not a plugin feature — MixMenuKit still has no
  `.bak`/undo mechanism of its own (same known gap logged above); the recovery only worked because
  a byproduct of this session's own tool usage happened to preserve the data elsewhere.
- **Second, distinct bug surfaced during the recovery itself**: restoring the file on disk while
  the plugin was still loaded didn't stick — reloading immediately re-damaged it. Cause: `c.reload`
  calls `Unload()` on the *currently running* instance before the new one loads, and that
  instance's in-memory `_config` was still the damaged copy; the v2.5.0 always-save-on-`Unload()`
  fix dutifully flushed that stale memory back over the freshly-restored file. This is a real,
  worth-knowing property of that fix, not a bug in it: **the live config file cannot be manually
  edited/restored on disk while the plugin stays loaded — in-memory always wins over disk on the
  next unload, by design.** Documented directly in `Unload()`'s own comment now, so it's not
  rediscovered the same way next time. Actual recovery required a 3-reload sequence: (1) deploy a
  build with `Unload()`'s save temporarily commented out, reload (the *old*, damaged-in-memory,
  not-yet-neutered instance still clobbers the file once more this transition, but no further data
  was at stake by that point), (2) restore the file again now that the *running* code no longer
  force-saves on unload, reload (memory now correct), (3) revert `Unload()` back to its proper
  always-save behaviour, redeploy, reload once more (uses the still-neutered outgoing instance's
  Unload(), so it doesn't touch the now-correct file; the new instance loads it into memory
  correctly) — confirmed stable across two additional reloads after that.
- Verified live: v2.6.2 compiled clean, zero exceptions, all 12 original plates (SALVO through F1
  KILL) confirmed byte-for-byte correct in both the live config and survived three consecutive
  `c.reload` cycles after the fix, not just the one that mattered.

### v2.6.2 → v2.6.3: max plates capped to 24, and a second real bug found on the very next test

Owner feedback: 36 was too many in practice (capped to 24 — `MaxButtons`/`MaxColumns`/
`PlatePresets` all updated together, same reasoning as before, just a lower ceiling) — and while
testing the cap-at-36 build, ended up "stuck at 36" with the same live "Standalone Mods" menu
missing its two original plates (SALVO, FREEBUILD) again, this time down to 34 then 32.

- **Root cause, `CRemove` (`-BTN`), not `CPlates` this time**: `EditState.Pick` defaulted to `0`.
  `Picked()` correctly falls back to `0` for *display* purposes (showing plate 1's fields when an
  editor first opens is the right default) — but `CRemove` read that same resolved value with no
  way to tell "genuinely picked plate 1" apart from "never picked anything, still at the
  fallback." Every -BTN click with nothing explicitly selected in the grid silently removed
  plate 1 — exactly the two hand-placed plates at the front of the list, on the very click meant
  to just trim the pile of blank "NEW" plates off the end. "Stuck at 36" was PLATES correctly
  refusing to shrink (working as designed after the previous fix) while the *real* tool for
  bringing the count down was quietly eating the wrong end of the list instead.
- Fixed (v2.6.3): `EditState.Pick` now defaults to `-1` (a genuine "nothing picked yet" sentinel,
  distinct from a real pick of index 0). `CRemove` reads that raw value directly rather than
  `Picked()`'s already-resolved fallback: an explicit, in-range pick removes exactly that plate
  (unchanged, correct); anything else — never picked, or a stale pick left over from switching
  menus — removes the *last* plate instead, and Pick is deliberately left at `-1` afterward (not
  advanced) so repeated un-picked clicks keep trimming from whatever the new end is rather than
  converging toward the front as the list shrinks. `Picked()`'s own display fallback (still plate
  1 when an editor first opens) is untouched — this only changes what an *unset* pick means to
  the one destructive action, not what's shown by default.
- **Recovery**: same manual-restore situation as the previous incident, same
  `Unload()`-clobbers-a-disk-restore mechanism already documented in its own comment — so this
  time straight to the known 3-reload neuter/restore/revert sequence without re-diagnosing it.
  The two lost plates were still known verbatim from the previous incident's own log, so no new
  reconstruction risk. Final live state: 12 original real plates (SALVO → F1 KILL) + 12 blank
  placeholders = 24, matching the new cap exactly.
- Verified live: v2.6.3 compiled clean, zero exceptions, all 24 plates confirmed correct in the
  live config and stable across an additional reload past the one that mattered.
- **Pattern worth naming**: two live-testing sessions in a row surfaced a real bug in a brand-new
  control within minutes of it actually being used by someone other than the person who wrote and
  reasoned about it in the abstract. Both were "obvious in hindsight, invisible in isolation" —
  the code read as correct on its own terms (defaulting to plate 1 for display is right; the
  paired-cycle-then-grow-only design was internally consistent) and the actual failure mode only
  showed up from the *sequence* of real actions a real admin takes, not from reading the function
  in question by itself. Any new destructive/near-destructive builder control (add/remove/resize/
  visibility-style actions, not cosmetic ones) should get an explicit "what does this do with
  nothing selected / at either boundary" pass before shipping, not just a correctness read of its
  own logic — the two incidents here were each individually correct-looking single functions.

### Full checklist re-run (v2.9.0) + first ILSpy ground-truth pass on this plugin

Re-ran all 11 categories against the complete current file (1416 lines — covers everything since
the last full pass: the style-picker rewrite, Public/Private, plate presets, column caps, the
grid height-cap/centering fix, and the new mixmods/Rogue Depot asset additions). Clean — no new
findings beyond what's already fixed above; every mutating console command still saves in the
same action, every command is still permission-gated before touching `_config`, per-player state
is still cleaned up on disconnect, no dead fields, no hot-path hooks exist to violate categories
2-3. Live: 5x `c.reload` in a row, zero exceptions, zero failed plugins, config re-inspected after
— all three live menus intact (the owner's own in-progress "Rogue Depot" Public menu included).

Then went further than a source read for the first time on this plugin — pulled the actual
Carbon/Facepunch assemblies from the live server (read-only, `_ilspy-assemblies/binarylane-au-
carbon/`, `carbon/managed/Carbon*.dll` + `RustDedicated_Data/Managed/Assembly-CSharp.dll` +
`Facepunch.Console.dll`) and decompiled the specific types this plugin actually depends on, to
check inference against ground truth rather than just "it compiled and didn't throw":

- **`ConsoleSystem.Arg` — confirmed, and better understood than before.** Lives in
  `Facepunch.Console.dll` (not `Assembly-CSharp.dll`, and not a Carbon-authored type at all —
  this is stock Facepunch/Rust, not something Carbon introduced). `Args` and `FullString` are
  genuinely declared `StringView[]`/`StringView` in the real source — confirms the root-cause
  fix from the input-save saga was correct, and clarifies it wasn't ever a "Carbon quirk"
  specifically, just a base-game type most Oxide/Carbon plugin authors never look past the
  implicit-conversion assumption for. `GetString`/`GetInt` are confirmed genuinely safe by
  design, not just "happened to work in testing": `GetString` converts `Args[i]` to a real
  `string` on first access and caches it in a private `_cachedArgs` array, so repeated calls for
  the same index always return the identical, stable value — exactly the API surface this
  plugin's fixed code already exclusively uses. (The exact mechanism behind the original
  *live-observed* double-read corruption — clean on a first touch, garbage moments later in the
  same handler — wasn't fully traced to a specific line; `ConsoleSystem.RunWithResult` allocates
  a fresh `Arg` per command, not a pooled/reused one, so the corruption most likely lived in
  Carbon's own Oxide-compatibility dispatch layer, `Carbon.Compat.dll`/`Carbon.Hooks.Oxide.dll`,
  not in the native type itself — not chased further since the defensive fix already in place is
  correct regardless of the exact mechanism, and has been stable in production for many versions.)
- **`BasePlayer.SendConsoleCommand` — confirmed safe, backing this plugin's security claim.**
  Decompiled body is one line: `ConsoleNetwork.SendClientCommand(net.connection, command, obj)` —
  a server-to-that-player's-own-client RPC telling their client to run the command as themselves,
  not an elevated/server-side execution. Directly confirms what was told to the owner when the
  Public/Private feature shipped: a Public menu with an admin-only command on one of its plates
  (their own "Standalone Mods" test menu has GOD MODE/NOCLIP/F1 KILL) is safe for a non-admin to
  press — it round-trips through their own client and gets rejected by whichever plugin owns that
  command's own permission check, the same as if they'd typed it themselves. Was previously stated
  from general Rust/Oxide knowledge, not verified against source — now is.
- **`FileStorage.Store` — confirmed CRC-keyed idempotent upsert, not a leak.** `Store()` computes
  a CRC32 of the bytes, then does `INSERT OR REPLACE` into the on-disk table and
  `_cache.Remove(crc); _cache.Add(crc, ...)` keyed by that same CRC. Re-storing identical PNG
  bytes on every `c.reload` (this plugin's `LoadPlates`, called from `OnServerInitialized`)
  produces the identical key and overwrites the same slot — confirms the answer already given to
  the owner's memory-footprint question, now backed by the real implementation instead of general
  inference, and consistent with the flat memory readings across every reload-stress-test this
  project has run.
- **`Command.RemoveChatCommand` (Carbon's Oxide-compat `Command` library, in `Carbon.Common.dll`)
  — confirmed matches this plugin's usage exactly.** `RemoveChatCommand(string command, BaseHookable
  plugin = null)` clears any registered command matching both the name *and* the given plugin
  reference — `RegisterTriggers()` passes `this`, so it can only ever remove commands this plugin
  itself registered, never another plugin's identically-named one. Verified, not just assumed to
  compile correctly because the live `c.reload` succeeded.
- **New finding, genuinely not known before this pass: `CuiHelper`'s own `ActivePanels` tracking
  only sees this plugin's `UiContent` panel after the first refresh, not `UiRoot`.** Reading
  `CuiHelper.cs` (also in `Carbon.Common.dll`) directly: `AddUi(player, List<CuiElement>)` records
  only `elements[0].Name` into its internal per-player `ActivePanels` set. `BeginUi`'s fresh-open
  branch adds the `UiRoot` shell first (correct — matches what `CClose`/`Unload()` actually
  destroy), but the refresh branch (used by nearly every click after the first) adds `UiContent`
  first instead, since `UiRoot` is deliberately left untouched to protect the cursor-jump fix.
  Practical effect: after any refresh, Carbon's own bookkeeping thinks `UiContent` is "the" active
  panel for that player, not `UiRoot`. This only matters if something calls the *other* public API
  on `CuiHelper`, `DestroyActivePanelList(player)`, which bulk-destroys every panel it currently
  has tracked for that player — if that fired while a refreshed MixMenuKit session were open, it
  would clear the content but leave `UiRoot`'s cursor-lock shell orphaned and stuck on screen.
  Checked how exposed this actually is: the only call site anywhere in Carbon is one admin utility
  console command in `CorePlugin.cs` ("Cleared {n} CUI panels") — not called automatically on
  disconnect, death, respawn, or any other lifecycle event. So this is a real, understood,
  low-probability edge case (an admin would have to deliberately run that specific cleanup command
  against a player who has MixMenuKit open past its first click) rather than something that can
  happen in normal play. Not fixed — the tracking-accuracy cost is the direct trade for the
  cursor-jump fix, which was a confirmed, guaranteed-every-click bug affecting everyone; this is
  neither, and the alternative (recreating `UiRoot` every refresh to keep tracking accurate)
  would bring the cursor bug straight back. Documented here so it's a known trade-off, not a
  surprise, if it's ever worth hardening later (e.g. periodically re-touching `UiRoot`'s tracking
  entry without actually recreating the panel, if `CuiHelper`'s API ever exposes a way to do it).
- Assemblies pulled for this pass, kept read-only in
  `E:\Projects\mix-apps\_ilspy-assemblies\binarylane-au-carbon\` for reuse: `Carbon.dll`,
  `Carbon.Common.dll`, `Assembly-CSharp.dll`, `Facepunch.Console.dll`, all from the live Carbon
  server's actual `carbon/managed/` and `RustDedicated_Data/Managed/` folders — a straight
  read-only `scp`, no server writes. `ConsoleSystem.Arg` specifically lives in
  `Facepunch.Console.dll`, not `Assembly-CSharp.dll` — worth remembering directly next time, since
  it isn't the obvious first place to look.

### v2.9.0 → v2.10.0 — proactive "what would an external reviewer catch" pass, two real fixes

Owner asked directly what a strict outside reviewer might still flag, with time set aside to find
out rather than wait for it to happen live. Beyond re-confirming the 11-category checklist and the
ILSpy ground-truth pass above (both already clean), went looking specifically for things outside
that framework — code-quality/practice issues rather than correctness bugs. Two real findings:

- **`CRun` had no rate limit.** A Public menu makes `mixmenukit.run <id> <plate>` reachable by any
  player with zero permissions — including by sending the console command directly rather than
  clicking the button, which has no natural human click-rate limit. Nothing stopped a script from
  firing it as fast as the network allows, repeatedly running whatever command sits on that plate.
  Fixed: a per-player 0.3s cooldown (`_lastRun`, cleaned up on disconnect/unload like every other
  per-player dictionary here) sits in front of the actual command dispatch in `CRun`, independent
  of whatever cooldown (or lack of one) the target command itself has.
- **Every text field's 48-character limit was enforced client-side only.** `CharsLimit = 48` on
  the input component stops a player *typing* past it in the builder; nothing stopped a longer
  value arriving via a raw console command instead. Not a security boundary (`CName`/`CTrigger`/
  `CLabel`/`CCmd` are already `mixmenukit.edit`-gated, so only an already-trusted admin could ever
  reach them) but genuinely not sound practice regardless — a client-side-only limit is not a
  limit. Fixed: a shared `Cap()` helper truncates to the same 48 characters server-side, applied
  in all four handlers right where the text is finalized.
- Verified live: v2.10.0 compiled clean (confirms `UnityEngine.Time.realtimeSinceStartup` is
  reachable from plugin code, not just assumed), zero exceptions, all three of the owner's live
  menus (including the in-progress "Rogue Depot" one) confirmed unaffected.

### v2.11.1 → v2.11.2 — the last two open review gaps closed: `.bak` on corrupt config, Oxide test

Two items owner explicitly asked to close before submission: the pack-wide "no `.bak` backup on a
corrupt config" gap (fixed here for MixMenuKit specifically, not retrofitted across the other 33
pack plugins — those are already-shipped/tested artifacts with no live deploy/test pipeline active
this session, same scoping reasoning already used for the copyright-header pass), and the
previously-untested "should work unmodified on plain Oxide" claim in `LISTING.md`/
`SUBMISSION-NOTES.md`.

- **`.bak` fix.** Used the same ILSpy ground-truth approach as the v2.9.0 pass rather than guess:
  decompiled `Oxide.Core.Configuration.ConfigFile` (the base class of `Config`'s actual type,
  `DynamicConfigFile`) from the already-pulled `Carbon.Common.dll` and confirmed `public string
  Filename { get; private set; }` is the real on-disk path property, set once in the constructor.
  Added `BackupCorruptConfig()`, called from `LoadConfig()`'s existing catch block (i.e. only on a
  genuine unreadable/corrupt file, not on a well-formed empty/first-run config) right before the
  method's own trailing `SaveConfig()` — which was the actual destructive step, since it
  unconditionally overwrites whatever's on disk with the kept in-memory config or fresh defaults
  immediately after the catch block runs. Copies `Config.Filename` to
  `<Filename>.<yyyyMMdd-HHmmss>.bak` (timestamped, not a single fixed name, so repeated corruption
  events don't clobber an earlier backup); wrapped in its own try/catch so a failed backup attempt
  (e.g. disk full, permissions) can never itself become a crash — same exception-containment
  discipline as everywhere else in this file. Verified live: v2.11.2 compiled clean, `c.reload`
  zero exceptions, config re-inspected after — 5 live menus intact, byte-for-byte, confirming the
  new code path didn't fire (correctly — the on-disk config wasn't actually corrupt) and didn't
  disturb normal load/save either.
- **Oxide live test — full pass.** Stopped `rust-mixmods-au-carbon.service`, started
  `rust-mixmods-au.service` (a pre-existing dual Carbon/Oxide systemd setup, confirmed live —
  both unit files `Conflicts=` each other on the same port, so only one can run at a time),
  deployed `MixMenuKit.cs` plus its full 138-file plate/frame asset folder to the Oxide-side
  `oxide/plugins/`/`oxide/data/MixMenuKit/` paths (distinct from Carbon's `carbon/plugins/`/
  `carbon/data/MixMenuKit/`). Loaded clean on first try — compiled with 0 errors, 120 plates
  loaded, 6x `o.reload` in a row with zero MixMenuKit-related exceptions (the only log errors are
  pre-existing generic Linux-dedicated-server noise — a companion-server Steamworks check, a
  boot-time `FreeConsole` call — present on Carbon too, unrelated to this plugin). Confirms the
  "should work unmodified on Oxide" claim is real, not just asserted; `LISTING.md`/
  `SUBMISSION-NOTES.md` to be updated to reflect verified rather than untested. One real
  process mistake made and owned during this pass: ran `o.unload MixMenuKit` to make room for
  copying the live 5-menu production config onto the Oxide side, without first checking whether
  the Oxide instance had unsaved live-edited state in memory — it did (the owner had been
  test-building an "OUTPOST" menu with 12 plates in the builder while this was happening); the
  unload's own `SaveConfig()` flushed that edit to disk a moment before it was overwritten by the
  production-config copy, and it wasn't captured first. Not recoverable beyond what the
  screenshot showed; owner confirmed not needed. Lesson for next time: check `o.reload`'s own
  compiled-diff or ask before any unload on an instance someone might be actively driving live,
  not just instances known to be idle test state.
- **Screenshot-prompted false alarm, resolved without a code change**: owner reported button/
  border images "not showing" from an in-game screenshot — investigated by pulling the actual
  source PNGs for the exact style combo shown (bone button + stencil border) and comparing
  side-by-side: both matched the screenshot exactly (bone = pale ivory riveted plate, stencil =
  thin dashed olive frame). These are just visually quiet styles compared to the punchier promo
  combos (copper+rivet), not a rendering failure. No fix needed — confirmed by direct asset
  comparison rather than guessing, which is what this log entry is for.

### v2.11.2 → v2.12.1 — per-menu click/close UI sound effects, owner-requested feature

Owner asked for the same in-game sound feedback MixCore already has, on plate presses and menu
close, admin-pickable from a curated list, without a major rewrite. Scoped and built the same
session, live-tested and corrected on the spot.

- **API, confirmed via ILSpy before writing any code**, not guessed: `EffectNetwork.Send(Effect,
  Connection target)` (in `Carbon.Common.dll`'s decompiled `EffectNetwork.cs`) sends to exactly
  one connection, bypassing the normal visibility-group broadcast — necessary since these menus
  are private per-player UI, not world objects, and a plate click must never be audible to anyone
  standing nearby. `Effect.server.Run(string, Vector3, Vector3, Connection, bool, List<Connection>,
  int, Type)`'s `targets` parameter is what reaches that path (confirmed from the decompiled
  `Effect.cs`): passing a one-player `List<Connection>` with `broadcast: false` routes through
  `EffectNetwork.Send`'s `effect.targets != null` branch, addressing that connection directly
  rather than falling through to the positional-visibility-group branch. `player.net.connection`
  itself was not independently ILSpy-confirmed (the `Networkable` class lives in an assembly not
  yet pulled into `_ilspy-assemblies/`) — used on general Rust/Oxide plugin-ecosystem knowledge
  instead, verified the fast way: it compiled clean on the very first live deploy.
- **Design**: `SoundOptions`, a curated `(Key, Label, Prefab)[]` — NONE/SELECT/CONFIRM/DENY/CASH/
  NOTICE — cycled per-menu from two new builder-dock buttons (CLICK, CLOSE SFX), single
  click-to-advance like SIZE/COLS/PLATES rather than the Button/Border `<`/`>` pair pattern (a
  short fixed list doesn't need bidirectional stepping). Per-menu, not global, matching how
  Button/Frame styles already work — different menus already carry different visual themes, so a
  shop menu and a kits menu may reasonably want different click "feel" too. Picking a new sound
  plays it immediately so the admin can preview without leaving the builder. `PlayUiSound` is its
  own try/caught helper (same exception-containment discipline as everywhere else in this file) —
  a bad sound must never break the menu it's decorating. `DrawMenu` now stamps
  `State(player).MenuId` on every open (not just the admin-editor path) so `CClose` — whose
  console command carries no menu id at all — can resolve which menu's `CloseSound` to play;
  `CRun` already had a direct `menu` reference for `ClickSound`, no new state needed there.
- **Dock layout**: no free space existed in the 5-row builder dock for a 6th row, so `DockTopY`
  grew 0.33 → 0.40 (a single named constant; `DrawMenu`'s `gridBot` already reads the same
  constant for its own headroom, so the plate grid adjusted automatically, no separate change
  needed) and all 6 rows' Y-anchors were evenly recomputed (0.14 row height + 0.02 gaps) rather
  than reflowed ad hoc, to avoid overlap risk.
- **Live-tested, two real path mistakes found and fixed on the spot (v2.12.0 → v2.12.1)**: DENY
  and CASH produced no sound in-game. Server log showed exactly why —
  `EffectNetwork.Send`'s own `"String ID is 0 - unknown effect <path>"` warning (silent no-op, no
  crash, exactly as designed) for both `lock.code.lock.deny.prefab` and
  `vending-machine-purchase.prefab` — both first-pass guesses, not ground-truthed. Corrected via
  a live web search against a maintained community Rust-prefab-path reference (not another guess):
  real paths are `lock.code.denied.prefab` and `vending-machine-purchase-human.prefab`. Redeployed
  as v2.12.1, reloaded clean, zero exceptions. SELECT/CONFIRM/NOTICE were correct on the first try.
  Worth remembering pack-wide: **any future `Effect.server.Run` prefab path should be treated as
  unverified until live-tested** — a wrong path is safe (no crash) but silent, so it won't surface
  on its own the way a compile error would.

### Full 11-category checklist re-run (v2.12.1), owner-requested before final submission

Re-ran all 11 categories against the complete current file (1591 lines), specifically covering
everything added this session (`.bak` backup, sound effects, dock reflow) on top of the
already-clean state from the v2.9.0/v2.10.0 passes above. Clean across the board:

- **Cat 1 (dead code)**: `ClickSound`/`CloseSound` have real read sites (dock labels, `PlayUiSound`
  call sites) and real write sites (`CClickSound`/`CCloseSound`) — not dead. `BackupCorruptConfig`
  has a real call site in `LoadConfig`'s catch block.
- **Cat 2/3 (hot-path/I/O)**: `PlayUiSound` from `CRun` is bounded by the existing
  `RunCooldownSeconds` check (checked *before* the sound plays, not after). `CClose`'s
  `PlayUiSound` (and its pre-existing `SaveConfig()`) have no rate limit, same as before this
  session — low severity since both only affect the calling player's own client/own save, not
  anyone else, and CLOSE has no natural spam incentive the way a Public menu's `CRun` does. Not
  fixed, matches the existing accepted-risk shape of that handler.
- **Cat 4/5 (state cleanup/permissions)**: no new per-player dictionaries, no new permissions.
  `CClickSound`/`CCloseSound` are `CanEdit`-gated like every other per-menu editing command.
- **Cat 6 (CUI lifecycle)**: the new dock row rides the same destroy/rebuild cycle as the rest of
  `MixMenuKit.Dock` — no new persistent panel. Confirmed live, not just by reading: the owner's own
  screenshot after the reflow shows all 6 rows rendering with no overlap.
- **Cat 7 (bounds/null-safety)**: `PlayUiSound` guards player/connection/key null; `SoundIndex`
  falls back to index 0 ("NONE" — silent, not a crash) exactly like `CButtonStyle`'s existing
  `idx < 0 ? 0` pattern. `DrawMenu`'s new `State(player).MenuId = menu.Id` sits after the
  existing null-player guard, no new NRE risk. One real, minor, non-bug quirk found: `CClose`
  resolves `CloseSound` from `State(p).MenuId`, which `DrawMenuList` never sets — closing from
  the menu-list screen (not an individual menu) plays whatever menu's sound was last viewed, or
  silently nothing if none was. Harmless (still just a sound, no functional effect, no exception)
  — noted here, not fixed, since the alternative (a dedicated "menu list close" sound) is more
  complexity than the actual impact justifies.
- **Cat 8 (exception containment)**: `PlayUiSound` and `BackupCorruptConfig` are each their own
  try/catch, same discipline as `StoreFile`/`SaveConfig`.
- **Cat 9/10 (cross-plugin/platform)**: unchanged — no `[PluginReference]`, no skin/DLC code.
- **Cat 11 (lifecycle)**: no new hooks; `DrawMenu`'s extra line is a plain assignment, not a hook.
- **Live-verified same session**: 5x `o.reload` in a row on the Oxide install at v2.12.1, zero
  MixMenuKit exceptions, 120 plates stable every time, config re-inspected after — all 5 live
  menus intact including the owner's own in-progress sound picks on "Standalone Mods" (NOTICE/
  NOTICE), confirming the reload cycle doesn't disturb per-menu sound settings either.
- **Then switched back to production Carbon and re-verified there too** (same session, before the
  v2.12.2 finding below): stopped
  `rust-mixmods-au.service`, started `rust-mixmods-au-carbon.service`, confirmed fresh boot loaded
  MixMenuKit v2.12.1 clean (120 plates), then 5x `c.reload` in a row — zero exceptions, `c.plugins`
  reported 0 failed plugins server-wide each check. Carbon's own production config (untouched
  since before this session's Oxide detour) still had all 5 real menus with correct plate counts
  (12/24/12/12/8), and `ClickSound`/`CloseSound` — fields that didn't exist when this config was
  last saved — backfilled correctly to `select`/`confirm` via `ClampMenu`, exactly as they did on
  the Oxide side. Confirms the same build is genuinely interchangeable between engines, not just
  independently working on each: same file, same config schema, same result on both.

### v2.12.2 — one real finding from a final adversarial "what would Ricky catch" pass

Owner asked for one last dedicated look across security/compliance/compatibility/best-practice,
beyond the 11-category checklist (which covers correctness/lifecycle patterns, not this kind of
edge case). Found one genuine, previously-missed issue; everything else re-confirmed clean.

- **Real finding: a menu's own `Trigger` could be set to the plugin's own reserved word,
  `"mixmenu"`, silently hijacking the base `/mixmenu` command for every player.**
  `RegisterTriggers()` calls `cmd.AddChatCommand(word, this, nameof(CmdMenuTrigger))` for every
  non-empty configured Trigger, with no exclusion for the plugin's own `[ChatCommand("mixmenu")]`-
  bound `CmdMixMenu`. `RegisterTriggers()` runs in `Init()` (after Oxide's own attribute-based
  binding) and again on every `SAVE`, so an admin naming any menu's Trigger `mixmenu` would
  re-point the base command at `CmdMenuTrigger` instead — silently breaking `/mixmenu edit` and
  `/mixmenu close`'s subcommand routing for every player on the server until someone noticed and
  renamed it. `CTrigger`'s existing collision checks only cover other *menus'* triggers (hard
  block) and a cross-plugin heads-up list, `KnownPackCommands` (soft warning, and didn't even
  include "mixmenu" itself) — neither caught this specific, entirely self-inflicted case. Fixed:
  `CTrigger` now hard-blocks `"mixmenu"` specifically (case-insensitive), same rejection pattern
  as the existing duplicate-trigger check, distinct from and stronger than the soft
  `KnownPackCommands` warning. Low real-world likelihood (an admin has to specifically choose
  the plugin's own name as a custom trigger, which is an unusual thing to type), but the failure
  mode if it happened would have been silent, server-wide, and exactly the kind of thing a
  careful outside reviewer tests for on purpose.
- **Re-confirmed clean, not just re-asserted**: every one of the 20 console commands re-checked
  individually for its permission gate (all correctly `CanEdit`-gated except `mixmenukit.close`
  and `mixmenukit.run`, which are gated per-menu instead — by design, both player-facing paths);
  `CPick`'s unbounded stored index never reaches an unchecked array access anywhere downstream
  (`CRemove`/`CLabel`/`CCmd` all independently bounds-check before use); no path-traversal
  surface (every file path segment in `StoreFile`/`LoadSet` comes from compile-time string
  arrays, never from config or player input); no external network calls, no `Process.Start`, no
  reflection, nothing that reads outside the plugin's own config/data folders; `RunPlayerCommand`
  only ever round-trips a command through the *pressing* player's own client (re-confirmed
  against the same decompiled `BasePlayer.SendConsoleCommand` ground truth as the v2.9.0 pass),
  so a Public menu can't grant elevated execution to anyone; `PlayUiSound` only ever targets the
  acting player's own connection, never broadcasts; `[Description]` still carries the
  "Fan-made. Not Facepunch." disclaimer.
- **Noted, not fixed — deliberately out of scope**: `CNewMenu` has no upper cap on total menu
  count (unlike `MaxButtons` for plates), so an admin could create an unbounded number of menus.
  `mixmenukit.edit`-gated already, so this is the same trust level as an admin editing the config
  file directly by hand — not a security boundary this plugin is expected to enforce, and not
  something a reviewer would reasonably flag as a defect rather than a hardening nice-to-have.
  Two admins editing the exact same menu simultaneously is last-write-wins with no lock — a
  known, industry-standard limitation of this entire genre of single-file-config admin tools, not
  unique to this plugin and not something reasonably fixable without a much bigger rewrite.
- Verified live: v2.12.2 deployed to the currently-running Carbon service, compiled clean, zero
  exceptions, config re-inspected after — all 5 production menus still intact.

### v2.15.1 — another adversarial "what would Ricky catch" pass, after the sidebar feature landed

Owner asked for the same dedicated look again, specifically because a substantial chunk of new
surface area had just landed this session (the "sidebar" menu size, a rewritten `BeginUi`, the
bind-info popup and its two new console commands, and the per-menu font picker) that hadn't been
through this process yet — the rest of the file had already had multiple passes (see v2.9.0/
v2.12.0/v2.12.1/v2.12.2 above) and wasn't re-audited line-by-line again this time; this pass
targeted the new code specifically. Found one genuine, real, previously-missed issue.

- **Real finding: `mixmenukit.showbindinfo`/`mixmenukit.hidebindinfo` had no permission check at
  all — any connected player could force-render any menu, including a private one, to their own
  screen.** Both commands went straight to `DeferredLiveMenu` → `DrawMenu(player, menu, false,
  false)`, which unconditionally renders the target menu's full title, frame, and every plate
  button — no `Public`/`CanUse` gate anywhere in that path, unlike every legitimate entry point
  (`CmdMenuTrigger`, `OpenDefaultMenu`) which explicitly checks `!menu.Public && !CanUse(player)`
  before ever calling `DrawMenu`. Any connected player, permission-free, could type
  `mixmenukit.showbindinfo <id>` by hand for *any* menu id on the server — including a private,
  non-Public admin menu — and have its complete contents (button labels, and via the bind-info
  popup, the menu's own `/trigger` word) forced onto their screen. Couldn't escalate to actually
  *running* a plate's command — `CRun` independently re-checks `Public`/`CanUse` per press, per
  its own existing comment on being "the actual security boundary" — so this was information
  disclosure, not privilege escalation, but a real, previously-missed bypass exactly matching the
  self-inflicted, silent-until-found class of bug the v2.12.2 pass caught on the `"mixmenu"`
  trigger collision. **Fixed**: both commands now require `State(p).Open && State(p).MenuId ==
  menu.Id` — i.e. the player must already have exactly this menu legitimately open — rather than
  re-deriving the `Public`/`CanUse` logic a second time. This is the more precise fix: the "i"
  icon these commands back only ever legitimately appears on an already-open, already-gated live
  menu in the first place, so requiring that state to be true covers both Public and private
  menus correctly with no duplicated permission logic to drift out of sync later.
- **Minor consistency fix, not a bug**: `ClampMenu` backfills empty `ClickSound`/`CloseSound` on
  old saves (belt-and-braces, since both also have field initializers) but hadn't been extended
  to do the same for the new `Font` field. `FontIndex` already falls back to index 0 ("regular")
  for anything unrecognized, so this was never a crash risk — just brought in line with the
  pattern every other per-menu key field already follows.
- **Re-checked, not just re-asserted, given how much of `BeginUi` changed this session**: the
  live-sidebar path's full-screen click-through fix (`Image = null` on `UiRoot`/`UiContent`,
  confirmed against the decompiled `CuiElementContainer.Add(CuiPanel)` — it only attaches an
  Image component `if (panel.Image != null)`) doesn't create any new server-side authority gap —
  it only ever changes what a specific player's own client can click through on their own screen,
  never anything that reaches another player or the server's own state.
- Verified live: v2.15.1 deployed to the currently-running Carbon service, 3x reload stress test,
  compiled clean, zero exceptions each time, config re-inspected after — all 6 production menus
  still intact.

## Full MixHub pack sweep (2026-09-02) — every live-deployed plugin, not just one at a time

Owner request: run the full checklist against the whole pack, not one plugin at a time — "the
whole thing... all the parts." Pulled all 27 `.cs` files actually running on AU
(`/home/rustserver/rust-carbon/carbon/plugins/`) directly via SSH rather than any of the several
older local mirrors (`submission-ready/`, `_carbon-conversion/carbon-source/`), which turned out to
be stale — `submission-ready/Salvo.cs` was still v2.9.3, predating this session's `OnUserDisconnected`
fix, confirming the live server itself is the only reliably current source of truth right now.

Ran systematic pack-wide sweeps (not spot checks) for every high-value pattern this checklist has
ever caught a real bug from, via targeted grep across all 27 files plus manual verification of every
hit: `OnUserDisconnected` usage, `entity.net` null-safety in `OnEntity*`/entity-hit hooks,
`Resources.FindObjectsOfType(All)`, chat/console command name collisions, every plugin's `Unload()`
CUI-destroy coverage, static collection fields (growth/LOH risk), `OnNewSave` presence, permission
gating on every `[ConsoleCommand]` handler (including ones that delegate to a shared dispatcher
rather than gate inline), and — new this pass — raw client-supplied entity IDs resolved via
`BaseNetworkable.serverEntities.Find(...)` with no ownership/proximity check before acting on the
result.

**9 real, fixed issues, across 4 plugins:**

1. **FreeBuild v2.3.0 → v2.3.1** — `OnUserDisconnected` (doesn't fire on this Carbon build, see the
   Salvo v2.9.6 entry above) was still the only disconnect hook, so `SetCreativeMode(p, false)`/
   `TouchPlayer`/`ClosePanel` never ran on a real disconnect. Same fix pattern as Salvo: both
   `OnUserDisconnected` and `OnPlayerDisconnected` now call a shared `CleanupOnDisconnect`. This is
   the one plugin from the original pack-wide `OnUserDisconnected` grep (done when Salvo was fixed)
   that was flagged but never actually checked at the time — closed now.
2. **MixGovern v0.9.5 → v0.9.6** — `OnHammerHit`'s chat/audit message read `entity.net.ID` with no
   null-check. Low real-world odds (a hammer-hit target is virtually always networked) but a real,
   unguarded NRE risk consistent with the established `entity.net` rule — guarded.
3–7. **MixApartmentHome v1.9.0 → v1.9.1, five `entity.net`/`resolved.net` accesses** — the room/wing
   strip functions (`StripEntitiesUnderTransform` ×2, `StripEntitiesAt`, `CatalogEntity` inside the
   asset-survey report) walk transform hierarchies and `Vis.Entities()` radius scans, both of which
   can genuinely return non-networked components — higher real risk than the MixGovern case since
   these are recursive scans, not a single raycast hit. A sixth, lower-stakes one in the `/apt info`
   diagnostic command was also guarded for consistency.
8. **MixApartmentHome, same version bump — `mixapt.fixwings` had no admin gate at all**, unlike
   every sibling console command in the same plugin (`mixapt.strip`/`.restorekits`/`.export`/
   `.import` all check `HasAdmin` — confirmed via their own inline comments that this exact gap was
   already caught and fixed on those siblings previously). `fixwings` was missed: any connected
   player could call it from F1 and trigger real `Kill()` + respawn of world building geometry
   server-wide — a genuine griefing/DoS surface, not just an inconsistency. Fixed with the same
   `HasAdmin` gate as its siblings. `mixapt.wings` (report-only, `Puts` to server console, never to
   the caller) was also gated for consistency, though its real exposure was already near-zero since
   it never sends anything back to an ungated caller.
9. **MixWorld v0.7.0 → v0.7.1 — `mixworld.furnace`/`mixrustqol.furnace` (`DoFurnaceSplit`) resolved
   a raw client-supplied numeric entity ID via `serverEntities.Find` and acted on it with no
   ownership or proximity check at all.** Found by comparing it against its own sibling in the same
   file, `mixworld.sort`/`DoQuicksort`, which resolves an ID the same way but correctly requires
   `player.inventory.loot.entitySource == box` (i.e., the player must actually be looting that exact
   entity server-side) before doing anything — `DoFurnaceSplit` never had the equivalent check. Real
   impact: any connected player could push their own held ore into *any* furnace on the map by ID,
   with no proximity or base-privilege check, bypassing the ownership model every other
   container/build interaction in Rust respects. Fixed with the same `entitySource` check as
   `DoQuicksort`. **Swept the whole pack for the same pattern afterward** (~70 total
   `serverEntities.Find` call sites across 7 plugins) — every other hit either resolves an ID from
   the plugin's own trusted internal tracking (raid event IDs, apartment room/IO tracking, blueprint
   capture IDs — never attacker-controlled) or already has an equivalent authorization check. This
   was an isolated case, not a pattern repeated elsewhere.

**Swept clean, no action needed** (real pack-wide checks, not skipped): `Resources.FindObjectsOfType`
(zero matches, all 27 files); command-name collisions (zero, all 27 files); every plugin's `Unload()`
already destroys its own CUI (several — MixCore, MixHud, MixRaidBases, MixSignboard, MixCommerce,
MixInstantBases — carry their own inline comments confirming this was already found and fixed in
earlier passes this project, not freshly checked here); static collection fields are all either
fixed-size lookup tables (icon aliases, embedded-asset file/chunk maps) or already self-cleaning
(`MixUiFix.LootEndGuard`, try/finally-wrapped).

**Known, pre-existing, deliberately not re-litigated this pass**: `OnNewSave` — 3/27 plugins have it
(MixEntityScale, MixGovern, MixSignboard, each a real per-plugin decision), the other 24 don't; this
is the same standing category logged earlier in this file (owner already confirmed: each plugin's
wipe-survival behavior needs a *deliberate* decision, not that every plugin needs `OnNewSave`) —
not a new finding, not something to unilaterally change without a product decision per plugin.
Corrupt-config backup-before-overwrite — only MixMenuKit has it (v2.11.2, logged above); the other
26 plugins share the same already-logged, already-deferred gap from earlier in this file. Neither
category was touched this pass.

**Deployed and verified**: all four fixed files pushed to AU, `chown`'d, reloaded individually, then
3x reload-stress-tested together — zero exceptions on any reload, `c.plugins` confirms all four at
their new versions. MixApartmentHome's live memory read 5.8mb → 19.4mb across the stress-test
window; noted rather than either dismissed or treated as a confirmed leak — one data point across
four quick reloads isn't enough to diagnose, and this plugin already carries substantial legitimate
caches (room/lease/wing tracking across a large complex). Worth a dedicated watch if it recurs, not
actioned as a bug from this single observation.

**Total: 9 confirmed, fixed, deployed, and verified issues across 4 plugins** (FreeBuild ×1,
MixGovern ×1, MixApartmentHome ×6, MixWorld ×1), found via a full, systematic sweep of all 27
live-deployed plugins — not a sample, not spot checks.

## The 16 "silently missing from Carbon" plugins — recovered, standalone-GUI'd, checklisted (2026-09-02)

Owner request: the 16 plugins flagged as live on the old pre-Carbon Oxide install but absent from
Carbon with zero acknowledgement anywhere (see the parity-gap note earlier this project) — pull
them, give them all a real MixServerPack-styled standalone GUI, checklist them, and load them on
the local dedicated server (`C:\RustServer`) alongside the existing pack. Owner had never looked at
any of them.

Pulled all 16 directly from `/home/rustserver/rust/oxide/plugins/` on BinaryLane (read-only —
the old Oxide install, untouched, separate from the live Carbon service). All 16 present, all
real, complete, working plugins — no stubs, no `TODO`s. **3 of the 16 are not standalone-GUI
candidates and were deliberately excluded, not silently skipped**: `MixCompanionFix` (a silent
background patch, zero UI/commands by design), `MixPackAssetsShim` (shared-asset infrastructure
other plugins depend on, not player-facing), `MixPlaytest` (2142 lines, "Grok playtest actor" —
the owner's own internal dev/testing tool, not something to submit).

**The other 13, each given a genuine MixServerPack-styled CUI panel** (the embedded `MixUiV2`
card-kit pattern every v2 plugin in this pack already carries — same colors, same
Card/Title/CloseBtn/Row/Btn primitives, one copy embedded per plugin, no shared dependency, kept
Oxide-single-file-compatible): `MixSky`, `MixTrophies`, `MixDayNight`, `MixAngler`, `MixExtract`,
`MixGrid`, `MixHorde`, `MixPlace`, `MixQuarry` had **zero CUI at all** before this pass (pure
chat-command plugins) — built from scratch. `BagTimer`, `MixAdminMove`, `MixToolBar`, `TwinBarrel`
already had real panels (`BagTimer` specifically already had a full separate, more sophisticated
shared kit — `BagTimerMixCui`, sourced from `rust-mods/shared/MixCui.Oxide.cs` — discovered while
checking it, not assumed) — those four got a checklist pass rather than new UI.

**Real, checklist-relevant issues found and fixed across the 13** (not just building UI —
every one got the same disconnect/registration/leak checks the rest of this file's audits use):

- **Per-player state never cleaned up on disconnect** (Cat 4, the pack's most common recurring bug
  class) — found and fixed in `MixAngler` (`_cd`/`_busy`), `MixExtract` (`_busy`/`_cd`, including a
  live channel timer that kept running for a disconnected player instead of being cancelled),
  `MixGrid` (`_pending`). `MixTrophies` had a related variant: barrel/node progress toward two
  trophy cards lived only in a bare in-memory dictionary — never saved, reset on every server
  restart, and never cleaned on disconnect. Moved both counters into the plugin's own persisted
  `PlayerRec` instead, which fixes the leak and the restart-reset in one move.
- **CUI never destroyed on a real disconnect, only on plugin `Unload()`** (Cat 6) — found and fixed
  in `BagTimer` and `TwinBarrel`, both of which already had real admin panels but no
  `OnPlayerDisconnected` at all.
- **Double command registration** — `MixToolBar`'s `mixtoolbar.ui` was registered both via
  `cmd.AddConsoleCommand` in `Init()` and via its own `[ConsoleCommand]` attribute, the same bug
  class found and fixed in `MixWorldTune`/`MixCommerce` earlier the same day. Third confirmed
  instance of this exact pattern — worth grepping for on any future plugin work
  (`grep -o '\[ConsoleCommand("[^"]*")\]'` vs `grep -o 'cmd\.AddConsoleCommand("[^"]*"'`, same name
  in both = bug). Also caught and fixed in my own first draft of `MixSky`'s new `mixsky.ui` before
  it ever shipped — the pattern is easy to reintroduce by habit when wiring a new panel.
- **`DevServerOpenPanelToAll`-style dev bypass flag** — `BagTimer` has one; checked its actual
  default (`false`, safe) rather than assume, since this exact flag defaulting to `true` was a
  real, previously-caught bug elsewhere in this pack (MixSprint/OSAutoTurrets, logged earlier in
  this file).
- **A compile error caught by testing, not just review**: my first draft of `MixSky`'s panel called
  the shared `MixUiV2.Card(...)` helper with a collapsed single min/max string instead of two
  separate arguments — failed to compile immediately on `c.load`, fixed before it ever reached the
  server's actual plugin list.

**Every one of the 13 deployed to the local dedicated server and verified individually** (compile
clean, `c.plugins` shows correct version, zero exceptions) **then stress-tested together** — final
state: 40 total scripts loaded, 0 unloaded, 0 failed, zero exceptions anywhere.

**Submission-ready copies produced** for all 13 (comment-stripped, license header added, matching
this pack's established no-comments-in-shipped-`.cs` convention) at
`_rogue-depot-packaging/unfinished-mods-submission-ready-20260902/` — spot-verified one
(`MixQuarry`) actually compiles identically to the working copy before trusting the strip script
on the rest, rather than assuming a mechanical comment-strip couldn't break anything.

**Known, separate, pre-existing gap noticed but out of scope for this pass**: the local dedicated
server's existing 27-plugin mirror is stale relative to AU (e.g. `MixApartmentHome` v1.8.0 there vs
v1.9.2 on AU, `MixCommerce` v0.8.0 vs v0.8.1, several others) — not touched here since the ask was
specifically about the 16 missing plugins, not re-syncing the dedicated server's existing mirror.
**Followed up and closed the same day — see the next entry.**

## Two real hub-button bugs found via actual functional testing (not just "did a panel open"), plus the full two-way sync (2026-09-02)

Owner ran real functional tests against the AU hub (via MixPlaytest, the owner's own playtest
actor — scrap wallet spend/restore, shop buy, kit give, sethome, apartment survey, world-tune
read all confirmed working end-to-end) and found two genuine breaks, plus one non-bug worth
recording as confirmed-correct rather than re-investigated later:

- **MixWorld backpack "Open" tile — real bug, fixed (v0.7.1 → v0.7.2).** The backpack storage
  entity spawned correctly (function ran, no error), but the native loot panel never visibly
  attached — not just while the hub was open, but still not 0.4s after closing it, ruling out a
  simple animation race. Root cause: `CmdBackpack` called `player.EndLooting()` (which only ends
  an active *loot* session) but never touched MixCore's own hub CUI overlay — a separate,
  `CursorEnabled=true` panel contesting cursor/input focus with the native loot panel RPC that
  followed. Fixed: destroy `MixCore.Hub`/`MixCore.Hub.Dim` by their known, stable root names
  (MixCore is only ever a soft `[PluginReference]` here — a direct `CuiHelper.DestroyUi` call by
  name needs no dependency beyond that) before opening the loot panel, so the hub is actually gone
  rather than layered under something that never yields it.
- **MixGovern "God" hub button — real bug, fixed (v0.9.6 → v0.9.7).** Owner granted
  `mixpack.admin.god` explicitly and it still didn't flip god — no confirmation chat message,
  nothing. Root cause, found by reading rather than guessing: `HandleAction`'s `self.god` case
  (reached via `API_HubAction`) already got a real fix in an earlier pass — its own comment says so
  — replacing a raw `permission.UserHasPermission`-only check with `HasToolPerm` (accepts either
  the specific grant *or* pack-admin status). But the sibling path, `HandleQuickCommand` (reached
  via `/mixgovern god` and the same `[ChatCommand]`-level admin gate), never got the same fix —
  its `god`/`vanish`/`noclip`/`nvg` cases all still used the raw, stricter permission-only check,
  silently `return`ing with no message for exactly the case the owner hit. Fixed all four cases to
  use `HasToolPerm`, matching the already-established, already-correct pattern next to it.
- **Kits tile vs. kit give — checked, not a bug.** The hub's Kits button only lists kits
  (`mixcore.hub run kits`); actually receiving one is `/kit spawn` — confirmed this is the button's
  actual, honest scope, not a broken promise. No action taken; recorded so it doesn't get
  re-flagged as a mystery later.

**Full two-way sync, both directions, backed up first**: full plugin-folder backups taken on both
AU (`/home/rustserver/backups/pre-sync-au-<timestamp>/`) and the dedicated server
(`C:\RustServer\backups\pre-sync-dedicated-<timestamp>\`) before touching either. Diffed every
plugin's version on both servers precisely (not by memory) — found AU ahead on 15 core plugins
(`FreeBuild`, `MixApartmentHome`, `MixCommerce`, `MixCore`, `MixGovern`, `MixImages`,
`MixInstantBases`, `MixMenuKit`, `MixRaidBases`, `MixSiloRaid`, `MixSprint`, `MixWorld`,
`MixWorldTune`, `RogueDepot`, `Salvo`) and the dedicated server ahead on the 13 new standalone
plugins from the entry above (AU didn't have them at all yet). Pushed AU's 15 (including both bug
fixes above) to the dedicated server; pushed the dedicated server's 13 to AU. Deliberately left
alone: `MixCarbonLeakGuard` and `MixPlaytest` (AU-only diagnostic/testing tools, not "our mods" to
mirror anywhere) and `MixLabActor` (dedicated-only, same category).

**Tooling fix found mid-sync**: `tools/rcon-lan.py` had the exact same single-frame-receive bug
already fixed in `rcon-mixmods-au.py` earlier the same day — a `c.plugins` reply came back as an
unrelated broadcast log line instead of the actual plugin table, under the same "many plugins
reloading close together" load that originally surfaced the bug in the AU tool. Applied the
identical fix (collect every Identifier-matching frame, short quiet period after the first match,
short grace period if none arrive at all) rather than re-diagnose from scratch.

**One thing investigated and cleared, not treated as a new bug — same pattern as earlier today**:
mid-sync, `MixApartmentHome` briefly showed 16 seconds of hook time, 168MB memory, and a nonzero
exception count with no logged stack trace, and `MixCore` (5749 lines, the largest plugin in the
pack) took long enough to compile that it briefly vanished from `c.plugins` entirely before
finishing. Both were caused by 15 simultaneous plugin reloads competing for the same compile
queue/GC on the dev machine — retested both in isolation immediately after: `MixApartmentHome`
came back at 220ms / 17.4mb / zero exceptions, `MixCore` finished compiling and loaded cleanly.
Not a regression in either plugin, just contention from the sync itself.

**Final state, both servers, verified with a second full stress-test pass after the sync**: AU —
41 scripts, 0 unloaded, 0 failed. Dedicated server — 40 scripts, 0 unloaded, 0 failed.
- 2026-09-04: **Part C of the AI Rust-Modding Bible cross-checked against this checklist for the
  first time.** Part C ("Grok Agent Ruleset," xAI/Grok with MiX apps, MIT license, `PART-C.md` in
  the public `AI-Rust-Modding-Bible` repo) was written independently — its own findings from
  roughly the same two-week window of live Rust modding work, not derived from reading this file
  or Part B. Checked item-by-item rather than assumed compatible: the overwhelming majority of its
  22 sections (§0–22) restate rules already in categories 1–11 here and in Part B — object-return
  hook cancel-on-non-null (Cat 2), explicit CUI destruction on unload/disconnect (Cat 6), no CUI in
  `OnPlayerConnected` (Cat 6), `Init`=perms/`OnServerInitialized`=cross-plugin refs tested across
  both hot-reload and restart (Cat 4/11), `SaveConfig()` in the same action that applies a toggle
  (Cat 2), dangerous bypasses default `false` (Cat 1), permissions registered as real strings and
  actually deny (Cat 5/7), no allocation/disk I/O in hot-path hooks (Cat 3), Facepunch DLC
  ownership-API-only compliance (Cat 10 — the exact rule `MixSkinOwnership.cs` was built to
  satisfy), and 10x-reload/deny-path/real-click verification (Cat 4/7). Treated as corroboration,
  not new information, for all of the above — independent convergence from a different model on
  the same rule set is worth noting but doesn't need a new checklist line for something already
  covered. **Two items were genuinely new** and promoted into the numbered categories rather than
  left in this log only: "no two loaded plugins independently do the same job off the same
  underlying state" (new bullet, Cat 9 — this is exactly the `MixDayNight`/`TimeOfDay` conflict
  from 2026-09-03, found and fixed before Part C was read, so this is a case of the checklist and
  Part C independently landing on the identical concrete bug class from opposite directions) and
  "don't stack a hub/admin overlay on top of an open native loot interface" (new bullet, Cat 6 —
  not yet confirmed as a live bug anywhere in this pack, added on principle same as Cat 12's
  struct-key item was). Deliberately **not** merging Part B and Part C into one document — kept as
  separate files in the public repo, since the differing vantage points (Claude's findings vs.
  Grok's) are themselves the value of running two independent audits rather than one that tries to
  be everything; this checklist stays the single practical, actively-applied source of truth that
  both feed into.
- 2026-09-04: **First genuine pack-wide checklist pass** (all 46 live AU plugins, not one plugin at
  a time) — automated cross-cutting greps for every mechanically-checkable item across the whole
  pack, plus a full manual pass on the plugins never before run through this checklist (the 9
  standalones added 2026-09-03/04, `MixSkinOwnership`). Real findings, all fixed same-day:
  1. **Cat 4 — `FreeBuild.cs` was relying solely on `OnUserDisconnected`**, which this Carbon build
     doesn't reliably fire (see the 2026-09-02 entry above). Creative-mode flag and panel state
     could leak across a real disconnect. Fixed: wired defensively to both hook names, same pattern
     as Salvo/BagTimer/MixCarbonLeakGuard. v2.3.0 → v2.3.1.
  2. **Cat 7 — `MixRackKit.cs`'s `OnRackedWeaponTake`/`OnRackedWeaponSwap`** accessed `rack.net.ID`
     with `rack` null-checked but not `rack.net` — the identical bug class already caught once in
     MixSignboard's `OnEntityKill` (2026-08-30 entry). Fixed both hook sites; 4 lower-priority
     admin/API-driven call sites in the same file share the pattern but are far less exposed (known
     good entities, not a universal server-wide hook) — left as optional hardening, not required.
     v1.0.2 → v1.0.3.
  3. **Cat 12 — the embedded-UI-kit-assets eager-load issue (logged 2026-09-01, "not yet fixed as
     of this writing") was re-confirmed still open, and found present in 4 plugins, not just
     Salvo**: FreeBuild, MixSignboard, and MixEntityScale all share the identical
     `MixPackAssetsEmbedded` pattern. Real fix applied to all 4: the `Chunks` static `Dictionary`
     field (built unconditionally at class-load, before `EnsureInstalled()`'s `File.Exists` check
     ever runs) was replaced with a `string[] AssetFiles` (filenames only) plus a
     `GetChunks(string fileName)` switch method — a file's base64 content is now only referenced,
     and only then does the runtime actually materialize those literals, when that specific file is
     confirmed missing from disk. Transform done via a purpose-built script
     (`fix_cat12_lazy_assets.py`, scratchpad) rather than hand-editing multi-megabyte base64 blobs,
     with a separate verification script decoding every asset from both the original and
     transformed files and confirming byte-for-byte identical output before anything was deployed —
     zero data-corruption risk despite the scale of the mechanical change. Confirmed live via
     `c.plugins` memory before/after the reload, not just "should work": Salvo 12.8mb → 3.2mb,
     FreeBuild 10.2mb → 1.4mb, MixSignboard 9.1mb → 3.9mb, MixEntityScale 6.0mb → 1.6mb — roughly
     28mb of pointless per-load memory freed across the pack. Salvo v2.9.6→2.9.7, FreeBuild
     v2.3.1→2.3.2, MixSignboard v1.8.0→1.8.1 (also picked up the rename below), MixEntityScale
     v1.4.0→1.4.1.
  4. **Naming collision, not a technical bug**: `MixSignboard` owned the chat command
     `/mixshowcase` (a long-standing public alias for its neon-sign feature) at the same time a
     genuinely separate plugin named `MixShowcase` (added 2026-09-03/04, console/admin-driven, no
     chat commands of its own) existed in the pack. No runtime collision, but real customer
     confusion — typing `/mixshowcase` after seeing "MixShowcase" on the mods page lands in the
     wrong plugin entirely. Renamed MixSignboard's command to `/signshow` (internal preset id
     `"mixshowcase"` left unchanged — unrelated namespace, just this feature's name); updated the 3
     places `MixModsConnectUI` advertised the old command in its welcome/connect-info panels (2
     of the 3 are config-backed default strings — the **live AU and dedicated config.json files**
     needed the same text fix directly, not just the C# defaults, since an existing config isn't
     regenerated from new defaults). MixSignboard v1.8.1, MixModsConnectUI v1.5.25 → v1.5.26.
  5. **`RescueSurface`'s two cooldown dictionaries had no cleanup at all** (correctly *not*
     disconnect-cleared, per the Cat 4 persistent-intent-state rule — but also no periodic sweep,
     so they only ever grew). Low severity (two floats per unique player ever seen — trivial cost),
     fixed anyway since it was cheap: added a 10-minute `timer.Every` sweep removing entries only
     for players confirmed absent from both `BasePlayer.FindByID`/`FindSleeping`, using the same
     `Facepunch.Pool.Get<List<T>>()`/`FreeUnmanaged` pattern already established in
     `MixCarbonLeakGuard`. v1.1.1 → v1.1.2.
  All 6 fixes deployed to AU and the dedicated server, reloaded individually, confirmed via
  `c.plugins`: 46/46 scripts loaded, 0 failed, 0 exceptions on any of the six.
- 2026-09-04: **Cat 12 superseded by a full architectural change — embedded assets removed
  entirely from all 4 plugins, not just made lazy.** Owner decided the lazy-decode fix from
  earlier the same day was the wrong end-state: the real goal is a single shared asset pool
  (already proven to work — MixMenuKit's own button/frame system, a different asset set) rather
  than each plugin privately embedding and self-installing its own copy. `MixPackAssetsShim`
  designated the conceptual owner of the shared pool at `oxide/data/MixPackAssets/uikit/`
  (Carbon actually resolves this to `carbon/data/MixPackAssets/uikit/` — confirmed live, not
  assumed). Real finding that shaped the implementation: **`MixUiV2`, used identically by all 4
  plugins, turned out not to be a shared cross-plugin reference at all — it's independently
  duplicated into 14 separate plugin files**, per an existing in-code comment: "Oxide listings
  must stay single-file/dependency-free." This is a deliberate, documented policy, not an
  oversight — so the rework does NOT have the 4 plugins call into `MixPackAssetsShim`'s code;
  each independently computes the same well-known shared-folder path (a plain `Path.Combine`, no
  runtime dependency), matching the established convention exactly. `MixPackAssetsEmbedded`
  (`Chunks`/`AssetFiles`/`GetChunks`/`EnsureInstalled`) removed completely from Salvo, FreeBuild,
  MixSignboard, MixEntityScale — replaced by a ~4-line `MixPackAssetsShared.SharedDir` constant.
  File sizes: Salvo 4.19MB → 96KB, FreeBuild 2.03MB → 78KB, MixSignboard 2.05MB → 104KB,
  MixEntityScale 2.15MB → 34KB. The 21 real PNG assets were extracted losslessly from the
  (already-decoded, already-verified) embedded data via a purpose-built script — no new art
  needed, no risk of introducing a mismatch between old and new visuals.
  **Placeholder/fallback behavior — checked, not assumed, and found to already be correctly built
  in months before this rework**: every button-drawing call (`PaintedButton`) already falls back
  to a plain `MixUiV2.Button` when its texture is missing, and — critically — the real `CuiButton`
  carrying the actual `Command` binding is always added regardless of whether the image loads, so
  a button never loses functionality, only its paint job. Same pattern for state-carrying elements
  (`PaintedPill` → `MixUiV2.Pill` with real ok/bad color; `MixEntityScale`'s scale-slider track/fill
  → flat-color `CuiPanel`). Only purely decorative, non-functional chrome (corner brackets, frame
  texture, divider) skips drawing entirely when missing — correct, since it carries no information
  and no click target. Verified identical across all 4 plugins individually, not assumed from one.
  **New addition this pass**: a startup diagnostic — `WarmUiKitAssets()` now compares each
  plugin's own expected filename list against what's actually on disk and, if anything's missing,
  prints one consolidated `PrintWarning` naming every missing file and pointing at install
  instructions, rather than degrading completely silently.
  **Both paths live-tested on AU, not just the happy path**: shared folder temporarily renamed
  aside, all 4 plugins reloaded — warning fired correctly naming the exact missing files, `c.plugins`
  confirmed 0 exceptions/0 failed for all 4 in the degraded state. Folder restored, all 4 reloaded
  again — `warmed N` confirmed, memory back to the same figures as the earlier lazy-load fix
  (Salvo 3.1mb, FreeBuild 1.4mb, MixSignboard 3.9mb, MixEntityScale 1.5mb — no regression from
  switching mechanisms). Dedicated server's local shared folder populated with the same 21 assets
  for consistency (server currently offline, files staged for its next start).
  Versions: Salvo 2.9.7→2.10.0, FreeBuild 2.3.2→2.4.0, MixSignboard 1.8.1→1.9.0, MixEntityScale
  1.4.1→1.5.0. **Not yet done**: repackaging these 4 for actual distribution (loose `assets/`
  folder alongside each `.cs` in the submission package) and writing the "skip if MixServerPack
  already installed" install-instruction wording — deferred, not part of this pass.
- 2026-09-04 (later): **Full core/standalone split executed, plus two real fixes found while
  building it.** Owner asked for a concrete distinction between "MixServerPack" (core) and
  "standalone mod" using only what's actually live on AU (no speculative future mods), delivered
  as two Desktop folders. Settled the split as: **14 core** (`MixCore, MixImages,
  MixPackAssetsShim, MixModsConnectUI, MixMenuKit, MixGovern, MixWorld, MixWorldTune, MixCommerce,
  MixHud, MixUiFix, TimeOfDay, SimpleFurnaces, MixAdminMove`) and **29 standalone** (everything
  else genuinely sellable, `MixSkinOwnership` bundled alongside `MixSkinsLight` as its dependency,
  `MixCompanionFix` kept as its own planned free listing per existing packaging notes) — 3 plugins
  excluded from both as AU-only test/demo infrastructure (`MixCarbonLeakGuard`, `MixPlaytest`,
  `RogueDepot`, the marketplace connector).
  1. **Stale third-party-adjacent comment found and fixed**: `TimeOfDay.cs` — already fully
     rewritten to original MiX authorship earlier this session — still had a comment naming the old
     joint credit it replaced. Removed; the code was already clean, only the comment wasn't. v3.0.1.
  2. **Real, live, user-facing bug found and fixed**: `MixWorldTune.cs` had 10 places (comments AND
     actual admin-panel/chat text) still saying *"controlled by MixDayNight"* after `MixDayNight`
     was removed and replaced by `TimeOfDay` earlier this session. The disable-logic itself
     (`return;` before the plugin's own day/night code, so it doesn't fight whichever plugin owns
     the clock) was already correct — only the name shown to admins was wrong. Fixed all 10. v0.6.2.
  Both deployed live to AU (and dedicated), reloaded, zero exceptions — not left as submission-copy-only
  fixes.
- 2026-09-04 (later still): **Two structural asks — unify the private per-plugin uikit folders
  into the one shared pool, and fix the `CorePackPlugins`/`ProtectedPlugins` mismatch flagged in
  the entry above.**
  - **Private-folder unification**: 6 core files (`MixCore`, `MixWorld`, `MixWorldTune`,
    `MixCommerce`, `MixGovern`, `MixHud`) each had their own private `oxide/data/<PluginName>/uikit/`
    folder — a third, separate asset-delivery pattern discovered while building the core/standalone
    split, unrelated to the shared-pool work done earlier the same day. Pulled the real files from
    all 6 (66 total), hash-compared every filename against the existing 21-file shared pool before
    merging — **zero conflicts**, every overlapping filename was byte-identical — so the merge to a
    54-file canonical pool carried zero risk of silently swapping one plugin's art under another's
    filename. Updated all 6 `.cs` files' `folder` path from their own private subfolder to the
    shared `MixPackAssets/uikit/` path, and added the same missing-asset warning pattern already
    proven on the earlier 4. `MixArena`/`MixArcade` — extensively wired into `MixCore` (hub tab,
    live-snapshot API calls, ticker HUD) but not currently live as their own plugin files — were
    deliberately left out of the merge scope; this is a real, substantial planned feature, not
    stale naming, and shouldn't be silently assumed either way.
  - **The `CorePackPlugins` mismatch — root cause was NOT what it looked like.** Fixing the
    hardcoded `canonicalCore` array (a local variable) looked sufficient, but the fix kept reverting
    on every reload — then on an explicit `c.unload`→edit-file→`c.load` sequence — then even on a
    genuine `systemctl restart` of the whole service, which should have ruled out every caching
    explanation. Root cause, found by diffing the corrupted result against the known-clean edit
    (see new Cat 11 item above): `CorePackPlugins`/`ProtectedPlugins`/`RetiredPlugins` are
    `List<string>` fields with non-empty C# field-initializer defaults and no
    `ObjectCreationHandling.Replace` — Newtonsoft was appending the correct saved JSON onto the
    stale hardcoded default on every single load, not replacing it. Fixed at the actual source (the
    field declarations), not just the symptom. Also found and fixed in the same pass: `RetiredPlugins`
    incorrectly listed the very-much-alive `MixModsConnectUI` as retired (harmless — the field is
    never actually read anywhere — but wrong and confusing), and a boot-log line unconditionally
    telling admins to "enable MixPack.cs" for permission registration, when `MixPack.cs` doesn't
    exist and `MixGovern` already handles `mixpack.*` registration correctly (confirmed in an
    earlier audit pass). Final corrected `CorePackPlugins` (14) matches the core/standalone split
    above exactly; `ProtectedPlugins` trimmed to the 5 entries genuinely outside that list
    (`MixCompanionFix, MixSkinOwnership, MixCarbonLeakGuard, MixPlaytest, RogueDepot`) rather than
    redundantly re-listing plugins already covered by core-pack membership.
  Verified live end-to-end: config confirmed holding correctly *after* a full service restart (the
  only way to be sure the fix, not a live process's cached state, was what got tested), MixCore
  reloaded clean, full pack health 46/46 loaded, 0 failed. Desktop submission folder, dedicated
  server, and its config all synced to match.
- 2026-09-04 (final pass): **Pack-wide re-check against a fully fresh pull, confirming the day's
  fixes actually held and catching what they missed.** No functional bugs found this pass — a real
  signal the earlier fixes were genuine, not papered over — but 4 more stale-reference comments
  turned up, same class as the `MixWorldTune`/`MixDayNight` find earlier today: `Salvo.cs`,
  `FreeBuild.cs`, `MixEntityScale.cs` each still said "self-installed (see MixPackAssetsEmbedded
  below)" in a comment above the class that was actually renamed to `MixPackAssetsShared` during
  the shared-pool rework — the class itself was already correct, only the comment describing it
  hadn't caught up. `MixSky.cs` had a doc comment crediting day/night-clock and skip-night-vote
  ownership to `MixDayNight`/`MixSkipNight`, neither of which exist — both live on `TimeOfDay` now.
  Fixed all 4 (comment-only, no behavior change, no version bump). Re-confirmed clean:
  `Resources.FindObjectsOfType` still zero matches pack-wide, `DevServerOpenPanelToAll` still
  defaults false everywhere, `FreeBuild`'s `OnPlayerDisconnected` pairing and `MixRackKit`'s
  `rack.net` null-checks both still in place, `CorePackPlugins`/`ProtectedPlugins`/`RetiredPlugins`
  still holding at 14/5/1 after this round of reloads too. **New finding, not yet acted on**: 4
  *standalone* mods (`MixInstantBases`, `MixApartmentHome`, `MixRaidBases`, `MixSprint`) have the
  same private-per-plugin `oxide/data/<PluginName>/uikit/` pattern the 6 core files had — same
  class of thing, just in standalone mods rather than core, and out of scope for today's "unify the
  6 core files" ask. Worth the same treatment eventually, flagged rather than assumed. Full pack
  health after this pass: 46/46 loaded, 0 failed.
- 2026-09-04 (new category — real, found via a live beta tester, not a static review): **A brand
  new player has zero access to anything by default — every single player-facing feature across
  the whole pack requires a permission that is registered but never actually granted to anyone.**
  Found via the owner's son (Joe) beta-testing as a genuine first-time player: no shop, no /outpost
  or /bandit teleport, no Salvo. Traced precisely: `MixGovern`'s `ManagedPerms` and every
  standalone's own `PermUse` constant are correctly `RegisterPermission`'d (Oxide/Carbon knows the
  permission exists) but nothing ever calls `GrantGroupPermission("default", ...)` — even Salvo's
  own boot log admits it: *"assign ranks in /salvo admin · or grant salvo.use ... manually"*.
  `MixMenuKit`'s own `CanUse` gate was checked and ruled out as the cause first — every real
  player-facing menu is `Public: true` in the live config, so the menu system itself was never the
  blocker, only the individual feature commands underneath it.
  **Fix**: added `permission.GrantGroupPermission("default", PermUse, this)` right after each
  plugin's own `RegisterPermission` call, in the plugin's `Init()` (runs every load — idempotent,
  self-healing, survives a fresh install or a permission wipe with zero admin action needed) — 20
  plugins fixed this way, plus a new `PlayerBaselinePerms` array + grant loop added to `MixGovern`
  for the shared `mixpack.*` namespace (`mixpack.use`, `mixpack.shop.use`, `mixpack.shop.transfer`,
  `mixpack.kits.use`, `mixpack.teleport.use`, `mixpack.report.use`).
  **Deliberately excluded, checked case-by-case rather than assumed from naming** — a `PermUse`
  name doesn't reliably mean "baseline player tier":
  - `osautoturrets.use` — read the actual code: this permission returns VIP1-equivalent turret
    status (`return "vip1";`), not baseline access. A naming-only judgment would have wrongly
    handed every new player a premium feature for free.
  - `mixmove.use` (`MixAdminMove`) — the plugin's own boot log ("`/move` drop · foundations ·
    look+LMB grab/place") and default authLevel≥1 gate make clear this is a trusted/staff building
    tool, not a player feature, despite having a non-admin permission tier.
  - `mixapartmenthome.move` — no config bypass (unlike its sibling `.use`, which has one), always
    admin-or-explicit-grant — a more restricted action within the apartment system, left alone.
  - Every `*.admin` permission, and every genuine VIP/earned tier (`salvo.vip1/2`,
    `mixpack.teleport.vip`, `mixpack.kits.builder/hazmat/spawn`, `mixpack.rules.bypass`,
    `mixpack.world.backpack.4/7`) — auto-granting these to everyone would defeat their purpose as a
    progression/monetization mechanic, not fix a bug.
  - `mixplaytest.use` — AU-only diagnostic tooling, not a real customer-facing feature.
  **Verification note, worth keeping**: RCON on this server does not echo any response for
  `oxide.grant`/`o.grant`-style commands — not even an "unknown command" error — so a live RCON
  command cannot be used to confirm a permission change here. The actual persisted proof is
  `carbon/data/oxide.groups.data` (binary/protobuf, not JSON — readable via a simple `[\x20-\x7e]{4,}`
  printable-string extraction). That file also does **not** save synchronously on every grant —
  it only updates on some autosave/trigger event; `server.save` reliably forces one. Checked once
  right after the batch of reloads and only 8 of 26 grants had persisted, which looked like a real
  problem until `server.save` + a re-check showed all 26 correctly present — don't conclude "grant
  failed" from an unforced read of this file being incomplete.
  Deployed to AU and dedicated, all 21 touched plugins reloaded individually, confirmed via
  `c.plugins`: 46/46 loaded, 0 failed. Final confirmation was reading the actual saved permission
  data on disk, not just the source code or a clean reload — the same discipline as the
  `CorePackPlugins` fix earlier today, and for the same reason: a clean-looking reload here still
  wasn't sufficient proof by itself.

## "Bought an assault rifle, never received it" — three real bugs, none of them in the shop (2026-09-04)

- **2026-09-04 — the report**: owner buys an AK from the shop (the 120-RP `rp_ak` listing), gets
  the green "Purchased" message, is charged, and the rifle is neither in the inventory (8 free
  slots) nor on the ground. First buy of a session works; the second doesn't. Six hypotheses were
  tested and killed with live evidence before the real cause surfaced — click not reaching the
  server (UI-debug proved it did), wrong listing (true, explained why *scrap* never moved, but not
  the missing item), full inventory, server lag (FPS 141–154), a cooldown in the buy path (there is
  none), an interfering `CanAcceptItem`/`OnItemAddedToContainer` hook (checked all 46 plugins).
  Reading Rust's own `BasePlayer.GiveItem` in the decompiled assembly showed a failed give *always*
  drops at the player's feet — so "gone from both" should be impossible, which meant the client
  view was wrong, not the server.
- **What actually broke it**: a server-side inventory dump (new `mixcommerce.inv <steamid>`,
  RCON-usable) showed `belt[0]: rifle.ak x2/5`. The second rifle had *merged into the first
  slot* — `rifle.ak`'s stack limit was 5, vanilla is 1. Every item on the server was at exactly
  5x vanilla (wood 5000, rifle ammo 640, torch 5, armour 5) while MixCore.json's saved
  `WorldTune.Stacks.DefaultMultiplier` said 1.0 and Carbon's own StackManager module was disabled.
- **Bug 1 (root cause, systemic — new Cat 9 item)**: MixCore's `API_SetWorldTune` and
  `API_GetWorldTune` were `public` methods with no `[HookMethod]`, which on Carbon means
  `MixCore.Call("API_SetWorldTune", ...)` returns null without a sound. So when an admin applied
  the 5x preset in the Tune panel today, the multiplier took effect in MixWorldTune's memory and
  on every `ItemDefinition`, the "save" to MixCore silently did nothing, and the config kept
  reporting 1.0x — a reload would have quietly undone the whole thing and hidden the evidence.
  Scanned the pack: **23 dead cross-plugin calls across 7 plugins**, all the same shape.
  Fixed by adding the attribute to every one (MixCore 0.9.30, MixCommerce 0.8.4, MixGovern 0.9.11,
  MixWorld 0.7.4, MixWorldTune 0.6.4, MixInstantBases 0.11.2, OSAuto-Turrets 1.7.1). Proven at
  runtime, not by reload: `ModuleVersions` in MixCore.json went from `{MixCore}` alone to listing
  the registering pack plugins within seconds of MixCore 0.9.30 loading.
- **Bug 2 (gameplay — new Cat 7 item)**: MixWorldTune's `ApplyStacks` multiplied *every* item,
  including everything with a vanilla stack of 1. Now skips those unconditionally (override or
  not, with a warning if an override targets one), and a repair pass splits already-merged
  non-stackables back into separate slots for online players — on deploy it logged
  "un-stacked 2 weapon/tool/armour item(s)" and the owner's belt went from `rifle.ak x2` to two
  `rifle.ak x1` slots. `SaveTuneToCore` now checks MixCore's answer and warns if the settings
  weren't accepted, instead of the old fire-and-forget.
- **Bug 3 (observability — new Cat 8 item)**: MixCommerce's `GiveItem` returned `true` once
  `ItemManager.Create` succeeded and never looked at where the item went; purchase results only
  ever went to `player.ChatMessage`, which doesn't reach `server.log`, so nothing about a
  disputed purchase was verifiable after the fact. Replaced on the buy path with
  `GiveItemVerified` (reports main/belt/wear slot, dropped-in-world, stacked-into-existing, or
  removed — and refunds only on a genuine loss) plus a `[Shop]` audit line per buy/sell with the
  balance delta and free-slot count. Stack-merge is explicitly treated as success by comparing
  held count before/after, so buying 100 wood into an existing wood stack doesn't get refunded.
- **Side note for the owner**: MixCore's playtime points accrue ~+1 RP/min while online, which
  is why "my balance went up after buying" was true and not a sign the purchase failed — don't
  diff balances across minutes without accounting for it. Also noticed and left alone for now:
  OSAuto-Turrets' boot line still prints a hard-coded "v1.6.9 (Oxide)" — stale string, same class
  as the Cat 1 stale-comment sweep, cosmetic.
- **Not touched**: `_rogue-depot-packaging/submission-ready/` (standing rule), and the older
  `live-source/` project copies — AU and the dedicated box are the source of truth for these
  seven files as they have been all session.

### Checklist pass over that round (2026-09-04, evening) — 2 minor findings, both fixed

Scope: the seven plugins above, reviewed against all 12 categories with the live server as
the oracle where one existed.
- **Cat 9, scripted re-scan of the deployed sources**: 101 cross-plugin `Call()` targets
  resolve, **0 broken** (was 23). The 60-odd `UNDEFINED` names are MixPlaytest probing optional
  third-party plugins (Clans/Friends/etc.) that aren't installed — expected, not findings.
- **Live oracle**: `c.plugins` on AU — 46 scripts, 0 failed, 0 hook exceptions on any of the
  seven. MixGovern shows one hook-lag tick; the log says that's `OnServerInitialized` at 1061ms
  on reload (catalog build) — pre-existing, not from this round. The shop's own audit line
  after the fix read `item → main[3] x1 · now holding 3` / `main[9] x1 · now holding 4` for two
  consecutive rifle buys with a full belt — the exact case that used to merge.
- **Cat 7 finding (fixed, MixWorldTune 0.6.5)**: the over-stack repair pass covered main + belt
  but skipped `containerWear`; a doubled-up armour piece in a clothing slot is the same breakage.
  Added wear — safe because `GiveItem()` routes the split copy to main/belt, never back into
  wear, so it can't loop. Also confirmed against the decompile that `Item.SplitItem(n)` returns
  null when `n >= amount` and decrements `amount` otherwise, so the `while (amount > 1)` loop
  always terminates; the split copy is a fresh full-condition item (condition isn't copied).
- **Cat 1 finding (fixed, OSAuto-Turrets 1.7.2)**: boot line hard-coded "v1.6.9 (Oxide)" while
  `[Info]` said 1.7.1 — now prints `{Version}`. Same stale-string class as the earlier sweep; the
  AU and dedicated copies of this plugin differ (dedicated carries a legacy-tier migration block
  the AU one doesn't), so each got its own two-line patch rather than one overwriting the other.
- **Cat 11**: MixWorldTune's `ApplyStacks` → repair runs from `OnServerInitialized` and
  `OnPluginLoaded(MixCore)`; at boot `activePlayerList` is empty so the repair is a no-op there,
  and on the mid-session reload it correctly found nothing left to split (the first deploy had
  already done it). The `SaveTuneToCore` "did not accept" warning did not fire after the MixCore
  attribute fix — that path is now silent for the right reason.
- **Dedicated box**: not running (last Carbon log 2026-09-03), so its compile of these files is
  unverified until it next starts; the AU copies of the identical sources compiled, and the
  dedicated-only OSAuto-Turrets variant differs by attribute lines + one string only.
- **Nothing else found**: no new statics (Cat 12), no per-player state added (Cat 4), no new
  CUI (Cat 6), `mixcommerce.inv` gates in-game callers on commerce admin and lets RCON through
  (Cat 5), every new `Call()`/console path already sits inside the existing try/catch shape
  (Cat 8), no skin/TOS surface (Cat 10).

### 5x preset re-applied and proven persistent; `mixtune.preset` added; checklist pass (2026-09-04, late)

- **Why a new command**: nothing in MixWorldTune was reachable from RCON — `/mixtune` and every
  `mixtune.ui`/`mixtune.action` route bail on `arg.Player() == null`. Added `mixtune.preset
  <1|2|5|10>` (RCON, or in-game with `HasAdmin`), calling the same `ApplyPreset` →
  `SaveTuneToCore` path as the panel button and then **reading the value back from MixCore**
  via `API_GetWorldTune` rather than echoing its own field — so the reply is proof, not a claim.
- **Persistence proven the right way**: applied 5x over RCON → MixCore.json written seconds
  later with Stacks/Cook/Craft/Gather/Loot 5.0, ActivePreset 5.0 → `c.reload MixWorldTune` →
  config still 5.0 and the plugin reported "active 5x · stacks 5x" from MixCore's saved state →
  owner confirmed the Tune panel shows **Global 5x** → owner bought a fifth AK with 5x live and
  the audit line read `item → main[10] x1 · now holding 5`, not `stacked`. Wood `/5000`, ammo
  `/640`, `rifle.ak /1`. Every link in that chain was previously either silently broken or
  unobservable.
- **Cat 7 finding (fixed, 0.6.7)**: `float.TryParse` accepts `"NaN"` and `"Infinity"`, and
  `Mathf.Clamp(NaN, 1, 10)` returns NaN — which would have gone into `_tune`, through
  `ApplyStacks` (every stack silently to 1) and into MixCore.json as a literal `NaN`. Guarded
  in the new command **and** in the pre-existing `/mixtune stacks <n>` chat path, which had the
  same hole. Verified live: `mixtune.preset NaN` and `mixtune.preset abc` both return usage and
  leave the config at 5.0.
- **Cat 8 finding (fixed)**: the read-back `JsonConvert` round-trip of MixCore's object was
  unguarded — a shape mismatch would have thrown out of a console handler. Wrapped; reply says
  "unreadable (…)" instead.
- **Cat 6 finding (fixed)**: an in-game caller's open Tune panel wasn't refreshed after the
  console path applied a preset (the panel's own action route does `RefreshTuneHub`). Added for
  `player != null`; RCON callers have no panel.
- **Cat 11**: registered via `[ConsoleCommand]` only, matching this file's deliberate
  attribute-only convention (see the 2026-09-02 note above about the removed double
  registration) — not added to `Init()`.
- **Live oracle**: `c.plugins` — 46 scripts, 0 failed, 0 hook exceptions; deployed to AU and
  mirrored to the dedicated box (still offline, compile unverified there).

### AU cold restart as the compile test, and the one thing it surfaced: MixImages 1.0.2 (2026-09-04, night)

- **Restart**: `systemctl restart rust-mixmods-au-carbon` (the unit is `Restart=on-failure`, so
  an RCON `quit` would have left it down — worth knowing before reaching for it), world saved
  first, zero players on. 376s to "Server startup complete"; **46/46 compiled from cold, 0
  failed**, every version from today present, 5x preset re-applied from MixCore.json on boot,
  B1's five rifles still `x1`, 218 FPS, `[Shop]` audit lines confirmed in the daily-archived
  Carbon log (server.log is truncated on every start — never rely on it for history).
- **Cat 11 finding (fixed, MixImages 1.0.2)**: 14 "CommunityEntity.ServerInstance never became
  ready, gave up storing" warnings for MixHud icons ~2.5 min into boot. Not new — 123–246 per
  day in the archives, on every boot since the Carbon move — and masked in practice because most
  callers warm their assets a second time in their own `OnServerInitialized`. Cause: v1.0.1
  handled "world not up yet" with a **5×1s timer retry**, while a real cold boot has minutes
  between plugin load and world-ready. A fixed retry budget against an event whose timing scales
  with world size is the wrong shape. Now: before `OnServerInitialized`, bytes are parked in a
  slot-keyed dictionary (same asset warmed twice = one copy) and flushed the moment the server
  reports ready; `Init()` also sets ready-state from `CommunityEntity.ServerInstance` directly so
  a mid-session reload never holds anything back waiting on hook ordering. Registry saves are
  coalesced to one write per burst (a 17-asset warm was 17 disk writes — Cat 3), with the
  pending save flushed in `Unload()` (Cat 11) and its timer destroyed (Cat 4/12).
- **Verified, mid-session reload first**: compiled clean, 0 failed; MixCore re-warmed and
  MixModsConnectUI re-queued through the new code with zero warnings; registry intact at 228
  slots including all 20 MixHud icons.
- **Verified, then the cold boot that actually matters** (second `systemctl restart` of the
  night, 21:35): MixImages loaded at 21:37:29, the world reported ready at 21:41:04 — a **3½
  minute** gap, seven times the old retry budget — and the flush fired: *"stored 41 image(s)
  that arrived before the world was ready."* **Zero** "gave up storing" lines (14 on the boot
  two hours earlier, 123–246/day in the archives). The whole boot log carries exactly one
  `[WARN]`, Carbon's own analytics notice. 46/46 loaded, 0 failed, 184 FPS. Two notes for
  whoever runs this next: a full AU boot plus poll now exceeds a 10-minute tool timeout — poll
  in a background task or in two calls; and the 41 figure is the true count of assets the pack
  hands over before world-ready, a useful baseline if it ever jumps.

### Checklist pass over MixImages 1.0.2 → 1.0.3 (2026-09-04, night) — 3 findings, all fixed

Reviewed the final file cold, against all 12 categories, after the cold-boot proof above.
- **Cat 8 (real, introduced by the fix itself)**: batching 41 stores into one `FlushDeferred`
  loop created a single point where one `FileStorage.Store` throw would abort the loop and
  silently drop every image after it — and `_deferred` had already been cleared, so nothing
  would retry them. v1.0.1's one-image-per-timer-callback shape never had that exposure. Now
  each store in the flush is individually contained, failures are counted and named, and the
  summary line says "stored 40 of 41 — 1 failed" instead of lying with a clean count. The
  lesson generalises: **when you batch previously-independent operations, you inherit a new
  failure mode — one bad item poisoning the batch — that the unbatched code never had.**
- **Cat 1 (stale comment, same class as the earlier sweep)**: `FlushDeferred`'s snapshot
  comment claimed `StoreBytes` "may re-defer" during the flush. It can't — `_serverReady` is
  already true at that point, so a not-yet-ready store goes to the timer-retry path, not back
  into `_deferred`. The snapshot is still correct, the *reason given* for it was wrong. Fixed
  the comment rather than leave a future reader trusting a false invariant.
- **Cat 11 (observability)**: unloading mid-boot with images still parked dropped them
  silently. Recoverable (callers re-warm on their own hooks) but now logged with the count.
- **Live oracle**: the cold boot's single hook-lag warning was MixApartmentHome's
  `OnServerInitialized` (337ms), not MixImages — the 41-image flush stayed under Carbon's
  100ms threshold, so no need to spread it across frames. 1.0.3 reloaded on AU: compiled
  clean, 0 failed, **0** warnings since, MixCore re-warmed and ConnectUI re-queued through it.
  Mirrored to the dedicated box. The three changes are defensive/observability only, so the
  cold-boot proof recorded for 1.0.2 stands for 1.0.3 without another restart.
- **Clean elsewhere**: no hot-path work (Cat 2), coalesced registry save unchanged (Cat 3), no
  per-player or static state (Cat 4/12), no perms/CUI/TOS surface (Cat 5/6/10), the six API
  methods stay non-public and therefore `Call()`-reachable on Carbon (Cat 9).

### Checklist pass over MixImages 1.0.3 → 1.0.4 (2026-09-04, night) — 1 finding, fixed; loop closed

- **Cat 8 (real, a consequence of the previous fix's shape)**: the try/catch added in 1.0.3
  wrapped only the *flush loop's* calls. `FileStorage.Store` is reached from three other places
  — a caller's own hook (`AddImageData` during a plugin's warm-up), the 1s timer retry, and the
  download coroutine — and on those paths a throw would propagate out as a hook exception
  **attributed to the calling plugin** (MixHud, Salvo, …) or to an anonymous timer callback.
  Whoever read the log would open the wrong file. Moved the containment *into* `StoreBytes`,
  which now returns `bool` (false only for oversize or a FileStorage throw; parked-for-later and
  retry-scheduled are both true), and the flush loop just counts. One guard, every path, and the
  failure is named as MixImages' own. Rule worth keeping: **contain at the sink, not at one of
  its callers** — when a shared method can throw and has several entry points, the try/catch
  belongs inside it, otherwise the exception is reported against whichever caller you forgot.
- **Live**: 1.0.4 reloaded on AU — compiled clean, 0 failed, 0 warnings since, MixCore
  re-warmed and ConnectUI re-queued. Mirrored to the dedicated box. Defensive-only change; the
  1.0.2 cold-boot proof still stands.
- **Stopping here**: three consecutive passes on one 200-line file went real bug → real
  consequence of that fix → consistency of *that* fix. The third pass found nothing beyond that
  one point, which is the signal the loop has converged — further passes on this file would be
  ceremony, not review.

## MixThirdPerson 1.0.1 → 1.0.2 — static checklist pass, not installed anywhere (2026-09-04, night)

Desktop `MixThirdPerson\` (667-line standalone, no `[PluginReference]`), reviewed cold from
source + README + LICENSE only. **Not deployed to AU or the dedicated box by instruction**, so
every "live oracle" item below is explicitly *unverified*; the original 1.0.1 is kept in the
session scratchpad for diffing.
- **What it does, and the part that needs eyes**: sets `PlayerFlags.ThirdPersonViewmode` per
  mount category with admin-chosen allow/deny, and — to get the client to accept a `camdist`
  convar — **pulses `PlayerFlags.IsAdmin` on and off around it, with a network update in
  between** (`PulseAdminForCamera`, default true). Checked the server decompile: the server never
  reads `ThirdPersonViewmode` (bare enum flag; the *client* decides whether to honour it), so
  whether a non-admin actually gets third person on the current client build is not decidable
  from source. **Must be tested with a genuinely non-admin account (Joe) before it ships**; if
  the flag alone works, `PulseAdminForCamera=false` is the safer default because the pulse
  broadcasts a momentary admin flag to every client in range. Left as the owner's call, not
  changed.
- **Interference check against the pack (Cat 9)**: MixGovern and MixCore also set `IsAdmin` —
  MixGovern's noclip keeps a player elevated for the whole noclip session and restores a
  captured `hadAdmin` afterwards. Traced both directions: this plugin's pulse is synchronous
  (set → send → set → send → `finally` restore in one call stack), so nothing can interleave;
  and when MixGovern has a player elevated, this plugin sees `IsAdmin` true and doesn't pulse.
  No command-name collisions: nothing in the 46 owns `/third`, `/mixthird`, `/view`,
  `/third.admin` or `mixthird.*`.
- **Cat 7 finding (fixed, real race)**: `wasAdmin` was `player.IsAdmin` alone. A genuine admin
  whose flag isn't set yet (connect window; or mid-restore by another plugin) would be pulsed,
  and the `finally` would then set the flag **false** — stripping a real admin. Now
  `IsAdmin || authLevel >= 2`, the same test MixGovern's `HasRustAdminCheat` uses.
- **Cat 5 finding (fixed, pack convention)**: registered `mixthirdperson.use` but never granted
  it — a fresh install works for nobody, the exact "new player has zero access" shape from
  earlier today; README even told the admin to grant it by hand. Now granted to `default` in
  `Init()` (idempotent), README updated.
- **Cat 7 finding (fixed, latent)**: `Rule(cat)` returned a *detached* `new CategoryRule()` for
  an unknown key; `SetRule` edits that and `SaveConfig()`s, so the edit would vanish. Unreachable
  today (LoadConfig backfills every category, NormalizeCategory only returns known ones) but one
  hand-edited config away from real. Now inserts into `_config.Rules`.
- **Cat 7 finding (fixed)**: `CamDistance` from config went straight to the client convar
  unclamped — 0 is a black screen, 500 a satellite view. Clamped 0.5–10 on load; negative
  cooldown floored at 0.
- **Cat 8 finding (fixed)**: `SaveData()` (called from `OnServerSave` and every toggle) and
  `SaveConfig()` were unguarded disk writes — a hiccup would surface as a hook exception against
  the save cycle or the player's command. Wrapped, same shape as MixCommerce.
- **Cat 1 finding (fixed)**: boot line hard-coded "1.0.1" — same stale-string class as
  OSAuto-Turrets earlier today; now `{Version}`.
- **Clean**: Cat 2 (no per-tick hooks; mount/dismount/respawn go through `NextTick` with
  re-validation; NPC mounts fall out on `!IsConnected`), Cat 4 (`_third`/`_lastToggle` cleared
  on disconnect, `Unload` restores first-person for everyone it changed), Cat 6 (no CUI), Cat
  10 (third person for players is permitted; the admin-flag pulse is the only judgement call,
  flagged above), Cat 11 (`Unload` restores + saves, `OnServerSave` saves, no `OnNewSave` by
  documented design), Cat 12 (only static is an immutable string array), Carbon hook-cache rule
  (every method private).
- **README nit, not changed**: the "delete your 1.0.0 config" note is over-cautious — an old
  `Policy`-shaped config deserialises with `AllowFirst/AllowThird` defaulting true, i.e. "both",
  which is exactly the 1.0.1 default. Harmless either way.
- **Compile finding (fixed, and the reason this plugin would have failed on arrival)**: the
  `mixthird.rule` console handler indexed `arg.Args[0]` / `arg.Args[1]` as `string`. On the
  current Rust build `ConsoleSystem.Arg.Args` is `Facepunch.StringView[]`, so that is three
  hard compile errors — which is exactly why every pack plugin goes through `arg.GetString(i)`
  or `.ToString()`. Reading the file did not catch it; an offline compile did. Same errors in
  the untouched 1.0.1, so it was already broken, not introduced here. Now `GetString(0/1)`.
- **How it was compiled without installing — new tool, keep it**:
  `E:\Projects\mix-apps\_ilspy-assemblies\binarylane-au-carbon\harness\compile.sh <plugin.cs>`
  — the .NET 8 SDK's csc against AU's *exact* `RustDedicated_Data/Managed` (pulled tonight,
  Assembly-CSharp dated 2026-09-04) plus Carbon's managed DLLs. Two things it needed that a
  naive reference set lacks, both discovered by running a known-good live plugin as a control:
  (1) a **publicized** Assembly-CSharp — Carbon publicizes in memory at boot, so live plugins
  legally read private fields (`PlayerBoat.Sails/Engines/Anchors` are private); `harness/
  publicizer/` is a 40-line Cecil tool that does the same to disk; (2) **`Carbon.Proxy.dll`**
  in the references — not a loader piece as its name suggests, it carries `RustProxies.*`,
  static shims for APIs Facepunch has removed (`BaseEntity.SetFlag` is gone in the 09-04 build;
  MixBoatHelm only compiles because of the proxy). Validated: MixBoatHelm and MixCommerce 0.8.4
  (both live on AU) compile clean, a removed-API probe compiles, and MixThirdPerson 1.0.1 fails
  on exactly its three bad lines. Refresh after each Rust/Carbon update. Not loaded: Carbon's
  CompilerPolyfills source generator (needs Roslyn 4.14+, SDK here is 4.11) — so a plugin that
  compiles on Carbon *only* through a generated polyfill would still show an error here; treat
  an unexplained one as "verify on Carbon", not as proof.
- **Cat 7 rule, new, general**: never index `ConsoleSystem.Arg.Args` as `string` — it's
  `StringView[]`; use `arg.GetString(i)` / `arg.GetString(i, fallback)`. Cheap to grep for:
  `Args\[\d+\]` outside a `.ToString()`.
- **Side finding, dedicated box**: its `Assembly-CSharp.dll` is dated 2026-08-30 vs AU's
  2026-09-04 — it has not taken the September forced update. The first attempt at this harness
  used its assemblies and the live control plugin *failed* on members that only exist in the
  newer build. When that server next starts it must update Rust before any of tonight's
  plugin mirrors are judged by what it does or doesn't compile.
- **Then uploaded to AU on the owner's say-so (22:17) for Joe to test as a non-admin.** Compiled
  and loaded clean: 47 scripts, 0 failed, config generated with the intended defaults, `use`
  granted to `default` (confirmed in `oxide.groups.data` after a forced `server.save`).
  **Deploy quirk worth remembering**: Carbon's file watcher did *not* pick up the file — every
  earlier deploy tonight overwrote an existing rustserver-owned `.cs`; this was a *new* file
  created root-owned by scp, and two minutes later there was no compile attempt at all. Fix was
  `chown rustserver:rustserver` + RCON `c.load MixThirdPerson` — loaded in 210ms. New plugins
  onto this box: chown first, then `c.load`; don't wait on the watcher.
- **Live test, answered (2026-09-04 ~22:20–22:50, tester 1968Datsun1000)**: third person
  works for a **genuine non-admin** on the bike, and the owner reports every command behaved.
  Non-admin is proven, not assumed: no "has auth level" line at his join (B1's join prints
  `has auth level 2`), zero hits in `users.cfg`, and `oxide.users.data` shows him in `default`
  only with no individual grants. Plugin-side evidence: `data/MixThirdPerson.json` holds
  `{"bike": "third", "Locked": false}` for his SteamID — a real toggle path wrote it — and the
  Carbon log has **zero** warnings or errors in the 28 minutes since load, no hook
  exceptions in `c.plugins`. So the client honours `ThirdPersonViewmode` without admin.
## Dedicated box brought level with AU, then the "BoatHelm will break" question answered — 14 plugins off Carbon's obsolete shim (2026-09-04, late night)

- **Dedicated update**: Rust via the box's own `Update-RustServer.bat` command (SteamCMD
  258550 validate) → build **25083359**, identical to AU; Carbon 2.0.257 (Aug 6) →
  **2.0.258** from the CarbonCommunity `production_build` release (owner approved the 21.4 MB
  download; `carbon/managed` + `native` + doorstop backed up first; zip inspected before
  extraction — it carries only empty placeholder dirs for plugins/configs/data). Cold boot:
  **43/43 compiled, 0 failed, 0 hook-lag**, one pre-existing data warning (MixRackKit wants
  `kits.json` generated). World gen from scratch took ~20 min because the update invalidated
  the cached map — expected, not a plugin symptom.
- **The real finding — Cat 9, new item**: Facepunch removed `BaseEntity.SetFlag(f, b,
  recursive, networkupdate)` in the 2026-09 build. Every call still compiled on AU only through
  `Carbon.Proxy.dll`'s `RustProxies.SetFlag`, which Carbon has already marked obsolete
  ("Use BaseEntity.StartSetFlags or BaseEntity.SetFlagLocal instead") — i.e. it can go in any
  Carbon release, and it never existed on Oxide. The owner asked about BoatHelm; a pack-wide
  compile **without the shim** (`NO_PROXY=1 compile.sh`) found **14 of 47 plugins, 78 call
  sites**, all the same one API: FreeBuild, MixApartmentHome, MixBoatHelm, MixHeadlamp,
  MixInstantBases, MixQuarry, MixRackKit, MixRaidBases, MixShowcase, MixSignboard, MixSiloRaid,
  MixSkinsLight, MixWorld, OSAutoTurrets.
- **Fix shape, chosen for zero call-site churn**: `harness/add_setflag_compat.py` drops a
  small `internal static class <Plugin>FlagCompat` with `SetFlag(this BaseEntity, Flags, bool,
  bool recursive = false, bool networkupdate = true)` into each plugin's own
  `namespace Oxide.Plugins`. C# resolves extension methods innermost-namespace-first, so it
  shadows the global-namespace shim with no ambiguity; the body does exactly what the shim's
  IL does (`StartSetFlags(networkupdate ? SendNetworkUpdate : Local)` → `scope.Set(f, b,
  recursive)` → dispose), verified by decoding both `RustProxies.SetFlag` and
  `FlagsUpdateScope` with Cecil. Handles file-scoped and block namespaces. Proven on scratch
  copies first, then all 14 (+ the dedicated-only OSAutoTurrets variant) compiled **with and
  without** the shim. Deployed to AU (all 14 reloaded, 47 scripts, 0 failed, only reload
  hook-lag warnings) and mirrored to the dedicated box.
- **Harness gained two things while doing this**: `NO_PROXY=1` mode (the "will it still build
  when Carbon drops a shim" check — worth running on every new plugin), and
  `polyfills/IsExternalInit.cs`, compiled alongside every plugin because Carbon's
  CompilerPolyfills generator (which the harness can't load) emits it for `init` accessors —
  MixSkinsLight uses them and was a false failure until then. Polyfills are harness-only, never
  shipped in a plugin file, where they could collide with Carbon's generated copy.
- **Rule, general**: *a plugin that compiles only because the loader ships a shim for a removed
  API is already broken, on a timer you don't control.* Compile without the shims; if it fails,
  fix it now while the replacement API is fresh and the shim still tells you what it did.

### Checklist pass over the SetFlag round (2026-09-04, late night) — semantics proven, 3 stale strings, 1 false alarm

- **Cat 2, the one that mattered — is the helper hot-path-neutral?** Old vanilla `SetFlag`
  early-returned when the flag already had the requested value: no `OnFlagsChanged`, no
  network update. FreeBuild's lock sweeps and MixApartmentHome's power loops call it in bulk
  on entities that are usually already in the target state, so if the new
  `StartSetFlags`/`Set`/`Dispose` path sent an update regardless, the "identical behaviour"
  claim would be false and traffic would go up. Decoded both with Cecil (extended the
  inspector to match explicit-interface `Dispose` and show branches): `Set` returns early on
  `HasFlag(f) == b`; `Dispose` compares `oldFlags` to `owner.flags` and skips both
  `OnFlagsChanged` and `HandleFlagsUpdateMode` when equal. A no-op stays a no-op. Recursion
  is handled inside `Set` via child scopes. Claim stands — with IL, not by assertion.
- **Cat 1 finding (fixed, 3 plugins)**: hard-coded version literals in boot lines —
  MixHeadlamp printed 1.0.4 (Info 1.0.7), MixQuarry 0.1.0 (Info 0.2.2), MixRaidBases 1.7.0
  (Info 1.8.1). Same class as OSAuto-Turrets earlier; found by grepping the post-reload log
  for `[Plugin] Plugin vX` lines and diffing against `[Info]`. Now `{Version}`; bumped to
  1.0.8 / 0.2.3 / 1.8.2, compiled both modes, deployed AU + dedicated, boot lines verified.
  The other 11 either use `{Version}` already or print no version line.
- **Cat 9 false alarm, run down rather than dismissed**: MixRaidBases logged `OS
  turrets=MISSING` during the 13-plugin reload. It was logged 6 s *before* OSAuto-Turrets
  finished its own reload; the cold boot two hours earlier said `linked`, and the plugin's
  `ResolveOsTurrets()` re-resolves lazily on every use by both names (`plugins.Find(
  "OSAuto-Turrets") ?? plugins.Find("OSAutoTurrets")`). Confirmed on the 1.8.2 reload:
  `linked`. Comment added at the boot line so the next reader doesn't chase it. Side fact
  worth keeping: Carbon indexes plugins by their `[Info]` title, so a class named
  `OSAutoTurrets` with title `OSAuto-Turrets` needs `[PluginReference("OSAuto-Turrets")]`
  (MixSiloRaid does exactly that) — a bare `[PluginReference] Plugin OSAutoTurrets` would
  silently stay null.
## Menu rework — Tab sidebar as the hub, 12 menus → 11, every new plugin reachable (2026-09-05, morning)

- **Why**: nine plugins (Angler, Extract, Grid, Quarry, Sky, Trophies, Place, TwinBarrel,
  ThirdPerson) had no menu presence; two overlapping menu families had grown side by side; one
  24-plate junk menu was twelve `NEW → /kit` plates; six plates pointed at commands that don't
  exist (`/tpm outpost`, `/godmode`, `/rescueme`, `/mixraidbases`, `/getting older`,
  `/TP banditcamp`). Owner's rule: the Tab inventory sidebar is the hub — HOME and OUTPOST
  direct, everything else a submenu; big menus (≤12 plates) rather than many small ones.
- **Method, not taste**: catalogued every chat command in all 47 plugins from source, read
  each new plugin's handler to see what the *bare* command does (e.g. `/place` bare prints
  help, `/place ui` opens the panel; `/angler` opens a panel, `/cast` casts; skip-night vote is
  `/skipnight` from TimeOfDay's config, not `/tod`), pulled the real kit ids from MixGovern's
  live config (`daily`/`vip` exist; admin/raid/oillarge are admin-gated), and measured the
  sidebar: 2×5 fits, 12 is cramped. The config was generated by a script that asserts ≤12
  plates, unique triggers, and that every submenu plate resolves to a real trigger; the eight
  new triggers were checked against the full command catalogue for collisions.
- **Layout**: sidebar `/i` (10) → `/travel` `/ks` `/build` `/ride` `/gear` `/world` (≤12
  each, all with `← BACK → /i`), `/hub` as the non-Tab twin, admin folded from five menus into
  `/admin` (12) + `/atools` + `/fbtools`, all non-public and off the player sidebar (a
  non-public menu answers players with "no access" — clutter, not security).
- **Deploy order, and the real finding of the round — Cat 11**: dedicated first (cold start
  after the PC reboot: 43/43, 0 failed, config round-tripped as 13 menus incl. its own two
  showcase menus kept), then AU. **First AU attempt silently reverted**: I wrote the file and
  `c.reload`ed, and the file came back as the *old* 12 menus. `Unload()` calls `SaveConfig()`,
  so a reload writes the in-memory config over the new file before `Load` reads it — the
  opposite race from the autosave one I had guarded against. Live procedure is therefore
  **unload → write → load**, verified by reading the file after each step (old after unload,
  new after write, new after load). Rule: *for any plugin that saves config on Unload, a
  "reload to pick up my edit" is a no-op that looks like success* — read the file back.
- **Owner's in-game test found the real bug (fixed, MixMenuKit 3.0.21, both boxes)**: open a
  submenu from the Tab sidebar, close it, and the sidebar was gone too — then the next Tab
  toggled the sidebar back ON with the inventory closing, leaving a cursor-locked panel on the
  main screen until the player typed `/i`. MixCore's hub button did *not* do this, which was the
  tell: MixHub draws in its own UI root, MixMenuKit drew every menu into one root, so a submenu
  destroyed the sidebar to draw itself and `CClose` destroyed the root outright. Fix, Cat 6:
  the Inventory Menu gets its own root (`MixMenuKit.InvRoot`) and its own open-state
  (`InvOpen`/`InvMenuId`), so normal menus stack over it and closing them leaves it in place;
  trigger toggle, force-close-on-loot, `/mixmenu close`, disconnect and unload all handle both
  roots. Second half: the sidebar no longer sets `CursorEnabled` — the inventory screen already
  supplies the cursor, so a sidebar that outlives the inventory (ESC, loot echo, any desync) is
  now inert instead of trapping the mouse. Rust still offers no server-side "own inventory is
  open" signal, so perfect sync stays impossible; this removes the *cost* of a desync instead.
  Scripted patch with per-hunk match assertions (14 draw calls re-parented), compiled both ways,
  dedicated first (hot reload, config intact at 13 menus), then AU (11 menus, 47 scripts, 0
  failed). **Rule**: *a panel meant to live inside another UI must not bring its own cursor
  lock, and a family of UIs that should coexist must not share one root.*
- **Second in-game round: SORT INV (MixWorld 0.7.6 + menu config v2, both boxes)**. Owner
  reported BUILD → BACKPACK "closes both menus", and a small pop-up button near the hotbar that
  turned out to be the quick-sort. Traced: `/backpack` opens a real loot session
  (`StartLootingEntity` + `RPC_OpenLootPanel`) and MixMenuKit force-closes the sidebar on any
  loot by design (the sidebar would sit over the box contents) — expected, not a bug, documented
  for the owner; the pop-up is MixWorld's loot-panel "Sort Inv" button drawn on the *Hud* layer,
  i.e. underneath the inventory screen, which is why it looked blurred yet still clicked. The
  sort routine itself never needed the box — the box was only a gate for the pop-up. Fix:
  `/sortinv` (+ `mixworld.sortinv`) sorts the main inventory directly; new config
  `Features.QuicksortLootButton` (default true for standalone users) hides the pop-up, set
  false here. Sidebar plate FIX UI → SORT INV, FIX UI moved into WORLD (now 10/12). Patched
  against the *current* 0.7.5 source (post-SetFlag), not the older pull — worth saying, because
  the older copy was still lying around. Config changes used the right procedure per plugin:
  MixWorld saves only on load/admin action → plain reload; MixMenuKit saves on Unload → unload →
  write → load, file read back after each step. Dedicated first, then AU; 0 failed on both.
  **Cat 6 rule**: *a UI element that belongs to a menu system should be a plate in it, not a
  free-floating overlay — and anything drawn on "Hud" renders under the inventory screen.*
- **Third round — a regression my own sidebar fix created (menu config v3, both boxes)**: every
  submenu's `← BACK` plate ran `/i`, which is the sidebar's *toggle*. Under 3.0.20 the submenu
  had already destroyed the sidebar, so `/i` re-opened it and BACK looked right; under 3.0.21 the
  sidebar survives beneath the submenu, so the same `/i` now closed it — BACK killed both. The
  plate wanted "close this submenu", not "toggle the sidebar": rewired all six to the console
  command `mixmenukit.close` (closes the normal root only; plates without a leading slash are
  sent as console commands). `← ADMIN → /admin` plates are fine — opening a different normal
  menu replaces the current one, no toggle involved. **Cat 6 rule**: *when a fix changes what
  stays open, re-read every plate/command that was written against the old lifetime — a toggle
  that "returned" you somewhere only because that thing had been destroyed is a latent
  inversion.* Also documented for the owner: `/backpack` opens a hidden small-stash entity
  (spawned 50 m underground, `Disabled` flag, `enableSaving=false`) as a loot panel — 1 row of 6
  slots by default, 4 or 7 rows via `mixpack.world.backpack.4/.7` — so "extra slots" means a
  stash-style loot window plus a "Backpack: N row(s)" chat line, not slots added to the
  inventory grid.
- **Backpack never opened — a wrong-typed RPC, live since the feature shipped (MixWorld 0.7.7,
  both boxes)**. Owner's report was precise enough to bisect on: "chat line appeared, no loot
  panel". The chat line prints *after* the open call, so the server side ran; the client drew
  nothing. Decompile: `RPC_OpenLootPanel` takes a **panel-name string** (`"generic"`,
  `"player_corpse"`, `"generic_resizable"`); MixWorld sent `container.net.ID` — the client got
  a nonsense name. And `StartLootingEntity()` on its own never adds the container to the loot
  session; Rust's `StorageContainer.PlayerOpenLoot` is the whole sequence (flags → AddContainers →
  SendImmediate → RPC with `panelName`). Switched to it, and set `panelName =
  "generic_resizable"` on the stash so a 4- or 7-row backpack renders every slot (the stash's own
  panel draws six). Checked `CanBeLooted` (only rejects transferring — the `Disabled` flag on the
  hidden stash is fine) and that `generic_resizable` exists in this build (workbench recycle bin
  uses it). **Cat 7 rule, general**: *a `ClientRPC` argument is an untyped payload — the
  compiler cannot tell you it's the wrong kind. Read the client handler's signature in the
  decompile before hand-rolling any RPC, and prefer the entity's own open/close method to
  re-implementing the sequence.* Compiled both ways, dedicated then AU.
- **Untested until someone presses Tab**: plate → submenu → BACK round-trips, and the Tab
  bind (unchanged trigger `/i`, so existing binds keep working). Backups: both boxes,
  timestamped `backups/menu-rework-*` with AU's full `data/MixMenuKit` alongside.

- **Clean**: Cat 7 (helper null-guards the entity; `using var` disposes on throw — Cat 8),
  Cat 12 (static class, no state), Cat 11 (no lifecycle surface), patcher handles both
  namespace styles and refuses to double-apply. AU: 47 scripts, 0 failed after every reload
  tonight; dedicated: all mirrored versions loaded, 0 failed.

- **Pulse question settled the same night (→ 1.0.3)**: set `PulseAdminForCamera=false` on AU,
  reloaded (config re-save confirmed the value stuck), same non-admin rode the same bike:
  *"same, works, toggle works, all working."* Camera distance identical with the pulse off,
  so it was broadcasting a momentary admin flag to every client in range for nothing.
  Shipped default flipped to `false` in the source (opt-in kept for a client build where
  `camdist` matters again), README updated. General rule this reinforces: **when a plugin
  elevates a player's privilege flag "briefly" for a cosmetic, test whether the cosmetic
  actually needs it before accepting the elevation** — here the answer was no, and a
  read-through could never have said so.
- **Not a regression, noted**: only MixCore and MixModsConnectUI re-register when MixImages
  reloads (they wire `OnPluginLoaded("MixImages")`); the other 14 warmers don't, and don't need
  to — their slots persist in `MixImages/registry.json` and Rust's server FileStorage across a
  plugin reload. A future MixImages change that *invalidated* stored slots would have to go
  through a full restart, not a reload, for the same reason.
