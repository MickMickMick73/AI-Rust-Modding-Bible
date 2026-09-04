# The AI Rust-Modding Bible — Part D: Verification Run Playbook

This document is for an AI agent asked to run a fresh QA/verification pass on the MiX Server Pack
and its standalone mods, **without** the conversation history of the session that built them. Part
A/B/C are rulesets — what to check and why. This is the runbook for actually doing a check: what
the product is, where things live, and how to report back.

**This document does not contain server access credentials.** If a live-server verification pass
is being requested, the project owner provides RCON/SSH access separately and directly — never
proceed on assumed, guessed, or previously-seen credentials, and never ask for them to be pasted
into a public or logged channel.

## 1. What you're checking

Two distinct products, drawn from a single pool of live-tested plugin source:

**MixServerPack (core, 14 plugins)** — the pack a customer gets as one purchase:
`MixCore, MixImages, MixPackAssetsShim, MixModsConnectUI, MixMenuKit, MixGovern, MixWorld,
MixWorldTune, MixCommerce, MixHud, MixUiFix, TimeOfDay, SimpleFurnaces, MixAdminMove`

**Standalone mods (~29, growing)** — sold individually, each fully self-contained (works with or
without MixServerPack installed): building/PvP/RPG/skin/utility mods such as `FreeBuild`, `Salvo`,
`MixSignboard`, `MixEntityScale`, `MixSkinsLight` (+ its `MixSkinOwnership` dependency), and others
— check the current live plugin list rather than assuming this document's snapshot is exhaustive,
since new standalones are added over time.

A shared visual-asset pool (`MixPackAssets/uikit/`, provisioned by `MixServerPack`) supplies
painted UI-kit art to any plugin that wants it. Every plugin that consumes it degrades gracefully
to plain flat-colour styling if the pool isn't present — this must never be a hard dependency. See
Part B/C for the shared-pool rationale and the exact fallback pattern to verify (a button always
keeps its real command binding regardless of whether its texture loads).

## 2. The actual checklist

Run every item in `MODDING-QA-CHECKLIST.md`, published alongside this file in this repo. It's a
living 12-category document built from real, found-and-fixed bugs, not a generic template — read
the "Log of additions" at the bottom too, not just the numbered categories: it's where the actual
reasoning and the real cases that justified each item live, and it's what makes this repeatable
rather than a list of principles to apply from memory. Treat a gap found during a real review as a
signal to add a new category, not just fix the one instance — and if you extend it, append a dated
entry in the same style rather than editing past entries, so the reasoning trail stays intact.

This file is snapshotted at the time this playbook was published — it may be newer by the time you
read it if the project has moved on since. If you have access to the live project folder (not just
this public repo), prefer that copy over this one.

## 3. How to run a pass

1. Confirm what's actually live before checking anything — don't assume this document's plugin
   list is current. If you have server access, pull the real plugin list and current source fresh
   rather than trusting any cached/prior snapshot (including this one).
2. Prefer automated, mechanical checks (grep for a pattern) across the *whole* pack at once over
   reading one plugin fully before moving to the next — cheaper, and catches a regression anywhere
   in one pass.
3. Reserve close reading for what a grep can't catch: hook-firing-order bugs, whether a fallback
   path is actually reachable, whether a "fix" changed behavior it shouldn't have.
4. For anything you cannot verify live (no server access, or a check that specifically requires
   watching a real event happen — a genuine player disconnect, a real reload-stress test), say so
   explicitly rather than presenting a static-code read as equivalent to a live-verified result.
   This distinction has mattered before: a hook that reads correctly in source has, at least once,
   turned out not to actually fire on this Carbon build in practice — see Cat 4 in the checklist.
5. Report real findings only. A category with nothing wrong is worth stating plainly ("clean,
   verified") — padding a report with restated principles instead of actual findings isn't useful
   to the person reading it.
6. If you make a fix: verify it compiles/loads (0 failed, 0 exceptions) before considering it done,
   and version-bump the plugin's `[Info(...)]` line for anything with real behavior impact.

## 4. Before publishing anything back to this repo

This repo is public. Never commit real credentials, IP addresses, or anything else that would let
a stranger reach the live server. Findings, fixes, and methodology are fine to publish here —
they're what this repo is for. Connection details are not, ever.
