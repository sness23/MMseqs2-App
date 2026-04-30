# Deploying this repo as a ColabFold-compatible MSA server

This is a deployment note for a specific use of this codebase: hosting the `mmseqs-server` backend as a **ColabFold MSA API server** that clients (e.g. `colabfold_batch --host-url`) can hit over HTTP. The MSA server route is one of the explicitly-maintained modes (per the top-level README), so this is a supported configuration — just not the one most of the upstream README walks through.

For the broader architecture of this repo (Go backend + Vue 2 frontend + Electron, job types, FRONTEND_APP switch), see the top-level `CLAUDE.md`.

## ⚠️ Status note (2026-04-30)

We attempted to self-host an MSA server on a 31 GB workstation Apr 18–29 and abandoned it — the DBs need ~28+ GB available RAM to index and ~23 GB working set per query. **For our actual MSA needs we now use the public `api.colabfold.com`** server (free, no published rate limit, serial queries from one IP). See `~/data/vaults/docs/ARCH-colabfold-msa-server.md` for the full decision context, what's still on disk, and a survey of self-hosting alternatives if we revisit.

This document still describes how to deploy this repo if/when we have suitable hardware. The minimum-RAM gotcha at the bottom is the load-bearing piece.

## TL;DR — how this repo fits in

- Clients use the ColabFold MSA API, POSTing to `/api/ticket/msa` (and `/api/ticket/pair`).
- Those endpoints are implemented in this repo: `backend/msajob.go` + `backend/server.go` + `backend/worker.go` case `MsaJob`.
- The canonical deployment is driven by ColabFold's `MsaServer/` scripts, which download a **prebuilt binary of this repo** from `mmseqs.com/archive/` and invoke it in `-local` mode.
- To run a customized version, `go build` the backend from `backend/` and swap it for the prebuilt binary — same CLI, same config.

## The ColabFold MsaServer convention

