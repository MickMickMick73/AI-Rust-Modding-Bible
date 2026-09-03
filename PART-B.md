# The AI Rust-Modding Bible
## Part B — Agent Ruleset

*This document is written for an AI coding agent, not a human. If you are an
AI agent reading this before writing or editing a Rust Oxide/Carbon plugin:
read this whole document first. It encodes verified findings from real
testing, real bug reports, and real marketplace-reviewer feedback — not
guesses. Every rule below states its own evidence. Follow the reasoning, not
just the rule text, so you can extend it correctly to cases it doesn't cover.*

*This is a living document. Section 10 explains how to add to it. Do not
treat it as closed or complete — treat it as the current known-good baseline.*

---

## 1. Plugin skeleton

Every plugin starts from the same shape. This is not stylistic preference —
consistency here is what lets one AI session's work be picked up cleanly by
the next one, and it's the shape every experienced Oxide/Carbon reviewer
expects to see.

```csharp
// Copyright © <year> <author>. All rights reserved.
// Licensed for use on Rust servers you own or operate — see LICENSE.md for full terms.
// Not for redistribution, resale, or reuse of this code or its artwork assets elsewhere,
// and not for use as a component of another product without a separate written agreement.

using System;
using System.Collections.Generic;
using Oxide.Core;
using Oxide.Game.Rust.Cui;
using UnityEngine;

namespace Oxide.Plugins;

[Info("PluginName", "AuthorName", "1.0.0")]
[Description("One sentence, plain language, no marketing fluff — what it does, not why it's great.")]
public class PluginName : RustPlugin
{
    // permissions, config, state, hooks, commands, UI — in that order
}
```

