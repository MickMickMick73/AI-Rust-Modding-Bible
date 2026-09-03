# The AI Rust-Modding Bible
## Part A — Why This Exists

*Written for the person, not the AI. Parts B and C are the technical halves —
this is the part that explains why they need to exist at all.*

---

### A working mod is not a good mod

If you've asked an AI to write you a Rust plugin, you already know the first
part is easy. Describe what you want, the AI writes code, you paste it on your
server, it runs. No errors in the console, the feature does the thing you
asked for. That part takes an afternoon, and it genuinely works — that's not
the problem.

The problem shows up later, when that plugin meets someone who actually knows
what they're looking at: a marketplace reviewer, an experienced Rust dev, a
server owner who's been burned before. And what they find isn't that the mod
doesn't work. It's things like: a config file that silently corrupts itself
on a bad read. A permission check that's missing on exactly the command where
it matters. Dead code and leftover debug clutter nobody ever cleaned up. A UI
built with patterns that only happen to work by accident. None of that shows
up when you're just testing "does the button do the thing" — all of it shows
up the moment someone experienced actually reads the code.

That gap is the entire reason this document exists. It took months of
building mods that *worked*, submitting them, and getting told — politely or
otherwise — that they weren't actually good, to understand that "make me a
mod" and "make me a mod that would pass review from someone who's shipped a
hundred of these" are two completely different requests. An AI given the
first request will happily give you exactly that: something that works. It
has no way to give you the second thing unless it's actually told what the
second thing requires — because nobody wrote that down anywhere it could
learn it from. Until now, it's been.

### What changed

The rules in Parts B and C aren't theory. Every one of them came from an actual
mistake, an actual rejection, or an actual conversation with someone who
reviews Rust plugins for a living and has genuinely no patience for sloppy
work. Config handling that looked reasonable but wasn't. Migration logic that
was solving a problem that no longer existed. A CUI system built one way when
the working, respected pattern is built a different way. Every rule exists
because something real broke, or someone who actually knew what "good" looked
like said "not like this" and explained why.

This project used **two** AI agents on purpose — **Claude** and **Grok** —
because they do not hit the same walls. Claude's findings are Part B. Grok's
are Part C. Neither file is a rewrite of the other. Together they are the
floor; either one alone is a hole.

That's the part a prompt alone can't give you. You can ask an AI to "write
clean, professional code" all day and it will agree enthusiastically and
still produce something with real problems in it, because "clean and
professional" isn't a fact any AI already has memorized for *this specific
ecosystem* — Rust, Oxide, Carbon, the actual conventions this community has
converged on. It has to be told. Parts B and C are that telling.

### What this document can't do for you

Be honest about this part, because overselling it would just recreate the
exact problem it's trying to fix: **this bible raises the floor. It doesn't
replace testing.**

Handing an AI agent this document before it writes a mod means it starts from
a real, hard-won baseline instead of guessing — no backup-file clutter, no
migration code solving problems you don't have, real permission gates, no
dead code, patterns that have actually been proven to work rather than
patterns that merely compile. That's a genuinely large head start.

What it can't do is catch the thing nobody's hit yet. Some of the most useful
fixes in this whole process came from actually clicking through a live
server, watching something behave strangely, and digging into *why* — not
from a checklist, because the checklist didn't know that problem existed
yet. A document is a snapshot of everything learned up to the point it was
written. The process that produced it — build something, test it for real,
have someone who knows better look at it, fix what's actually wrong — is
still the thing that makes a mod good. This bible just means you're not
starting that process from zero.

### Who this is for

If you're using AI to build Rust plugins and you want the result to be
something you could actually hand to a marketplace reviewer without wincing —
this is the missing piece nobody hands you at the start. Parts B and C are
written to be given directly to your AI agent, in full, before it starts
writing your mod. Paste both in (or point the agent at both raw links), let
it read them, then ask for what you actually want built.

It won't make your AI as good as having an experienced developer looking over
its shoulder in real time. But it will mean the AI knows what that developer
would have said — before you ever have to ask.

---

*Part B (Claude) and Part C (Grok) — the two technical rulesets — follow
separately. Load both.*
