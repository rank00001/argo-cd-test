# argocd-test — a free, local Argo CD GitOps demo

Learn Argo CD end-to-end on Docker Desktop Kubernetes with a dummy app (no build, no
registry, no Azure). A tiny `http-echo` container serves one line of text; you change
that text on a branch, push, and watch Argo CD update the cluster.

- **stage** branch → Argo app `hello-stage` → namespace `demo-stage` (**auto-sync**)
- **main** branch → Argo app `hello-prod` → namespace `demo-prod` (**manual sync**)

Everything here is **free**: Docker Desktop k8s + Argo CD (open source) + a public
GitHub repo + the public `hashicorp/http-echo` image.

```
argocd-test/                 (this folder = the GitHub repo  rank00001/argo-cd-test)
├── manifests/               ← what Argo DEPLOYS (Argo watches this path)
│   ├── deployment.yaml      ← the -text line you edit to test
│   └── service.yaml
└── argocd/                  ← the two Argo Applications (you register these once)
    ├── app-stage.yaml
    └── app-prod.yaml
```

---

## 0. Prerequisites
- Docker Desktop → Settings → Kubernetes → **Enabled** (green).
- `kubectl config use-context docker-desktop`
- A GitHub account (repo `https://github.com/rank00001/argo-cd-test` — public).

---

## 1. Push this folder to GitHub (branches: main + stage)

Run from **inside** this `argocd-test/` folder — it becomes the repo root, so
`path: manifests` in the Argo apps resolves.

```bash
cd "D:/clean-kit-gateway-frontend/kubernetes-deploy-test/argocd-test"

git init -b main
git add .
git commit -m "hello gitops demo"
git remote add origin https://github.com/rank00001/argo-cd-test.git
git push -u origin main

# create the stage branch from main and push it too
git branch stage
git push -u origin stage
```

You now have two identical branches. (They diverge later — that's the test.)

---

## 2. Install Argo CD into the local cluster

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait -n argocd --for=condition=available --timeout=180s deploy --all
```

Open the Argo CD UI (optional but nice to watch):

```bash
# initial admin password:
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d ; echo

# port-forward the UI, then open https://localhost:8080  (user: admin)
kubectl port-forward -n argocd svc/argocd-server 8080:443
```

> Public repo = no credentials to configure. Argo can clone it anonymously.

---

## 3. Register the two Applications

```bash
kubectl apply -f argocd/app-stage.yaml
kubectl apply -f argocd/app-prod.yaml

kubectl get applications -n argocd        # STATUS should reach Synced / Healthy (stage)
```

- `hello-stage` auto-syncs → `demo-stage` comes up on its own.
- `hello-prod` shows **OutOfSync** and waits (manual). Sync it once to start it:
  ```bash
  # via UI: click hello-prod → Sync.  Or install the CLI and:
  # argocd app sync hello-prod
  ```

---

## 4. See it running

```bash
kubectl get pods -n demo-stage
kubectl get pods -n demo-prod

# view stage (in one terminal):
kubectl port-forward -n demo-stage svc/hello 8081:5678
curl http://localhost:8081        # → "Hello from GitOps ... (v1)"

# view prod (in another terminal):
kubectl port-forward -n demo-prod svc/hello 8082:5678
curl http://localhost:8082
```

---

## 5. THE TEST — change stage, watch only stage update

1. On the **stage** branch, edit `manifests/deployment.yaml` — change the `-text` line,
   e.g. `... (v2 STAGE)`:
   ```bash
   git checkout stage
   # edit the -text= value in manifests/deployment.yaml
   git commit -am "stage: bump message to v2"
   git push
   ```
2. Argo CD notices the new commit on `stage` (within ~3 min, or click **Refresh** in the
   UI to check now) and **auto-syncs**. The pod rolls over.
3. Verify — only stage changed:
   ```bash
   curl http://localhost:8081     # → "... (v2 STAGE)"   ← updated
   curl http://localhost:8082     # → "... (v1)"          ← prod untouched
   ```

That's the whole point: **the branch is the environment.** `main` didn't change, so
prod didn't move.

---

## 6. Promote to prod (the manual gate)

```bash
git checkout main
git merge stage
git push
```
Now `hello-prod` goes **OutOfSync** (main changed) but does **not** deploy on its own.
Release it deliberately:
```bash
# UI: hello-prod → Sync.   (or  argocd app sync hello-prod)
curl http://localhost:8082     # → "... (v2 STAGE)"  after you sync
```

---

## 7. Bonus — see self-heal

With `hello-stage` running, hand-edit the live deployment:
```bash
kubectl -n demo-stage scale deploy/hello --replicas=3
```
Because the Application has `selfHeal: true`, Argo reverts it back to `replicas: 1`
(what Git says) within seconds. Git is the source of truth.

---

## 8. Cleanup

```bash
kubectl delete -f argocd/app-stage.yaml -f argocd/app-prod.yaml
kubectl delete namespace demo-stage demo-prod
# remove Argo CD itself if you want:
kubectl delete namespace argocd
```

---

## What this teaches (maps 1:1 to the real setup)

| Here (demo) | Real (`kubernetes-deploy-test`) |
|-------------|--------------------------------|
| `hello-stage` watches `stage` → `demo-stage` | `rapid-stage` watches `stage` → `rapid-stage` ns |
| `hello-prod` watches `main` → `demo-prod`, manual | `rapid-prod` watches `main` → `rapid-prod` ns, manual |
| edit `-text`, push → Argo syncs | pipeline bumps image tag, push → Argo syncs |
| `selfHeal` reverts manual drift | same |
