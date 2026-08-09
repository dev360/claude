---
name: worktree-fleet
description: Bootstrap an orchestrator repo for parallel worktree development — shared Postgres/Redis/NATS, per-worktree databases and namespaces, per-worktree docker-compose projects, and a slot-based port allocator so N feature branches run at once with zero manual port juggling. Wires `mise worktree:new|dev|logs|pull|sync|kill` and per-service `worktree:<service>:{bash,shell,dbshell,logs,migrate,makemigration}` tasks. Enforces conventional-commit branch prefixes at branch-creation time. Invoke when the user says "set up worktrees", "worktree fleet", or asks to standardize a new project's parallel-dev workflow.
---

# worktree-fleet

Run N feature branches in parallel, each with its own database and services, without stepping on each other's ports, containers, or data. The tooling (mise scripts, port registry, shared compose file) lives in its own git repo — the **orchestrator** — while the actual project code stays in git submodules. That separation lets a team share the workflow and rebase the tooling independently of the projects it drives.

**You always end up with at least two git repos**: the orchestrator repo (which you create) plus one or more project repos (which already exist and get pulled in as submodules). The section labels below — "one project" vs "several projects" — refer to *how many project repos* the orchestrator manages, not to the total git-repo count. In both cases the orchestrator itself is its own git repo.

> Skip the orchestrator repo (leave the top folder as an untracked scratch dir) only when it's just for you and you have no intention of sharing the workflow. In that case the project repo is the only git repo involved.

## Repo shape

### One project (orchestrator + one project repo = 2 git repos)

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

### Several projects (orchestrator + N project repos = N+1 git repos)

```
<project>-orchestrator/
├── .gitignore
├── .mise/scripts/
├── mise.toml
├── registry.toml
├── services.toml                 # only when there are multiple projects: name → remote + framework
├── docker-compose.yaml
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

Two rules that make the shape work:

- **Tooling and project code do not share a git repo.** Registry, compose file, mise scripts — orchestrator. App code — submodules.
- **No port literals outside `registry.toml`.** Compose files, Dockerfiles, app config, tests, and READMEs all read `${*_PORT}` from `.env.local`.

## Mise tasks

All tasks are invoked from anywhere inside the orchestrator (scripts walk up to find `registry.toml`).

### Worktree-level

| Task | What it does |
|---|---|
| `worktree:new <branch>` | Validate branch (must match `^(feat\|fix\|docs\|chore\|refactor\|perf\|test\|style\|build\|ci\|revert)/[a-z0-9][a-z0-9-]*$` — hard error otherwise). Derive slug from the part after `/`. Take flock on `registry.toml` and allocate the lowest free slot. Run `git worktree add` from inside each submodule under `main/` targeting `worktrees/<slug>[/<service>]`. Create per-worktree Postgres DB `<project>_<slug>`. Write `.env.local` with `WORKTREE_SLUG`, `COMPOSE_PROJECT_NAME=wt-<project>-<slug>`, `DATABASE_URL`, `REDIS_KEY_PREFIX=wt:<slug>:`, `NATS_SUBJECT_PREFIX=wt.<slug>.`, and `<SERVICE>_PORT=base+slot*10` per service class. When several projects are attached, launch the sync TUI to pick which of them to check out into this worktree. |
| `worktree:dev [<slug>]` | `docker compose up -d` for the worktree. No arg = detect the slug from CWD via the `.worktree-slug` sentinel. |
| `worktree:logs [<slug>]` | Follow logs of every service in the worktree. |
| `worktree:pull [<slug>]` | `git pull --rebase origin main` in each service directory of the worktree. |
| `worktree:sync [<slug>]` | Only meaningful when several projects are attached. TUI to add/remove projects from an existing worktree; adds do `git worktree add` from the matching submodule, removes call `git worktree remove`. |
| `worktree:kill <slug>` | `docker compose down -v` for the worktree, drop the per-worktree Postgres DB, `git worktree remove` each service dir, delete the folder, free the slot in `registry.toml`. Refuses to kill `main`. |
| `worktree:ls` | Print the registry as a table (slug, slot, branch, services, created-at). |

### Per-service (generated from `services.toml`, or from framework detection when there's only one project)

| Task | What it does |
|---|---|
| `worktree:<svc>:logs` | `docker compose logs -f <svc>` |
| `worktree:<svc>:bash` | `docker compose exec <svc> bash` |
| `worktree:<svc>:shell` | Django `shell_plus` (or `shell` if `django-extensions` isn't present), Rails `console`, Node `repl`, etc. |
| `worktree:<svc>:dbshell` | Django `dbshell`, otherwise `psql` against the worktree's DB. |
| `worktree:<svc>:migrate` | Django `manage.py migrate` (skipped for non-Django services). |
| `worktree:<svc>:makemigration` | Django `manage.py makemigrations` (skipped for non-Django services). |
| `worktree:<svc>:test` | `pytest` if available, otherwise the framework default. |

## Port math

```
port(service, slot) = base(service) + slot * 10
```

`main` is slot 0. Worktree slots 1..N are allocated in creation order; freed slots are reused (lowest free wins). Bases live in `registry.toml`; new service classes take the next `+100` band (so if `web=3000`, `api=8000`, the fourth class goes at `8100`).

## Shared services are namespaced by slug, not by port

One Postgres, one Redis, one NATS across the whole orchestrator. Worktrees don't get their own — they share via:

- **Postgres**: per-worktree database `<project>_<slug>`.
- **Redis**: every key wrapped by `REDIS_KEY_PREFIX=wt:<slug>:`. Raw `SET foo bar` in app code is a bug — worktrees will clobber each other.
- **NATS**: publishes/subscribes wrapped by `NATS_SUBJECT_PREFIX=wt.<slug>.`. Same failure mode.

The skill sets the env vars; it does not rewrite app code.
