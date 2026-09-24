---
name: bootstrap-cloud-env
description: Bootstrap Cursor Cloud Agent environments. Use when the user wants to set up or speed up Cloud Agents, create or update .cursor/environment.json, write install scripts, prepare Builds, list secrets by name, add AGENTS.md cloud instructions, or says "bootstrap cloud env".
---

# Bootstrap Cloud Agent environment

Write a correct, Build-friendly Cloud Agent environment **in one pass** so the first Build is right. Cursor still runs that Build on its VMs. This skill removes discovery and redo — it does not take the snapshot itself.

Official docs (read when field-level accuracy matters):

- Setup: https://cursor.com/docs/cloud-agent/setup
- Builds: https://cursor.com/docs/cloud-agent/builds
- Schema: https://cursor.com/schemas/environment.schema.json
- Secrets: https://cursor.com/dashboard/cloud-agents (Secrets / environment-scoped secrets)

Do **not** add a `$schema` property to `.cursor/environment.json`; the live schema rejects undeclared fields.

## When to use

- "Set up Cloud Agents", "speed up cloud agents", "bootstrap cloud env"
- Create or fix `.cursor/environment.json`
- Prepare Builds / install scripts / a tight validate path
- Audit secrets **names** that must live in the dashboard, not in git

If the user only wants an audit, use the `review-cloud-env` skill instead of rewriting files.

## Non-negotiables

1. **Do not invent.** Scan the workspace. Only emit install/start/validate commands that match files and scripts you actually found.
2. **Never write secret values.** Checklist names only. Never copy values from `.env`, `.env.local`, CI, or chat into git, `environment.json`, Dockerfiles, or the reply.
3. **Do not silently destroy a working config.** If `.cursor/environment.json` exists, show a diff, explain each change, and preserve unknown-but-valid fields (`snapshot`, `user`, `ports`, `egress*`, `mcpServerAllowlist`, and so on).
4. **`install` is disk-only.** Dependencies, codegen, compile, cache warming. No long-running processes. Builds snapshot disk, not processes.
5. **Leave full CI to PR CI.** Agent validation is typecheck, lint, or one package / affected tests — not the entire suite.

This skill does **not** require an MCP server. After files are committed, the human creates or refreshes the Cloud Agent environment in the dashboard so Builds bake from this config. If Cloud MCP tools (`environment-info`, `trigger-environment-build`, `propose-environment-json`) happen to be available **and** the user asked to test a Build in this session, you may use them after the files are written. Do not block on them.

---

## Workflow

### 1. Scan the workspace

Run a read-only inventory. Prefer glob/grep over guessing. Record **what exists**, not what a typical app might have.

Look for:

| Area | Files / signals |
| --- | --- |
| Node | `package.json`, `pnpm-lock.yaml`, `package-lock.json`, `yarn.lock`, `bun.lock`, `bun.lockb`, `packageManager` / `engines` in `package.json`, `.nvmrc`, `.node-version` |
| Python | `pyproject.toml`, `uv.lock`, `poetry.lock`, `Pipfile`, `Pipfile.lock`, `requirements.txt`, `requirements-dev.txt`, `requirements/*.txt` |
| Go | `go.mod`, `go.sum` |
| Rust | `Cargo.toml`, `Cargo.lock` |
| Monorepo | `pnpm-workspace.yaml`, `turbo.json`, `nx.json`, `lerna.json`, `go.work` |
| Containers | `Dockerfile`, `docker-compose.yml` / `docker-compose.yaml` / `compose.yaml`, `.cursor/Dockerfile` |
| Cursor env | `.cursor/environment.json`, `.cursor/install.sh`, `.cursor/Dockerfile`, `AGENTS.md` |
| Env templates | `.env.example`, `.env.sample`, `.env.template`, `*.env.example`, `.env.local.example` — **names only** |

Read `package.json` `scripts`, `packageManager`, and `engines`. In a monorepo, read the **root** `package.json` plus workspace globs so you do not invent a filter that does not exist.

Skip generated trees (`node_modules`, `.next`, `dist`, `vendor`, `.git`) when searching.

If the repo has no app stack (for example this is a plugin-only repository), say so and stop. Do not fabricate a Node/Python environment.

### 2. Detect the install command

Prefer the **lockfile-accurate** installer. Do not upgrade lockfiles during bootstrap unless the user asked.

| Detected | `install` |
| --- | --- |
| `pnpm-lock.yaml` (or `packageManager` starts with `pnpm`) | `pnpm install --frozen-lockfile` |
| `package-lock.json` | `npm ci` |
| `yarn.lock` + Yarn Berry (`packageManager` `yarn@2+` or `.yarnrc.yml`) | `yarn install --immutable` |
| `yarn.lock` (classic) | `yarn install --frozen-lockfile` |
| `bun.lock` / `bun.lockb` | `bun install --frozen-lockfile` |
| `uv.lock` | `uv sync --frozen` |
| `poetry.lock` | `poetry install --no-interaction` |
| `requirements.txt` (no uv/poetry) | `python -m pip install -r requirements.txt` |
| `go.mod` | `go mod download` |
| `Cargo.toml` | `cargo fetch` |

