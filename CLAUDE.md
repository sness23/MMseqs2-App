# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A web/desktop frontend + API server for the [MMseqs2](https://github.com/soedinglab/MMseqs2), [Foldseek](https://github.com/steineggerlab/foldseek), [FoldMason](https://github.com/steineggerlab/foldmason), and FoldDisco sequence/structure search tools. Also serves as the MSA API server backing [ColabFold](https://github.com/sokrypton/ColabFold). Upstream lives at [soedinglab/MMseqs2-App](https://github.com/soedinglab/MMseqs2-App); this checkout is a fork.

Per the upstream README: only the ColabFold backend mode and the MMseqs2/Foldseek webservers are actively maintained. The Electron desktop app is on pause.

## The FRONTEND_APP switch

A single codebase produces three differently-branded products selected at build time via the `FRONTEND_APP` env var:

- `mmseqs` — sequence search
- `foldseek` — structure search (also surfaces FoldMason/FoldDisco/interface/multimer routes)
- `foldmason` — MSA-only variant

Webpack (`frontend/webpack.frontend.config.js`) hard-requires `FRONTEND_APP` to be one of those three, and swaps in translated strings from `frontend/assets/*.en_US.po` plus app-specific icons. The Makefile uses it to pick which upstream binary hash to download. Any frontend or build-config change must work for all three unless explicitly scoped.

## Three-tier architecture

```
┌──────────┐    HTTP/JSON    ┌────────┐    Redis queue    ┌──────────┐
│ Frontend │ ─────────────▶  │ Server │ ───────────────▶  │ Worker(s)│
│ (Vue 2)  │                 │ (Go)   │                   │ (Go)     │
└──────────┘                 └────────┘                   └──────────┘
                                                                │
                                                                ▼
                                                      shells out to
                                                      mmseqs/foldseek/…
```

- **Backend** (`backend/`, Go 1.18, module `github.com/soedinglab/MMseqs2-App`): one binary with three run modes selected by flags in `main.go` — `-server`, `-worker`, `-local` (local combines server + N in-process workers + local filesystem queue; used by Electron and dev). `-config <path>` points at a JSON config; a default is written if missing.
- **Job system** (`jobsystem.go`): Redis-backed in server/worker mode, filesystem-backed in local mode. Job types live in `JobType` constants — `search`, `structuresearch`, `complexsearch`, `interfacesearch`, `msa`, `pair`, `foldmasoneasymsa`, `folddisco`, `index`. Each type has its own `*job.go` file implementing the request/validation and corresponding `worker.go` logic to shell out to the MMseqs2/Foldseek/FoldMason/FoldDisco/foldcomp binaries.
- **Frontend** (`frontend/`, Vue 2 + Vuetify 2 + vue-router 3): single-page app. Two entrypoints:
  - `main.js` — the full web/electron app with router (Search, MultimerSearch, InterfaceSearch, FoldMasonSearch, FoldDiscoSearch, Queue, Result*).
  - `main_local.js` + `LOCAL=1` — a standalone static "result viewer" bundle that hydrates a precomputed result blob (used for exporting single results). Webpack branches heavily on `isLocal`.
  - Globals defined via `webpack.DefinePlugin`: `__APP__`, `__ELECTRON__`, `__LOCAL__`, `__TITLE__`, `__CONFIG__`. Check these before assuming an environment.
- **Electron wrapper** (`electron/`): packages the Go backend binary + the MMseqs2/Foldseek CLI as extra files and launches them in `-local` mode on a free port, wiring Basic auth creds between the main process and renderer via `@electron/remote`.

## Common commands

All npm scripts run from the repo root. Most require `FRONTEND_APP` set:

```bash
# Frontend dev server (proxies /api → :3000, so start a backend in -server or -local mode)
FRONTEND_APP=foldseek npm run frontend:dev

# Production frontend bundle → dist/
FRONTEND_APP=mmseqs npm run frontend

# Standalone result-viewer bundle (LOCAL=1)
FRONTEND_APP=foldseek npm run result            # prod
FRONTEND_APP=foldseek npm run result:watch      # dev watch

# Electron dev (hot-reloaded renderer + live main process)
FRONTEND_APP=mmseqs npm run electron:dev

# Electron full packaged build (runs `make all` first to fetch binaries + cross-compile Go)
FRONTEND_APP=mmseqs npm run electron:build

# Backend: plain Go
cd backend && go build ./...
cd backend && go test ./...                     # only decoder_test.go currently exists
cd backend && go test -run TestDecoder ./...    # single test
```

