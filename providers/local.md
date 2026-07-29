# Local: Grafana on the host (default, no account, no token)

This is what the kit ships with. Both `mcp-grafana` and the `grafana-client`
runbook talk to a Grafana running on your host, reached from the sandbox over
`host.docker.internal:3000`, so no cloud account and no API token are needed.

| | |
|---|---|
| Runs where | Host (your machine's Docker) |
| Grafana URL | `http://host.docker.internal:3000` (baked) |
| Credential | none (anonymous Admin) |
| MCP server | `~/.local/bin/mcp-grafana` |
| Python lib | `grafana-client` |

## Prerequisites

Bring up Grafana (and a Prometheus datasource to query) on the host with the
bundled compose stack. Run this on your **host**, before launching the kit:

```console
docker compose -f compose/docker-compose.yml up -d
```

That starts Grafana on `http://localhost:3000` with anonymous Admin access
enabled, so the sandbox can reach its API with no token. Already running your
own Grafana on port 3000? Skip the compose step — the kit only needs *something*
answering on `host.docker.internal:3000`.

> Anonymous Admin is convenient for a local demo. Do not expose that instance
> beyond localhost.

## Run

Published as the Hub image (`:latest`, also tagged `:local`), or run the
standalone spec from this repo:

```console
sbx run --kit docker.io/ajeetraina777/sbx-grafana-kits:latest claude
# or from this repo:
sbx run --kit ./kits/local claude
```

## Verify (inside the sandbox)

The binary is installed and wired:

```console
!/home/agent/.local/bin/mcp-grafana --help | head
!env | grep GRAFANA_URL
```

End-to-end, through the local Grafana:

```console
!python3 ~/runbooks/grafana_report.py
```

Expect the instance health, the `Prometheus` datasource, and any dashboards.

## Notes

The MCP server is registered with the Claude agent automatically (best-effort at
startup). For other agents, import the portable definition the kit writes at
`~/.grafana/mcp.json`, or run `claude mcp add grafana -- ~/.local/bin/mcp-grafana`
yourself.
