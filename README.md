# helm-charts-devops-toolchain

Hand-written Helm charts for a self-hosted CI/CD toolchain on Kubernetes: **Jenkins**, **Argo CD**, **Nexus Repository** and **SonarQube** with its own **PostgreSQL**. Each chart is standalone and installs into its own namespace behind an NGINX Ingress.

> **Lab / learning use.** These charts were written to understand what the upstream charts do under the hood. For anything long-lived, use the official charts ([jenkinsci/helm-charts](https://github.com/jenkinsci/helm-charts), [argo-helm](https://github.com/argoproj/argo-helm), [sonatype/helm3-charts](https://github.com/sonatype/helm3-charts), [SonarSource/helm-chart-sonarqube](https://github.com/SonarSource/helm-chart-sonarqube)).

## Charts

| Chart | App | Deploys | Default host |
|---|---|---|---|
| [`jenkins-chart`](jenkins-chart) | Jenkins `2.426.1-lts` | Deployment, Service (8080 + 50000 agents), PVC (10Gi), ServiceAccount + RBAC (pods/exec/log for the Kubernetes plugin), ConfigMap with a plugin list and **JCasC** config (local admin user, logged-in-users-can-do-anything), Secret, Ingress | `jenkins.local` |
| [`argocd-chart`](argocd-chart) | Argo CD `v2.12.4` | A single `argocd-server` Deployment (`--insecure`, TLS ends at the ingress), Service, PVC, RBAC, ConfigMap, Secret, Ingress | `argocd.yourdomain.com` |
| [`nexus-chart`](nexus-chart) | Nexus 3 `3.68.1` | Deployment with tuned JVM heap, Service (8081), PVC (20Gi), ConfigMaps for `nexus.properties` / datastore, ServiceAccount + RBAC, Secret, Ingress (unlimited body size for artifact uploads) | `nexus.local` |
| [`sonarqube-chart`](sonarqube-chart) | SonarQube `10.3.0-community` | Deployment with a busybox init container (sets `vm.max_map_count`), Service, PVCs for data/logs/extensions, ConfigMap (`sonar.properties`), Secret, Ingress, plus a **PostgreSQL 13** Deployment + Service + PVC when no external DB is set | `sonarqube.yourdomain.com` |

The Argo CD chart runs only the API/UI server. It has no repo-server, application-controller or Redis, so it shows the UI but won't sync applications. Use the upstream chart for a working Argo CD.

## Prerequisites

- A Kubernetes cluster (minikube, kind, AKS, …) with a default `StorageClass` called `standard`, or set `*.persistence.storageClass`.
- [ingress-nginx](https://kubernetes.github.io/ingress-nginx/deploy/) installed.
- Helm 3.8+.
- Hostnames resolving to the ingress, e.g. `/etc/hosts` entries for `jenkins.local` and `nexus.local`.

## TLS

No certificates or keys are stored in this repo. Each Ingress points at a TLS Secret **by name** (`argocd-tls`, `jenkins-tls`, `nexus-tls`, `sonarqube-tls`), and you create that Secret in the cluster.

**Option A: self-signed certificate (lab):**

```bash
HOST=jenkins.local NS=jenkins
openssl req -x509 -nodes -newkey rsa:2048 -days 365 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=${HOST}" -addext "subjectAltName=DNS:${HOST}"
kubectl create namespace "$NS" --dry-run=client -o yaml | kubectl apply -f -
kubectl -n "$NS" create secret tls jenkins-tls --cert=tls.crt --key=tls.key
rm tls.key tls.crt
```

**Option B: cert-manager.** Install [cert-manager](https://cert-manager.io/docs/installation/helm/) with a `ClusterIssuer`, then add the annotation to the chart's ingress (Jenkins/Nexus take `ingress.annotations`; for Argo CD/SonarQube add it to `templates/ingress.yaml`):

```yaml
cert-manager.io/cluster-issuer: letsencrypt   # your issuer's name
```

cert-manager then creates and renews the Secret named in `tls.secretName` / `ingress.tls[].secretName`.

To turn TLS off, set `argocd.ingress.tls.enabled=false` / `sonarqube.ingress.tls.enabled=false`, or `ingress.tls=[]` for Jenkins/Nexus.

## Install

```bash
helm install jenkins-release   ./jenkins-chart   -n jenkins   --create-namespace
helm install nexus-release     ./nexus-chart     -n nexus     --create-namespace
helm install sonarqube-release ./sonarqube-chart -n sonarqube --create-namespace
helm install argocd-release    ./argocd-chart    -n argocd    --create-namespace
```

The Argo CD and SonarQube charts pick their namespace from `argocd.namespace` / `sonarqube.namespace`, so keep `-n` matching. The backup scripts assume the release names above.

## Credentials

No passwords are stored in `values.yaml`. For every chart-managed password:

1. if you pass a value (`--set …password=…`), that value is used
2. otherwise the value already stored in the release's Secret is reused, so `helm upgrade` doesn't rotate it
3. otherwise a random 24-character password is generated on first install

Or bring your own Secret with `<app>.existingSecret=<name>`. The chart then creates no Secret and every `secretKeyRef` points at yours.

| Chart | Password values | `existingSecret` must contain |
|---|---|---|
| Jenkins | `jenkins.admin.password` | `jenkins-admin-user`, `jenkins-admin-password` (+ `database-*` if `database.external.enabled`) |
| Argo CD | `argocd.admin.password` | `admin-password` |
| SonarQube | `sonarqube.admin.password`, `sonarqube.database.postgresql.password`, `sonarqube.database.password` (external DB, required) | `admin-username`, `admin-password`, `db-username`, `db-password` |
| Nexus | none: Nexus generates its own initial admin password | n/a |

Read the generated passwords:

```bash
kubectl -n jenkins get secret jenkins-release-jenkins-chart-secret \
  -o jsonpath='{.data.jenkins-admin-password}' | base64 -d; echo

kubectl -n nexus exec deploy/nexus-release-nexus-chart -- cat /nexus-data/admin.password; echo
```

SonarQube's own admin login starts as `admin` / `admin`, and SonarQube makes you change it on first login.

## Important values

| Value | Default | Notes |
|---|---|---|
| `jenkins.installPlugins` | list of ~25 plugins | Written to `plugins.txt` and installed at startup |
| `jenkins.jcasc.configScripts` | welcome, local security realm, authz | Jenkins Configuration-as-Code snippets |
| `jenkins.persistence.size` / `nexus.persistence.size` | `10Gi` / `20Gi` | |
| `nexus.env` | heap 1024m–2048m | JVM and Nexus settings |
| `nexus.resources` / `sonarqube.resources` | 2–4Gi memory | Both are memory-hungry; give the node enough RAM |
| `sonarqube.database.external` | `false` | `true` + `host`/`port`/`name`/`username`/`password` to use your own PostgreSQL |
| `*.ingress.hostname` / `ingress.hosts` | see table above | |
| `*.persistence.storageClass` | `standard` | |

## Backups

`jenkins-chart/backup-jenkins.sh` and `nexus-chart/backup-nexus.sh` stream the data directory out of the running pod with `kubectl exec … tar czf - | tar xzf -` into a timestamped folder under `/tmp`:

```bash
./jenkins-chart/backup-jenkins.sh   # copies /var/jenkins_home  -> /tmp/jenkins-backup-YYYYMMDD-HHMMSS
./nexus-chart/backup-nexus.sh       # copies /nexus-data        -> /tmp/nexus-backup-YYYYMMDD-HHMMSS
```

They assume the namespaces `jenkins` / `nexus` and the release names `jenkins-release` / `nexus-release`, so edit the variables at the top if yours differ. These are point-in-time file copies of a live app, which is fine for a lab. For consistent Nexus backups, use its built-in *Admin → Tasks → Backup* task, or volume snapshots.

## Uninstall

```bash
helm uninstall jenkins-release -n jenkins   # repeat per chart
kubectl delete pvc --all -n jenkins          # PVCs (and your data) are kept until you do this
```
