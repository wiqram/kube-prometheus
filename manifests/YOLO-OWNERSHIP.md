# YOLO's Grafana content is NOT owned by this repo

`yolo-grafana-extras.yaml` used to live in this directory and defined two
ConfigMaps in the `monitoring` namespace:

- `grafana-dashboard-yolo-pipeline`
- `grafana-alerting-yolo`

**It was deleted on 2026-08-03 and must not be recreated here.**

## Why

That file had drifted badly. It carried **4 of YOLO's 20 alert rules and no
`contactPoints` and no notification policy at all** — so those four safety
alerts (kill-switch, reconcile-drift, quarantine-spike, broker-errors) were
evaluating on schedule inside this Grafana and firing into a void. It went
unnoticed for weeks because YOLO also ran a second Grafana of its own in the
`yolo` namespace which *did* have a working ntfy contact point and was
delivering the same alerts.

That second Grafana has now been retired in favour of this cluster-wide one, so
this Grafana is the only thing standing between a YOLO safety condition and a
human. A stale copy of those ConfigMaps in this repo is therefore actively
dangerous: a routine `kubectl apply -f manifests/` — the workflow this repo's
own README documents — would **silently** revert YOLO to four rules that
notify nobody. No error, no diff, just alerts that stop arriving.

## Where it lives now

Both ConfigMaps are built and applied by the **IG-Trading-Microservices**
repo's Jenkins deploy stage, from:

| ConfigMap | Source of truth |
|---|---|
| `grafana-dashboard-yolo-pipeline` | `monitoring/grafana/*.json` |
| `grafana-alerting-yolo` | `monitoring/grafana-alerting/alerting.yaml` |

## What this repo DOES still own — do not remove it

`grafana-deployment.yaml` declares the volumes and mounts that make the above
work, and they are still required:

- `grafana-dashboard-yolo-pipeline` → `/grafana-dashboard-definitions/yolo/yolo-pipeline`
- `grafana-alerting-yolo` → `/etc/grafana/provisioning/alerting`

`grafana-dashboardSources.yaml` (the `YOLO` folder provider, path
`/grafana-dashboard-definitions/yolo`) and the `yolo-loki` datasource in
`grafana-dashboardDatasources.yaml` are also still owned here.

**Renaming either ConfigMap in this repo silently unmounts it** and the other
repo has no way to know. If a rename is ever needed, it has to be a coordinated
change across both repos.
