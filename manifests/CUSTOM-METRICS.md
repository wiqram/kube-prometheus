# Custom metrics — `custom.metrics.k8s.io`, served by prometheus-adapter

This is how an application metric that Prometheus scrapes becomes something an
`autoscaling/v2` HPA can actually scale on. It is **cluster infrastructure and is owned
here**, in this repo — not by the app repos.

## The two metrics APIs are not the same thing, and confusing them has cost us before

| APIService | Served by | Serves | Used by |
|---|---|---|---|
| `v1beta1.metrics.k8s.io` | `kube-system/metrics-server` | node & pod cpu/memory | `kubectl top`, `type: Resource` HPAs |
| `v1beta1.custom.metrics.k8s.io` | `monitoring/prometheus-adapter` | arbitrary app metrics from Prometheus | `type: Pods` / `type: Object` HPAs |

Each is a **singleton** object: whoever registers it owns it, cluster-wide.

`prometheusAdapter-apiService.yaml` — which claimed **`metrics.k8s.io`** for the adapter —
was deleted in `88c88ce8` and **must not come back**. On this node the adapter cannot serve
resource metrics at all: the kubelet's cAdvisor series arrive without `pod`/`namespace`/
`container` labels, so `kubectl top pod` returned empty while `top node` worked. Re-adding
that file re-claims the API from metrics-server on the next `kubectl apply -f manifests/`
and silently re-breaks `top` — and every CPU-based HPA with it.

`prometheusAdapter-apiServiceCustomMetrics.yaml` (added 2026-08-04) claims the **other**
API. It does not touch `metrics.k8s.io` and cannot re-break `top`. Both are expected:

```bash
kubectl get apiservice v1beta1.metrics.k8s.io          # kube-system/metrics-server   True
kubectl get apiservice v1beta1.custom.metrics.k8s.io   # monitoring/prometheus-adapter True
```

## What is served today

Defined by the `rules:` block in `prometheusAdapter-configMap.yaml`. Scope is an explicit
allowlist — currently the single prefix group `^(yolo)_` — so the adapter is not re-planning
every series in Prometheus once a minute on a box that is also running the apps.

| Custom metric | Built from | Query |
|---|---|---|
| `yolo_grpc_requests_per_second` | `yolo_grpc_requests_total` | `sum(rate(…[2m])) by (pod)` |
| `yolo_grpc_in_flight_requests` | same name (gauge) | `sum(…) by (pod)` |
| `yolo_grpc_request_duration_p95_milliseconds` | `yolo_grpc_request_duration_seconds_bucket` | `histogram_quantile(0.95, …) * 1000` |

Counters are exposed as a **rate**, never raw: a counter only climbs, so an HPA on one
would scale out once and never come back down.

The p95 is in **milliseconds** because custom metric values cross the API as
`resource.Quantity` — `0.0095` seconds round-trips as `9m` (nine *milli*-units), which is
unreadable in both `kubectl get --raw` and an HPA target.

## Checking it works

```bash
# every custom metric the adapter currently offers
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 | jq -r '.resources[].name' | sort

# a live value, per pod
kubectl get --raw \
  '/apis/custom.metrics.k8s.io/v1beta1/namespaces/yolo/pods/*/yolo_grpc_in_flight_requests' | jq
```

An empty `resources` list means the rules matched no series — check that the app is
actually being scraped (`/api/v1/targets` in Prometheus) before touching the rules.

## Using one in an HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  metrics:
    - type: Pods
      pods:
        metric:
          name: yolo_grpc_in_flight_requests
        target:
          type: AverageValue
          averageValue: "20"       # scale out above 20 in-flight per pod
```

`type: Pods` targets are **AverageValue** only — there is no `Utilization` for a custom
metric, because there is no request/limit to compute a percentage against.

Keep the existing `type: Resource` CPU metric alongside it unless you have a reason not to:
with several metrics listed, the HPA computes a replica count for each and takes the
**highest**, so CPU stays as a floor if the custom metric ever stops reporting.

⚠️ A pod with **no traffic in the window** has no p95 — `histogram_quantile` over an empty
bucket is `NaN` and the adapter omits `NaN` samples. The HPA then reports `<unknown>` for
that metric and falls back to its other metrics. That is correct behaviour, but it means a
latency HPA is a poor fit for a service that idles for long stretches.

## Adding another app

1. The app exposes Prometheus metrics named with its own prefix, on a Service port, and
   ships a **ServiceMonitor plus a `prometheus-k8s` Role/RoleBinding in its own namespace**
   (kube-prometheus only grants that Role in `default`, `kube-system` and `monitoring` — without
   it the ServiceMonitor is accepted and simply never produces a target, with no error
   anywhere). Those live in the app's repo, not here.
2. Confirm the series carry `pod` and `namespace` labels. The adapter needs them to map a
   series onto a Kubernetes object; a metric without them cannot be served here at all.
3. Extend the prefix group in **all three** `seriesQuery` regexes in
   `prometheusAdapter-configMap.yaml`: `^(yolo)_` → `^(yolo|newapp)_`.
4. `kubectl apply -f manifests/prometheusAdapter-configMap.yaml` and
   `kubectl -n monitoring rollout restart deployment/prometheus-adapter` — the adapter reads
   its config once at startup.

## Ownership

Same split as `YOLO-OWNERSHIP.md`: the adapter, its rules and the APIService are **this**
repo's. ServiceMonitors, namespace RBAC, dashboards, alert rules and HPAs belong to the app
repos. An app must never ship a competing adapter config — there is one adapter and one
config, and the last apply wins silently.
