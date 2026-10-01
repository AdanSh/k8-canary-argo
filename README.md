# k8-canary-argo

A local lab for canary deployments: Argo CD for GitOps, Argo Rollouts for the canary steps, and NGINX Ingress for traffic splitting. It runs on k3d.

```
Git (values.yaml) ──► Argo CD ──► Rollout ──► Argo Rollouts ──► ingress-nginx weights
                      (sync)       (spec)      (20% → 50% → 100%)
```

## Layout

```
argocd/demo-app.yaml          # Argo CD Application (bootstrap once)
canary-lab/helm/demo-app/     # Helm chart: Rollout, stable/canary Services, Ingress
  values.yaml                 # image.tag = the version that runs
```

## Prerequisites

You need Docker, k3d, kubectl, Helm, and the `kubectl-argo-rollouts` plugin. On Windows, run everything in WSL2 with Docker Desktop's WSL integration enabled.

## Setup

```bash
# Cluster (Traefik disabled so ingress-nginx can own ports 80/443)
k3d cluster create canary -p "80:80@loadbalancer" -p "443:443@loadbalancer" \
  --k3s-arg "--disable=traefik@server:0" --wait

# ingress-nginx
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
kubectl wait -n ingress-nginx --for=condition=ready pod -l app.kubernetes.io/component=controller --timeout=180s

# Argo Rollouts (server-side apply: its CRDs are too large for client-side apply)
kubectl create namespace argo-rollouts
kubectl apply --server-side -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# Argo CD
kubectl create namespace argocd
kubectl apply --server-side -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait -n argocd --for=condition=available deploy --all --timeout=300s

# Hosts entry (also add it to C:\Windows\System32\drivers\etc\hosts for a Windows browser)
echo "127.0.0.1 demo.local" | sudo tee -a /etc/hosts

# Bootstrap the app
kubectl apply -f argocd/demo-app.yaml
```

## Argo CD UI

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
kubectl port-forward -n argocd svc/argocd-server 8080:443
# https://localhost:8080  (user: admin)
```

## Releasing a new version (GitOps)

1. Change `image.tag` in `canary-lab/helm/demo-app/values.yaml`, for example `green` to `blue`.
2. Commit and push to `main`.
3. Argo CD syncs. It polls about every 3 minutes, or you can click **Refresh** in the UI.
4. Argo Rollouts sends 20% of traffic to the new version and then pauses.
5. Test, then promote. You can use **Resume** on the Rollout in the Argo CD UI, or:
   ```bash
   kubectl argo rollouts promote demo-app -n canary-demo
   ```
6. The rollout goes to 50%, waits 30s, goes to 100%, and the new version becomes stable.

Watch and test:

```bash
kubectl argo rollouts get rollout demo-app -n canary-demo --watch
for i in $(seq 100); do curl -s demo.local/color; echo; done | sort | uniq -c
curl -s -H "X-Canary: true" demo.local/color
```

## Rolling back

- **During the canary:** `kubectl argo rollouts abort demo-app -n canary-demo` sends traffic back to stable straight away.
- **Afterwards:** `git revert` the tag change and push. Git stays the source of truth.

`selfHeal` is on, so manual changes such as `kubectl argo rollouts set image` are reverted to what Git says. Make version changes in Git.

## Cleanup

```bash
k3d cluster delete canary
sudo sed -i '/demo.local/d' /etc/hosts
```
