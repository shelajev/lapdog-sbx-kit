# Lapdog for Claude Code — Sandbox Kit

Docker Sandboxes mixin kit that installs [Datadog Lapdog](https://docs.datadoghq.com/llm_observability/lapdog/) and transparently wraps the sandbox's `claude` binary so every prompt, tool call, and permission request is captured. The lapdog dashboard runs inside the sandbox; you reach it from your host browser by publishing a port. Optionally, set the `datadog` secret on your host to also forward spans to Datadog LLM Observability.

## Quick start

```bash
sbx run --kit git+https://github.com/shelajev/lapdog-sbx-kit.git claude .
```

Then, in another terminal on your host, publish the dashboard port and open it:

```bash
sbx ports <sandbox-name> --publish 8127:8127
open http://localhost:8127
```

The kit also nudges Claude (via `CLAUDE.md`) to announce at the start of each session that it is running under lapdog and to print the `sbx ports` command, so you don't need to remember it.

### One-liner: launch + publish dashboard

If you'd rather get the dashboard published from the start, chain the three commands. The first creates the sandbox and starts lapdog at boot, the second publishes the dashboard port to your host, the third attaches the agent:

```bash
sbx create --kit git+https://github.com/shelajev/lapdog-sbx-kit.git --name lapdog-claude claude . && \
  sbx ports lapdog-claude --publish 8127:8127 && \
  sbx run lapdog-claude
```

Open <http://localhost:8127> in your browser while the agent is running. To re-attach later, just `sbx run lapdog-claude` — the published port persists with the sandbox.

## Optional: forward to Datadog LLM Observability

If you want spans to land in your Datadog account in addition to the local dashboard, store your Datadog API key as the `datadog` sandbox secret:

```bash
sbx secret set -g datadog
# paste your DD_API_KEY when prompted
```

The kit declares `DD_API_KEY` as a proxy-managed credential: inside the sandbox `DD_API_KEY` is a sentinel; the real value never enters the VM and is substituted into the `DD-API-KEY` header by the egress proxy. When `DD_API_KEY` is present, the shim invokes `lapdog --forward claude` automatically; otherwise it just runs the local capture agent.

## Named sandbox

```bash
sbx create --name lapdog-claude-current \
  --kit git+https://github.com/shelajev/lapdog-sbx-kit.git claude .

sbx run lapdog-claude-current
```

For custom kits, pass `--kit` again when re-running an existing sandbox if `sbx` does not resolve it automatically.

## How it works

- **Install (once at sandbox creation):**
  - `uv tool install ddapm-test-agent` — pulls the `lapdog` CLI into the agent user's `~/.local/bin`. Python 3 + `uv` are already in the `claude-code-docker` base image.
  - The image ships `claude` at `~/.local/bin/claude`. The kit moves it to `~/.local/bin/claude.real` and drops a tiny shell shim in its place.
- **Shim behaviour** (each time the user runs `claude`):
  - Materializes a `/tmp/.lapdog-shim/claude` symlink → `claude.real`, so lapdog's internal `shutil.which("claude")` resolves to the real binary instead of looping back into the shim.
  - Sets `WEB_UI_PORT=8127` so the dashboard is reachable.
  - Adds `--forward` iff `DD_API_KEY` is present.
  - `exec`s `lapdog [--forward] claude "$@"`.
- **Startup (every boot):**
  - `lapdog start` runs the capture agent (`127.0.0.1:8126`) and the dashboard (`127.0.0.1:8127`) before the user launches `claude`, so port-forwarding works immediately.
  - Idempotent append of a lapdog block to `CLAUDE.md` instructing Claude to announce itself at session start and print the port-forward command.

## Network policy

The kit only allows:

- `pypi.org`, `files.pythonhosted.org` — for the initial `uv tool install`
- `*.datadoghq.com`, `*.datadoghq.eu` — Datadog intake for `--forward`

The `datadog` egress service intercepts requests to known Datadog endpoints and substitutes the real `DD_API_KEY` into the `DD-API-KEY` header.

If you need additional package registries (npm, etc.) or your own services, fork the kit and extend `network.allowedDomains` in `spec.yaml`.

## Smoke test

From your host, after the sandbox is up:

```bash
sbx exec <sandbox-name> -- sh -lc 'lapdog status'
```

Should print `Lapdog running at http://127.0.0.1:8126/info (...)`. Then publish the dashboard port and open it in your browser:

```bash
sbx ports <sandbox-name> --publish 8127:8127
open http://localhost:8127
```

## Local clone

If you clone this repo, `run.sh` runs a named sandbox using the local kit path:

```bash
./run.sh lapdog-claude-current
```

Use any sandbox name as the first argument:

```bash
./run.sh my-sandbox
```

## Caveats

- This kit targets `claude` only. Lapdog also supports `codex` and `pi` (the package ships those wrappers), but the shim here only replaces the `claude` entrypoint.
- The base image's `claude` is renamed once on first install. If you reattach the kit to a sandbox where `claude` was already renamed, the install step is a no-op.
- The dashboard is bound to `127.0.0.1` inside the sandbox; `sbx ports --publish` is required to reach it from the host.

## License

Apache 2.0. See [LICENSE](LICENSE).