Append **existing** codegen / Prisma / `prepare` work only when a script already exists (`generate`, `prisma generate`, codegen binaries). Quote the script name you found.

If both a language lockfile and a Node lockfile exist, install **both**, in a stable order (system/lang modules, then JS). Explain the order.

Corepack: if `package.json` has `"packageManager": "pnpm@…"`, you may prefix with `corepack enable` in a Dockerfile or at the start of `install.sh`. Do not assume Corepack is present in the default image unless a Dockerfile installs it.

### 3. Propose, then write

Create or carefully update these files. Show the proposed contents (or a diff) **before** overwriting an existing `.cursor/environment.json`.

#### `.cursor/environment.json`

Minimal default-image shape (most Node/Next apps):

```json
{
  "install": "pnpm install --frozen-lockfile"
}
```

With a Dockerfile (paths are **relative to `.cursor`**):

```json
{
  "build": {
    "dockerfile": "Dockerfile",
    "context": ".."
  },
  "install": "pnpm install --frozen-lockfile && ./custom_script.sh"
}
```

Field rules:

- `install` — idempotent, non-interactive, must **exit**. Disk-preparable work only.
- `start` — per-boot services (example: `sudo service docker start`). Omit when unused.
- `terminals` — named tmux processes (dev servers) only if the agent benefits from them on every run. Prefer documenting `pnpm dev` in AGENTS.md over always starting a server.
- `build.dockerfile` / `build.context` — relative to `.cursor`. `"dockerfile": "Dockerfile"` means `.cursor/Dockerfile`. Omitting `context` defaults to `.cursor`. `"context": ".."` is the repo root. Cursor clones the repo separately; do not rely on the image containing project source.
- `name` — optional human label from the repo/package name.
- `user` — only if the Dockerfile creates a specific user (often `ubuntu`).
- Choose **one** base: Dockerfile `build`, `image`, or `snapshot`. Do not combine them.
- Keep existing `snapshot` IDs unless the user asked to switch to a Dockerfile.

Inline `install` is fine when it is one or two commands. Use `.cursor/install.sh` when there are several steps, conditionals, or codegen.

**Do not put in `install`:** `pnpm dev`, `next start`, `docker compose up`, databases, watchers, `tailscaled`, tunnels, `sleep infinity`.

**Do put in `start` (if needed):** `sudo service docker start`. Put compose/dev servers in `terminals` or AGENTS.md.

#### Optional `.cursor/Dockerfile`

Add a Dockerfile **only** when the default Ubuntu image is missing a required system package, Node/Python/Go version, or OS tool (Docker engine, `build-essential`, a specific Node major from `.nvmrc` / `engines.node`).

Do **not** `COPY` the full project. Cursor clones the repo. The Dockerfile is for OS packages and toolchains; lockfile installs belong in `install`.

Example when Node 22 is pinned:

```dockerfile
FROM node:22-bookworm-slim

RUN apt-get update \
    && apt-get install -y --no-install-recommends git ca-certificates \
    && rm -rf /var/lib/apt/lists/* \
    && corepack enable

USER node
```

If `docker-compose.yml` / `compose.yaml` exists and the agent must run those services, the Dockerfile may install Docker. Pair it with `"start": "sudo service docker start"`. Follow the complex Docker notes in the setup docs (`fuse-overlayfs`, `iptables-legacy`) only when compose/build-in-docker is actually required — do not add that boilerplate for a plain Next app.

#### `.cursor/install.sh` (or inline install)

Idempotent bash. Example for a Next + pnpm repo with Prisma:

```bash
#!/usr/bin/env bash
set -euo pipefail

pnpm install --frozen-lockfile
if grep -q '"prisma"' package.json 2>/dev/null; then
  pnpm exec prisma generate
fi
```

Make the script executable in git (`chmod +x`). Point `install` at `./.cursor/install.sh` or `bash .cursor/install.sh`. The `install` command runs from the **project root**.

#### `AGENTS.md` — section `Cursor Cloud specific instructions`

Create `AGENTS.md` or append this section if missing. Do not delete other sections. Keep it short.

Include:

1. How to run the app (the actual script you found: `pnpm dev`, `uv run …`, `go run .`).
2. A **short** validate path (see presets). Explicitly: leave the full CI suite to PR CI.
3. Pointer that secrets are dashboard-managed, never committed.

Template (replace commands with detected ones):

```markdown
## Cursor Cloud specific instructions

Cloud Agents boot from a Build that already ran `install`. Do not reinstall dependencies unless lockfiles changed.

### Run the app

pnpm dev

### Validate (keep this short)

pnpm typecheck

Run lint or one package / affected tests when you touch a small surface. Leave the full CI suite (e2e, every workspace package) to PR CI.

### Secrets

Do not commit secret values. Configure the names from the Secrets checklist in [Cloud Agents Secrets](https://cursor.com/dashboard/cloud-agents).
```

### 4. Secrets checklist (names only)

