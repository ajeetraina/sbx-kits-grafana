# Grafana Cloud

Points `mcp-grafana` and the runbook at your Grafana Cloud stack
(`https://<stack>.grafana.net`), authenticated with a service-account token that
the sbx proxy injects on the wire — the token never enters the sandbox.

| | |
|---|---|
| Runs where | Grafana Cloud (`*.grafana.net`) |
| Grafana URL | you set it (per-stack) |
| Credential | `GRAFANA_SERVICE_ACCOUNT_TOKEN` via `sbx secret set -g grafana` |
| MCP server | `~/.local/bin/mcp-grafana` |

## 1. Create a service-account token

In your stack: *Administration → Users and access → Service accounts → Add
service account*. Give it a role (`Viewer` for read-only MCP tools; `Editor` to
create dashboards/annotations), then *Add service account token* and copy the
value. Details: <https://grafana.com/docs/grafana/latest/administration/service-accounts/>.

## 2. Store the token as a secret (never on the command line)

`sbx run` has no `-e` flag. Store the token once with sbx's secret manager; the
proxy attaches it to outbound requests to your stack and it never enters the
sandbox, shell history, or `ps`:

```bash
echo "$GRAFANA_TOKEN" | sbx secret set -g grafana   # -g = all sandboxes
# or run `sbx secret set -g grafana` for an interactive prompt
```

## 3. Set your stack URL

The kit can't know your stack slug, so it ships a placeholder `GRAFANA_URL`. Set
yours one of two ways:

- Run from a local clone and edit `kits/cloud/spec.yaml` (`GRAFANA_URL` and the
  `mcp.json` initFile), then `sbx run --kit ./kits/cloud claude`.
- Or, inside the sandbox, `export GRAFANA_URL=https://<stack>.grafana.net`
  before running the runbook / re-registering the MCP server.

## Run

```bash
echo "$GRAFANA_TOKEN" | sbx secret set -g grafana
sbx run --kit docker.io/ajeetraina777/sbx-grafana-kits:cloud claude
# or from an edited clone:
sbx run --kit ./kits/cloud claude
```

## What the kit contains

`kits/cloud/spec.yaml` already wires everything:

- `network.allowedDomains` includes `*.grafana.net`.
- `network.serviceDomains` maps `*.grafana.net` to the `grafana` service, and
  `serviceAuth` sets `Authorization: Bearer %s`, so the proxy attaches the token.
- `credentials.sources.grafana` reads the token from the stored secret, and
  `environment.proxyManaged` lists `GRAFANA_SERVICE_ACCOUNT_TOKEN` so its
  in-sandbox value stays a placeholder that the proxy replaces on the wire.

## Verify (inside the sandbox)

```console
!export GRAFANA_URL=https://<stack>.grafana.net   # if not baked in
!python3 ~/runbooks/grafana_report.py
```

Expect your stack's health, datasources, and dashboards.
