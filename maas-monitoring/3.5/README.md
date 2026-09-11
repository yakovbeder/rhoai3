# MaaS Token metrics (Perses) — RHOAI 3.5 / COO

> This is a Perses adaptation of the Grafana dashboard created by **Guy Rakover**.
>
> Original dashboard: [rockocoop/openshiftai3 — maas/maas-monitoring](https://github.com/rockocoop/openshiftai3/tree/main/maas/maas-monitoring)

Customer-facing YAML for a **Token metrics** tab on OpenShift AI **Observe & monitor → Dashboard**. It sorts **after Usage**. On 3.5 that Dashboard page is GA.

Apply the YAML on the cluster. OpenShift AI lists the new tab next to Usage when the object name starts with `dashboard-`. Leave the product Usage dashboard (`dashboard-3-maas-usage-admin`) in place.

**3.5 does not use User Workload Monitoring.** Queries, scrape, and cost recording all stay on Cluster Observability Operator (COO), same as Usage and the other 3.5 dashboards.

The 3.4 UWM snapshot is in [`../3.4/`](../3.4/). Do not apply 3.4 and 3.5 together.

## Apply

```bash
oc apply -k maas-monitoring/3.5
```

That creates:

| File | Object | Namespace |
|---|---|---|
| `dashboard-4-maas-token-metrics-admin.yaml` | `PersesDashboard` | `redhat-ods-monitoring` |
| `maas-cost-rates.yaml` | `PrometheusRule` (`monitoring.rhobs/v1`) | `redhat-ods-monitoring` |

If a Token metrics dashboard from 3.4 is still live, delete it so there is not a second tab:

```bash
oc delete persesdashboard dashboard-4-maas-token-metrics-admin -n redhat-ods-applications --ignore-not-found
oc delete prometheusrule maas-cost-rates -n kuadrant-system --ignore-not-found
```

Reload **Observe & monitor → Dashboard**. Admins should see **Cluster**, **Model**, **Usage**, **Token metrics**.

### Prerequisites

- `OdhDashboardConfig.spec.dashboardConfig.observabilityDashboard: true`
- Perses operator + COO `PersesDatasource` **`data-science-prometheus-datasource`** in `redhat-ods-monitoring` (same as Usage on RHOAI 3.5). The dashboard must live in that namespace so Perses can resolve the datasource.
- Live Usage tab name starts with `dashboard-`. This CR is `dashboard-4-…` so it sorts after `dashboard-3-maas-usage-admin`. If Usage is not `dashboard-3-…`, rename this CR so the `dashboard-N-` prefix still sorts after it.
- Schema matches this cluster: `perses.dev/v1alpha2` with panels under `spec.config`.

The `-admin` suffix follows the same access rule as Cluster / Usage.

### Dashboard naming

The UI lists every `PersesDashboard` whose **name** starts with `dashboard-`, then sorts those names lexicographically. Tab text is `spec.display.name` (on this cluster that lives at `spec.config.display.name` because the CRD storage version is `v1alpha2`).

```
dashboard-{order}-{name}[-admin]
```

| Piece | This CR |
|---|---|
| `order` | `4` — after `dashboard-3-maas-usage-admin` (`dashboard-3-maas-tokens-admin` would sort **before** Usage) |
| `name` | `maas-token-metrics` |
| `-admin` | Present — only users with cluster Prometheus/`prometheuses/api` access see it (same gate as Usage) |

Do **not** add a variable named `namespace`. OpenShift AI substitutes the signed-in user’s projects for that name. Limitador’s Kubernetes `namespace` label is `kuadrant-system`, not the model project.

Multi-tenant filter is **Project / route** (`serving_route`). That is Limitador `limitador_namespace` = `{project}/{HTTPRoute}`, for example `beder/gpt-oss-20b-kserve-route`. It is not an OpenShift Route “service”. Filters: `user`, `subscription`, `model`, `serving_route`, with `customAllValue: ".*"` so **All** works in `=~"$var"` matchers.

## Disable Kuadrant observability

That flag is UWM-only. It creates `PodMonitor/kuadrant-limitador-monitor`, which duplicates hits (Usage ~8K, Token metrics ~16K if Thanos merges both). Deleting the PodMonitor is not enough; Kuadrant recreates it while the flag is true.

```bash
oc patch kuadrant kuadrant -n kuadrant-system --type merge \
  -p '{"spec":{"observability":{"enable":false}}}'
oc get podmonitor kuadrant-limitador-monitor -n kuadrant-system --ignore-not-found
```

Do not re-enable `observability.enable` for Token metrics. The same flag also owns `PodMonitor/istio-pod-monitor` in `openshift-ingress` (UWM Istio metrics). Token metrics and Usage do not use it.

## Metrics

This dashboard queries **`authorized_hits_total`** (same as Usage on 3.5), labels `user`, `subscription`, `model`. There is no `tier` label; grouping is by **subscription** (`MaaSSubscription` name).

Rate-limit success/blocked (`authorized_calls` / `limited_calls`) stays on the **Usage** tab. Token metrics is hits, subscriptions, and demo cost.

## Dashboard panels

Same layout as [3.4](../3.4/), with PromQL on `authorized_hits_total` instead of untyped `authorized_hits`.

| Panel | Type | Purpose |
|---|---|---|
| Total authorized hits | Stat | `sum(increase(authorized_hits_total[$__range]))` |
| Active users | Stat | Distinct `user` labels on series that exist **now** |
| Total revenue (USD) | Stat | Hits in the window × `maas:cost_rate` per subscription |
| Authorized hits rate by subscription | Timeseries | `rate(...[$__rate_interval])` |
| Authorized hits by user and subscription | Timeseries | Hit rate over time |
| Top 10 users by hits | Table | Ranking for the selected window |
| Top 5 users by cost (USD) | Table | `increase(authorized_hits_total) * maas:cost_rate` |
| Hourly authorized hits by user | Timeseries (bars) | `increase[1h]`, `minStep: 1h`. Use Last **6h** or **24h** |
| Total authorized hits by subscription | Timeseries | Sliding `increase(...[$__range])` |
| Authorized hits by model | Timeseries | Same sliding increase, one series per model |
| Top models by hits | Table | Ranking for the selected window |

## Cost rates (per subscription)

Prices are **not** in the dashboard PromQL. Cost panels multiply hits by recording metric `maas:cost_rate{subscription="<name>"}`.

A subscription with no `maas:cost_rate` series still appears in hit charts; it is omitted from revenue/cost.

Default in `maas-cost-rates.yaml`: `gpt-premium` and `bedrock-free` = `0.008` USD per authorized hit (demo). Change it with `oc patch`.

List live subscription names:

```bash
oc get maassubscription -A
# or Prometheus: label_values(authorized_hits_total, subscription)
```

### Add or set rates (replace the full rule list)

`NS=redhat-ods-monitoring`. Copy every subscription you want billed; merge replaces `spec.groups`.

```bash
oc patch prometheusrule.monitoring.rhobs maas-cost-rates -n redhat-ods-monitoring --type merge -p "$(cat <<'EOF'
spec:
  groups:
  - name: maas-cost
    interval: 30s
    rules:
    - record: maas:cost_rate
      expr: vector(0.008)
      labels:
        subscription: gpt-premium
    - record: maas:cost_rate
      expr: vector(0.012)
      labels:
        subscription: acme-enterprise
EOF
)"
```

Wait ~30s, then confirm against COO Prometheus (not UWM). The web listener is HTTPS:

```bash
oc exec -n redhat-ods-monitoring prometheus-data-science-monitoringstack-0 -c prometheus -- \
  curl -skG 'https://localhost:9090/api/v1/query' --data-urlencode 'query=maas:cost_rate'
```

JSON patch of `rules/0` is **not** the supported path. That index is whatever happened to be first after the last merge. Replace the full list above.

Keep the rule in `redhat-ods-monitoring` as `prometheusrule.monitoring.rhobs` so COO evaluates it. Do not use `monitoring.coreos.com/v1` or `openshift.io/prometheus-rule-evaluation-scope: leaf-prometheus` (those are UWM / platform Prometheus).

## If panels are empty

1. Kuadrant `observability.enable` is still true and `PodMonitor/kuadrant-limitador-monitor` duplicates hits. Disable it.
2. Datasource is not `data-science-prometheus-datasource` in `redhat-ods-monitoring`.
3. You are not in the `-admin` audience (need `prometheuses/api` like Usage).
4. Cost tiles only: hits work, revenue is missing → no `maas:cost_rate` for that `subscription`. Patch `prometheusrule.monitoring.rhobs` (full `spec.groups` list).
5. Hourly chart is a single bar or looks empty → widen the time range to Last 6h or 24h.
