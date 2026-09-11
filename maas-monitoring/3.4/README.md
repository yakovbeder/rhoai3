# MaaS Token metrics (Perses) — RHOAI 3.4 / UWM

> This is a Perses adaptation of the Grafana dashboard created by **Guy Rakover**.
>
> Original dashboard: [rockocoop/openshiftai3 — maas/maas-monitoring](https://github.com/rockocoop/openshiftai3/tree/main/maas/maas-monitoring)

Customer-facing YAML for a **Token metrics** tab on OpenShift AI **Observe & monitor → Dashboard (Tech Preview)**. It sorts **after Usage**.

This is a **component-owned** dashboard (MaaS), not a PR into `odh-dashboard`. The UI picks it up automatically when the name follows the convention in the [ODH observability dashboards guide](https://github.com/opendatahub-io/odh-dashboard/blob/main/docs/observability.md#observability-dashboards).

Do not replace the Usage dashboard (`dashboard-3-maas-usage-admin`).

**3.4 uses User Workload Monitoring / OpenShift Thanos.** Queries, scrape, and cost recording stay on that path (`kuadrant-prometheus-datasource`, `authorized_hits`).

The 3.5 COO snapshot is in [`../3.5/`](../3.5/). Do not apply 3.4 and 3.5 together.

## Screenshots

From a live OpenShift AI 3.4 cluster, time range **Last 24 hours**. After apply, admins see **Cluster**, **Models**, **Usage**, **Token metrics**.

Tab order:

![Dashboard tabs with Token metrics selected](docs/token-metrics-tabs.png)

Filters and Overview — total hits, active users, demo revenue, hits rate by subscription:

![Token metrics overview with filters](docs/token-metrics-overview.png)

Users and subscriptions — hits over time, top users, top cost, hourly bars, totals by subscription:

![Users and subscriptions section](docs/token-metrics-users.png)

Models — hits over time and top models:

![Models section](docs/token-metrics-models-cost.png)

## Apply

```bash
oc apply -k maas-monitoring/3.4
```

That creates:

| File | Object | Namespace |
|---|---|---|
| `dashboard-4-maas-token-metrics-admin.yaml` | `PersesDashboard` | `redhat-ods-applications` |
| `maas-cost-rates.yaml` | `PrometheusRule` | `kuadrant-system` |

`limitador-servicemonitor.yaml` is an optional fallback scrape. It is not listed in `kustomization.yaml`. Do not apply it if Limitador is already scraped.

Complete the following steps in order. Apply the manifests only in step 4, after you confirm Limitador is already being scraped. If a Limitador scrape already exists, do not add another ServiceMonitor or PodMonitor.

Reload **Observe & monitor → Dashboard**. Admins should see **Cluster**, **Model**, **Usage**, **Token metrics**.

### Prerequisites

- `OdhDashboardConfig.spec.dashboardConfig.observabilityDashboard: true`
- Perses operator + a Prometheus `PersesDatasource`. This CR uses **`kuadrant-prometheus-datasource`** (same as Usage on RHOAI 3.4). Change the datasource `name` if your Usage dashboard uses another (for example `data-science-prometheus-datasource`).
- Live Usage tab name starts with `dashboard-`. This CR is `dashboard-4-…` so it sorts after `dashboard-3-maas-usage-admin`. If Usage is not `dashboard-3-…`, rename this CR so the `dashboard-N-` prefix still sorts after it.
- Schema matches this cluster: `perses.dev/v1alpha2` with panels under `spec.config` (not ACM-unrelated; this is what RHOAI 3.4 stores).

The `-admin` suffix follows the same Thanos access rule as Cluster / Usage.

### ODH naming (from the guide)

The UI lists every `PersesDashboard` whose **name** starts with `dashboard-`, then sorts those names lexicographically. Tab text is `spec.display.name` (on this cluster that lives at `spec.config.display.name` because the CRD storage version is `v1alpha2`).

```
dashboard-{order}-{name}[-admin]
```

| Piece | This CR |
|---|---|
| `order` | `4` — after `dashboard-3-maas-usage-admin` (`dashboard-3-maas-tokens-admin` would sort **before** Usage) |
| `name` | `maas-token-metrics` |
| `-admin` | Present — only users with cluster Prometheus/`prometheuses/api` access see it (same gate as Usage) |

Do **not** add a variable named `namespace`. The guide’s special `namespace` handling substitutes the user’s OpenShift projects. Limitador’s Kubernetes `namespace` label is `kuadrant-system`, not the model project.

Multi-tenant filter is **Project / route** (`serving_route`). That is Limitador `limitador_namespace` = `{project}/{HTTPRoute}`, for example `beder/gpt-oss-20b-kserve-route`. It is not an OpenShift Route “service”. Filters: `user`, `subscription`, `model`, `serving_route`, with `customAllValue: ".*"` so **All** works in `=~"$var"` matchers.

## Procedure

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

This dashboard queries **`kuadrant-prometheus-datasource`** (RHOAI 3.4 Usage). Confirm:

```bash
oc get persesdatasource -n redhat-ods-applications
oc get persesdashboard dashboard-3-maas-usage-admin -n redhat-ods-applications \
  -o yaml | grep -i datasource | head
```

If Usage uses another name (for example `data-science-prometheus-datasource`), change every datasource `name` in `dashboard-4-maas-token-metrics-admin.yaml` to match before you apply.

### 3. Confirm Limitador `/metrics` is already scraped

Empty hit panels almost always mean Prometheus never saw `authorized_hits`.

```bash
oc get servicemonitor,podmonitor -A | grep -iE 'limitador|kuadrant'
oc get kuadrant -n kuadrant-system -o jsonpath='{.items[0].spec.observability}{"\n"}'
# In Prometheus / Thanos used by the datasource:
#   authorized_hits   or   authorized_hits_total
```

If `podmonitor/kuadrant-limitador-monitor` exists in `kuadrant-system`, or the series already exists in the User Workload Prometheus, **stop**. A second scrape duplicates series.

If scrape is missing, prefer Kuadrant observability:

```bash
oc patch kuadrant kuadrant -n kuadrant-system --type merge \
  -p '{"spec":{"observability":{"enable":true}}}'
```

Only if that stays off and there is still no monitor, apply the fallback (not in kustomize):

```bash
oc apply -f maas-monitoring/3.4/limitador-servicemonitor.yaml
```

That YAML has no `bearerTokenFile` (User Workload Monitoring rejects file-based tokens). Selector is `app: limitador`, port `http`, path `/metrics`.

This folder’s ServiceMonitor is **not** applied on clusters that already scrape Limitador.

### 4. Apply the dashboard and cost rule

```bash
oc apply -k maas-monitoring/3.4
```

### 5. Reload the UI

Open **Observe and monitor → Dashboard**. Admins should see **Cluster**, **Models**, **Usage**, **Token metrics**.

The CR name is `dashboard-4-maas-token-metrics-admin` so it sorts after `dashboard-3-maas-usage-admin`. If your Usage tab is not `dashboard-3-…`, rename this CR so the `dashboard-N-` prefix still sorts after it. The `-admin` suffix is the same Thanos access gate as Cluster and Usage.

Schema on RHOAI 3.4: `perses.dev/v1alpha2` with panels under `spec.config`.

## Filters

| Filter | Prometheus label | Purpose |
|---|---|---|
| User | `user` | Token identity |
| Subscription | `subscription` | `MaaSSubscription` name. Grafana grouped by `tier`; live MaaS does not |
| Model | `model` | Served model |
| Project / route | `limitador_namespace` (`serving_route` variable) | `{project}/{HTTPRoute}`, for example `beder/gpt-oss-20b-kserve-route` |

Do **not** add a dashboard variable named `namespace`. ODH substitutes the signed-in user’s OpenShift projects. Limitador’s scrape `namespace` is `kuadrant-system`, not the model project.

**All** on each filter uses `customAllValue: ".*"` so PromQL `=~"$var"` matchers stay valid.

## Metrics

On RHOAI 3.4 + Limitador this dashboard queries untyped **`authorized_hits`** (same as Usage), labels `user`, `subscription`, `model`. There is no `tier` label; grouping is by **subscription** (`MaaSSubscription` name).

If your Prometheus only has `authorized_hits_total`, use the [3.5](../3.5/) folder instead of rewriting this YAML.

Rate-limit success/blocked (`authorized_calls` / `limited_calls`) stays on the **Usage** tab. Token metrics is hits, subscriptions, and demo cost.

## Dashboard panels

Type is how the panel is drawn: **stat** (single number), **timeseries** (line or bars), **table** (rows). A gray bar in a table is often one row that has not painted yet, or no series.

### Overview

| Panel | Type | Purpose |
|---|---|---|
| Total authorized hits | Stat | `sum(increase(authorized_hits[$__range]))` — hits in the selected window, not the raw counter |
| Active users | Stat | Distinct `user` labels on series that exist **now** (filters apply). Not limited to the selected time range |
| Total revenue (USD) | Stat | Hits in the window × `maas:cost_rate` per subscription. Subscriptions with no rate are omitted |
| Authorized hits rate by subscription | Timeseries | `rate(...[$__rate_interval])`, one series per subscription |

### Users and subscriptions

| Panel | Type | Purpose |
|---|---|---|
| Authorized hits by user and subscription | Timeseries | Hit **rate** over time, one series per `user` + `subscription` |
| Top 10 users by hits | Table | Ranking for the selected window (`increase` of hits) |
| Top 5 users by cost (USD) | Table | Same window, `increase(authorized_hits) * maas:cost_rate` |
| Hourly authorized hits by user | Timeseries (bars) | One bar per clock hour (`increase[1h]`, `minStep: 1h`). Not a trailing 1h rate. Use Last **6h** or **24h** — Last 1h is a single bar |
| Total authorized hits by subscription | Timeseries | `increase(...[$__range])` at each step (sliding window), one series per subscription. Looks stepped when traffic is bursty; not a running total from the start of the range |

### Models

| Panel | Type | Purpose |
|---|---|---|
| Authorized hits by model | Timeseries | Same sliding `increase([$__range])` as the subscription totals, one series per model |
| Top models by hits | Table | Ranking for the selected window |

## Cost rates (per subscription)

Prices are **not** in the dashboard PromQL. Cost panels multiply hits by recording metric `maas:cost_rate{subscription="<name>"}`.

A subscription with no `maas:cost_rate` series still appears in hit charts; it is omitted from revenue/cost.

Default in `maas-cost-rates.yaml`: `gpt-premium` = `0.008` USD per authorized hit (demo). Change it with `oc patch`.

List live subscription names:

```bash
oc get maassubscription -A
# or Prometheus: label_values(authorized_hits, subscription)
```

### Add or set rates (replace the full rule list)

`NS=kuadrant-system`. Copy every subscription you want billed; merge replaces `spec.groups`.

```bash
oc patch prometheusrule maas-cost-rates -n kuadrant-system --type merge -p "$(cat <<'EOF'
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

Wait about 30 seconds, then confirm on the same Prometheus that scrapes Limitador. If `wget` is missing in the pod, use the Thanos / User Workload query UI instead:

```bash
oc exec -n openshift-user-workload-monitoring prometheus-user-workload-0 -c prometheus -- \
  wget -qO- --post-data='query=maas:cost_rate' http://localhost:9090/api/v1/query
```

JSON patch of `rules/0` is **not** the supported path. That index is whatever happened to be first after the last merge. Replace the full list above.

The PrometheusRule must keep label `openshift.io/prometheus-rule-evaluation-scope: leaf-prometheus` or OpenShift UWM will not evaluate it. Keep it in `kuadrant-system` so Thanos `?namespace=kuadrant-system` (Usage datasource) can see `maas:cost_rate`.

## If panels are empty

1. Scrape: no `authorized_hits` / `authorized_hits_total` in the datasource Prometheus. Re-run step 3.
2. Datasource name does not match Usage. Re-run step 2.
3. You are not in the `-admin` audience (need `prometheuses/api` like Usage).
4. Usage CR is not `dashboard-3-…`, so this tab sorted somewhere unexpected. Rename the Token metrics CR.
5. Cost tiles only: hits work, revenue is missing → no `maas:cost_rate` for that `subscription`. Patch the PrometheusRule (full `spec.groups` list).
6. Hourly chart is a single bar or looks empty → widen the time range to Last 6h or 24h.
