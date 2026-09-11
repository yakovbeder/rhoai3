# MaaS Token metrics (Perses)

> This is a Perses adaptation of the Grafana dashboard created by **Guy Rakover**.
>
> Original dashboard: [rockocoop/openshiftai3 — maas/maas-monitoring](https://github.com/rockocoop/openshiftai3/tree/main/maas/maas-monitoring)

YAML for a **Token metrics** tab on OpenShift AI **Observe & monitor → Dashboard**. It sorts **after Usage**.

On **3.5** that Dashboard page is GA. The **3.4** snapshot was Tech Preview.

| Folder | RHOAI | Metrics path | Apply |
|---|---|---|---|
| [3.4](3.4/) | 3.4 (Tech Preview) | UWM / OpenShift Thanos (`kuadrant-prometheus-datasource`, `authorized_hits`) | `oc apply -k maas-monitoring/3.4` |
| [3.5](3.5/) | 3.5 | COO only (`data-science-prometheus-datasource`, `authorized_hits_total`) | `oc apply -k maas-monitoring/3.5` |

Do not apply both: two `PersesDashboard` objects with the same name in different namespaces become two Token metrics tabs.

Leave the product Usage dashboard (`dashboard-3-maas-usage-admin`) in place. OpenShift AI lists any `PersesDashboard` whose name starts with `dashboard-`.

There is no kustomize overlay at this directory. Apply a version folder, not `oc apply -k maas-monitoring`.
