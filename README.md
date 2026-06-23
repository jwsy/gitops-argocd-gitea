# Customer Workshop: Gitea + Argo CD Self-Hosting GitOps Bootstrap on Rancher Desktop

This workshop starts with an empty Rancher Desktop Kubernetes cluster and ends with a self-hosting GitOps environment:

```text
Gitea
  │
  ▼
gitops-argocd-gitea repository
  │
  ▼
Argo CD Root App
  │
  ├── Argo CD
  ├── Gitea
  └── jade-shooter
```

After bootstrap:

- Gitea hosts the GitOps repository.
- Argo CD reads the GitOps repository from Gitea.
- Argo CD manages itself.
- Argo CD manages Gitea.
- Argo CD manages the jade-shooter sample application.

This workshop uses:

```text
gitea.rancher.localhost
argocd.rancher.localhost
```

---

## Prerequisites

You need:

```text
Rancher Desktop
kubectl
helm
git
curl
```

Verify cluster access:

```bash
kubectl get nodes
```

Verify Traefik exists:

```bash
kubectl get svc traefik -n kube-system
```

---

## 0. Optional clean reset

Use this only if you want a clean workshop environment.

```bash
helm uninstall gitea -n gitea || true

kubectl delete statefulset,pod,svc,secret,configmap,pvc,ingress \
  -n gitea \
  -l app.kubernetes.io/instance=gitea \
  --ignore-not-found

kubectl delete namespace gitea --wait=true --ignore-not-found
kubectl delete namespace argocd --wait=true --ignore-not-found
```

---

## 1. Clone the workshop repository

The `start` branch contains the wrapper charts and bootstrap assets you need. Clone it and move into the directory:

```bash
git clone --branch start \
  https://github.com/jwsy/gitops-argocd-gitea.git \
  ~/code/gitops-argocd-gitea

cd ~/code/gitops-argocd-gitea
```

Verify the starting files:

```bash
find . -not -path './.git/*' -type f | sort
```

Expected:

```text
./.gitignore
./argocd/bootstrap-repo/commit-argocd-bootstrap-job.yaml
./argocd/bootstrap-root-app/gitea-repo-secret.yaml
./argocd/chart/Chart.yaml
./argocd/chart/values.yaml
./gitea/chart/Chart.yaml
./gitea/chart/values.yaml
```

---

## 2. Add Helm repositories

The wrapper charts declare upstream repos as dependencies. Add them so `helm dependency update` can resolve them:

```bash
helm repo add gitea-charts https://dl.gitea.com/charts/ || true
helm repo add argo https://argoproj.github.io/argo-helm || true
helm repo update
```

---

## 3. Review the Gitea wrapper chart

`gitea/chart/Chart.yaml` declares the upstream Gitea chart as a Helm dependency so the chart version and values both live in this repo:

```yaml
apiVersion: v2
name: gitea
version: 0.1.0
dependencies:
  - name: gitea
    version: 12.3.0
    repository: https://dl.gitea.com/charts/
```

`gitea/chart/values.yaml` contains the Gitea configuration. Values are nested under the dependency name (`gitea:`) so Helm scopes them to the sub-chart:

```yaml
gitea:
  gitea:
    admin:
      username: admin
      password: rootroot
      email: admin@localhost
  ...
```

---

## 4. Install Gitea

Resolve the chart dependency (downloads the upstream Gitea chart into `gitea/chart/charts/`, which is gitignored):

```bash
helm dependency update gitea/chart
```

Install from the local wrapper chart:

```bash
helm upgrade --install gitea \
  gitea/chart \
  --namespace gitea \
  --create-namespace
```

Watch startup:

```bash
kubectl get pods -n gitea -w
```

Expected:

```text
One Gitea pod eventually reaches Running.
```

Verify ingress:

```bash
kubectl get ingress -n gitea
curl -k -I https://gitea.rancher.localhost
```

Open:

```text
https://gitea.rancher.localhost
```

Login:

```text
admin
rootroot
```

---

## 5. Generate Gitea admin token

```bash
POD=$(kubectl get pod -n gitea \
  -l app.kubernetes.io/name=gitea \
  -o jsonpath='{.items[0].metadata.name}')

TOKEN=$(kubectl exec -n gitea "$POD" -- \
  gitea admin user generate-access-token \
    --username admin \
    --token-name bootstrap \
    --scopes all \
    --raw)
```

Store token in Kubernetes:

```bash
kubectl create secret generic gitea-admin-token \
  -n gitea \
  --from-literal=token="$TOKEN" \
  --dry-run=client \
  -o yaml | kubectl apply -f -
```

