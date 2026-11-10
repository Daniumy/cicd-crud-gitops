# cicd-crud-gitops

Desired state of the cluster. Nothing is applied by hand from here except `infra/`: ArgoCD watches `main` and makes the cluster match it.

```mermaid
flowchart TD
  jenkins(["Jenkins<br/>(from the webapp repo)"]) -- "commit: newTag = sha" --> overlay

  subgraph REPO["This repo (branch main)"]
    direction TB
    appdev["argocd/app-dev.yaml"]
    appprod["argocd/app-prod.yaml"]
    base["app/base<br/>Deployment, Service"]
    overlay["app/overlays/dev | prod<br/>namespace, Ingress,<br/>config.js, image + newTag"]
    infra["infra/<br/>ClusterIssuer, ArgoCD ingress"]
    base --> overlay
  end

  appdev -- "path: app/overlays/dev" --> argo
  appprod -- "path: app/overlays/prod" --> argo
  argo["ArgoCD<br/>auto-sync, prune, selfHeal"]

  argo -- "kustomize build + apply" --> ns

  subgraph ns["Cluster: one namespace per env"]
    direction LR
    dep["Deployment web<br/>nginx image"]
    cm["ConfigMap web-config<br/>mounted as /config.js"]
    svc["Service web"]
    ing["Ingress<br/>TLS via cert-manager"]
    ing --> svc --> dep
    cm -.-> dep
  end

  infra -. "kubectl apply (manual, not watched by ArgoCD)" .-> cluster["cert-manager + ArgoCD UI exposure"]
```

## Layout

| Path | What it does |
|---|---|
| `app/base/` | Env-neutral `Deployment` + `Service`. Image is a placeholder name (`chollos-web`). |
| `app/overlays/dev/`, `prod/` | Per-env namespace, host/Ingress, `config.js` (banner, colors, email) and the real image + `newTag`. |
| `argocd/` | Two ArgoCD `Application`s, one per overlay, both auto-sync with prune and self-heal. |
| `infra/` | Cluster-wide pieces: Let's Encrypt `ClusterIssuer` and the Ingress for the ArgoCD UI. |

## How a deploy happens

- **Image bump:** Jenkins rewrites `newTag:` in the overlay's `kustomization.yaml`. That commit *is* the deploy.
- **Config change:** edit the overlay's `config.js`. Kustomize hashes the ConfigMap name, so pods restart by themselves.
- **Drift:** manual `kubectl` edits get reverted (`selfHeal`); files deleted from Git get deleted from the cluster (`prune`).
- **Rollback:** `git revert` the `deploy(...)` commit.
