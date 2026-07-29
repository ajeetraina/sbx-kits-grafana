# Self-hosted Grafana (OSS / Enterprise)

Points `mcp-grafana` and the runbook at any self-hosted Grafana you can reach
over the network, authenticated with a service-account token the sbx proxy
injects on the wire.

| | |
|---|---|
| Runs where | Your Grafana host |
| Grafana URL | you set it |
| Credential | `GRAFANA_SERVICE_ACCOUNT_TOKEN` via `sbx secret set -g grafana` |
| MCP server | `~/.local/bin/mcp-grafana` |

Because only you know your Grafana hostname, this target is meant to be run from
a local clone you edit — the sandbox's egress allow-list must name your host, and
a baked image can't.

## 1. Create a service-account token

In your Grafana: *Administration → Service accounts → Add service account*,
assign a role, then mint a token. See
<https://grafana.com/docs/grafana/latest/administration/service-accounts/>.

## 2. Store the token

```bash
echo "$GRAFANA_TOKEN" | sbx secret set -g grafana
```

## 3. Point the kit at your host

Edit `kits/oss/spec.yaml` and replace `grafana.example.com` with your Grafana
host in all three places:

- `network.allowedDomains` (so the sandbox may reach it)
- `network.serviceDomains` (so the proxy attaches the token to it)
- `environment.variables.GRAFANA_URL` (and the `mcp.json` initFile)

If your Grafana runs on the same host as Docker, use
`http://host.docker.internal:3000` and add `host.docker.internal` to `NO_PROXY`
(already set) — no token or serviceDomains needed in that case; it behaves like
the [local](./local.md) target.

## Run

```bash
echo "$GRAFANA_TOKEN" | sbx secret set -g grafana
sbx run --kit ./kits/oss claude
```

## Verify (inside the sandbox)

```console
!python3 ~/runbooks/grafana_report.py
```

Expect your instance's health, datasources, and dashboards. If it can't connect,
confirm your host is in `allowedDomains` and the token is stored
(`sbx secret ls`).
