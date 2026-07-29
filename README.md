# sbx kits for Grafana

This is a standalone [Docker Sandboxes](https://docs.docker.com/ai/sandboxes/) kit
(`kind: mixin`) that gives any sandbox agent hands-on access to
[Grafana](https://grafana.com/) — dashboards, datasources, and Prometheus/Loki
queries — through the official
[Grafana MCP server](https://github.com/grafana/mcp-grafana) (`mcp-grafana`),
plus the [`grafana-client`](https://pypi.org/project/grafana-client/) Python
library for scripting.

A local Grafana on the host is the zero-config default. It works with no Grafana
account and no API token; the target is swappable to Grafana Cloud or any
self-hosted instance. See [providers/](./providers/) for copy-paste config.

## What the kit does

Layered onto an agent, the mixin does four observable things:

1. Installs the pinned `mcp-grafana` v1.0.0 binary at `~/.local/bin/mcp-grafana`.
2. Installs the `grafana-client` Python library.
3. Sets `GRAFANA_URL` (and, for cloud/oss, wires the service-account token
   through the sbx proxy) and writes a portable MCP definition to
   `~/.grafana/mcp.json`.
4. Registers the MCP server with the Claude agent (best-effort) and injects a
   memory note so the agent knows the tooling is there.

## Prerequisites

### 0. Login to Docker Hub

```console
sbx login
```

### 1. Bring up a Grafana for the default (local) target

Run the bundled stack on your **host** — it starts Grafana on
`http://localhost:3000` (anonymous Admin) with a Prometheus datasource, so the
sandbox can reach it over `host.docker.internal:3000` with no token:

```console
docker compose -f compose/docker-compose.yml up -d
```

Already have a Grafana on port 3000? Skip this — the kit only needs something
answering there. (Cloud / self-hosted users skip it too; see step 2.)

### 2. Store a token (cloud / self-hosted only)

The local default needs no token. For Grafana Cloud or a self-hosted instance,
create a [service-account token](https://grafana.com/docs/grafana/latest/administration/service-accounts/)
and store it once with sbx's secret manager. The token is never baked into the
kit; the sbx proxy injects it into outbound requests at runtime (`sbx run` has no
`-e` flag):

```console
echo "$GRAFANA_TOKEN" | sbx secret set -g grafana   # -g = all sandboxes
```

Confirm it's stored:

```console
sbx secret ls
```

### 3. Launch the sandbox with the kit

Each target is published as its own image tag — pick the one matching your setup:

```console
# Local Grafana on the host (default, no token) — :latest is the same as :local
sbx run --kit docker.io/ajeetraina777/sbx-grafana-kits:latest claude

# Grafana Cloud — store the token, set your stack URL (see providers/cloud.md)
sbx secret set -g grafana && sbx run --kit docker.io/ajeetraina777/sbx-grafana-kits:cloud claude

# Self-hosted — edit kits/oss/spec.yaml with your host, then run from the clone
sbx secret set -g grafana && sbx run --kit ./kits/oss claude
```

Or straight from this repo over git (uses the default local target):

```console
sbx run --kit "git+https://github.com/ajeetraina/sbx-kits-grafana.git" claude
```

Or from a local clone (the kit lives at the repo root):

```console
git clone https://github.com/ajeetraina/sbx-kits-grafana.git
sbx run --kit ./sbx-kits-grafana/ claude
```

#### Choosing the agent

The trailing argument (`claude` above) is the **coding agent** that runs inside
the sandbox. It is a separate axis from the target kit tag. The tag
(`:local`, `:cloud`, `:oss`) decides which Grafana the tooling points at; the
agent decides which assistant you interact with. Any supported agent pairs with
any tag. `sbx run --help` lists them:

```
claude, claude-bedrock, codex, copilot, cursor, docker-agent, droid, gemini, kiro, opencode, shell
```

So you can swap `claude` for any of these, e.g. Codex against a local Grafana:

```console
sbx run --kit docker.io/ajeetraina777/sbx-grafana-kits:latest codex
```

Arguments meant for the agent itself go after a `--` separator, e.g.
`sbx run --kit ...:latest codex -- --help`.

### 4. Confirm the kit installed correctly

Inside the agent session, use `!` shell escapes to prove the mixin is really
inside.

**4a. The MCP server binary is installed:**

```console
!/home/agent/.local/bin/mcp-grafana --help | head
```

**4b. The mixin's env is present** (a fingerprint that the kit wired things up):

```console
!env | grep -E 'GRAFANA_URL|NO_PROXY'
```

Expect `GRAFANA_URL=http://host.docker.internal:3000` for the local kit.

**4c. The portable MCP definition the kit wrote exists:**

```console
!cat /home/agent/.grafana/mcp.json
```

**4d. End-to-end functional proof** — reach Grafana and read back its health,
datasources, and dashboards through `grafana-client`. This single command
exercises the Python lib, the env vars, and the connection to Grafana, so if you
only run one check, run this one:

```console
!python3 ~/runbooks/grafana_report.py
```

Expect the instance health, the `Prometheus` datasource, and any dashboards.

### 5. Use the MCP server from the agent

On the Claude agent the server is registered automatically. Confirm and use it:

```console
!claude mcp list
```

Then just ask the agent, e.g. *"list my Grafana datasources"* or *"run the PromQL
`up` and summarize"* — it drives the `mcp-grafana` tools. For other agents,
import `~/.grafana/mcp.json` or run
`claude mcp add grafana -- ~/.local/bin/mcp-grafana`.

### 6. Try a runbook

The kit ships runnable demos under `~/runbooks/`. They are plain files under
[`files/home/runbooks/`](./files/home/runbooks/) in this repo (the
[sbx-kits-contrib][contrib] `files/home/` convention — everything under it is
mirrored into `/home/agent/`), **not** hard-coded into `spec.yaml`:

```console
!python3 ~/runbooks/grafana_report.py
```

To add a runbook, drop a `*.py` in `files/home/runbooks/` — it ships
automatically, no `spec.yaml` change.

[contrib]: https://github.com/docker/sbx-kits-contrib

## Switching the Grafana target

| Target | Runs where | Credential | Doc |
|---|---|---|---|
| local (default) | Host `host.docker.internal:3000` | none (anonymous) | [providers/local.md](./providers/local.md) |
| cloud | Grafana Cloud `*.grafana.net` | `sbx secret set -g grafana` | [providers/cloud.md](./providers/cloud.md) |
| oss | Any self-hosted Grafana | `sbx secret set -g grafana` | [providers/oss.md](./providers/oss.md) |

Each page has the exact `spec.yaml`, run command, and setup notes. Overview:
[providers/README.md](./providers/README.md).

## Troubleshooting

**`mount policy denied: /Users/<you>`** when running `sbx run --kit docker.io/..`:
the sbx runtime refuses to mount your home directory into the sandbox. Run
`sbx run` from any directory other than your home directory.

**`grafana_report.py` can't connect on the local target:** confirm Grafana is up
on the host (`docker compose -f compose/docker-compose.yml ps`) and that the
compose stack (or your own Grafana) is listening on port 3000.

**Cloud/oss `Unauthorized`:** confirm the token is stored (`sbx secret ls` shows
a `grafana` entry) and that `GRAFANA_URL` points at your stack (the kit ships a
placeholder — see [providers/cloud.md](./providers/cloud.md)).

## License

[Apache 2.0](./LICENSE).