Collect variable **names** from `.env.example`, `.env.sample`, `.env.template`, `*.env.example`, and docs that list required env vars (`README` setup sections, `AGENTS.md`). Deduplicate. If keys collide across apps in a monorepo, note the file path and suggest namespacing (`NEXTJS_*`, `CONVEX_*`) without rewriting app code unless asked.

Output a checklist, for example:

```markdown
## Secrets checklist

Configure these **names** in [Cloud Agents Secrets](https://cursor.com/dashboard/cloud-agents) (user, team, or environment-scoped). Do not paste values here.

| Name | Source |
| --- | --- |
| `DATABASE_URL` | `.env.example` |
| `NEXT_PUBLIC_SUPABASE_URL` | `.env.example` |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | `.env.example` |

Team and environment-scoped secrets are available during Builds (private registries). User secrets are injected only when an agent starts — they are not in the shared Build snapshot.

If none found: say so. Do not invent `OPENAI_API_KEY`.
```

Never print values, even if a real `.env` is present. Do not `cat .env`.

### 5. Tell the human the next step

After writing files, the agent cannot bake a dashboard environment by committing alone. Instruct:

1. Commit `.cursor/environment.json`, optional Dockerfile/install script, and `AGENTS.md`.
2. Create or refresh the Cloud Agent environment in the dashboard so Builds use this config.
3. Link:
   - https://cursor.com/docs/cloud-agent/setup
   - https://cursor.com/docs/cloud-agent/builds

Be explicit: **the first Build still runs on Cursor's VMs.** This plugin makes that Build correct; it does not skip it.

### 6. Existing config: diff, don't clobber

If `.cursor/environment.json` already exists:

1. Read it and any referenced Dockerfile / install script.
2. Propose a unified diff (or a before/after JSON block).
3. Explain why each change helps Builds (idempotent install, moved a server out of `install`, lockfile-accurate installer, tighter validate path).
4. Keep working fields you do not understand well enough to replace.
5. Apply only after the explanation. If the current `install` is already lockfile-accurate and disk-only, say "no environment.json changes" and only add AGENTS.md / secrets checklist if those are missing.

---

## Presets (use when detection matches)

### Node / Next (pnpm preferred when the lockfile exists)

- **install:** `pnpm install --frozen-lockfile` when `pnpm-lock.yaml` exists; otherwise npm/yarn/bun per the table. Add `pnpm exec prisma generate` or `pnpm generate` only if those scripts/deps exist.
- **start:** empty unless Docker/compose is required.
- **Dockerfile:** only if Node major is pinned and likely missing, or OS packages are required.
- **validate** (first match that exists in `scripts`):
  1. `pnpm typecheck` (or `tsc --noEmit` / `next typegen` only if that is the project's typecheck)
  2. `pnpm lint`
  3. `pnpm test --filter <pkg>` when this is a pnpm workspace and a relevant package name is known
  4. Never `pnpm test` across the whole monorepo, Playwright, or `pnpm build` of every app as the default agent validate path

Next.js app router: `pnpm dev` is the run command. Do not put `next build` in `install` unless the user needs production-build artifacts on disk for every agent (unusual).

### Python

- **uv:** `uv.lock` or `pyproject.toml` with uv → `uv sync --frozen`
- **Poetry:** `poetry.lock` → `poetry install --no-interaction`
- **pip:** `requirements.txt` → `python -m pip install -r requirements.txt` (include extra requirements files only if they exist)
- **validate:** `uv run ruff check` / `pytest` for a **narrow** path if those tools are in the project; otherwise a single module test. Not the full tox matrix.
- **run:** whatever the README or `scripts` already document (`uv run uvicorn …`, `python -m app`).

### Go

- **install:** `go mod download` and, when the module is a binary the agent will run often, `go build -o /tmp/app ./…` is acceptable for cache warming. Prefer `go build ./...` only if it finishes quickly.
- **validate:** `go test ./<touched-package>` or `go test ./...` when the module is small. Do not add race + every integration test as the default agent path.
- **Dockerfile:** only for a non-default Go version (`go.mod` `go 1.xx` vs image) or extra CGO/system libs.

### Monorepo (pnpm workspace / turbo / nx)

- Install **once at the root** with the frozen lockfile. Do not loop `pnpm install` per package.
- Validate with an affected/filter command you **verified** exists (`pnpm --filter web typecheck`, `pnpm turbo typecheck --filter=web`). If turbo/nx pipelines are unclear, pick one workspace's `typecheck`/`lint` and say so.
- Secrets: list names from every `.env.example`; note collisions.

---

## Response format

Lead with what you detected and what you wrote.

1. **Detected stack** — package manager, lockfile, app framework, monorepo, Docker, existing env files.
2. **Files written or updated** — paths. Include the diff when updating `environment.json`.
3. **Secrets checklist** — names only, plus the dashboard link.
4. **Validate path** now in AGENTS.md.
5. **Next human step** — create/refresh the environment so Builds bake; link setup + Builds docs.

If you could not detect a stack, stop after the inventory. Do not emit a generic `npm install` environment.
