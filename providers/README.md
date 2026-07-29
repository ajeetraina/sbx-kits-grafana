# Grafana targets for the kit

The kit adds the same two things to your agent regardless of target: the
official [Grafana MCP server](https://github.com/grafana/mcp-grafana)
(`mcp-grafana`) and the [`grafana-client`](https://pypi.org/project/grafana-client/)
Python library. What changes per target is *which Grafana* they point at and how
the credential is handled.

The kit ships wired to a **local Grafana on the host**, so it works with no
Grafana account and no API token. But you can point it at Grafana Cloud or any
self-hosted Grafana instead. This folder has a focused page per target with
copy-paste config.

Why local is the default: it needs no account, no token, and no cloud egress —
you bring up Grafana + Prometheus on the host with the bundled
[`compose/`](../compose) stack and the sandbox reaches it over
`host.docker.internal`. Cloud and self-hosted users authenticate with a Grafana
[service-account token](https://grafana.com/docs/grafana/latest/administration/service-accounts/),
stored once with sbx so it never enters the sandbox.

## Target matrix

| Target | Runs where | Grafana URL | Credential | How the token reaches Grafana |
|---|---|---|---|---|
| [local](./local.md) (default) | Host (`host.docker.internal:3000`) | baked | none (anonymous Admin) | n/a |
| [cloud](./cloud.md) | Grafana Cloud (`*.grafana.net`) | you set it | `sbx secret set -g grafana` | proxy injects `Authorization: Bearer` on the wire |
| [oss](./oss.md) | Any self-hosted Grafana | you set it | `sbx secret set -g grafana` | proxy injects `Authorization: Bearer` on the wire |

## Two notes that apply to every target

1. **The token is a service-account token, not your login.** Create one in
   Grafana under *Administration → Service accounts*, give it a role
   (`Viewer` is enough for read-only MCP tools; `Editor`/`Admin` to create
   dashboards or annotations), then mint a token. See the
   [Grafana docs](https://grafana.com/docs/grafana/latest/administration/service-accounts/).
2. **The stack URL is the one value the kit cannot know for you.** Cloud and
   self-hosted URLs are per-user, so the `cloud`/`oss` kits ship a placeholder
   `GRAFANA_URL`. Set yours by running from a local clone and editing the spec
   (`sbx run --kit ./kits/cloud`) or `export GRAFANA_URL=...` inside the sandbox
   before you use the runbook. See each target's page.

## How to switch target

Each target is published as an image tag (`:local`, `:cloud`, `:oss`), and the
same specs live under [`kits/`](../kits). Pick one, store its token if it's not
local, and run it:

```bash
sbx secret set -g grafana                 # cloud / oss only
sbx run --kit docker.io/ajeetraina777/sbx-grafana-kits:cloud claude
# or from this repo: sbx run --kit ./kits/cloud claude
```

Keys are never stored in the kit; the sbx proxy injects them from the stored
secret, so they never enter the sandbox (`sbx run` has no `-e` flag).
