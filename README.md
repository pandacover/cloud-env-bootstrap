# cloud-env-bootstrap

Cursor plugin that helps an agent bootstrap a **correct Cloud Agent environment in one pass**. Cold Cloud Agent runs waste time on discovery, installs, and validation. Builds fix later runs — but only if `.cursor/environment.json`, the install script, and the `AGENTS.md` cloud section are right the first time.

This plugin does **not** take a VM snapshot. Cursor still runs the first Build on its machines. The plugin removes the guess-and-redo loop so that Build is worth keeping.

- [Cloud environment setup](https://cursor.com/docs/cloud-agent/setup)
- [Cloud Agent Builds](https://cursor.com/docs/cloud-agent/builds)

## What you get

| Component | Purpose |
| --- | --- |
| Skill `bootstrap-cloud-env` | Scan the repo, write Build-friendly config, list secret **names** |
| Skill `review-cloud-env` | Audit an existing env for install/idempotency/secrets/validate-path |
| Command `/bootstrap-cloud-env` | Run the bootstrap workflow on the current workspace |
| Rule `prefer-cloud-builds` | When Cloud Agents come up, prefer Builds + tight validation over full CI |

No MCP server in v1.

## Install

Public source: [https://github.com/pandacover/cloud-env-bootstrap](https://github.com/pandacover/cloud-env-bootstrap)

### Local path (testing this repo)

Cursor loads Cursor Plugins from `~/.cursor/plugins/local/<plugin-name>/`. The plugin root must contain `.cursor-plugin/plugin.json` (do not nest an extra directory). **Copy** the files; Cursor skips symlinks whose target is outside `~/.cursor/plugins/local`.

```bash
git clone https://github.com/pandacover/cloud-env-bootstrap.git
cd cloud-env-bootstrap

mkdir -p ~/.cursor/plugins/local/cloud-env-bootstrap
rsync -a --delete \
  --exclude .git \
  ./ ~/.cursor/plugins/local/cloud-env-bootstrap/
```

Then **Developer: Reload Window**. Open **Customize** and confirm the plugin, `/bootstrap-cloud-env`, both skills, and the rule.

On Teams/Enterprise, an admin may need to enable **Allow Local Plugin Imports** (Dashboard → Settings → Security & Identity → Marketplace and Plugins). A marketplace plugin with the same name wins over the local copy.

### Marketplace (later)

When this plugin is listed, install it from **Customize** → Marketplace (or your team marketplace) at user or project scope. Until then, use the GitHub repo: [https://github.com/pandacover/cloud-env-bootstrap](https://github.com/pandacover/cloud-env-bootstrap).

## Use

In Agent chat, in a real app repo (Next/pnpm, Python, Go, …):

```text
/bootstrap-cloud-env
```

Or ask in natural language: “bootstrap the Cloud Agent environment”, “speed up Cloud Agents”, “write `.cursor/environment.json`”.

To audit without writing:

```text
Review this repo's Cloud Agent environment for Build-friendliness.
```

That triggers **review-cloud-env**.

## What bootstrap writes

Only what the scan supports. The skill will not invent a stack.

| File | When |
| --- | --- |
| `.cursor/environment.json` | Always, if a stack is detected. `install` is disk work only (deps, codegen, compile). `start` / `terminals` only for live services (e.g. `sudo service docker start`). |
| `.cursor/install.sh` | When install is more than a short inline command. Idempotent. |
| `.cursor/Dockerfile` | Only if system packages, a pinned Node/Python/Go version, or OS tools are required. **Does not `COPY` the project** — Cursor clones the repo. Paths in `build` are relative to `.cursor`. |
| `AGENTS.md` | Creates or appends **Cursor Cloud specific instructions**: how to run the app, a **short** validate path (`pnpm typecheck` / `pnpm lint` / one package), and “leave full CI to PR CI”. |

If `.cursor/environment.json` already exists, the agent **diffs and explains** instead of silently replacing it.

### Example (Next.js + pnpm)

Detected: `package.json` with `next`, `pnpm-lock.yaml`, scripts `lint` and `typecheck`.

```json
{
  "install": "pnpm install --frozen-lockfile"
}
```

No Dockerfile. No `start`. Validate: `pnpm typecheck` (or `pnpm lint` if typecheck is missing).

With a Dockerfile when Node is pinned:

```json
{
  "build": {
    "dockerfile": "Dockerfile",
    "context": ".."
  },
  "install": "pnpm install --frozen-lockfile"
}
```

## Secrets checklist

Bootstrap prints env var **names** collected from `.env.example` and similar templates. It never writes values.

Add those names in [Cloud Agents Secrets](https://cursor.com/dashboard/cloud-agents) (user, team, or environment-scoped). Team and environment-scoped secrets are available during Builds (private registries). User secrets are injected when an agent starts and are **not** baked into the shared snapshot.

## After the files are committed

1. Push the branch that contains `.cursor/environment.json`.
2. Create or refresh the Cloud Agent environment in the dashboard so a **Build** runs `install` and snapshots the disk.
3. New Cloud Agents start from that Build instead of installing from scratch.

See [setup](https://cursor.com/docs/cloud-agent/setup) and [Builds](https://cursor.com/docs/cloud-agent/builds).

## Layout

```text
.
├── .cursor-plugin/plugin.json
├── README.md
├── LICENSE
├── assets/logo.svg
├── commands/bootstrap-cloud-env.md
├── rules/prefer-cloud-builds.mdc
└── skills/
    ├── bootstrap-cloud-env/SKILL.md
    └── review-cloud-env/SKILL.md
```

## License

MIT
