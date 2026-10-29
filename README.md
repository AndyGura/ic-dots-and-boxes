# Dots & Boxes

Classic dots and boxes for two players on the Internet Computer, built on
[duel-game-core](https://github.com/research-ag/duel-core). Take turns
drawing one line between two dots; the fourth side of a box claims it and
earns another turn; most boxes when the sheet is full wins. The table's
creator picks the board: 3×3, 4×4, 5×5 or 6×6 boxes (the rules accept 2
to 8). The page looks like pencil on ruled paper, with synthesized pencil
sounds (mute with the 🔊 button) and animated strokes. A player waiting at
a table can add "Pencil Bot" from the 🤖 dialog: Easy takes a box when it
is offered and otherwise draws at random; Hard takes boxes, plays safe
lines and, when forced, gives up the cheapest chain. Every finished game
rates both seats on the Elo leaderboard (🏆 in the header; bots are rated
per level).

| Canister | Id              | Runs                                  |
| -------- | --------------- | ------------------------------------- |
| backend  | `kozn2-waaaa-aaaau-ahhsq-cai`  | the game (`src/Host.mo`)             |
| frontend | `kh2gg-aiaaa-aaaau-ahhta-cai` | https://kh2gg-aiaaa-aaaau-ahhta-cai.icp0.io/     |
| bot      | `ka3as-nqaaa-aaaau-ahhtq-cai` | "Pencil Bot" (`bot/`), Easy and Hard |

## Layout

- `src/DotsAndBoxesRules.mo` — the rules as a pure `#turnBased` module:
  `State`, `Action`, `View`, `Options`, `spec`, and the `SEMANTICS` text
  the backend serves at `GET /semantics`.
- `src/Host.mo` — the host actor from the skill template (90 s idle
  takeover, 60 s claim window) with the BOTS and LEADERBOARD lines on;
  `src/BotIface.mo` is the inter-canister call to a seated bot.
- `bot/Bot.mo`, `bot/BotLogic.mo` — the bot canister and its two ways
  of playing; `test/Bot.test.mo` checks them.
- `test/RulesUnit.test.mo` — unit checks for the rules.
- `frontend/src/dots-and-boxes-plugin.js` — the `GamePlugin`: Candid
  types, the SVG paper board, hit areas for undrawn lines, `applyLocal`.
- `frontend/src/sounds.js` — Web Audio pencil sounds.
- `frontend/src/app.js`, `index.html`, `style.css` — the shell and the
  paper theme over duel-game-core's default screens, with the 🤖 and 🏆
  overlays; `frontend/src/morph-screen.js` morphs each redraw onto the
  existing DOM (morphdom) so an incoming update never flickers.
- `frontend/play-test.mjs` — headless two-player end-to-end test;
  `frontend/bot-test.mjs` — a game against each bot level plus the
  leaderboard; `frontend/check-upgrade.mjs` — stable-type and Candid
  check before upgrading the live backend.
- `.claude/skills/duel-game-core/` — the framework's skill; read its
  `SKILL.md` before changing anything.

## Build

Tools: `icp` (icp-cli), `mops`, `ic-wasm` (used by icp-cli's Motoko
recipe), Node 22.

```bash
npm install -g @icp-sdk/icp-cli ic-mops
cargo install ic-wasm --locked      # or a release binary from github.com/dfinity/ic-wasm
```

The Motoko package `duel-game-core` is a path dependency on a sibling
checkout of the framework (it is not on mops.one yet):

```bash
git clone https://github.com/research-ag/duel-core.git ../duel-core
mops install
```

```bash
cd frontend && npm install && npm run build && cd ..
icp build backend                   # .icp/cache/artifacts/backend
icp build bot                       # .icp/cache/artifacts/bot
```

## Test

```bash
mops test                           # rules unit tests
icp network start -d                # once; a private local Internet Computer
icp deploy                          # all three canisters, locally; prints the page URL
icp canister call bot register "(principal \"$(icp canister status backend -i)\", \"Pencil Bot\")"
cd frontend && node play-test.mjs   # two headless players finish a game per board size
node bot-test.mjs                   # a game against Easy and one against Hard, then the leaderboard
```

`play-test.mjs <backend-id> --ic` runs the same games against the live
backend. In a Claude Code cloud session the launcher for `icp network
start` must be fetched by tag from `github.com/dfinity/icp-cli-network-launcher`
and set as `ICP_CLI_NETWORK_LAUNCHER_PATH` (the `releases/latest` lookup
is blocked there).

## Update the live game

Everything on mainnet goes through hosty.live, in a deploy session the
owner approves in the hosty.live web app (`hosty_request_deploy_session`).

1. Test locally (above). Then `cd frontend && node check-upgrade.mjs kozn2-waaaa-aaaau-ahhsq-cai`
   must exit 0 (stable types and Candid compatible with the live module).
2. Backend: `icp build backend` and `gzip -9 -k .icp/cache/artifacts/backend`
   (the cloud session's proxy refuses the raw 540 KB upload; the gzipped
   module installs identically), then `hosty_prepare_wasm_upload` with
   `canisterId: kozn2-waaaa-aaaau-ahhsq-cai`, `wasmPath: .icp/cache/artifacts/backend.gz`,
   `mode: upgrade`, `snapshotBeforeUpgrade: true`; run the returned
   `curl` from the project root (absolute file paths are safest); `hosty_wait_for_deployment`. The module
   lands in hosty.live's module store by itself. Roll back with
   `hosty_restore_snapshot` if the upgrade breaks the state.
3. Frontend: `cd frontend && npm run build && zip -r dist.zip dist -x "*.map"`,
   then `hosty_prepare_zip_upload` with `canisterId: kh2gg-aiaaa-aaaau-ahhta-cai`,
   `pureAssets: true`, `outputDir: dist` (the canister keeps
   `PUBLIC_CANISTER_ID:backend = kozn2-waaaa-aaaau-ahhsq-cai`); run the `curl` from
   `frontend/`; `hosty_wait_for_deployment`.
4. Bot (only when `bot/` changed): `icp build bot`, gzip it, then
   `hosty_prepare_wasm_upload` with `canisterId: ka3as-nqaaa-aaaau-ahhtq-cai`,
   `mode: upgrade`; run the `curl`; `hosty_wait_for_deployment`. After a
   REINSTALL of the backend (never after an upgrade) register it again:
   `hosty_call_canister` on the bot, method `register`, argument
   `(principal "kozn2-waaaa-aaaau-ahhsq-cai", "Pencil Bot")`, `update: true`,
   in a deploy session.
5. `cd frontend && node play-test.mjs kozn2-waaaa-aaaau-ahhsq-cai --ic` and
   `node bot-test.mjs kozn2-waaaa-aaaau-ahhsq-cai --ic`, then check
   `https://kozn2-waaaa-aaaau-ahhsq-cai.raw.icp0.io/semantics`.

A change to `View`, `Action`, `Options` or the rules updates `SEMANTICS`
in `src/DotsAndBoxesRules.mo` in the same change. A change to the bot's
play is `bot/BotLogic.mo` plus `test/Bot.test.mo`
(`.claude/skills/duel-game-core/references/canister-player-bots.md`).

The game is listed for players at https://games.ic0.info/ once the
frontend canister is registered there; until then it is online but
unlisted.