### `make` targets (drive Electron builds)

`make all` (with `FRONTEND_APP` set) cross-compiles the Go backend for linux/mac/windows × amd64/arm64 and downloads the matching MMseqs2/Foldseek binaries from `mmseqs.com/archive/<hash>/…`. The hashes are pinned at the top of the `Makefile` — bump them when upgrading the bundled CLI.

### Server/worker entry points

```bash
# In-process everything (used by Electron and for dev)
./mmseqs-web -local -config config.json -app foldseek

# Production split
./mmseqs-web -server -config config.json -app foldseek
./mmseqs-web -worker -config config.json -app foldseek
```

Config keys are defined by `ConfigRoot` in `backend/config.go`; defaults are written if the file doesn't exist. `-app` selects runtime behavior parallel to `FRONTEND_APP` at build time.

### Docker-compose deployment

`docker-compose/docker-compose.yml` composes four services: `mmseqs-web-redis`, `mmseqs-web-api` (server), `mmseqs-web-worker`, and `mmseqs-web-webserver` (nginx serving the frontend image + proxying to api). `APP`, `PORT`, `DB_PATH`, `JOBS_PATH`, `THREADS_PER_WORKER`, `REPO_OWNER`, `TAG` are read from `.env`. A one-shot `db-setup` profile runs `setup_db.sh`. `docker-compose.local.yml` overrides api/worker to run a single `-local` container with redis/worker stubbed out as `busybox`.

## Things that trip people up

- **Vue 2, not Vue 3.** `vue@^2.7.16`, `vue-router@<4`, `vuetify@<=2.6.12`. Don't import Composition-API-only patterns or Vue 3 libs. The svelte MCP tools listed in your environment do not apply here.
- **`vue-loader/lib/plugin` import path** is the Vue 2 variant — webpack config will break if "upgraded" to the Vue 3 path.
- **axios is pinned at `^0.26.1`** (pre-1.0). API quirks differ from modern axios.
- **`FRONTEND_APP` is mandatory.** Webpack refuses to build without it; `make` silently produces the wrong artifacts if it's unset.
- **Two frontend bundles, shared components.** When editing a `Result*.vue`, verify both the routed `main.js` path and the `main_local.js` (`LOCAL=1`) path still work — the latter has no router and loads a pre-baked result payload.
- **Go module is `github.com/soedinglab/MMseqs2-App`** — imports must use that path, not the fork path.
- **Structure viewers** use NGL (`ngl@2.0.1`), TM-align via WASM (`tmalign-wasm`), and Pulchra WASM. TM-align runs inside a web worker (`TMAlignWorker.js`) and has its own webpack splitChunk in `LOCAL` mode.
- **Binary resources directory** (`resources/<os>/<arch>/`) is generated by `make` and gitignored. A missing `resources/` is normal on a fresh clone.

## Layout pointers

- `backend/*job.go` — one file per job type; start here when adding/modifying a search flavor.
- `backend/worker*.go` — platform-specific worker process control (`worker_linux.go`, `worker_unix.go`, `worker_windows.go`).
- `backend/databases.go`, `backend/dbreader.go` — database metadata + index management.
- `frontend/Search.vue` / `MultimerSearch.vue` / `InterfaceSearch.vue` / `FoldMasonSearch.vue` / `FoldDiscoSearch.vue` — the five entry UIs.
- `frontend/Result*.vue` + `ResultMixin.vue` — result rendering; mixins hold the shared data-loading logic.
- `frontend/lib/` — generic helpers (axios compression, color scales, debounce/throttle, identicon, po-loader, simple portal, blob DB, zip).
- `docs/api.html` + `docs/api_example.py` — the public HTTP API reference; regenerate/update these when adding endpoints in `server.go`.