Verify:

```bash
kubectl get secret gitea-admin-token -n gitea
```

---

## 6. Create Gitea organization and repository

Create the `platform` organization:

```bash
curl -k \
  -H "Authorization: token $TOKEN" \
  -H "Content-Type: application/json" \
  -X POST \
  https://gitea.rancher.localhost/api/v1/orgs \
  -d '{"username":"platform"}' || true
```

Create the GitOps repository:

```bash
curl -k \
  -H "Authorization: token $TOKEN" \
  -H "Content-Type: application/json" \
  -X POST \
  https://gitea.rancher.localhost/api/v1/orgs/platform/repos \
  -d '{
    "name": "gitops-argocd-gitea",
    "private": false,
    "auto_init": false
  }' || true
```

---

## 7. Push the workshop repository to Gitea

Temporarily allow Git to tolerate the lab self-signed certificate:

```bash
git config --global http.https://gitea.rancher.localhost.sslVerify false
```

Point the repo at your Gitea and push:

```bash
git remote set-url origin \
  https://gitea.rancher.localhost/platform/gitops-argocd-gitea.git

git \
  -c http.extraHeader="Authorization: token $TOKEN" \
  push -u origin main
```

Expected output:

```
Enumerating objects: ...
Writing objects: 100% ...
To https://gitea.rancher.localhost/platform/gitops-argocd-gitea.git
 * [new branch]      main -> main
branch 'main' set up to track 'origin/main'.
```

---

## 8. Patch the Argo CD wrapper chart with your cluster IP

`argocd/chart/Chart.yaml` declares the upstream argo-cd chart as a dependency.

`argocd/chart/values.yaml` contains the ArgoCD configuration. Values are nested under the dependency name (`argo-cd:`).

⚠️ One value needs to match your cluster: the Traefik ClusterIP. ArgoCD's repo-server needs to reach Gitea through the cluster ingress, and the `hostAliases` entry makes `gitea.rancher.localhost` resolve correctly inside ArgoCD pods.

Get the IP and patch the values file:

```bash
GITEA_IP=$(
kubectl get svc traefik \
  -n kube-system \
  -o jsonpath='{.spec.clusterIP}'
)

sed -i.bak "s/ip: \".*\"/ip: \"$GITEA_IP\"/" argocd/chart/values.yaml
rm argocd/chart/values.yaml.bak
```

Verify:

```bash
grep -A2 hostAliases argocd/chart/values.yaml
```

Commit the patched values — this must reach Gitea before ArgoCD starts managing itself, otherwise the repo-server can't resolve `gitea.rancher.localhost`:

```bash
git add argocd/chart/values.yaml
git commit -m "Patch ArgoCD hostAlias with cluster Traefik IP"
```

Login will be:

```text
admin / rootroot
```

---

## 9. Install Argo CD

Resolve the chart dependency (downloads the upstream argo-cd chart into `argocd/chart/charts/`, which is gitignored):

```bash
helm dependency update argocd/chart
```

Install from the local wrapper chart:

```bash
helm upgrade --install argocd \
  argocd/chart \
  --namespace argocd \
  --create-namespace
```

Expected output:
```
Release "argocd" does not exist. Installing it now.
NAME: argocd
LAST DEPLOYED: Mon Jun 22 20:51:10 2026
NAMESPACE: argocd
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
TEST SUITE: None
NOTES:
In order to access the server UI you have the following options:

1. kubectl port-forward service/argocd-server -n argocd 8080:443

    and then open the browser on http://localhost:8080 and accept the certificate

2. enable ingress in the values file `server.ingress.enabled` and either
      - Add the annotation for ssl passthrough: https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#option-1-ssl-passthrough
      - Set the `configs.params."server.insecure"` in the values file and terminate SSL at your ingress: https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#option-2-multiple-ingress-objects-and-hosts


After reaching the UI the first time you can login with username: admin and the random password generated during the installation. You can find the password by running:

kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

(You should delete the initial secret afterwards as suggested by the Getting Started Guide: https://argo-cd.readthedocs.io/en/stable/getting_started/#4-login-using-the-cli)
```

Verify:

```bash
kubectl get pods -n argocd
kubectl get ingress -n argocd
curl -k -I https://argocd.rancher.localhost
```

Verify host alias on repo-server:

```bash
kubectl get deploy argocd-repo-server \
  -n argocd \
  -o yaml | grep -A6 hostAliases
```

Open:

```text
https://argocd.rancher.localhost
```

Login:

