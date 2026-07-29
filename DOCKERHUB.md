# Grafana kit for Docker Sandboxes

A standalone [Docker Sandboxes](https://docs.docker.com/ai/sandboxes/) kit
(`kind: mixin`) that gives any sandbox agent access to
[Grafana](https://grafana.com/) — dashboards, datasources, and Prometheus/Loki
queries — through the official
[Grafana MCP server](https://github.com/grafana/mcp-grafana) (`mcp-grafana`),
plus the `grafana-client` Python library. This image ships in three target
flavors, one per tag.

Source and full docs: https://github.com/ajeetraina/sbx-kits-grafana

## Image tags

| Tag | Grafana target | Credential |
|-----|----------------|------------|
| `latest`, `local` | Grafana on the host (`host.docker.internal:3000`) | none (anonymous Admin) |
| `cloud` | Grafana Cloud (`*.grafana.net`) | `sbx secret set -g grafana` |
| `oss` | Any self-hosted Grafana | `sbx secret set -g grafana` |

Local is the default because it needs no Grafana account and no token: bring up
Grafana on the host and the sandbox reaches it over `host.docker.internal`. Cloud
and self-hosted users authenticate with a Grafana service-account token, stored
once with sbx so it never enters the sandbox.

## Quick start

Local default. Bring up Grafana on the host, then launch:

    docker compose -f compose/docker-compose.yml up -d   # from the repo
    sbx run --kit docker.io/ajeetraina777/sbx-grafana-kits:latest claude

Grafana Cloud. Store a service-account token once (never on the command line),
set your stack URL (see the repo's providers/cloud.md), then run:

    echo "$GRAFANA_TOKEN" | sbx secret set -g grafana
    sbx run --kit docker.io/ajeetraina777/sbx-grafana-kits:cloud claude

Self-hosted:

    echo "$GRAFANA_TOKEN" | sbx secret set -g grafana
    sbx run --kit ./kits/oss claude   # edit kits/oss/spec.yaml with your host first

The cloud/oss tags hold no token. The sbx proxy injects it from the stored
secret, so the token never enters the sandbox. `sbx run` has no `-e` flag by
design.

## How it works

Each kit installs the pinned `mcp-grafana` v1.0.0 binary and `grafana-client`,
sets `GRAFANA_URL`, writes a portable MCP definition to `~/.grafana/mcp.json`,
and (best-effort) registers the server with the Claude agent. The cloud/oss tags
additionally route requests to your Grafana host through the sbx proxy with an
`Authorization: Bearer` header sourced from the stored `grafana` secret. It also
ships a runbook (`~/runbooks/grafana_report.py`) that prints the instance's
health, datasources, and dashboards.

Per-target setup notes, validation steps, and the raw `spec.yaml` for each kit
live on GitHub:
https://github.com/ajeetraina/sbx-kits-grafana/tree/main/providers