**Rules:**
- File-scoped `namespace Oxide.Plugins;` (C# 10+ syntax), not the older
  block-brace namespace — matches every plugin in a modern pack and avoids an
  extra indent level across the whole file.
- The `[Info]` version string is the *only* place version lives — never
  duplicate it in a comment or a config field. When you bump it, bump it
  here and nowhere else.
- Only `using` what's actually referenced. An unused `using` is exactly the
  kind of thing an experienced reviewer notices in the first ten seconds —
  see Section 7.
- **Only import what you can find a real call site for.** Every `using`
  directive should trace to at least one actual usage elsewhere in the file.
  If you remove the last thing that needed a namespace, remove the `using`
  in the same edit — don't leave it "just in case."

---

## 2. Config — the exact pattern to use

This is the single most consequential section in this document. It exists
because a proposed rewrite from an actual marketplace reviewer replaced an
earlier, more "defensive" version of this pattern, and the instruction that
came with it was explicit: *implement it exactly as given, do not add your
own safety logic on top.* That instruction itself is the lesson — a reviewer
with real experience across many submissions has already made this tradeoff
deliberately. Don't second-guess it by re-adding complexity that was
consciously removed.

```csharp
private sealed class MenuButton
{
    public string Label = "BTN";
    public string Command = "/kit";
}

private sealed class MenuDef
{
    public int Id;
    public string Name = "MENU NAME";
    public string Trigger = "";
    public bool Default;
    public bool Public;
    public List<MenuButton> Buttons = new();
}

private sealed class PluginConfig
{
    [JsonProperty("NextId")] public int NextId = 1;
    [JsonProperty("Menus")] public List<MenuDef> Menus = new();

    public static PluginConfig DefaultConfig() => new()
    {
        NextId = 3,
        Menus =
        {
            new MenuDef { Id = 1, Name = "MAIN MENU", Trigger = "main", Default = true, Public = true,
                Buttons = { new MenuButton { Label = "KIT", Command = "/kit" } } },
        }
    };
}

protected override void LoadDefaultConfig() => _config = PluginConfig.DefaultConfig();

protected override void LoadConfig()
{
    base.LoadConfig();
    try
    {
        _config = Config.ReadObject<PluginConfig>();
        if (_config == null) LoadDefaultConfig();
        if (_config.Menus == null) _config.Menus = new List<MenuDef>();
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

protected override void SaveConfig() => Config.WriteObject(_config, true);
```

**What is deliberately *not* here, and why:**

- **No `.bak`-on-corrupt-config backup file.** An earlier, more defensive
  version of this pattern wrote a timestamped backup of the corrupt config
  before overwriting it. A real reviewer's instruction removed this
  outright: it clutters a server owner's config folder with files they never
  asked for and rarely need. If the config is unreadable, log the exception
  (`Debug.LogException`) and move on — the exception itself is the audit
  trail; a stray `.bak` file sitting on disk forever is not a service to the
  server owner, it's a byproduct nobody asked for.
- **No "preserve the in-memory config on a transient read failure" guard.**
  An earlier version only fell back to defaults if there was no in-memory
  config already loaded (`if (!hadConfig)`), to protect a good live config
  from being wiped by a one-off read glitch. This was also removed by
  explicit instruction. The reasoning given: keep the logic simple and
  predictable: read, and if it fails, you get defaults. Don't build in
  silent recovery paths that make it harder to reason about what state
  the plugin is actually in.
- **No migration logic for a brand-new plugin.** See Section 3 — migration
  is a real, sometimes-necessary thing, but only when there's an actual old
  format in the wild to migrate *from*. A first submission has never shipped
  before, so there is nothing to migrate.

**The generalizable rule:** don't add defensive code for a failure mode that
can't currently occur. Every extra branch is something a reviewer has to
read, question, and satisfy themselves is necessary. If it isn't necessary
yet, it's not defensive — it's bloat wearing defensive code's clothes.

---

## 3. Migration logic — when it's real, when it's not

Migration code (reading an old config shape and converting it to a new one)
is legitimate *only* when a plugin has already shipped and has real installs
running the old format. Concretely: a plugin new to a marketplace has zero
installs of any prior format, so migration code in a first submission is
dead code from the moment it ships — it's solving a problem that cannot
exist yet.

**Before-and-after, from an actual removal done under this rule:**

```csharp
// BEFORE — a shipped plugin's LoadConfig, complete with migration for a
// genuinely-old field that some real, already-running installs still had:
protected override void LoadConfig()
{
    base.LoadConfig();
    try
    {
        _config = Config.ReadObject<PluginConfig>();
        _config ??= new PluginConfig();
        _config.Levels ??= new Dictionary<string, LevelDef>();
        MigrateLegacyConfig();
    }
    catch
    {
        PrintWarning("Invalid config — using defaults");
        LoadDefaultConfig();
    }
    SaveConfig();
}

private void MigrateLegacyConfig()
{
    try
    {
        var raw = Config.ReadObject<JObject>();
        if (raw == null || raw["GroundOnPaste"] != null) return;
        if (raw["PasteStability"]?.Value<bool>() is bool pasteStability)
        {
            _config.GroundOnPaste = !pasteStability;
            raw.Remove("PasteStability");
            raw["GroundOnPaste"] = _config.GroundOnPaste;
            Config.WriteObject(raw);
        }
    }
    catch { /* non-fatal — defaults apply */ }
}
```

That code was *correct and necessary* — real, already-running servers had
configs with the old `PasteStability` field, and this let them upgrade
without losing their settings. It only became removable when the same
pattern showed up in a **brand-new** submission with no install base at all
— at which point it was pure dead weight, deleted along with its
now-unused `Newtonsoft.Json.Linq` import.

**Decision rule:** before writing or keeping migration code, answer one
question honestly — *does a real, currently-running install of the old
format exist anywhere?* If you don't know, ask rather than assume either
way. Assuming "yes" when the real answer is "no" ships dead code and extra
attack surface for nothing. Assuming "no" when the real answer is "yes"
breaks someone's live server.

---

## 4. CUI patterns

### 4.1 Standard Oxide CUI, not Carbon-native, unless the whole plugin buys in

Two different CUI systems exist in this ecosystem: the standard Oxide
`CuiElementContainer`/`CuiHelper.AddUi` pattern (portable, works on Oxide
*and* Carbon), and Carbon's own native CUI builder (`Carbon.Components.CUI`,
Carbon-only). **Default to the Oxide pattern.** It is what almost every
existing plugin in a pack is already built on, and mixing the two inside one
plugin is a real, verified cost, not a hypothetical one: Carbon's
`[ProtectedCommandAttribute]` — which hides a console command's real name
from a client-side console log, worth having for admin-power commands like
`god`/`noclip` — only works with Carbon-native CUI button bindings. A plugin
built on the standard `CuiElementContainer` pattern cannot adopt it without
rebuilding its entire UI layer on the Carbon-only builder. Verified by
decompiling `Carbon.Core.ModLoader` directly, not assumed from docs.

