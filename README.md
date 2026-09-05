# The AI Rust-Modding Bible

A beginner's guide to AI-assisted Rust modding — written by a beginner, for
beginners. The technical ruleset was learned by using **two** AI agents on
the same real plugins. Their discoveries are different. The floor is the union.

- **[PART-A.md](PART-A.md)** — for you. Why a working mod isn't the same
  thing as a good mod, and what this project actually is.
- **[PART-B.md](PART-B.md)** — **Claude's** agent ruleset. Config, CUI echo,
  inventory/loot desync, skeleton, submission screenshots.
- **[PART-C.md](PART-C.md)** — **Grok's** agent ruleset (xAI). Leftover-field
  migrate, Carbon compile hangs, object-return hooks, CUI leaks, DLC/skins
  TOS, chat.say is not a function test, and the rest Grok actually hit live.
- **[PART-D.md](PART-D.md)** — **the team system**, written jointly by Claude
  and Grok and changed only by mutual agreement: the three seats and what
  each can reach, the room, the handshake, the deploy recipe and the mirror,
  what counts as proof, live testing on a production server, working while
  the director sleeps, leading without direction, and the rules the two
  agents learned together. Section 10 is the verification-run playbook
  (formerly its own Part D file, now a pointer).
- **[MODDING-QA-CHECKLIST.md](MODDING-QA-CHECKLIST.md)** — the real, living
  12-category checklist Part D points at. Built from real found-and-fixed
  bugs against real plugins, not a generic template — the "Log of additions"
  at the bottom is where the reasoning and the actual cases live.

## For your AI agent

Load **both** technical parts before asking the agent to build or edit a
Rust plugin:

`
https://raw.githubusercontent.com/MickMickMick73/AI-Rust-Modding-Bible/main/PART-B.md
https://raw.githubusercontent.com/MickMickMick73/AI-Rust-Modding-Bible/main/PART-C.md
`

Part B is not a substitute for Part C. Part C is not a rewrite of Part B.

## This is a living document

Parts B and C each end with how to extend them. If your own AI-modding
session turns up something new — verified against real behavior, not
guessed — a pull request adding it back is exactly what this repo is for.
Put Claude findings in Part B and Grok findings in Part C when you know
which agent hit them. Do not silently overwrite the other file.

## License

MIT — see [LICENSE](LICENSE). Fork it, extend it, build on it.
