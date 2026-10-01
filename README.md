# k8-canary-argo

A local lab for automated canary deployments on k3d:

- **Argo CD** syncs the app from this repo (GitOps).
- **Argo Rollouts** runs the canary steps, and a smoke test promotes or aborts automatically.
- **ingress-nginx** splits traffic by weight, header or cookie.

```
git push ─► Argo CD ─► Rollout ─► Argo Rollouts ─► ingress-nginx canary weight 20% → 50% → 100%
            (sync)     (spec)     + background smoke test (Job curls the canary every 10s)
```

## Repository layout

```
argocd/demo-app.yaml                      # Argo CD Application (applied once, at bootstrap)
canary-lab/helm/demo-app/                 # Helm chart for the app
├── Chart.yaml                            # chart name and version
├── values.yaml                           # image.tag, replicas, steps, analysis: what you change
└── templates/
    ├── _helpers.tpl                      # shared names and labels
    ├── rollout.yaml                      # Rollout: pod template + canary strategy
    ├── service-stable.yaml               # Service for the stable ReplicaSet
    ├── service-canary.yaml               # Service for the canary ReplicaSet
    ├── ingress.yaml                      # Ingress demo.local -> stable Service
    ├── analysistemplate.yaml             # smoke test used during the canary
    └── NOTES.txt
```

---

## 1. Prerequisites (Windows 11 + WSL2)

Run every command in this README in the **WSL2 Ubuntu** terminal, from the repo root
(`/mnt/c/Projects/K8-Argo`).

### Docker Desktop with WSL integration

1. Install and start Docker Desktop.
2. Go to Settings → Resources → WSL Integration and turn on your Ubuntu distro, then click Apply & Restart.
3. Check:
   ```bash
   which docker     # /usr/bin/docker
   docker info      # must show a server version
   ```
   If `/var/run/docker.sock` is missing, restart the distro from PowerShell
   (`wsl --terminate Ubuntu-24.04`) or toggle the integration off and on.

### CLI tools (installed to `~/.local/bin`, no sudo)

`~/.local/bin` must be on your `PATH`. Ubuntu's `~/.profile` adds it once the folder exists; open a new terminal after creating it.

```bash
mkdir -p ~/.local/bin

# k3d
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh \
  | USE_SUDO=false K3D_INSTALL_DIR="$HOME/.local/bin" bash
k3d version

# Helm (only needed for linting and rendering the chart locally)
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4 \
  | USE_SUDO=false HELM_INSTALL_DIR="$HOME/.local/bin" bash
helm version --short

# kubectl: use the one from Docker Desktop, or any recent kubectl
kubectl version --client
```

---

## 2. Cluster

k3s ships Traefik, which would take ports 80/443, so it's disabled here to let ingress-nginx have them.

```bash
k3d cluster create canary \
  -p "80:80@loadbalancer" -p "443:443@loadbalancer" \
  --k3s-arg "--disable=traefik@server:0" \
  --wait

kubectl config current-context    # k3d-canary
kubectl get nodes                 # Ready
```

After a reboot: `k3d cluster start canary`.

## 3. ingress-nginx

Use the **cloud** manifest. k3s's built-in load balancer exposes the controller's LoadBalancer Service on ports 80/443, and k3d maps those to localhost.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
kubectl wait -n ingress-nginx --for=condition=ready pod \
  -l app.kubernetes.io/component=controller --timeout=180s

curl -s -o /dev/null -w "%{http_code}\n" localhost    # 404 = nginx is answering
```

The ingress-nginx project was retired in March 2026 and gets no more fixes. That's fine for a local lab.

## 4. Argo Rollouts controller and kubectl plugin

Use `--server-side`. The `rollouts` and `analysisruns` CRDs are too large for client-side apply, which fails on them with `metadata.annotations: Too long`.

```bash
kubectl create namespace argo-rollouts
kubectl apply --server-side -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
kubectl wait -n argo-rollouts --for=condition=available deploy/argo-rollouts --timeout=180s

kubectl get crd | grep argoproj.io    # rollouts, analysisruns, analysistemplates, ...

