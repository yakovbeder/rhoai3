# MaaS Token metrics (Perses) — RHOAI 3.5 / COO

Adds a **Token metrics** tab on OpenShift AI **Observe & monitor → Dashboard** (GA on 3.5). It sorts after **Usage**. Leave the product Usage dashboard (`dashboard-3-maas-usage-admin`) in place.

**3.5 does not use User Workload Monitoring.** Queries, scrape, and cost recording stay on Cluster Observability Operator (COO), same path as Usage (`data-science-prometheus-datasource`, `authorized_hits_total`).

The 3.4 UWM YAML is in [`../3.4/`](../3.4/). Do not apply 3.4 and 3.5 together.

Run every `oc` apply command from the **repo root**.

## Procedure

Complete these steps in order. Apply the dashboard only in step 4.

That apply creates:

| File | Object | Namespace |
|---|---|---|
| `dashboard-4-maas-token-metrics-admin.yaml` | `PersesDashboard` | `redhat-ods-monitoring` |
| `maas-cost-rates.yaml` | `PrometheusRule` (`monitoring.rhobs/v1`) | `redhat-ods-monitoring` |

### 1. Turn on the Dashboard page

```bash
oc get odhdashboardconfig odh-dashboard-config -n redhat-ods-applications \
  -o jsonpath='{.spec.dashboardConfig.observabilityDashboard}{"\n"}'
```

If that is not `true`:

```bash
oc patch odhdashboardconfig odh-dashboard-config -n redhat-ods-applications --type merge \
  -p '{"spec":{"dashboardConfig":{"observabilityDashboard":true}}}'
```

### 2. Use the same Prometheus datasource as Usage

This dashboard queries **`data-science-prometheus-datasource`** in `redhat-ods-monitoring` (RHOAI 3.5 Usage). The dashboard must live in that namespace so Perses can resolve the datasource.

```bash
oc get persesdatasource -n redhat-ods-monitoring
oc get persesdashboard dashboard-3-maas-usage-admin -n redhat-ods-monitoring \
  -o yaml | grep -i datasource | head
```

### 3. Disable Kuadrant observability

Kuadrant `spec.observability.enable` is UWM-only. When it is true, Kuadrant creates `PodMonitor/kuadrant-limitador-monitor`, which duplicates hits in Token metrics. Deleting the PodMonitor is not enough; Kuadrant recreates it while the flag is true.

```bash
oc patch kuadrant kuadrant -n kuadrant-system --type merge \
  -p '{"spec":{"observability":{"enable":false}}}'
oc get podmonitor kuadrant-limitador-monitor -n kuadrant-system --ignore-not-found
```

Do not re-enable `observability.enable` for Token metrics. The same flag also owns `PodMonitor/istio-pod-monitor` in `openshift-ingress` (UWM Istio metrics). Token metrics and Usage do not use it.

### 4. Apply the dashboard and cost rule

```bash
oc apply -k maas-monitoring/3.5
```

If a Token metrics dashboard from 3.4 is still live, delete it so there is not a second tab:

```bash
oc delete persesdashboard dashboard-4-maas-token-metrics-admin -n redhat-ods-applications --ignore-not-found
oc delete prometheusrule maas-cost-rates -n kuadrant-system --ignore-not-found
```

### 5. Reload the UI

Open **Observe & monitor → Dashboard**. Admins should see **Cluster**, **Models**, **Usage**, **Token metrics**.

The object name is `dashboard-4-maas-token-metrics-admin` so it sorts after `dashboard-3-maas-usage-admin`. If Usage is not `dashboard-3-…`, rename this CR so the `dashboard-N-` prefix still sorts after it. The `-admin` suffix is the same access gate as Cluster and Usage.

## Dashboard naming

OpenShift AI lists every `PersesDashboard` whose name starts with `dashboard-`. Tab text is `spec.config.display.name` (**Token metrics**).

Do **not** add a variable named `namespace`. OpenShift AI substitutes the signed-in user’s projects for that name. Limitador’s Kubernetes `namespace` label is `kuadrant-system`, not the model project.

## Filters

| Filter | Prometheus label | Purpose |
|---|---|---|
| User | `user` | Token identity |
| Subscription | `subscription` | `MaaSSubscription` name |
| Model | `model` | Served model |
| Project / route | `limitador_namespace` (`serving_route` variable) | `{project}/{HTTPRoute}`, for example `beder/gpt-oss-20b-kserve-route` |

**All** on each filter uses `customAllValue: ".*"` so PromQL `=~"$var"` matchers stay valid.

## Metrics

This dashboard queries **`authorized_hits_total`** (same as Usage on 3.5), labels `user`, `subscription`, `model`. Grouping is by **subscription** (`MaaSSubscription` name).

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

Copy every subscription you want billed; merge replaces `spec.groups`.

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

Wait about 30 seconds, then confirm against COO Prometheus (not UWM). The web listener is HTTPS:

```bash
oc exec -n redhat-ods-monitoring prometheus-data-science-monitoringstack-0 -c prometheus -- \
  curl -skG 'https://localhost:9090/api/v1/query' --data-urlencode 'query=maas:cost_rate'
```

Replace the full `spec.groups` list. Do not JSON-patch `rules/0`; that index is whatever happened to be first after the last merge.

Keep the rule in `redhat-ods-monitoring` as `prometheusrule.monitoring.rhobs` so COO evaluates it. Do not use `monitoring.coreos.com/v1` or `openshift.io/prometheus-rule-evaluation-scope: leaf-prometheus` (those are UWM / platform Prometheus).

## If panels are empty

1. No `authorized_hits_total` in the COO datasource. If Usage is also empty, scrape is missing.
2. Datasource is not `data-science-prometheus-datasource` in `redhat-ods-monitoring`.
3. You are not in the `-admin` audience (need `prometheuses/api` like Usage).
4. Cost tiles only: hits work, revenue is missing → no `maas:cost_rate` for that `subscription`. Patch `prometheusrule.monitoring.rhobs` (full `spec.groups` list).
5. Hourly chart is a single bar or looks empty → widen the time range to Last 6h or 24h.

If hit counts look about twice Usage, Kuadrant `observability.enable` is still true. Re-run step 3.
