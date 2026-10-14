# dashr-nova-deploy

What runs where. Flux watches this repo (wired up in dashr-platform-gitops under
`tenants/dashr-nova/`) and applies `overlays/<env>` to the `dashr-nova`
namespace. Nothing here watches a registry: a deployment is a commit.

```
base/                    GitRepository for the chart + HelmRelease defaults
overlays/local/          loaded image (pullPolicy Never), chart from main
overlays/staging/        pinned chart tag + image tag
overlays/production/     pinned chart tag + image tag, protected by CODEOWNERS
```

## Who provides what

| Values | Provided by | Where |
|---|---|---|
| Image, replicas, resources, `APP_ENV` | This repo | `overlays/<env>/helmrelease-patch.yaml` |
| Chart version | This repo | GitRepository `ref` in `overlays/<env>/kustomization.yaml` |
| Database Secret, Temporal address, OPA URL, hostname | The platform | ConfigMap `dashr-nova-platform-values`, read via `valuesFrom` |
| Credentials | The platform | OpenBao, synced to Secret `dashr-nova-db` by ESO |

This repo never contains endpoints or credentials. If you must override a
platform value for one environment, set it under `spec.values` in that
overlay: it takes precedence over `valuesFrom`.

## Deploy a new version

| To change | Edit |
|---|---|
| Application image | `image.tag` in `overlays/<env>/helmrelease-patch.yaml` |
| Helm chart | the `tag` in the GitRepository patch in `overlays/<env>/kustomization.yaml` |

```bash
git commit -am "dashr-nova 0.1.1 to staging" && git push
flux reconcile kustomization dashr-nova -n dashr-nova --with-source   # optional, skip the wait
helm test dashr-nova -n dashr-nova                                    # check dependencies
```

**Promote to production** by copying the image tag (and chart tag) verified
in staging, in a pull request.

**Roll back** with `git revert`. If an upgrade fails, Flux rolls the release
back automatically; still revert the commit so Git matches the cluster.

## What the platform provides

The `dashr-nova` namespace, the deploy service account (admin of this
namespace only), the platform values ConfigMap, the `dashr-nova-db` Secret,
the shared Istio gateway, and image policies (allowed registry prefix, no
`:latest`).