**The actual tradeoff, stated plainly:** protecting a handful of admin
console commands is not worth rebuilding an entire plugin's UI system and
losing Oxide compatibility. Skip `[ProtectedCommandAttribute]` on plugins
built the standard way. It's a real feature with a real cost — don't reach
for it by default.

### 4.2 Debounce pattern for text inputs

Rust's client-side `InputField` UI can echo a stale value back to the server
for a few frames after the player actually changes it (an artifact of the
render pipeline, not a bug in your code). Left unhandled, this causes a
player's freshly-typed text to flash and revert. The verified fix is a
short time-windowed staleness check, not a longer debounce timer that would
make the UI feel laggy:

```csharp
private const float EchoWindowSeconds = 0.05f;

private static bool IsStaleEcho(string incoming, string baseline, string current, float lastEditTime) =>
    baseline != null && current != null
    && string.Equals(incoming, baseline, StringComparison.Ordinal)
    && !string.Equals(current, baseline, StringComparison.Ordinal)
    && UnityEngine.Time.realtimeSinceStartup - lastEditTime < EchoWindowSeconds;

// Usage in a console command handling an input field's OnTextChanged:
[ConsoleCommand("plugin.name")]
private void CName(ConsoleSystem.Arg arg)
{
    var p = arg.Player();
    // ... resolve target object ...
    var text = arg.GetString(0, "");
    if (IsStaleEcho(text, State(p).BaselineName, target.Name, State(p).BaselineNameEditTime)) return;
    State(p).BaselineNameEditTime = UnityEngine.Time.realtimeSinceStartup;
    target.Name = text;
}
```

Track one `Baseline<Field>` string and one `Baseline<Field>EditTime` float
per editable text field, per player, in your per-player state object.

### 4.3 Self-healing pagination for any list a player can grow unbounded

If a UI shows a list a player controls the size of (menus, entries, saved
items), do not cap it with "show the first N, plus a static '+N more' label."
That's a real, reported UX failure: it silently defeats "unlimited" as a
feature, and it requires the player to remember exact names to reach
anything past the cap. The fix is real pagination with a self-clamping page
index:

```csharp
private const int MaxShown = 8;

[ConsoleCommand("plugin.page")]
private void CPage(ConsoleSystem.Arg arg)
{
    var p = arg.Player();
    var dir = arg.GetInt(0, 0);
    var pageCount = Math.Max(1, (int)Math.Ceiling(_items.Count / (float)MaxShown));
    var st = State(p);
    st.Page = Math.Max(0, Math.Min(pageCount - 1, st.Page + dir));
    Redraw(p);
}

// In the draw method — clamp defensively every time, not just on navigation,
// so a page that's now out of range (e.g. items were deleted) self-heals
// instead of rendering blank:
var pageCount = Math.Max(1, (int)Math.Ceiling(_items.Count / (float)MaxShown));
st.Page = Math.Max(0, Math.Min(pageCount - 1, st.Page));
var startIdx = st.Page * MaxShown;
var shown = Math.Min(MaxShown, _items.Count - startIdx);
```

