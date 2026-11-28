# kubesentinel-gitops

Desired state of the `kubesentinel` cluster, synced by Argo CD. Application source code and the
cluster/Argo CD installation live in the [kubesentinel](https://github.com/AmineBarguellil7/kubesentinel) repo.

## MVP scope

The platform is trimmed to the minimum to keep RAM low on a laptop:

- **Running:** the demo shop, Prometheus, Alertmanager, node-exporter, kube-state-metrics,
  metrics-server, the three demo alerts, and the KubeSentinel agent (incident -> AI diagnosis ->
  fix PR -> approval in Slack; read-only on the cluster).
- **Off:** Grafana (one-line switch, see [Re-enabling Grafana](#re-enabling-grafana)). The demo
  dashboard JSON stays in the repo.
- **Removed:** Loki, Alloy (logs) and Chaos Mesh. They are in the Git history if needed again.

## Layout

```
bootstrap/
  root-app.yaml            App of apps. Applied once (by Terraform); points Argo CD at apps/
apps/
  demo-app.yaml            Demo shop -> apps/demo
  demo/                    Kustomize manifests: namespace, redis, payments, orders, api,
                           ServiceMonitor, PrometheusRule, Grafana dashboard
  monitoring-app.yaml      kube-prometheus-stack Helm chart (Prometheus, Alertmanager; Grafana off)
  metrics-server-app.yaml  metrics-server Helm chart, for `kubectl top`
  agent-app.yaml           KubeSentinel agent -> apps/agent, namespace kubesentinel
  agent/                   ServiceAccount, read-only Role/RoleBinding in demo, Deployment, Service
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
   Helm chart, skip step 1 and copy `metrics-server-app.yaml` instead.
3. Commit and push; the root app creates it.

## Observability

Everything runs in the `monitoring` namespace (metrics-server in `kube-system`) and is sized for a
laptop: Prometheus keeps 2 days of metrics and every container has small requests and limits.
Prometheus and Alertmanager use `emptyDir`, so their data is lost when the pod is recreated.

| Component | What it does here |
| --- | --- |
| Prometheus | Scrapes the cluster and the demo services. Discovers `ServiceMonitor`, `PodMonitor` and `PrometheusRule` objects in every namespace, whatever their labels. |
| Alertmanager | Receives the alerts. Alerts with `namespace="demo"` go to the KubeSentinel agent's webhook, grouped by namespace (see below); everything else, Watchdog included, goes to the `null` receiver. |
| Grafana | **Disabled** (`grafana.enabled: false`). When on: datasources Prometheus (default) and Alertmanager; loads every ConfigMap labelled `grafana_dashboard: "1"` from any namespace. |

etcd, kube-scheduler, kube-controller-manager and kube-proxy scraping (and their alert rules) is
disabled: kind does not expose those endpoints.

### Access

```bash
kubectl -n monitoring port-forward svc/monitoring-kube-prometheus-prometheus 9090:9090  # http://localhost:9090
kubectl -n monitoring port-forward svc/monitoring-kube-prometheus-alertmanager 9093:9093
```

### Re-enabling Grafana

One line in `apps/monitoring-app.yaml`:

```yaml
        grafana:
          enabled: true   # was false
```

Commit and push. Argo CD deploys Grafana, and its sidecar picks up the demo dashboard ConfigMap
(still generated from `apps/demo/dashboards/demo-shop.json`). Then:

```bash
kubectl -n monitoring port-forward svc/monitoring-grafana 3001:80   # http://localhost:3001
```

Login: `admin` / `kubesentinel`. The password is a plain Helm value in `apps/monitoring-app.yaml`,
which is acceptable only because this cluster is local and throwaway. Anywhere else it belongs in a
Secret (`grafana.admin.existingSecret`). Set it back to `false` after recording.

### Demo shop

- `apps/demo/servicemonitor.yaml` scrapes `/metrics` on the `http` port of `api`, `orders` and
  `payments` every 15s. In queries the `service` label is the Kubernetes Service name
  (`api`, `orders`, `payments`).
- `apps/demo/prometheusrule.yaml` defines three alerts:

  | Alert | Fires when | Severity |
  | --- | --- | --- |
  | `DemoHighErrorRate` | more than 10% of a service's requests over 2m are 5xx, for 1m | critical |
  | `DemoPodCrashLooping` | a demo pod restarted more than 2 times in 10m | warning |
  | `DemoServiceDown` | api, orders or payments has no target up (scrape failing, or no pods at all) for 1m | critical |

- `apps/demo/dashboards/demo-shop.json` is the Grafana dashboard "KubeSentinel - Demo shop"
  (requests/s, 5xx rate and share, p95 latency, pod restarts, memory per pod). Kustomize turns it
  into a ConfigMap, which is still deployed while Grafana is off (it costs nothing), so the
  dashboard is there as soon as Grafana is re-enabled.

## KubeSentinel agent

`apps/agent/` runs the agent (code in `agent/` of the kubesentinel repo): one small pod in the
`kubesentinel` namespace that turns demo alerts into ONE Slack incident with an AI diagnosis and,
when it can, a fix PR on this repo that a human approves or rejects from Slack.

```
Alertmanager --webhook--> agent /alert
  gather_evidence -> diagnose (Groq) -> propose_fix (LLM + code reviewer -> PR) -> notify (Slack)
Slack "Approve and merge" -> squash-merge PR -> Argo CD deploys -> agent watches the rollout
```

- **One incident, one message.** `alertmanager.config` in `apps/monitoring-app.yaml` groups
  `namespace="demo"` alerts by `namespace` only (`group_wait: 30s`, `group_interval: 5m`,
  `send_resolved: true`), so alerts that fire together arrive as one group. The agent posts one
  message per group listing every firing alert, updates it when alerts join or clear, and replies
  in its thread when the group resolves. One incident = one diagnosis = at most one PR.
- **Root cause, not nearest dependency.** The prompt carries the call chain
  (api -> orders -> payments, orders -> redis), and code computes which services are themselves
  broken. The suspected service must be the first broken one in the chain; otherwise the model is
  asked again, then overridden with low confidence.
- **Fix PRs.** The LLM proposes one change; a plain-code reviewer accepts only, for a Deployment
  in a file under `apps/demo/`: `replicas` (1-5), the image **tag**, cpu/memory requests and
  limits, literal env var values, httpGet probe paths/ports. The old value must match `main`, and
  the diff must be at most 4 changed lines. Accepted changes are committed (formatting preserved)
  on a `fix/<alertname>-<incident id>` branch and opened as a PR with the diagnosis and the diff.
  An open KubeSentinel PR for the same change is reused, never duplicated.
- **Approval in Slack** (Socket Mode, no public endpoint). The incident message shows the PR link,
  the diff and two buttons. *Approve and merge* squash-merges the PR, deletes the branch, and the
  agent then watches the Deployment (read-only) until it runs the new value with all replicas
  ready, and posts the time to recovery in the thread. *Reject* closes the PR and deletes the
  branch. Each button works once. `SLACK_APPROVERS` (Slack user IDs) can restrict who may click.
- **Never pushes to main.** The agent writes only to `fix/*` branches; `main` changes only through
  the merge of a PR a human approved. Branch protection on `main` (require a PR) is a good second
  guard.
- **Read-only on the cluster.** The `agent` ServiceAccount is bound to a Role in `demo` with only
  `get/list/watch` on pods, pods/log, events, deployments and replicasets. No Secrets, no write
  verbs, no ClusterRole.
- **Secret not in Git.** The Deployment reads `GROQ_API_KEY`, `SLACK_BOT_TOKEN`,
  `SLACK_APP_TOKEN`, `SLACK_CHANNEL_ID` and `GITHUB_TOKEN` (fine-grained: this repo only,
  contents + pull requests read/write) from the Secret `kubesentinel-agent`, created from local
  files by `make agent-secrets` in the kubesentinel repo.
- **State is in memory** (MVP): a restart forgets open incidents; their buttons then answer
  "not known any more" and the PR has to be handled in GitHub.
- Image `kubesentinel-agent:dev` is built and loaded into kind by `make images`. Non-secret
  settings are env vars in `apps/agent/deployment.yaml`: `GROQ_MODEL`, `PROMETHEUS_URL`,
  `TARGET_NAMESPACE`, `GITOPS_REPO`, `GITOPS_BASE`, `SLACK_APPROVERS`.

Logs: `make agent-logs` in the kubesentinel repo.

## Notes

- The demo images (`kubesentinel-demo-*:dev`) are built locally and loaded into kind with
  `make images` in the kubesentinel repo. They are not in a registry.
- Bootstrap by hand, if Argo CD is installed without Terraform:
  `kubectl apply -n argocd -f bootstrap/root-app.yaml`
