---
name: bootstrap-cloud-env
description: Scan this workspace and write a Build-friendly Cursor Cloud Agent environment (environment.json, install script, AGENTS.md cloud section, secrets checklist).
---

# /bootstrap-cloud-env

Run the **bootstrap-cloud-env** skill on the **current workspace**.

Follow `skills/bootstrap-cloud-env/SKILL.md` end to end. Do not skip the scan. Do not invent scripts, lockfiles, or secret names.

## Do this now

1. Inventory the repo: package managers and lockfiles, Python/Go/Rust, monorepo markers, Dockerfiles/compose, existing `.cursor/environment.json`, `AGENTS.md`, and `.env.example` files (**names only**).
2. Choose a lockfile-accurate `install` command. Put disk work in `install`; put Docker/dev servers in `start` / `terminals` / AGENTS.md — never long-running processes in `install`.
3. Propose then write:
   - `.cursor/environment.json`
   - `.cursor/install.sh` only if install is more than a short inline command
   - `.cursor/Dockerfile` only if OS packages or a pinned runtime are required (do **not** `COPY` the project)
   - `AGENTS.md` section titled `Cursor Cloud specific instructions` with run + short validate + "leave full CI to PR CI"
4. If `.cursor/environment.json` already exists, show a diff and explain. Do not silently replace a working config.
5. Print a **Secrets checklist** of env var **names** only. Point at https://cursor.com/dashboard/cloud-agents for Secrets / environment-scoped secrets. Never write values.
6. Tell the user the next human step: create or refresh the Cloud Agent environment in the dashboard so Builds bake from this config.

Links to include:

- https://cursor.com/docs/cloud-agent/setup
- https://cursor.com/docs/cloud-agent/builds

This command does not take a VM snapshot. Cursor still runs the first Build on its machines; bootstrap makes that Build correct.
