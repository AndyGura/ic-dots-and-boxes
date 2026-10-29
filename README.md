# Dots & Boxes

Classic dots and boxes for two players on the Internet Computer, built on
[duel-game-core](https://github.com/research-ag/duel-core). Take turns
drawing one line between two dots; the fourth side of a box claims it and
earns another turn; most boxes when the sheet is full wins. The table's
creator picks the board: 3×3, 4×4, 5×5 or 6×6 boxes (the rules accept 2
to 8). The page looks like pencil on ruled paper, with synthesized pencil
sounds (mute with the 🔊 button) and animated strokes.

| Canister | Id              | Runs                                  |
| -------- | --------------- | ------------------------------------- |
| backend  | `__BACKEND_ID__`  | the game (`src/Host.mo`)             |
| frontend | `__FRONTEND_ID__` | https://__FRONTEND_ID__.icp0.io/     |

## Layout

- `src/DotsAndBoxesRules.mo` — the rules as a pure `#turnBased` module:
  `State`, `Action`, `View`, `Options`, `spec`, and the `SEMANTICS` text
  the backend serves at `GET /semantics`.
- `src/Host.mo` — the host actor from the skill template (90 s idle
  takeover, 60 s claim window).
- `test/RulesUnit.test.mo` — unit checks for the rules.
- `frontend/src/dots-and-boxes-plugin.js` — the `GamePlugin`: Candid
  types, the SVG paper board, hit areas for undrawn lines, `applyLocal`.
- `frontend/src/sounds.js` — Web Audio pencil sounds.
- `frontend/src/app.js`, `index.html`, `style.css` — the shell and the
  paper theme over duel-game-core's default screens.
- `frontend/play-test.mjs` — headless end-to-end test; `frontend/check-upgrade.mjs`
  — stable-type and Candid check before upgrading the live backend.
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
```

## Test

```bash
mops test                           # rules unit tests
icp network start -d                # once; a private local Internet Computer
icp deploy                          # both canisters, locally; prints the page URL
cd frontend && node play-test.mjs   # two headless players finish a game per board size
```

`play-test.mjs <backend-id> --ic` runs the same games against the live
backend. In a Claude Code cloud session the launcher for `icp network
start` must be fetched by tag from `github.com/dfinity/icp-cli-network-launcher`
and set as `ICP_CLI_NETWORK_LAUNCHER_PATH` (the `releases/latest` lookup
is blocked there).

## Update the live game

Everything on mainnet goes through hosty.live, in a deploy session the
owner approves in the hosty.live web app (`hosty_request_deploy_session`).

1. Test locally (above). Then `cd frontend && node check-upgrade.mjs __BACKEND_ID__`
   must exit 0 (stable types and Candid compatible with the live module).
2. Backend: `icp build backend`, then `hosty_prepare_wasm_upload` with
   `canisterId: __BACKEND_ID__`, `wasmPath: .icp/cache/artifacts/backend`,
   `mode: upgrade`, `snapshotBeforeUpgrade: true`; run the returned
   `curl` from the project root; `hosty_wait_for_deployment`. The module
   lands in hosty.live's module store by itself. Roll back with
   `hosty_restore_snapshot` if the upgrade breaks the state.
3. Frontend: `cd frontend && npm run build && zip -r dist.zip dist -x "*.map"`,
   then `hosty_prepare_zip_upload` with `canisterId: __FRONTEND_ID__`,
   `pureAssets: true`, `outputDir: dist` (the canister keeps
   `PUBLIC_CANISTER_ID:backend = __BACKEND_ID__`); run the `curl` from
   `frontend/`; `hosty_wait_for_deployment`.
4. `cd frontend && node play-test.mjs __BACKEND_ID__ --ic`, then check
   `https://__BACKEND_ID__.raw.icp0.io/semantics`.

A change to `View`, `Action`, `Options` or the rules updates `SEMANTICS`
in `src/DotsAndBoxesRules.mo` in the same change. Bot support and a
leaderboard are the lines marked BOTS / LEADERBOARD in `src/Host.mo`,
`frontend/src/app.js` and `frontend/src/index.html`
(`.claude/skills/duel-game-core/references/changing-a-live-game.md`).

The game is listed for players at https://games.ic0.info/ once the
frontend canister is registered there; until then it is online but
unlisted.
