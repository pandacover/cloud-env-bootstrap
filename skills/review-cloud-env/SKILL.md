---
name: review-cloud-env
description: Audit an existing Cursor Cloud Agent environment for Build-friendliness. Use when reviewing .cursor/environment.json, install scripts, Dockerfiles, or AGENTS.md cloud instructions; when Builds are slow or fail; or when checking that install is idempotent and validation is tighter than full CI.
---

# Review Cloud Agent environment

Audit the current workspace's Cloud Agent config. Report **pass/fail** with prioritized fixes. Do not rewrite files unless the user asks you to apply the fixes (then follow `bootstrap-cloud-env` for the write path).

Docs:

- Setup: https://cursor.com/docs/cloud-agent/setup
- Builds: https://cursor.com/docs/cloud-agent/builds
- Schema: https://cursor.com/schemas/environment.schema.json

## Scope

Inspect, if present:

- `.cursor/environment.json`
- `.cursor/Dockerfile` or repo-root `Dockerfile` referenced by `build.dockerfile`
- Install script referenced by `install` (`.cursor/install.sh`, `scripts/*.sh`, inline string)
- `AGENTS.md` section titled `Cursor Cloud specific instructions`
- `.env.example` / env templates (names only — never values)
- Lockfiles and `package.json` scripts, to judge whether `install` and validate match reality

If `.cursor/environment.json` is missing, fail the audit with: "No repo-managed environment; run `/bootstrap-cloud-env` or the bootstrap-cloud-env skill."

## Checks

Grade each check **PASS**, **FAIL**, or **N/A**. A single FAIL makes the overall review **FAIL**. List FAIL items first, highest impact first.

### 1. Install is complete and idempotent

**PASS when:**

- `install` uses the lockfile-accurate installer (`pnpm install --frozen-lockfile`, `npm ci`, `yarn install --immutable` / `--frozen-lockfile`, `bun install --frozen-lockfile`, `uv sync --frozen`, `poetry install --no-interaction`, `go mod download`, …).
- Codegen that the app needs to compile/typecheck (`prisma generate`, `pnpm generate`) is in `install` **if** those scripts exist.
- The command is non-interactive (`set -euo pipefail` for scripts) and safe to re-run on already-prepared disk.
- It does not rewrite lockfiles or run broad upgrades (`pnpm update`, `npm install` without ci/frozen when a lockfile exists).

**FAIL when:** install is missing, uses a floating installer that will mutate the lockfile, skips required generate steps, or is clearly incomplete vs the stack you detected.

### 2. No long-running processes in install

**PASS when:** `install` exits. Dev servers, `docker compose up`, databases, watchers, tunnels, and `sleep infinity` are absent.

**FAIL when:** `install` starts Docker daemon, compose, `pnpm dev`, `next start`, `uvicorn --reload`, or anything that must still be running after the Build snapshot. Builds preserve disk, not processes. Those belong in `start` or `terminals` (or AGENTS.md if optional).

Note: `sudo service docker start` belongs in **`start`**, not `install`.

### 3. Secrets are not hardcoded

**PASS when:** `environment.json`, Dockerfiles, and install scripts contain no tokens, passwords, private keys, or `.env` values. Env templates are names-only references.

**FAIL when:** a secret value or `ENV KEY=sk-…` appears in committed env config. Do **not** reprint the value in the review — cite the file and key name only.

Also fail if the repo clearly requires env vars (`.env.example`) but AGENTS.md / the reply never points at [Cloud Agents Secrets](https://cursor.com/dashboard/cloud-agents).

### 4. AGENTS.md has a tight validate path

**PASS when:** `AGENTS.md` has a section **`Cursor Cloud specific instructions`** that tells the agent:

- How to run the app
- A **short** validate command that exists in the repo (`pnpm typecheck`, `pnpm lint`, `pnpm test --filter …`, `uv run ruff`, `go test ./pkg`)
- To leave the full CI suite to PR CI

**FAIL when:** the section is missing, the validate command does not exist as a script, or it tells the agent to run full e2e / every workspace package / the entire CI workflow on every change.

### 5. Dockerfile and paths (if used)

**PASS when:** `build.dockerfile` / `context` are relative to `.cursor`; the Dockerfile does **not** `COPY` the full project; OS/toolchain belongs in the image and lockfile installs belong in `install`; only one of `build` / `image` / `snapshot` is used.

**FAIL when:** dockerfile path would miss `.cursor/Dockerfile`, context copies the app source as the primary workspace, or `build` is combined with `snapshot`/`image` without explanation.

**N/A** when there is no Dockerfile — default image + `install` is valid.

### 6. Start vs install vs terminals

**PASS when:** live services are in `start`/`terminals`; `install` is disk work.

**FAIL when:** `start` re-runs full `pnpm install` on every boot, or `terminals` is the only place dependencies are installed.

### 7. Detected stack matches config

**PASS when:** the installer matches the lockfile you found (pnpm lockfile → pnpm frozen, and so on).

**FAIL when:** `npm install` is configured but `pnpm-lock.yaml` is the source of truth, or Python/Go deps are ignored in a polyglot repo that needs them.

## Output format

```markdown
# Cloud env review: PASS | FAIL

## Summary
One paragraph: Build-ready or not, and the single most important fix.

## Checks
| Check | Result | Evidence |
| --- | --- | --- |
| Install complete + idempotent | PASS/FAIL | … |
| No long-running processes in install | PASS/FAIL | … |
| Secrets not hardcoded | PASS/FAIL | … |
| AGENTS.md tight validate path | PASS/FAIL | … |
| Dockerfile / paths | PASS/FAIL/N/A | … |
| start vs install | PASS/FAIL/N/A | … |
| Stack matches config | PASS/FAIL | … |

## Prioritized fixes
1. (blocking) …
2. (should) …
3. (nice) …

## Secrets names seen (no values)
- `NAME` — from `.env.example`

## Next step
If FAIL: apply fixes with the bootstrap-cloud-env skill, commit, then create or refresh the Cloud Agent environment so a Build bakes the new install.
Docs: https://cursor.com/docs/cloud-agent/setup · https://cursor.com/docs/cloud-agent/builds
```

Do not propose a full rewrite when one line in `install` is wrong. Prefer the smallest change that makes the next Build correct.