The clamp in the *draw* method (not just the navigation command) matters —
it's what makes the page index recover automatically if the underlying list
shrinks between draws (an item gets deleted from a different code path,
say), rather than needing its own separate bounds-checking bug fix later.

**Also:** insert new player-created items at the *front* of the list, not
the back. It means the picker's first page is always "what was touched most
recently" — the thing the player is most likely still working on, or most
likely to have forgotten about and need to find again. If you do this,
check whether anything elsewhere falls back to "the first item in the list"
as a default (e.g. re-promoting a deleted default item) — that fallback now
means "the newest item" instead of "the oldest," which is usually the wrong
direction. Flip it to the *last* item in the list to preserve the original
"pick something stable and established" intent.

---

## 5. Real Rust/Oxide/Carbon behavior — verified, not assumed

Everything in this section was confirmed by directly reading engine state or
by reproducing the behavior live — not inferred from documentation or
convention, because in more than one case the actual behavior does not match
what would be reasonable to assume.

### 5.1 There is no server-side flag for "the player's own inventory is open"

Confirmed by reflecting over every public field and property on `BasePlayer`,
`BasePlayer.inventory`, and `BasePlayer.inventory.loot` in a live session, in
three states (nothing open / player's own inventory open / looting an
external entity open), and diffing the results. `player.inventory.loot.IsLooting()`
reliably flips `true` only while looting an external entity (a box, a
corpse). It stays `false` for the player's own bare inventory — identical to
the "nothing open" state. No other field anywhere in those three objects
changes with UI state either.

**Implication:** if you need to know whether a player's own inventory panel
is open, you cannot ask the server. You can only reliably detect looting an
external entity, via:

```csharp
private void OnLootEntity(BasePlayer player, BaseEntity entity) { /* loot session started */ }
private void OnLootEntityEnd(BasePlayer player, BaseEntity entity) { /* loot session ended */ }
```

If a feature needs to track "is the player's inventory-adjacent UI up" and a
client-side keybind is involved (see 5.3), design around the fact that a
naive open/close toggle *will* drift out of sync the moment the player loots
anything — because looting silently changes native inventory visibility
without ever going through that keybind.

### 5.2 A plain reload does not guarantee a fresh disk read of a config file

If a config file is replaced on disk by something other than the plugin
itself (an external file copy, a git checkout, a backup restore) while the
server is still running, `c.reload <plugin>` — and even a full
`c.unload` followed by `c.load` in the same process — is **not** reliable
for picking up the externally-written file. The plugin's own `SaveConfig()`
(which most `LoadConfig()` implementations call at the end of a successful
load) can end up re-writing whatever was still in memory back over the
freshly-restored file, because the reload didn't actually re-read from disk
the way it appears to. Confirmed by reproducing it three separate times
against a live server.

**The only fix that reliably works:** a full process restart. No in-memory
state survives that, so the next `LoadConfig()` call is guaranteed to read
whatever is actually on disk.

```bash
systemctl restart <your-carbon-service>
```

If you ever need to externally replace a config file on a running server,
plan for a restart, not a reload — and say so plainly rather than silently
hoping a reload is enough.

### 5.3 Client keybinds combining a native command with a plugin command will desync

A common pattern for player-configured menus: a bind that fires both a
native Rust command and a plugin's own chat-triggered toggle in one
keypress, e.g.:

```
bind tab inventory.toggle chat.say /mymenu
```

This looks reasonable and works most of the time — until the *native*
command's effect changes through some path that never goes through the
bind. Looting a box is exactly that path: it opens/closes the native
inventory panel through Rust's own interaction system, not through the
player's keybind, so the plugin's chat-triggered toggle never fires to stay
in sync — but the *next* time the bind key is pressed (say, to close that
box), the toggle fires again anyway, now out of phase with reality.

