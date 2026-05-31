# Boutique Microservices — Kubernetes Deployment Troubleshooting Report

**Date:** 2026-05-31
**Cluster:** AWS EKS (`ip-10-1-1-81.ec2.internal`, v1.34.8-eks)
**Namespace:** `boutique`

---

## Issue 1 — PostgreSQL Container Failed on Docker Compose (`exit code 126`)

### Symptom
`docker compose up -d` failed with:
```
dependency failed to start: container boutique-postgres exited (126)
```

### Root Cause
The database init script `database/init/10-create-databases.sh` had Windows-style line endings (`CRLF`). When the Linux container tried to execute it, bash saw `/bin/bash^M` (the `\r` carriage return) as the interpreter path, which doesn't exist.

```
/docker-entrypoint-initdb.d/10-create-databases.sh: /bin/bash^M: bad interpreter: No such file or directory
```

### Fix
Converted the file from CRLF to LF using PowerShell:
```powershell
$content = [System.IO.File]::ReadAllText($file)
$content = $content -replace "`r`n", "`n"
[System.IO.File]::WriteAllText($file, $content, [System.Text.UTF8Encoding]::new($false))
```
Then ran `docker compose down -v && docker compose up -d` to restart with a clean state.

---

## Issue 2 — Services Crashing on Kubernetes (`ENOTFOUND boutique-postgres`)

### Symptom
`auth`, `product-service`, `user-service`, and `order-service` pods were repeatedly crashing (4 restarts each):
```
ENOTFOUND boutique-postgres
errno: -3008, code: 'ENOTFOUND', syscall: 'getaddrinfo'
```

### Root Cause
The services attempted a PostgreSQL connection immediately on startup, before the `boutique-postgres` StatefulSet pod was fully ready to accept connections. Kubernetes has no built-in dependency ordering between pods, so services started racing against Postgres.

### Fix
Added two layers of protection to all affected deployment manifests (`auth.yml`, `order-service.yml`, `orders.yml`, `product-service.yml`, `user-service.yml`):

**1. `initContainer` — blocks pod startup until Postgres is reachable:**
```yaml
initContainers:
  - name: wait-for-postgres
    image: busybox:1.35
    command: ['sh', '-c', 'until nc -z boutique-postgres 5432; do echo waiting for postgres; sleep 2; done']
```

**2. `readinessProbe` — prevents traffic routing until the service is healthy:**
```yaml
readinessProbe:
  httpGet:
    path: /health
    port: <service-port>
  initialDelaySeconds: 5
  periodSeconds: 10
```

**3. `livenessProbe` — auto-restarts the pod if it becomes unhealthy:**
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: <service-port>
  initialDelaySeconds: 15
  periodSeconds: 20
```

All services expose a `/health` HTTP endpoint, confirmed by grepping the source code.

---

## Issue 3 — `frontend` and `orders` Pods Stuck in `ImagePullBackOff`

### Symptom
```
Back-off pulling image "003344631323.dkr.ecr.us-east-1.amazonaws.com/frontend:1ebec3729cd10fe1eeac6f0858fba58e2da07956"
Back-off pulling image "003344631323.dkr.ecr.us-east-1.amazonaws.com/orders:1ebec3729cd10fe1eeac6f0858fba58e2da07956"
```

### Root Cause
The Kubernetes manifests referenced images tagged with a specific git commit hash (`1ebec3729cd10fe1eeac6f0858fba58e2da07956`), but only the `latest` tag had been pushed to ECR. The CI/CD pipeline had not pushed the commit-tagged versions.

### Fix
1. Logged into ECR:
```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 003344631323.dkr.ecr.us-east-1.amazonaws.com
```
2. Tagged local images with the required commit hash:
```bash
docker tag .../frontend:latest .../frontend:1ebec3729cd10fe1eeac6f0858fba58e2da07956
docker tag .../orders:latest   .../orders:1ebec3729cd10fe1eeac6f0858fba58e2da07956
```
3. Pushed both to ECR, then force-restarted the deployments:
```bash
kubectl rollout restart deployment/frontend deployment/orders -n boutique
```

---

## Issue 4 — New Pods Stuck in `Pending` (`Too many pods`)

### Symptom
After applying the updated manifests with `initContainers`, the rolling update for `product-service`, `user-service`, and `orders` got stuck:
```
0/1 nodes are available: 1 Too many pods. preemption: 0/1 nodes: 1 No preemption victims found.
```

### Root Cause
The cluster has a single EKS node which was at its pod capacity limit. The default `RollingUpdate` strategy tries to spin up a new pod **before** terminating the old one (`maxSurge: 1`), but there was no room on the node for the extra pod.

### Fix
Patched the affected deployments to use the `Recreate` strategy, which terminates the old pod first before starting the new one:
```bash
kubectl patch deployment product-service -n boutique \
  -p '{"spec":{"strategy":{"type":"Recreate","rollingUpdate":null}}}'

kubectl patch deployment user-service -n boutique \
  -p '{"spec":{"strategy":{"type":"Recreate","rollingUpdate":null}}}'

kubectl patch deployment orders -n boutique \
  -p '{"spec":{"strategy":{"type":"Recreate","rollingUpdate":null}}}'
```
> **Note:** `rollingUpdate` must be explicitly nulled out, otherwise Kubernetes rejects the patch as invalid.

---

## Final State

All 8 pods running with 0 restarts:

| Pod | Status | Restarts |
|-----|--------|----------|
| `boutique-postgres-0` | Running | 0 |
| `gateway` | Running | 0 |
| `auth` | Running | 0 |
| `product-service` | Running | 0 |
| `order-service` | Running | 0 |
| `orders` | Running | 0 |
| `user-service` | Running | 0 |
| `frontend` | Running | 0 |

---

## Recommendations

| # | Recommendation |
|---|----------------|
| 1 | Configure `.gitattributes` to enforce LF line endings for `.sh` files to prevent CRLF issues on Windows |
| 2 | Add `initContainers` as a standard pattern to all DB-dependent services going forward |
| 3 | Update the CI/CD pipeline to push both `latest` and the commit-hash tag to ECR on every build |
| 4 | Add a second EKS node to the node group to avoid pod capacity issues during rolling updates, or configure the cluster autoscaler |
