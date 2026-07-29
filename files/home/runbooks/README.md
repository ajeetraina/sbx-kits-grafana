# Runbooks

Runnable demos shipped with the Grafana kit. They live at `~/runbooks/` in the
sandbox and use the `grafana-client` library the kit installs, reading
`GRAFANA_URL` / `GRAFANA_SERVICE_ACCOUNT_TOKEN` from the environment the kit
sets up. They work against the `local`, `cloud`, or `oss` kit unchanged.

## grafana_report.py

Prints a quick health report for the wired Grafana instance — its health
status, its datasources, and its dashboards:

```console
python3 ~/runbooks/grafana_report.py
```

Against the default local kit this talks to Grafana on the host
(`http://host.docker.internal:3000`) with no token. Against the cloud/oss kits
it uses `GRAFANA_URL` and the token the sbx proxy injects from the stored
`grafana` secret.

To add a runbook, drop a `*.py` in `files/home/runbooks/` — it ships
automatically, no `spec.yaml` change.