**This is not fixable by changing which key the bind uses.** The failure
mode is about *what native state changed without going through the bind*,
not about which key triggers it — any key bound the same way has the
identical problem.

**The actual fix**, combining 5.1 and this section: force-close on both
ends of any detectable loot session (5.1's hooks), *and* use a short,
consumed suppression window to swallow the specific echo that follows
immediately after a loot session ends — rather than trying to prove the
menu "should" be open, which Section 5.1 already established you cannot do:

```csharp
private const float SuppressSeconds = 0.6f;

private void OnLootEntityEnd(BasePlayer player, BaseEntity entity)
{
    ForceCloseMyMenu(player);
    var st = PeekState(player); // see 5.4 — do not use the allocating lookup here
    if (st != null) st.SuppressMenuOpenUntil = UnityEngine.Time.realtimeSinceStartup + SuppressSeconds;
}

// In the trigger handler, before opening:
if (UnityEngine.Time.realtimeSinceStartup < st.SuppressMenuOpenUntil)
{
    st.SuppressMenuOpenUntil = float.NegativeInfinity; // consume it — one swallow, not a standing block
    return;
}
```

The window length is a real, stated tradeoff, not a proven-perfect value:
long enough to comfortably cover normal network latency between the box
closing and the bind's second command arriving, short enough not to block a
genuine standalone open a moment later. 0.6 seconds was chosen and tested
live, not derived mathematically — treat it as a starting point, not a
constant to copy blindly onto an unrelated problem.

### 5.4 Hooks that fire for every player server-wide need an allocation-free lookup

`OnLootEntity`/`OnLootEntityEnd` (and most global player hooks) fire for
*every* player's interaction with the game, not just players who use your
plugin. If your handler calls a "get-or-create" state lookup, you are
silently allocating a tracking entry for every player on the server who
touches a loot container, whether they've ever used your plugin or not —
verified as a real, unnecessary allocation, not a hypothetical one, on a
plugin whose loot hooks did exactly this.

```csharp
// Allocating — creates an entry for ANY player, every call:
private EditState State(BasePlayer p)
{
    if (!_edit.TryGetValue(p.userID, out var st))
        _edit[p.userID] = st = new EditState();
    return st;
}

// Read-only — costs nothing for a player who's never touched this plugin:
private EditState PeekState(BasePlayer p) =>
    _edit.TryGetValue(p.userID, out var st) ? st : null;
```

**Rule:** use the allocating "get-or-create" lookup only inside a code path
that's already gated to your plugin's own actual users (a command handler
behind your plugin's own trigger, say). Any handler for a hook that fires
for *every* player regardless of whether they use your plugin should use
the read-only variant and simply no-op if there's nothing there yet.

---

## 6. Code hygiene — what an experienced reviewer actually notices first

These are small, individually low-stakes, and collectively the fastest way
to signal "AI-generated without review" to someone who's read a lot of
plugin code. Every example below is a real instance caught in this project,
not a hypothetical.

- **Self-assignment dead code.** `p.cloak = p.cloak;` — caught by a linter's
  `no-self-assign` rule, not by reading. Decay/update logic for a value like
  this belongs where the actual computation already happens (in this case,
  a `Math.Max(0, p.cloak - dt)` a few lines earlier already did the real
  work) — don't leave a no-op statement that looks like it's doing
  something.
- **Vestigial conditionals with an empty/comment-only body.**
  ```csharp
  if (e.hurt > 0.04) { /* still apply */ }
  e.hp -= dmg; // this runs unconditionally regardless of the branch above
  ```
  If a conditional's body is just a comment, the conditional itself is
  leftover from a refactor and should be deleted, not preserved "in case."
- **Unused locals and unused imports.** Run a linter before considering
  anything done. `dotnet build`/an IDE's own unused-variable warnings catch
  most of these; don't rely on visual scanning.
- **Empty catch blocks with no explanation.** `catch {}` reads as an
  oversight even when the swallow is intentional. If a failure is meant to
  be silently ignored, say so:
  ```csharp
  catch {
      // Malformed input — fall through to the safe default below.
  }
  ```
- **Stale comments that describe removed behavior.** If a feature or
  integration is removed, grep the *whole* codebase for its name — plugin
  file, `[Description]` attributes, other plugins' `[PluginReference]`
  fields calling into it, listing/doc files — not just the obvious source
  file. A `[Description]` that still advertises an integration with
  something that no longer exists is exactly the kind of thing that erodes
  trust the moment someone notices it.
- **Balance-check every file before considering an edit done.** A single
  Python one-liner catches a huge fraction of "the edit tool matched wrong
  and silently broke a brace/paren pair" mistakes before they ever reach a
  compiler:
  ```bash
  python3 -c "
  c = open('PluginName.cs', encoding='utf-8').read()
  print('braces:', c.count('{') - c.count('}'))
  print('parens:', c.count('(') - c.count(')'))
  "
  ```
  Both numbers must be exactly `0`. Run this after *every* edit to a file
  you're not immediately going to compile, not just before a final deploy.

---

## 7. Security baseline

- **Permission-check at the top of every command handler**, before any
  other logic runs — not buried after other work has already happened.
  ```csharp
  private void CmdSomething(BasePlayer player, string command, string[] args)
  {
      if (player == null) return;
      if (!permission.UserHasPermission(player.UserIDString, PermUse)) { Deny(player); return; }
      // ... actual logic only after both checks pass
  }
  ```
- **Never trust a client-supplied string or number for anything
  security-relevant** without validating it server-side — length limits on
  text fields, range clamps on numeric input, existence checks before using
  a client-supplied ID to look anything up.
- **`ConsoleSystem.Arg` values can arrive malformed** in ways worth an
  explicit guard for (verified: a Carbon-specific quirk where an argument
  can come through as a raw `stringview` wrapper rather than a plain
  string in certain call paths) — don't assume `arg.GetString(0)` always
  returns exactly what a normal chat/console input would.
- **Admin-power commands (god mode, noclip, vanish, entity removal, server
  config) need an explicit admin check separate from your plugin's own
  "can use this plugin at all" permission**, verified as a real distinction
  in a plugin that legitimately mixes both tiers in one file — some of its
  commands (home, kit, teleport) are meant for every player, others
  (god, noclip, vanish) are gated behind an actual admin-level check. Don't
  assume "has permission to use this plugin" is the same thing as "should
  have admin powers through it." Read the actual gating in the handler
  body, not just the registration list — the list alone doesn't show you
  which commands are actually gated and which aren't.

---

## 8. The verification checklist

Run this — actually run it, with real tool calls, not from memory — before
considering any change to a live or shippable plugin finished. This is not
a suggestion; treat every unchecked item as a plausible undiscovered bug.

1. **Structural balance** — brace/paren count both `== 0`, on every file
   touched (Section 6).
2. **Config/save safety** — does any new field touch a `[JsonProperty]`
   config class, or is it purely in-memory runtime state? If it's
   in-memory, it should never appear in a save-format discussion at all —
   confirm by checking the class it lives on has no `[JsonProperty]`
   attributes anywhere.
3. **Correctness scope** — re-read the actual conditional that gates a fix,
   not your memory of writing it. Confirm it can't fire somewhere you didn't
   intend.
4. **Null safety** — every new method that takes a `BasePlayer` (or any
   nullable game object) checks for null before touching it.
5. **Self-expiry / no stuck state** — any new timestamp-based flag ages out
   on its own if never explicitly consumed; any new per-player tracking
   entry gets cleaned up on `OnPlayerDisconnected`.
6. **Dead code sweep** — grep the diff for anything that used to be called
   and no longer is; grep the whole codebase (not just the file you edited)
   for the name of anything you removed.
7. **Version bump** — the `[Info]` attribute version incremented, and every
   deployed copy (dev source, submission copy, any live server) updated to
   match — a mismatch between what a live server reports and what's in your
   source tree is itself a bug waiting to cause confusion later.
8. **Deploy and reload-test** — push the change, then reload the plugin
   **three times in a row**, not once. A single clean reload doesn't rule
   out a race or an intermittent compile issue; three consecutive clean
   reloads with 0 failures/0 exceptions is the actual bar.
   ```
   c.reload PluginName
   c.reload PluginName
   c.reload PluginName
   ```
9. **Live functional verification** — actually exercise the changed
   behavior in-game (or via a real test account/tool), don't infer
   correctness from a clean reload alone. A plugin can compile and load
   perfectly and still be functionally wrong.
10. **Config integrity re-check on any server with real production data** —
    after any deploy that touches a config-adjacent code path, read the
    live config back and confirm real data is still intact, not reverted to
    a default/test state.

---

## 9. Submission-readiness — what a real reviewer actually checks

- **No backup files, no dead migration code** — Sections 2 and 3.
- **Assets self-install where possible**, rather than requiring a separate
  manual asset-folder copy step, when the pattern is available — reduces
  install friction and support burden.
- **Real, in-game screenshots — not AI-generated art.** Explicitly and
  negatively called out by an actual reviewer on a real submission. If a
  listing needs visual material, capture it from the actual running plugin.
- **Never describe a fixed bug in the past tense as if it's a feature.**
  "No longer accidentally does X" tells a customer nothing useful — they
  never experienced X in the first place, because they're seeing the fixed
  version. State the current, correct behavior plainly instead. If a
  changelog needs to mention a fix, describe what's different *now*, not
  what was wrong *before*.
- **Changelogs list only genuinely user-visible changes**, in plain
  before/after language, closed with an explicit statement of what did
  *not* change when true. A pure internal refactor or performance fix with
  zero observable behavior difference doesn't earn an entry. No names of
  external people, no narration of internal development process (false
  starts, who found what) — state only the final, correct behavior.
- **Config is generally not shipped with a submission** — it's expected to
  generate fresh on first load, matching the convention already established
  by accepted listings on at least one real marketplace.

---

## 10. How to extend this document

This is the actual goal, not a footnote: this document should grow every
time an agent (any agent, on any project) discovers something new the same
way the entries above were discovered — real testing, a real reviewer's
feedback, a real bug traced to its actual cause. Not a guess, not "this
seems like good practice."

**When you add an entry:**

1. **State what you actually did to find out**, not just the conclusion.
   "Confirmed by reflecting over every field on X in three live states" is
   verifiable and trustworthy. "This is probably because..." is not — if
   you're not sure, say so explicitly rather than presenting a guess with
   the same confidence as a verified fact.
2. **Include a real, complete code example** wherever the finding is
   code-shaped — not pseudocode, not "something like this." A future agent
   should be able to use the example directly, not have to guess the parts
   you left out.
3. **Explain the reasoning, not just the rule.** A rule without its reason
   gets applied mechanically to cases it doesn't actually fit. State *why*
   a decision was made so a future reader can extend it correctly to a
   situation this document didn't anticipate.
4. **Say what's still uncertain.** If something is a reasonable inference
   rather than a directly-verified fact, mark it as such, the way Section
   5.3's suppression-window length is explicitly flagged as "a starting
   point, not a constant to copy blindly."
5. **Don't overwrite an existing entry's evidence to make room for a
   correction — append the correction and say what changed.** If a later
   finding contradicts an earlier one, that contradiction is itself useful
   information (something changed, or the earlier finding was wrong for a
   reason worth recording) — don't quietly erase the trail.

The value of this document is entirely in how honestly it distinguishes
*verified* from *assumed*. The moment it stops doing that, it becomes just
another source of confident-sounding guesses — which is the exact problem
it exists to solve.
