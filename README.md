# kubesentinel-gitops

Desired state of the `kubesentinel` cluster, synced by Argo CD. Application source code and the
cluster/Argo CD installation live in the [kubesentinel](https://github.com/AmineBarguellil7/kubesentinel) repo.

## Layout

```
bootstrap/
  root-app.yaml            App of apps. Applied once (by Terraform); points Argo CD at apps/
apps/
  demo-app.yaml            Demo shop -> apps/demo
  demo/                    Kustomize manifests: namespace, redis, payments, orders, api,
                           ServiceMonitor, PrometheusRule, Grafana dashboard
  monitoring-app.yaml      kube-prometheus-stack Helm chart (Prometheus, Alertmanager, Grafana)
  logging-app.yaml         Loki Helm chart, single binary, filesystem storage
  alloy-app.yaml           Grafana Alloy Helm chart, DaemonSet shipping pod logs to Loki
  metrics-server-app.yaml  metrics-server Helm chart, for `kubectl top`
```

- `bootstrap/root-app.yaml` is the only manifest applied from outside Git. It watches the top level
  of `apps/` (non-recursive) and creates every `Application` it finds there.
- `apps/demo-app.yaml` points at the plain manifests in `apps/demo/`, tied together by
  `kustomization.yaml`.
- The other Applications point straight at a Helm chart, pinned to a version, with their values
  inline (`spec.source.helm.valuesObject`). Nothing is installed with `helm install`.

All Applications sync automatically with `prune` and `selfHeal`: what is in `main` is what runs,
and manual changes in the cluster are reverted.

## Making a change

Edit a manifest, commit, push to `main`. Argo CD polls the repo about every 3 minutes; to sync
immediately:

```bash
kubectl -n argocd annotate application demo argocd.argoproj.io/refresh=normal --overwrite
```

## Adding an application

1. Put its manifests in `apps/<name>/`.
2. Add `apps/<name>-app.yaml` (copy `demo-app.yaml`, change the name, path and namespace). For a
   Helm chart, skip step 1 and copy `logging-app.yaml` instead.
3. Commit and push; the root app creates it.

## Observability

Everything runs in the `monitoring` namespace (metrics-server in `kube-system`) and is sized for a
laptop: Prometheus keeps 2 days of metrics, Loki 48 hours of logs, every container has small
requests and limits. Prometheus, Alertmanager and Grafana use `emptyDir`, so their data is lost when
the pod is recreated; Loki has a 2Gi volume.

| Component | What it does here |
| --- | --- |
| Prometheus | Scrapes the cluster and the demo services. Discovers `ServiceMonitor`, `PodMonitor` and `PrometheusRule` objects in every namespace, whatever their labels. |
| Alertmanager | Receives the alerts. No receiver is configured: alerts are only visible in its UI/API. |
| Grafana | Datasources: Prometheus (default), Alertmanager, Loki. Loads every ConfigMap labelled `grafana_dashboard: "1"` from any namespace. |
| Loki + Alloy | Alloy runs on each node, tails `/var/log/pods` and pushes to Loki with the labels `namespace`, `pod`, `container`, `app`, `node`. |

etcd, kube-scheduler, kube-controller-manager and kube-proxy scraping (and their alert rules) is
disabled: kind does not expose those endpoints.

### Access

```bash
kubectl -n monitoring port-forward svc/monitoring-grafana 3001:80                       # http://localhost:3001
kubectl -n monitoring port-forward svc/monitoring-kube-prometheus-prometheus 9090:9090  # http://localhost:9090
kubectl -n monitoring port-forward svc/monitoring-kube-prometheus-alertmanager 9093:9093
```

Grafana login: `admin` / `kubesentinel`. The password is a plain Helm value in
`apps/monitoring-app.yaml`, which is acceptable only because this cluster is local and throwaway.
Anywhere else it belongs in a Secret (`grafana.admin.existingSecret`).

### Demo shop

- `apps/demo/servicemonitor.yaml` scrapes `/metrics` on the `http` port of `api`, `orders` and
  `payments` every 15s. In queries the `service` label is the Kubernetes Service name
  (`api`, `orders`, `payments`).
- `apps/demo/prometheusrule.yaml` defines two alerts:

  | Alert | Fires when | Severity |
  | --- | --- | --- |
  | `DemoHighErrorRate` | more than 10% of a service's requests over 2m are 5xx, for 1m | critical |
  | `DemoPodCrashLooping` | a demo pod restarted more than 2 times in 10m | warning |

- `apps/demo/dashboards/demo-shop.json` is the Grafana dashboard "KubeSentinel - Demo shop"
  (requests/s, 5xx rate and share, p95 latency, pod restarts, memory per pod). Kustomize turns it
  into a ConfigMap; edit the JSON and push to change the dashboard.

Logs in Grafana: Explore, datasource Loki, for example `{namespace="demo", app="payments"}`.

## Notes

- The demo images (`kubesentinel-demo-*:dev`) are built locally and loaded into kind with
  `make images` in the kubesentinel repo. They are not in a registry.
- Bootstrap by hand, if Argo CD is installed without Terraform:
  `kubectl apply -n argocd -f bootstrap/root-app.yaml`