# kubectl plugin
curl -Lo ~/.local/bin/kubectl-argo-rollouts \
  https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x ~/.local/bin/kubectl-argo-rollouts
kubectl argo rollouts version
```

## 5. Argo CD

```bash
kubectl create namespace argocd
kubectl apply --server-side -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait -n argocd --for=condition=available deploy --all --timeout=300s
```

## 6. Hosts entry for `demo.local`

```bash
# WSL (curl). WSL regenerates /etc/hosts from the Windows file on restart.
echo "127.0.0.1 demo.local" | sudo tee -a /etc/hosts
```

For a Windows browser, and to keep the entry across WSL restarts, also add
`127.0.0.1 demo.local` to `C:\Windows\System32\drivers\etc\hosts` using an editor run as administrator.

## 7. Bootstrap the app (once)

```bash
kubectl apply -f argocd/demo-app.yaml

kubectl get application demo-app -n argocd -w    # wait for Synced / Healthy, then Ctrl+C
kubectl argo rollouts get rollout demo-app -n canary-demo
curl -s demo.local/color; echo                   # the color in values.yaml image.tag
```

From now on, **Git is the only way to change the app**. Argo CD has `automated`, `prune` and `selfHeal` turned on. It deploys every commit to `main` under `canary-lab/helm/demo-app/`, and it reverts manual `kubectl` changes.

`argocd/demo-app.yaml` isn't watched by Argo CD. If you edit it, apply it again with `kubectl apply`.

---

## 8. Releasing a new version

1. Change `image.tag` in `canary-lab/helm/demo-app/values.yaml`. The `argoproj/rollouts-demo` image has the tags `blue`, `green`, `yellow`, `red`, `purple` and `orange`.
2. Commit and push:
   ```bash
   git commit -am "release yellow" && git push
   ```
3. Argo CD notices within 2–3 minutes. To make it check now:
   ```bash
   kubectl annotate application demo-app -n argocd argocd.argoproj.io/refresh=normal --overwrite
   ```
4. Argo Rollouts runs the steps with no further commands:
   ```
   20% ─► 1m ─► 50% ─► 1m ─► 100% ─► new version becomes stable, old ReplicaSet scales down after 30s
   ```
   During the whole canary, a background **AnalysisRun** starts a Job every 10s that runs
   `curl -f http://demo-app-canary/color`. Two failures (`failureLimit: 1`) **abort** the rollout,
   and traffic goes back to stable straight away. If canary pods don't become ready within 120s (for example, a bad image tag), the rollout also aborts (`progressDeadlineAbort`).

### Seeing an automatic abort

Set a bad check path and a new tag in `values.yaml`, then commit and push:

```yaml
image:
  tag: red
analysis:
  path: /broken
```

Within about 20s the AnalysisRun fails, the rollout is aborted, and the app shows Degraded. Fix it in Git by putting `path: /color` back and choosing the tag you want, or with `git revert HEAD && git push`.

### Manual controls (optional)

```bash
kubectl argo rollouts promote demo-app -n canary-demo          # skip the current pause
kubectl argo rollouts promote demo-app -n canary-demo --full   # skip to 100%
kubectl argo rollouts abort   demo-app -n canary-demo          # back to stable now
```

For a manual gate, use `pause: {}` in `canary.steps`. The rollout then waits for a `promote`.

After an abort, Git still holds the rejected tag. Revert or fix the commit so Git matches the cluster again.

---

## 9. Watching a release

Use one terminal for each:

```bash
# Argo CD: OutOfSync -> Synced/Progressing -> Healthy (Suspended while a pause: {} waits)
kubectl get application demo-app -n argocd -w

# Canary steps, weights, ReplicaSets, AnalysisRun
kubectl argo rollouts get rollout demo-app -n canary-demo --watch

# Real traffic split, refreshed every 2s
while true; do
  for i in $(seq 50); do curl -s demo.local/color; echo; done | sort | uniq -c | tr '\n' ' '
  echo; sleep 2
done
```

Routing checks during a canary:

```bash
curl -s -H "X-Canary: true" demo.local/color; echo   # always the canary
curl -s -b "canary=always"  demo.local/color; echo   # always the canary
curl -s -b "canary=never"   demo.local/color; echo   # always stable
```

