# Monitoring OpenShift AI MaaS with Perses

> This repository is a Perses adaptation of the Grafana dashboard created by **Guy Rakover**.
>
> Original dashboard: [rockocoop/openshiftai3 — maas/maas-monitoring](https://github.com/rockocoop/openshiftai3/tree/main/maas/maas-monitoring)

> In this repository we add a **Token metrics** tab to OpenShift AI **Observe and monitor → Dashboard (Tech Preview)**. It sorts after the product **Usage** tab. Apply the YAML with Kustomize; the UI lists any `PersesDashboard` whose name starts with `dashboard-`. Do not replace Usage, and do not open a PR against `odh-dashboard`.

Tested with:

- OpenShift AI 3.4
- Perses `perses.dev/v1alpha2` (`spec.config`)
- Kuadrant / Limitador (`authorized_hits`)

## Table of contents

- [About](#about)
- [Dashboard](#dashboard)
- [Repository structure](#repository-structure)
- [Quick start](#quick-start)

## About

- Aimed at cluster admins who need hits, subscription, and demo cost for Model as a Service (MaaS) traffic on OpenShift AI.
- Metrics come from Limitador (`authorized_hits`), the same series Usage already queries. Grouping is by **subscription** (`MaaSSubscription` name), not Grafana `tier`.
- Demo USD rates live in a `PrometheusRule` (`maas:cost_rate`). Patch the rule to change prices; do not edit PromQL in every panel.
- Multi-tenant filter is **Project / route** (`limitador_namespace` = `{project}/{HTTPRoute}`). Do not name a variable `namespace` — OpenShift AI would bind the signed-in user’s projects, not the Limitador scrape.

## Dashboard

YAML lives in [`maas-monitoring/`](maas-monitoring/). After apply, admins see **Cluster**, **Models**, **Usage**, **Token metrics**.

Screenshots from a live OpenShift AI 3.4 cluster, time range **Last 24 hours**.

Tab order:

![Dashboard tabs with Token metrics selected](maas-monitoring/docs/token-metrics-tabs.png)

Filters and Overview — total hits, active users, demo revenue, hits rate by subscription:

![Token metrics overview with filters](maas-monitoring/docs/token-metrics-overview.png)

Users and subscriptions — hits over time, top users, top cost, hourly bars, totals by subscription:

![Users and subscriptions section](maas-monitoring/docs/token-metrics-users.png)

Models — hits over time and top models:

![Models section](maas-monitoring/docs/token-metrics-models-cost.png)

Deployment procedure and panel reference: [`maas-monitoring/README.md`](maas-monitoring/README.md).

## Repository structure

```
maas-monitoring/
  dashboard-4-maas-token-metrics-admin.yaml   # PersesDashboard (redhat-ods-applications)
  maas-cost-rates.yaml                        # PrometheusRule maas:cost_rate (kuadrant-system)
  limitador-servicemonitor.yaml               # optional scrape; not in kustomize
  kustomization.yaml
  docs/                                       # live screenshots
```

## Quick start

```bash
cd maas-monitoring
oc apply -k .
```

Full procedure: [`maas-monitoring/README.md`](maas-monitoring/README.md).
