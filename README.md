# kubesentinel-gitops

Desired state of the `kubesentinel` cluster, synced by Argo CD. Application source code and the
cluster/Argo CD installation live in the [kubesentinel](https://github.com/AmineBarguellil7/kubesentinel) repo.

## Layout

```
bootstrap/
  root-app.yaml      App of apps. Applied once (by Terraform); points Argo CD at apps/
apps/
  demo-app.yaml      Argo CD Application for the demo shop -> apps/demo
  demo/              Kustomize manifests: namespace, redis, payments, orders, api
```

- `bootstrap/root-app.yaml` is the only manifest applied from outside Git. It watches the top level
  of `apps/` (non-recursive) and creates every `Application` it finds there.
- Each `apps/<name>-app.yaml` is one Application pointing at its `apps/<name>/` folder.
- `apps/demo/` holds plain Kubernetes manifests tied together by `kustomization.yaml`.

Both Applications sync automatically with `prune` and `selfHeal`: what is in `main` is what runs,
and manual changes in the cluster are reverted.

## Making a change

Edit a manifest, commit, push to `main`. Argo CD polls the repo about every 3 minutes; to sync
immediately:

```bash
kubectl -n argocd annotate application demo argocd.argoproj.io/refresh=normal --overwrite
```

## Adding an application

1. Put its manifests in `apps/<name>/`.
2. Add `apps/<name>-app.yaml` (copy `demo-app.yaml`, change the name, path and namespace).
3. Commit and push; the root app creates it.

## Notes

- The demo images (`kubesentinel-demo-*:dev`) are built locally and loaded into kind with
  `make images` in the kubesentinel repo. They are not in a registry.
- Bootstrap by hand, if Argo CD is installed without Terraform:
  `kubectl apply -n argocd -f bootstrap/root-app.yaml`
