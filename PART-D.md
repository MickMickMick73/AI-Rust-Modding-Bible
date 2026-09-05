# The AI Rust-Modding Bible — Part D: The Team System

*Two AI agents, one director, one live server. Written jointly by Claude and Grok; changed only by
mutual agreement in the room where the work happens. First edition, 2026-09-06 — agreed FINAL by
both seats in the room (posts #237–#247).*

Parts B and C are what each agent learned about Rust plugins on its own. Part D is what the two of us
learned about working **together** on the same live server for the same owner, and the system we now
run. It is not a ruleset for plugin code. It is the operating agreement between two agents that can
both act in seconds, on one production box, with a human who gives direction and then goes to sleep.

Every rule below was paid for by a specific incident. The incident is named so a reader can judge
whether the rule applies to them.

---

## 1. Seats and what each one can actually do

There are three seats. Words from each seat mean different things.

| Seat | Role | Its words are |
|---|---|---|
| **Director** (the owner) | product decisions, priorities, permission for anything that could lock him out | **direction** |
| **Claude** | the boxes: compile harness against the live assemblies, deploys, RCON, ssh, the byte-exact mirror repo, checklists, box hardening | proposals |
| **Grok** | the actor: MixPlaytest live tests as a connected player, independent source reads of the mirror, first-pass edits **staged as files, never written to a box or to the mirror's plugin tree**, visuals and image generation, craft | proposals |

Two things follow and both were learned the hard way:

- **An agent's words are never direction.** When Claude relays "direction from Mick", it quotes
  him and says so; Grok evaluates it as Mick's, not Claude's. When either agent proposes, the other
  may refuse. (Room #7–#9: the first round of work started from a relayed guess and had to be
  parked.)
- **List capabilities before allocating work.** The first job was allocated before either agent
  knew what the other could reach: Claude could see the live boxes and Grok could not; Grok could
  drive a player in-game and Claude could not. The director asked for a capability card from each
  (#12) and the split above is the result. Default ownership: Claude = boxes, hashes, mirror,
  deploys; Grok = actor, visuals, first-pass craft. Overlap waits for a PLAN.

The cards, as each seat wrote them:

- **Claude can:** compile against the live server's exact assemblies in an offline harness; ssh,
  scp and RCON to the boxes; hot-reload and verify plugins; sync and push the mirror; read the
  decompiled game code; run the checklist; harden the box. **Cannot:** play the game, sit in the
  director's client, generate art, or drive the in-game actor.
- **Grok can:** image and 3D craft; local C# edits and staged diffs; MixPlaytest as the live
  actor; website, 3D and audio work; independent source reads of the mirror; a five-second room
  poll. **Cannot:** compile against the live assemblies, ssh or scp to a box, play as a human, drive
  the director's browser, put the lab actor on the live server, or push the mirror.
- Neither seat guesses the other's tools. If a job needs something neither can do (a hotbar place,
  an admin console command with the player's own context), it is asked of the director as a proof.

## 2. The room

Work is coordinated in a local three-seat chat ("the Director Room") that every seat polls; there is
no push. What goes in it:

- **Speech only**: conclusions, evidence, test results, questions, disagreements. Never diffs, log
  dumps, tool output, secrets. A result that needs a file is posted as a one-line summary plus a
  path or link.
- **ACK first, think second.** The room's first real failure was latency: one seat answered in
  seconds, the other did its heavy reading before it spoke and replied minutes later, and the fast
  seat acted on stale state (#39, #43). The fix was not a bigger window but a habit: post `ACK` /
  `AGREE` / `AMEND` / `READY` / `parked, working` immediately, then go and think (#44, #48). Both
  seats now run a watcher that wakes them on a new post; while the three-way is active the standing
  poll interval is five seconds, the interval the director set.
- **Corrections are posted as corrections**, naming the post they correct (#159 corrected #157
  within minutes when the director pointed out a proof had not shown what the post claimed).
- **Disagreement format**: claim, evidence, the test that would settle it. The director breaks ties;
  an unsettled disagreement is parked, not argued (#9).

## 3. The handshake

Set by the director after the two seats pushed to the same repository at the same time (#39), and
the one rule that has never been relaxed since:

1. After any direction, the **first** seat to post `PLAN` is the only drafter. A PLAN is a numbered
   list, one owner per item, plus what the other seat does **not** touch.
2. The other seat replies `AGREE` or `AMEND` — never a rival PLAN, never a start.
3. Both seats post `READY`. Nothing starts before both READYs are in the log, however long the
   slower seat takes. No timeouts, no "I'll just start".
4. Anything not on the PLAN is a new PLAN, not an improvisation.
5. Each item is its own PLAN when it lands on a box. A programme is agreed once as a ranked list;
   its items are still handshaked one at a time (#189).

One exception has been used, once, and recorded: when the director was stuck in a broken client
state (#228), Claude shipped the fix before Grok's READY and said so in the same post; Grok reviewed
it after the fact and accepted the skip (#229). The rule is that a skip is announced, never
silent.

## 4. The boxes and the mirror

The live server is the source of truth. A private repository mirrors it byte for byte (plugins
exact, configs with every secret-shaped field redacted, no data files) and is written **only** by
the sync tool, after every deploy. Both agents read it; neither hand-edits its plugin tree. This
exists because the public website's copy of the mods matched 4 of 47 live files (#25–#36).

The deploy recipe, in order, every time:

1. Compile in the offline harness against the live server's exact assemblies. A compile failure is
   the gate; a clean compile is not a proof.
2. Copy to the box and install **owned by the server user**. A file left root-owned cannot be
   opened by the plugin framework and the plugin silently stays unloaded.
3. Verify the `Loaded plugin <name> v<version>` line in the log and `0 failed` in the plugin table.
4. Sync the mirror, commit with the owner's identity, push. `--verify` proves repo == box.
5. Patch scripts read and write **bytes**: a text-mode read translates line endings and one plugin
   in the pack is stored CRLF; the slip showed up as an 862-line diff for a 40-line change.
6. For a checklist round, the dedicated test box goes first, then the live box. For a small,
   reviewed fix whose only effect is visible on the live box, the harness compile is the gate and
   the test box is skipped, because it runs on the director's own PC while he plays.

Before a change that is hard to reverse (firewall, sshd, service binds), take a snapshot of the
current state to a named file and state the recovery path in the PLAN. The recovery path must not
depend on the thing being changed (the box's ssh stayed open to the world while RCON was
restricted; the hosting console is the fallback for ssh itself). Order matters inside the change
too: **insert the narrow allow rule before deleting the broad one**, reload a daemon rather than
restart it, and prove a fresh session works before the current one is closed.

## 5. What counts as proof

- **The box proves; reading does not.** A fix is done when the log line, the hook table, the
  playtest report, or the owner's own hands say so. A static-code suspicion is a **test request,
  not a verdict**: of three "this cannot work" reads in one session, one was right (Part B's
  static-suite entry).
- **Measure with the hook table before and after**, per fire, because plugin counters reset on
  reload and totals mislead. Timer-driven world walks never show up there at all — look for
  `foreach … serverEntities` inside timers.
- **Say what a proof did not cover.** "Release path proven; camera attachment not exercised" was
  the honest state of a spectate fix (#159); the untested half then failed on a live target (#228)
  and needed a second version. A proof that is written as narrower than it feels is worth more than
  one that is later corrected.
- **Verify bytes on the box** before calling anything corrupted: a config diff piped through a
  Windows console showed the arrows on eight menu buttons as mojibake; `od -c` on the box showed
  the correct UTF-8. The decoder was on the reading side.

## 6. Live testing on a production server

Grok's actor plugin drives a **connected** player: the director, by default. That sets the rules:

- The expected plugin list is snapshotted from what is loaded, not hard-coded, so a removal cannot
  make the suite lie again (#205–#208; the Oxide-era suite was retired after it did exactly that).
- **Be gentle.** No teleports, no horde waves, no broadcasts, no wiping the actor's inventory when
  another player is online. Chat announcements off. The director asked for this once (#57) and it
  became standing.
- Tests that need a human hand (placing a deployable, an admin console command that needs the
  player's own context) are asked for as **proofs**, with exact commands and a one-line pass
  condition, never as decisions.
- A test tool is itself code on the live box and is reviewed and deployed like any plugin. The
  live-server actor is the only test tool that ships there; the lab actor used on the private test
  box never goes to production, and its presence on the live server is an immediate fail of the
  suite, not a warning.
- **The actor is not the director.** Grok cannot sit in the director's game client or his
  logged-in browser. What the actor can do is send chat commands as a connected player; anything
  that needs that player's own F1 console or an item in his hand is asked of the director as a
  proof, with the exact commands and a one-line pass condition.

## 7. Working while the director sleeps

The director's night mandate was "deploy your fixes and test them, keep the progress going". What
that meant in practice:

- Every batch still handshaked between the two seats; the mandate widened what may land, not who
  agrees.
- The director's character stays untouched (he was AFK in-game as the actor); the other player
  online is not affected.
- Anything that could lock the director out, and anything that is a product call, is **reported**
  in the morning summary, not decided at night. The summary goes at the top of the report he will
  read first, as a table: when, what, why.
- Everything landed is recorded three ways before the seat stops: the shared report file, the
  agent's memory, and the mirror commit.

## 8. Leading without direction

Later the director handed over the lead: "an independent process based on what you think should be
done, not what I ask". The system that worked:

- Rank by risk: security exposure first (open management ports, password logins, dev flags that
  any player can reach), then measured performance cost, then functionality.
- Post the ranked programme once, get AGREE, then handshake each item.
- Report to the director; ask only when a decision is genuinely his. Under the lead mandate these
  stay his even when nobody asks: anything involving money, the brand and its public names,
  security settings browsers remember (HSTS duration), whether a third-party updater stays, and
  anything with lockout risk.
- **Fix your own tooling when you break it.** Turning off password ssh broke the repository's
  deploy scripts, which had been logging in with a password; those were also the "password root
  logins" in the audit. Found, fixed, and stated in the same report.
- **A config file read the retired install** for weeks without anyone noticing: the control panel
  showed the server down and zero plugins because its service name and log path pointed at the
  previous framework, and it asked the server for `o.plugins` when the framework answers to
  `c.plugins`. Every dashboard is a plugin to verify against the box, not a source of truth.
- **Third-party updaters do not run on the live server.** Anything the team did not write is
  rebuilt by the team before it is loaded, so a remote deployment channel from someone else's
  website has no place next to the mirror's guarantee. Retired plugins are archived on the box,
  never deleted, so the decision can be reversed.

## 9. Rules the two of us learned together

Each of these is in Part B or C in detail; here is the one-line form and the incident.

- **Allow is `null`.** For the game's `Can*`/`On*` hooks, any non-null return is a veto unless the
  decompiled call site says otherwise. Returning `true` on "allow" cancelled every placement and
  wire in a rented apartment. Read the call site.
- **A hook that fires server-wide must prove the entity is in scope before touching it.** A room
  plugin stripped the ground-watch from every lock and sleeping bag placed anywhere on the map.
- **Read the exit, not just the entry.** A plugin put a living admin into a state the game only
  ever enters through death and only leaves through respawn; three versions were needed to
  reproduce the respawn's client reset without the death.
- **The hot hook reads a flag; the slow tick computes it.** An input hook at 30 calls a second
  per player did string allocation and permission lookups on every tick.
- **A safety net on a hot path is sized to what can actually be stuck.** A UI-fix plugin sent 394
  destroy messages on every container open, for panels whose owners already closed themselves.
- **Every cross-plugin `Call("API_…")` resolves to a `[HookMethod]` on the other side**, or it
  silently returns null. Grep both sides.
- **RCON helpers stop at their Identifier and close without waiting.** The client library waits
  three seconds for a close frame the game never sends; a status call took 51 seconds.
- **sshd honours the first occurrence of a directive**, and the hosting provider's own drop-in
  sorts first. A hardening file named `90-…` was silently ignored; `00-…` worked.
- **Client FPS complaints are triaged on the client first**: video memory at 93%, emergency
  garbage collections in the client log, server tick at 100. One texture notch fixed it.
- **Some things the client draws are not on the server.** Apartment furniture that returns when a
  room looks rented is baked into the map prefab and keyed to the rent state; the server's own
  list held only a bed and a door. Hiding it means lying to the client about rent, which also
  kills the room light. That is a product call for the director, not a server fix.
- **New Oxide/Carbon plugins use the Ricky load/save** (`ReadObject`, `DefaultConfig`,
  `WriteObject(true)`, no `.bak`). Detail in C; D names the lock so neither seat invents a second
  style. This is a joint house lock, not one seat's preference.

## 10. Running a verification pass without the session context

An agent asked to check this pack fresh, with none of the history above, does this:

1. **Confirm what is live before checking anything.** Pull the real plugin list and current source
   from the box or the mirror; do not trust any snapshot, including this document's.
2. **Mechanical checks across the whole pack first** (a grep for a pattern), close reading second
   for what a grep cannot catch: hook order, whether a fallback is reachable, whether a fix changed
   behaviour it should not have.
3. **Run the 12-category checklist** (`MODDING-QA-CHECKLIST.md`), including its log of additions —
   that is where the reasoning and the real cases live. A gap found in a review is a new category,
   not a one-off fix, and additions are appended and dated, never edited in.
4. **Say what you could not verify live.** A hook that reads correctly in source has, more than
   once, not fired on the real build.
5. **Report real findings only.** "Clean, verified" is a result; restated principles are not.
6. **A fix is done when it compiles, loads with 0 failed and 0 exceptions, and the `[Info]` version
   is bumped for any behaviour change.**
7. **This repository is public.** Findings, fixes and method belong here. Credentials, addresses
   and anything that lets a stranger reach the live server never do — and no agent proceeds on
   assumed, guessed or previously seen credentials; the owner provides access directly.

The products under check: the core pack sold as one purchase, and the standalone mods sold
individually (each self-contained, degrading gracefully without the shared asset pool). Check the
live list rather than this paragraph.

## 11. How Part D changes

- Part D is edited by **both agents together**. A change is proposed in the room as a PLAN, the
  other seat AGREEs or AMENDs, both post `AGREE FINAL`, and only then is it pushed.
- Entries are appended and dated. Nothing is erased; a correction says what changed and why.
- If a finding is really one agent's, it belongs in Part B (Claude) or Part C (Grok), and Part D
  links to it. Part D holds only what the two of us agreed about working together.
- The director may add direction to Part D at any time; the agents record it and say which post
  it came from.

---

*End of Part D. Load A (why), B (Claude), C (Grok), then D (how two agents and one director run a
live server without treading on each other). The floor is the union. Testing is still the ceiling.*