Analysis results:

```bash
kubectl get analysisrun -n canary-demo
kubectl describe analysisrun -n canary-demo <name>
```

Has the new version become stable? It has when `stableRS` equals `currentPodHash`:

```bash
kubectl get rollout demo-app -n canary-demo \
  -o jsonpath='stable={.status.stableRS}  current={.status.currentPodHash}{"\n"}'
```

## 10. UIs

**Argo CD:**

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
kubectl port-forward -n argocd svc/argocd-server 8080:443
```

Open https://localhost:8080 and log in as `admin`.

**Argo Rollouts dashboard:**

```bash
kubectl argo rollouts dashboard -n canary-demo
```

Open http://localhost:3100.

**The app:** http://demo.local/. Its bubbles change color as traffic shifts.

---

## 11. Validating chart changes locally

```bash
helm lint canary-lab/helm/demo-app
helm template demo-app canary-lab/helm/demo-app -n canary-demo
helm template demo-app canary-lab/helm/demo-app -n canary-demo \
  | kubectl apply --dry-run=server -n canary-demo -f -
```

Bump `version` in `Chart.yaml` when templates or the structure of `values.yaml` change. A tag change alone doesn't need a bump.

## 12. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `no matches for kind "Rollout"` | Argo Rollouts CRDs missing. Re-run step 4 with `--server-side` |
| `503` from `demo.local` right after deploy | Pods are still pulling the image. Wait for `Healthy` |
| Canary stuck at `ActualWeight: 0`, pod `ImagePullBackOff` | Bad image tag. It aborts after 120s; fix the tag in Git |
| `.spec.selector is immutable` | Resources with the same names exist from another deploy method. Delete the namespace and let Argo CD recreate it |
| `kubectl argo rollouts set image` has no lasting effect | Expected: `selfHeal` restores Git's tag. Release through Git |
| `helm list` shows nothing | Expected: Argo CD renders the chart with `helm template` and applies it itself |

Useful logs:

```bash
kubectl get events -n canary-demo --sort-by=.lastTimestamp | tail -20
kubectl logs -n argo-rollouts deploy/argo-rollouts -f | grep demo-app
kubectl logs -n argocd statefulset/argocd-application-controller -f | grep demo-app
```

---

## 13. Cleanup

Pick the level you need. Each level also covers everything in the levels above it. To remove the whole lab but keep the tools and repo, use levels 2 and 3.

### Level 1: just the app (keep the cluster and controllers)

Delete the **Application first**. While it exists, Argo CD (`automated` + `selfHeal` +
`CreateNamespace`) recreates anything you delete within seconds. Its finalizer removes everything it deployed: the Rollout (and with it the ReplicaSets, pods and canary Ingress), the Services, the Ingress and the AnalysisTemplate.

```bash
kubectl delete application demo-app -n argocd
kubectl delete namespace canary-demo      # CreateNamespace doesn't delete the namespace
```

### Level 2: the whole cluster

This removes the app, Argo CD, Argo Rollouts and ingress-nginx, plus the k3d containers, the Docker network and the `k3d-canary` kubeconfig context.

```bash
k3d cluster delete canary

k3d cluster list                  # canary should be gone
docker ps --filter name=k3d       # no containers
```

### Level 3: hosts entries

```bash
sudo sed -i '/demo.local/d' /etc/hosts
```

Also remove the `127.0.0.1 demo.local` line from `C:\Windows\System32\drivers\etc\hosts`, using an editor run as administrator.

### Level 4: CLI tools (optional)

```bash
rm ~/.local/bin/k3d ~/.local/bin/helm ~/.local/bin/kubectl-argo-rollouts
```

### Level 5: leftover Docker images (optional, frees disk space)

```bash
docker image ls | grep -E 'rancher/k3s|ghcr.io/k3d-io'
docker image rm <image:tag> ...   # use the tags listed above
```

### Level 6: the code (permanent)

- Local folder: `rm -rf /mnt/c/Projects/K8-Argo`. Push anything you want to keep first.
- GitHub repo: `gh repo delete AdanSh/k8-canary-argo`, or on github.com go to Settings → Danger Zone → Delete.