```text
admin
rootroot
```

---

## 10. Create Gitea credentials in Argo CD namespace

```bash
TOKEN=$(
kubectl get secret gitea-admin-token \
  -n gitea \
  -o jsonpath='{.data.token}' | base64 -d
)

kubectl create secret generic gitea-admin-creds \
  -n argocd \
  --from-literal=username=admin \
  --from-literal=password=rootroot \
  --from-literal=token="$TOKEN" \
  --dry-run=client \
  -o yaml | kubectl apply -f -
```

## 11. Push to Gitea

The bootstrap Job clones this repo from inside the cluster. Everything it needs — the wrapper charts and the patched `argocd/chart/values.yaml` — must be in Gitea before the Job runs.

Verify the remote:

```
git remote -v
```

Expected:

```
origin https://gitea.rancher.localhost/platform/gitops-argocd-gitea.git
```

You should have two unpushed commits at this point: the initial push from step 7 already happened, and step 8 added the IP patch commit. Push both:

```
git log --oneline origin/main..HEAD
```

Expected:

```
<hash> Patch ArgoCD hostAlias with cluster Traefik IP
```

Push to Gitea:

```
TOKEN=$(
kubectl get secret gitea-admin-token \
  -n gitea \
  -o jsonpath='{.data.token}' | base64 -d
)
git \
  -c http.sslVerify=false \
  -c http.extraHeader="Authorization: token $TOKEN" \
  push -u origin main
```

Validate the repository contains the bootstrap assets:

```
git ls-tree -r HEAD --name-only
```

Expected files include:

```
.gitignore
gitea/chart/Chart.yaml
gitea/chart/values.yaml
argocd/chart/Chart.yaml
argocd/chart/values.yaml
argocd/bootstrap-repo/commit-argocd-bootstrap-job.yaml
argocd/bootstrap-root-app/gitea-repo-secret.yaml
```
Optionally verify from Gitea:

```
https://gitea.rancher.localhost/platform/gitops-argocd-gitea
```

You should see the repository contents in the web UI.

⸻

## 12. Create the bootstrap commit Job

This job clones the GitOps repo from inside the cluster and commits the ArgoCD Application manifests into it. It uses the `alpine/git` image.

It creates:

```text
argocd/root-app/application.yaml
argocd/apps/argocd/application.yaml
argocd/apps/gitea/application.yaml
argocd/apps/jade-shooter/application.yaml
argocd/apps/jade-shooter/values.yaml
```

The ArgoCD and Gitea Applications point to the wrapper chart paths (`argocd/chart` and `gitea/chart`) in the GitOps repo, so ArgoCD resolves Helm dependencies and manages the releases entirely from Git.

The job manifest is at `argocd/bootstrap-repo/commit-argocd-bootstrap-job.yaml` and is already committed to the repo.

---

## 13. Run the bootstrap commit Job

```bash
kubectl -n argocd delete job commit-argocd-bootstrap --ignore-not-found
kubectl apply -f argocd/bootstrap-repo/commit-argocd-bootstrap-job.yaml
kubectl logs -n argocd job/commit-argocd-bootstrap  -f
```

Expected final log line:

```text
Cloning into 'repo'...
[main b164003] Add Argo CD root app, self-management, Gitea, and jade-shooter
 5 files changed, 120 insertions(+)
 create mode 100644 argocd/apps/argocd/application.yaml
 create mode 100644 argocd/apps/gitea/application.yaml
 create mode 100644 argocd/apps/jade-shooter/application.yaml
 create mode 100644 argocd/apps/jade-shooter/values.yaml
 create mode 100644 argocd/root-app/application.yaml
remote: . Processing 1 references
remote: Processed 1 references in total
To http://gitea-http.gitea.svc.cluster.local:3000/platform/gitops-argocd-gitea.git
   a009c4a..b164003  main -> main
```

---

## 14. Pull generated files locally

```bash
git -c http.sslVerify=false pull --rebase origin main
```

You should now have:

```bash
find argocd -maxdepth 3 -type f | sort
```

Expected files include:

```text
argocd/apps/argocd/application.yaml
argocd/apps/gitea/application.yaml
argocd/apps/jade-shooter/application.yaml
argocd/apps/jade-shooter/values.yaml
argocd/root-app/application.yaml
```

---

## 15. Create Argo CD repository Secret

Argo CD needs repository credentials before it can read the GitOps repo from Gitea.

