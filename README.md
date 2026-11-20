# Yarrow-Mullein Bank: Kubernetes GitOps Deployment

**COMP-4001: Developing in a DevOps Environment | Module 6, Part 3**
**Student:** Olamilekan Ajose | Application Development and Delivery, RRC Polytech
**GitHub:** [@oajose6073](https://github.com/oajose6073)

A microservice banking application deployed to Kubernetes through a fully automated CI/CD pipeline. A `git push` to `main` builds and tests the code, publishes new container images, updates the deployment manifests, and Argo CD rolls the change out to the cluster with no manual steps.

---

## About me

I'm an Application Development and Delivery student at RRC Polytech in Winnipeg, focused on full-stack and back-end development. This project brings together everything from the DevOps course so far: Linux and Vagrant, Docker and Docker Compose, Bash automation, Kubernetes, and now GitOps with GitHub Actions and Argo CD.

---

## Architecture

```
Browser
   │
   ▼
nginx (NodePort 30080)  ── reverse proxy, single entry point
   │
   ├── /api/transactions/ ──► transactions (Node.js / Express) ──► mongo-transactions
   ├── /api/              ──► backend (Python / Flask)        ──► mongo-backend
   └── /                  ──► studentportfolio (static HTML)

syncer ── copies transactions from mongo-backend to mongo-transactions on a schedule
```

| Service | Language / image | Replicas | Purpose |
|---|---|---|---|
| `nginx` | nginx:alpine | 2 | Routes traffic by URL path |
| `backend` | Python / Flask | 3 (HPA 3 to 10) | Register, login, deposit, withdraw |
| `transactions` | Node.js / Express | 2 (HPA 2 to 8) | Monthly transaction reports |
| `studentportfolio` | nginx (static) | 2 | Student portfolio front end |
| `syncer` | Python | 1 | Keeps the transactions database in sync |
| `mongo-backend` | mongo:4.4.18 (StatefulSet) | 1 | Banking data |
| `mongo-transactions` | mongo:4.4.18 (StatefulSet) | 1 | Transaction report data |

---

## CI/CD pipeline

```
git push ──► GitHub Actions ──► GHCR ──► GitOps repo ──► Argo CD ──► Kubernetes
```

1. **Build:** GitHub Actions builds an image for each service that has a Dockerfile.
2. **Push:** each image is pushed to the GitHub Container Registry, tagged with `latest` and with the commit SHA, so every build is unique and traceable.
3. **Update manifests:** the workflow copies `k8s/` into the GitOps repository and changes every image tag to the new commit SHA, then commits and pushes.
4. **Deploy:** Argo CD watches the GitOps repository, sees the new commit, and syncs the cluster. Kubernetes performs a rolling update.

Argo CD is set to **auto-sync with self-heal and prune**, so any manual change made in the cluster is reverted to match Git, and resources removed from Git are removed from the cluster.

---

## Repository layout

```
.
├── .github/workflows/ci-cd.yml   # CI pipeline (build, push, update GitOps repo)
├── backend/                      # Flask banking API + tests
├── transactions/                 # Node.js transactions service
├── studentportfolio/             # Static portfolio site
├── syncer/                       # Database sync job
├── k8s/                          # Kubernetes manifests
├── nginx.conf                    # Reverse proxy routing rules
└── apply_minikube_dockerenv.sh   # Local deployment script (Part 2)
```

---

## Required secrets

Set in **Settings → Secrets and variables → Actions**:

| Secret | Purpose |
|---|---|
| `CR_PAT` | Token with `repo`, `workflow` and `write:packages` scopes, used to push images to GHCR |
| `GITOPS_PAT` | Token with `repo` scope, used to push updated manifests to the GitOps repository |
| `GITOPS_REPO` | The GitOps repository, e.g. `oajose6073/module_6_part3gitops` |

---

## Running it

**Install Argo CD with Helm:**
```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
minikube start --driver=docker
kubectl create namespace argocd
helm install argocd argo/argo-cd --namespace argocd
```

**Connect Argo CD to the GitOps repository and create the registry secret:**
```bash
kubectl apply -f argocd-application.yaml
kubectl create secret docker-registry ghcr-creds --namespace default --docker-server=ghcr.io --docker-username=oajose6073 --docker-password=<CR_PAT>
```

**Open the Argo CD UI:**
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:80
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```
Log in at `http://localhost:8080` as `admin`.

**Open the banking app:**
```bash
minikube service nginx
```

---

## Tech stack

Python (Flask) · Node.js (Express) · MongoDB · nginx · Docker · Kubernetes · Minikube · Helm · GitHub Actions · GitHub Container Registry · Argo CD
