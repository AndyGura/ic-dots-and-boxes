# Dots & Boxes

A 2-player game on duel-game-core
(https://github.com/research-ag/duel-core): `src/DotsAndBoxesRules.mo`
holds the rules, `src/Host.mo` the host actor, `frontend/` the page.
Read `.claude/skills/duel-game-core/SKILL.md` before changing anything;
for a change to the live game, its `references/changing-a-live-game.md`.

## Live

| Canister | Id                            | Runs                                          |
| -------- | ----------------------------- | --------------------------------------------- |
| backend  | `kozn2-waaaa-aaaau-ahhsq-cai` | the game; the module store has it (gzipped)   |
| frontend | `kh2gg-aiaaa-aaaau-ahhta-cai` | https://kh2gg-aiaaa-aaaau-ahhta-cai.icp0.io/  |

Deployed through hosty.live (`going-live.md`, setup 1: a Claude Code
cloud session; the user approves a deploy session in the hosty.live web
app). The README has the exact upgrade steps.

## What the game runs

- Mode: `#turnBased` (a seat that closes a box moves again).
- Options: `{ size : Nat }`, boxes per side, 2..8; the page offers 3–6.
- Bot support (the Host's and the page's BOTS lines): off.
- Leaderboard (LEADERBOARD lines): off.
- A bot canister: none.
- duel-game-core: a path dependency on `../duel-core/backend` (mops;
  clone https://github.com/research-ag/duel-core next to this repo) and
  `@research-ag/duel-game-core` 0.4.1 (npm).

## Working here

- The default branch is the game: merge finished work there, and
  update this file whenever a canister or a feature changes.
- `mops test`; `cd frontend && npm run build && node play-test.mjs`.
- Before upgrading the backend: `cd frontend && node check-upgrade.mjs
kozn2-waaaa-aaaau-ahhsq-cai` must exit 0.
- A change to `View`, `Action`, `Options` or the rules updates
  `SEMANTICS` in the same change.
- In a cloud session: `icp network start` needs the launcher fetched by
  tag (`git ls-remote --tags https://github.com/dfinity/icp-cli-network-launcher`)
  into `/opt/icp-launcher` and `ICP_CLI_NETWORK_LAUNCHER_PATH` set;
  `ic-wasm` comes from `cargo install ic-wasm --locked`.