```bash
TOKEN=$(
kubectl get secret gitea-admin-creds \
  -n argocd \
  -o jsonpath='{.data.token}' | base64 -d
)

cat > argocd/bootstrap-root-app/gitea-repo-secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: gitea-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository

stringData:
  url: https://gitea.rancher.localhost/platform/gitops-argocd-gitea.git
  username: admin
  password: "$TOKEN"
  insecure: "true"
EOF

kubectl apply -f argocd/bootstrap-root-app/gitea-repo-secret.yaml
```

---

## 15. Apply the root app

```bash
kubectl apply -f argocd/root-app/application.yaml
```

This creates the app-of-apps controller application:

```text
gitops-apps
```

The root app recursively watches:

```text
argocd/apps
```

and creates:

```text
argocd
gitea
jade-shooter
```

---

## 16. Validate applications

```bash
kubectl get applications -n argocd
```

Expected:

```text
NAME          SYNC STATUS   HEALTH STATUS
argocd        Synced        Healthy
gitea         Synced        Healthy
gitops-apps   Synced        Healthy
jade-shooter  Synced        Healthy
```

Check the workload:

```bash
kubectl get pods -n jade-shooter
```

Check Helm releases:

```bash
helm list -A
```

Expected releases include:

```text
argocd
gitea
```

---

## 17. Validate URLs

Open:

```text
https://gitea.rancher.localhost
https://argocd.rancher.localhost
```

Gitea login:

```text
admin
rootroot
```

Argo CD login:

```text
admin
rootroot
```

---

## 18. Demonstrate GitOps reconciliation

Change jade-shooter replica count:

```bash
sed -i.bak 's/replicaCount: 1/replicaCount: 2/' \
  argocd/apps/jade-shooter/values.yaml

rm -f argocd/apps/jade-shooter/values.yaml.bak

git add argocd/apps/jade-shooter/values.yaml

git commit -m "Scale jade-shooter to two replicas"

git -c http.sslVerify=false push
```

Watch Argo CD reconcile:

```bash
kubectl get application jade-shooter -n argocd -w
```

Verify pods:

```bash
kubectl get pods -n jade-shooter
```

---

## 19. Operating model after bootstrap

After bootstrap, treat Git as the source of truth.

Use Git for:

```text
Argo CD configuration
Gitea configuration
Application configuration
Ingress configuration
Replica counts
Helm values
```

Avoid direct manual changes:

```text
helm upgrade
kubectl edit
kubectl patch
kubectl delete application
```

Manual Helm and kubectl changes are reserved for disaster recovery or workshop reset.

---

## 20. Troubleshooting

### Gitea is not reachable from browser

Check ingress:

```bash
kubectl get ingress -n gitea
kubectl describe ingress gitea -n gitea
```

Check Traefik:

```bash
kubectl get pods -n kube-system
kubectl get svc traefik -n kube-system
```

Try:

```bash
curl -k -I https://gitea.rancher.localhost
```

---

### Argo CD cannot fetch repo

Check repository secret:

```bash
kubectl get secret gitea-repo -n argocd -o yaml
```

Check repo-server host aliases:

```bash
kubectl get deploy argocd-repo-server \
  -n argocd \
  -o yaml | grep -A6 hostAliases
```

Check repo-server logs:

```bash
kubectl logs -n argocd deploy/argocd-repo-server
```

---

### Bootstrap job cannot clone

Check token secret:

```bash
kubectl get secret gitea-admin-creds -n argocd
```

Check job logs:

```bash
kubectl logs -n argocd job/commit-argocd-bootstrap
```

Confirm Gitea repo URL:

```text
https://gitea.rancher.localhost/platform/gitops-argocd-gitea.git
```

---

### Application is OutOfSync

Inspect:

```bash
kubectl describe application <app-name> -n argocd
```

Common causes:

```text
Chart version mismatch
Values file typo
Repo credential issue
Self-signed certificate issue
Namespace missing
```

---

## 21. Workshop reset

```bash
kubectl delete namespace argocd --wait=true --ignore-not-found

helm uninstall gitea -n gitea || true

kubectl delete statefulset,pod,svc,secret,configmap,pvc,ingress \
  -n gitea \
  -l app.kubernetes.io/instance=gitea \
  --ignore-not-found
  
kubectl delete namespace gitea --wait=true --ignore-not-found
```

Remove local repo if desired:

```bash
rm -rf ~/code/gitops-argocd-gitea
```

---

## Final state

At the end of the workshop:

```text
Gitea hosts Git.
Argo CD reads Git from Gitea.
Argo CD manages itself.
Argo CD manages Gitea.
Argo CD manages jade-shooter.
```

This is the core self-hosted GitOps pattern.
