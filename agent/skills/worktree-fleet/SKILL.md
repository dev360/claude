---
name: worktree-fleet
description: Bootstrap an orchestrator repo for parallel worktree development — shared Postgres/Redis/NATS, per-worktree databases and namespaces, per-worktree docker-compose projects, and a slot-based port allocator so N feature branches run at once with zero manual port juggling. Wires `mise worktree:new|dev|logs|pull|sync|kill` and per-service `worktree:<service>:{bash,shell,dbshell,logs,migrate,makemigration}` tasks. Enforces conventional-commit branch prefixes at branch-creation time. Invoke when the user says "set up worktrees", "worktree fleet", or asks to standardize a new project's parallel-dev workflow.
---

# worktree-fleet

Run N feature branches in parallel, each with its own database and services, without stepping on each other's ports, containers, or data. The tooling (mise scripts, port registry, shared compose file) lives in its own git repo — the **orchestrator** — while project code stays in git submodules under `main/`. That gives you at least **two git repos**: the orchestrator (which you create) plus one or more project repos pulled in as submodules. Section labels below — "one project" vs "several projects" — count *project repos*, not total git repos.

> You *can* skip the orchestrator repo and leave the top folder untracked. Then only project code is versioned; the mise scripts, `registry.toml`, and compose file live nowhere. Fine solo — but the tooling has to move into an orchestrator repo the moment anyone else adopts the workflow.

## Repo shape

### One project (2 git repos)

```
<project>-orchestrator/           # git repo — holds tooling only
├── .gitignore                    # ignores worktrees/, .venv/, .env.local
├── .mise/scripts/                # worktree_new, registry, slug validator, …
├── mise.toml                     # worktree:* task definitions
├── registry.toml                 # port bases + slot table (source of truth for ports)
├── docker-compose.yaml           # shared postgres/redis/nats
├── main/                         # git submodule → the project repo, tracked to `main`
└── worktrees/                    # .gitignored — ephemeral feature checkouts
    └── <slug>/                   # `git -C main worktree add ../worktrees/<slug> <branch>`
        ├── .env.local            # generated: ports, DB URL, prefixes
        └── … (project files)
```

### Several projects (N+1 git repos)

```
<project>-orchestrator/
├── … (same tooling files as above)
├── services.toml                 # name → remote + framework, one entry per project
├── main/
│   ├── ui/                       # git submodule → the ui repo
│   ├── api/                      # git submodule → the api repo
│   └── …
└── worktrees/
    └── <slug>/
        ├── ui/                   # `git -C main/ui  worktree add ../../worktrees/<slug>/ui  <branch>`
        ├── api/                  # `git -C main/api worktree add ../../worktrees/<slug>/api <branch>`
        └── .env.local
```

## Mise tasks

Invoke from anywhere inside the orchestrator — scripts walk up to find `registry.toml`.

**`[slug]` is optional on every task except `worktree:new` and `worktree:kill`.** Inside a `worktrees/<slug>/` directory (at any depth), the slug is inferred from the `.worktree-slug` sentinel. So from `worktrees/checkout-v2/api/` you can just run `mise run worktree:dev`, `worktree:logs`, etc. Pass the slug explicitly when you're outside a worktree tree. `worktree:kill` demands it so you can't accidentally destroy the one you're standing in.

### Worktree-level

| Task | What it does |
|---|---|
| `worktree:new <branch>` | Validate branch (must match `^(feat\|fix\|docs\|chore\|refactor\|perf\|test\|style\|build\|ci\|revert)/[a-z0-9][a-z0-9-]*$` — hard error otherwise). Derive slug from the part after `/`. Under flock on `registry.toml`, allocate the lowest free slot, `git worktree add` from each `main/` submodule into `worktrees/<slug>[/<service>]`, create Postgres DB `<project>_<slug>`, write `.env.local` (`WORKTREE_SLUG`, `COMPOSE_PROJECT_NAME=wt-<project>-<slug>`, `DATABASE_URL`, `REDIS_KEY_PREFIX=wt:<slug>:`, `NATS_SUBJECT_PREFIX=wt.<slug>.`, `<SERVICE>_PORT=base+slot*10`). Multi-project: launch the sync TUI to pick which projects to check out. |
| `worktree:dev [<slug>]` | `docker compose up -d` for the worktree. |
| `worktree:logs [<slug>]` | Follow logs of every service in the worktree. |
| `worktree:pull [<slug>]` | `git pull --rebase origin main` in each project dir of the worktree. |
| `worktree:sync [<slug>]` | Multi-project only. TUI to add/remove projects; adds `git worktree add` from the matching submodule, removes call `git worktree remove`. |
| `worktree:kill <slug>` | `docker compose down -v`, drop the Postgres DB, `git worktree remove` each project dir, delete the folder, free the slot. Refuses to kill `main`. |
| `worktree:ls` | Print the registry as a table (slug, slot, branch, projects, created-at). |

### Per-service (generated from `services.toml`, or from framework detection in single-project)

| Task | What it does |
|---|---|
| `worktree:<svc>:logs` | `docker compose logs -f <svc>` |
| `worktree:<svc>:bash` | `docker compose exec <svc> bash` |
| `worktree:<svc>:shell` | Django `shell_plus` (or `shell`), Rails `console`, Node `repl`, etc. |
| `worktree:<svc>:dbshell` | Django `dbshell`, otherwise `psql` against the worktree's DB. |
| `worktree:<svc>:migrate` | Django `manage.py migrate` (Django only). |
| `worktree:<svc>:makemigration` | Django `manage.py makemigrations` (Django only). |
| `worktree:<svc>:test` | `pytest` if available, otherwise the framework default. |

## Ports

```
port(service, slot) = base(service) + slot * 10
```

`main` is slot 0; worktree slots 1..N are allocated in creation order and freed slots are reused (lowest free wins). Bases live in `registry.toml`; new service classes take the next `+100` band (so if `web=3000`, `api=8000`, the fourth class goes at `8100`).

**No port literals outside `registry.toml`** — Dockerfiles, compose files, app config, tests, and READMEs read `${*_PORT}` from `.env.local`.

## Shared services are namespaced by slug, not by port

One Postgres, one Redis, one NATS across the orchestrator. Worktrees share them via:

- **Postgres**: per-worktree database `<project>_<slug>`.
- **Redis**: every key wrapped by `REDIS_KEY_PREFIX=wt:<slug>:`. Raw `SET foo bar` in app code will clobber sibling worktrees.
- **NATS**: publishes/subscribes wrapped by `NATS_SUBJECT_PREFIX=wt.<slug>.`. Same failure mode.

The skill sets the env vars; it does not rewrite app code.
