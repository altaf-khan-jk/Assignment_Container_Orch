**Repo:** `Assignment_Container_Orch`  
**Primary URL:** https://probable-pancake-4j6xxg74vvq9h6jp-3000.app.github.dev/  

Launch an **open-source CI/CD pipeline** on a Kubernetes cluster together with **Gitea Actions**, making it accessible to the public via **Cloudflare Tunnel**, and employing an **external MySQL** with persistence.  
Moreover, set up **GitHub → Gitea mirroring** (one-way) and a **Gitea CI** that shows commit metadata & changed files for every push.


Gitea reachable at **https://probable-pancake-4j6xxg74vvq9h6jp-3000.app.github.dev/** via **Code Space**.
**Note:NOt Able to access the cloudflare**
- Gitea Uses **external MySQL 8.0** with persistent volume (no bundled Postgres/Valkey).
- **Runner** online with **`ubuntu-latest`** label mapping to a Node-ready image, so **`actions/checkout@v4`** works.
- **GitHub → Gitea mirroring**
- Gitea CI Workflow show error because of the cloudflare (robust even when `before` commit is missing).

Repositories & Workflows

### 1) GitHub → Gitea Mirror (in GitHub repo)
`/.github/workflows/mirror-to-gitea.yml`  
Uses secrets:
- `GITEA_USERNAME=altaf-khan-jk`
- `GITEA_TOKEN=<Gitea PAT with repo scope>`

Mirrors current branch to Gitea. Skips if remote already has the same commit (SHA check). Uses `--force-with-lease` if histories diverge.

### 2) Gitea CI: Commit Info (in Gitea repo)
`/.gitea/workflows/ci.yml`  
On each push, prints:
- Repository, branch, **pusher**, **author**, **SHA**, timestamp, message  
- **Changed files** between `before` and `after` (or fallback to parent/root)

---

## 🛠️ Install & Ops (from zero)

> Run these on the Ubuntu VM that already talks to the cluster.

### 0) Tooling
```bash
sudo apt-get update && sudo apt-get install -y git curl python3-pip unzip
python3 -m pip install --user ansible kubernetes
ansible-galaxy collection install kubernetes.core --force
export PATH="$HOME/.local/bin:$PATH"
```

### 1) Namespace & Secrets (MySQL)
```bash
kubectl create ns gitea || true
MYSQL_ROOT_PASSWORD=$(openssl rand -base64 24)
MYSQL_GITEA_PASSWORD=$(openssl rand -base64 24)
kubectl -n gitea create secret generic mysql-secrets \
  --from-literal=mysql-root-password="$MYSQL_ROOT_PASSWORD" \
  --from-literal=mysql-gitea-password="$MYSQL_GITEA_PASSWORD"
```

### 2) MySQL 8.0 (official image) with PVC
Create `k8s-mysql-official.yaml` (Service + StatefulSet, 10Gi PVC, `local-path` storage class), then:
```bash
kubectl apply -f k8s-mysql-official.yaml
kubectl -n gitea rollout status statefulset/mysql --timeout=5m
```

### 3) Gitea via Helm (external MySQL, persistence)
`values-prod.yaml` **(key items)**:
```yaml
postgresql: { enabled: false }
postgresql-ha: { enabled: false }
memcached: { enabled: false }
redis: { enabled: false }
redis-cluster: { enabled: false }
valkey: { enabled: false }
valkey-cluster: { enabled: false }

persistence:
  enabled: true
  data:   { enabled: true, size: 10Gi, storageClass: local-path }
  config: { enabled: true, size: 1Gi,  storageClass: local-path }

service: { http: { type: ClusterIP } }

gitea:
  config:
    database:
      DB_TYPE: mysql
      HOST: mysql.gitea.svc.cluster.local:3306
      NAME: gitea
      USER: gitea
      PASSWD: "__MYSQL_GITEA_PASSWORD__"
    server:
      ROOT_URL: "https://probable-pancake-4j6xxg74vvq9h6jp-3000.app.github.dev/altaf-khan-jk/Assignment_Container_Orch"
      DOMAIN:   "probable-pancake-4j6xxg74vvq9h6jp-3000.app.github.dev"
      PROTOCOL: http
      HTTP_PORT: 3000
      DISABLE_SSH: true
```

Install/update:
```bash
helm repo add gitea https://dl.gitea.com/charts/ || true
helm repo update
MYSQL_GITEA_PASSWORD=$(kubectl -n gitea get secret mysql-secrets -o jsonpath='{.data.mysql-gitea-password}' | base64 -d)

helm -n gitea upgrade --install gitea gitea/gitea -f values-prod.yaml \
  --set gitea.config.database.PASSWD="$MYSQL_GITEA_PASSWORD" \
  --set postgresql.enabled=false --set postgresql-ha.enabled=false \
  --set memcached.enabled=false --set redis.enabled=false \
  --set redis-cluster.enabled=false --set valkey.enabled=false --set valkey-cluster.enabled=false \
  --set deployment.strategy.type=Recreate
```

## 🧪 How to Trigger

- **Normal dev flow:** push to **GitHub** → GH Action mirrors to **Gitea** → Gitea CI runs, prints commit info (no loops).
- **Direct Gitea push (rare):** Gitea CI runs and prints info. (No mirroring back.)

---