Upstream points ops toward [`sokrypton/ColabFold/MsaServer`](https://github.com/sokrypton/ColabFold/tree/main/MsaServer) for MSA-server setup. That directory contains:

- `setup-and-start-local.sh` — downloads MMseqs2 + this repo's backend (prebuilt), downloads the ColabFold DB set (~1 TB), starts the server.
- `setup_databases.sh` (one level up) — does only the DB portion, with fine-grained `UNIREF30_READY` / `COLABDB_READY` / `PDB_READY` sentinels.
- `config.json` — server config scaffold (port, paths, workers, optional auth/rate-limit).
- `systemd-example-mmseqs-server.service` + `restart-systemd.sh` — optional service promotion.

Our live deployment lives at `/Users/sness/data/colabfold-msa-server/` on this workstation. End-to-end ops notes (layout, databases, commands, troubleshooting) are in `~/data/vaults/docs/ARCH-colabfold-msa-server.md`. This doc covers the parts specific to this codebase.

## Three relevant run modes

The `mmseqs-server` binary (built from `backend/`) supports three run modes, selected by flag in `backend/main.go`:

| Flag | When | Job queue |
|------|------|-----------|
| `-server` | Front-end API only; pair with `-worker` | Redis |
| `-worker` | Workers that pick up jobs and shell out to mmseqs | Redis |
| `-local`  | Server + N in-process workers in one process | Filesystem |

The ColabFold MsaServer setup uses **`-local`** — no Redis container, single process, filesystem-backed queue under `paths.results`. This is the recommended shape for a single-host MSA server. The `-server`/`-worker` split is for horizontally scaled search deployments (see `docker-compose/docker-compose.yml`).

## Config specifics for MSA-server mode

ColabFold's `config.json` differs from `docker-compose/config.json` in this repo in three important ways:

1. **`"app": "colabfold"`** — selects the ColabFold codepath (vs. `"mmseqs"` or `"foldseek"`).
2. **`paths.colabfold.{uniref,pdb,environmental,pdb70,pdbdivided,pdbobsolete}`** — ColabFold's DBs are wired in here, not via `.params` files in a databases directory. Consequence: `GET /api/databases` returns `{"databases":[]}`, which confuses people. That endpoint only enumerates classic search DBs; ColabFold DBs are not listed.
3. **`server.pathprefix: "/api/"`** — all endpoints are under `/api/`. Clients must include the prefix.

The commented `ratelimit`, `auth`, and GPU blocks in that `config.json` are directly consumed by this repo's config parser (`backend/config.go`).

## Endpoints the MSA API exposes (from this repo)

Relevant handlers registered in `backend/server.go`:

| Route | Method | Purpose |
|-------|--------|---------|
| `/api/ticket/msa` | POST | Submit an MSA job (`MsaJob` in `backend/msajob.go`) |
| `/api/ticket/pair` | POST | Submit an MSA-pair job (`PairJob` in `backend/pairjob.go`) |
| `/api/ticket/{id}` | GET | Status |
| `/api/ticket/type/{id}` | GET | Job type metadata |
| `/api/result/download/{id}` | GET | tarball of result artifacts (including `.a3m`) |

`colabfold_batch --host-url http://<host>:<port>` speaks exactly this protocol.

## Building a custom `mmseqs-server` from this checkout

If behavior of any job handler needs to change (new endpoints, modified pipeline, extra logging), build the binary from this repo and swap it in:

```bash
cd backend
CGO_ENABLED=0 go build -o mmseqs-server   # matches the release binary's name
# copy into the MsaServer dir
cp mmseqs-server /Users/sness/data/colabfold-msa-server/ColabFold/MsaServer/mmseqs-server/bin/mmseqs-server
# restart
pkill -f mmseqs-server
cd /Users/sness/data/colabfold-msa-server/ColabFold/MsaServer
nohup ./mmseqs-server/bin/mmseqs-server -local -config config.json \
  >> /Users/sness/data/colabfold-msa-server/logs/server.log 2>&1 &
disown
```

The `setup-and-start-local.sh` script checks the binary's `-version` output against a pinned commit and re-downloads on mismatch — run the server by hand (as above) to keep your custom build.

Go ≥ 1.18 is required. Current prebuilt release pins mmseqs commit `05ae20cbc628d9911ad0aa421fba029cc457b76e` and this repo's commit `01365aa4735539ba95b417f73fb5326c77410394`; custom builds don't need to match either, but the DB format must remain compatible.

## Gotchas worth knowing before touching backend code

- **ColabFold job paths** live in `paths.colabfold.*` and are resolved by `backend/config.go`. Adding a new DB-backed feature to ColabFold mode means wiring config there.
- **`MsaJob.Hash()` is content-addressable** — identical query + mode + db set yields the same ticket ID and job directory. Changing the hash function will fork caches.
- **`-local` worker count** is `local.workers` in config (default 1). On 16-core / 31 GB machines, 1 is safe because each MMseqs2 search is already multi-threaded. Increase only after measuring memory headroom.
- **`setup-and-start-local.sh` treats `databases/` dir existence as "DBs ready"** — so if setup fails mid-run, re-running the outer script skips setup entirely and starts the server against half-built DBs. Recover by calling `setup_databases.sh` directly; it honors per-DB `*_READY` sentinels.
- **`mmseqs createindex` wrapper swallows inner exit codes**: when its child `indexdb` dies, the wrapper still returns 0. Setup scripts using `set -e` will *not* catch this — the `*_READY` sentinels get touched on un-indexed DBs and you don't notice until live queries fail. After running setup, manually verify each DB has `db.idx` + `db.idx.dbtype` + `db.idx.index` files (compare against PDB100, which actually does index correctly).
- **`FAST_PREBUILT_DATABASES=1` does NOT mean indexes ship in the tarball.** It only skips the `tsv2exprofiledb` conversion step. `createindex` still has to run and build the actual `.idx*` files. The tarballs ship raw DBs only.
- **Hardware floor for indexing**: env DB requires ~22.7 GB *just for the dbreader buffer* before any work, plus 5–10 GB per-split working memory. Real minimum: **~32 GB available RAM**. UniRef30 is similar. 31 GB-total machines won't cut it. The ColabFold public server runs on 256+ GB hosts.
- **Hardware floor for serving** is also significant — live MSA queries report `Estimated memory consumption: 23G` for the search step. A box that can barely build the indexes will still OOM serving.

## Links

- Deployment ops doc (non-code side): `~/data/vaults/docs/ARCH-colabfold-msa-server.md`
- This repo's architecture: top-level `CLAUDE.md`
- Upstream MsaServer docs: `https://github.com/sokrypton/ColabFold/tree/main/MsaServer`
- Protenix (MSA consumer): `~/github/sness23/Protenix/docs/colabfold_compatible_msa.md`
