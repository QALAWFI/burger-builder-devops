# Operations Runbook (Day-2)

Routine operational procedures. All `kubectl` runs on **`k8s-control`**.

---

## 1. Image Build → Deploy Workflow (the micro-cycle)

The end-to-end loop used every time backend code changes:

```bash
# 1. Build & push a new tag (Windows host)
cd backend
docker build -t 127.0.0.1:8082/backend:v3 .
docker push 127.0.0.1:8082/backend:v3

# 2. Pre-pull on the worker (containerd can't auto-pull HTTP registry — see Troubleshooting)
#    Run on k8s-worker1:
sudo ctr -n k8s.io images pull --hosts-dir /etc/containerd/certs.d \
  -u <NEXUS_USER>:<NEXUS_PASSWORD> 192.168.221.1:8082/backend:v3

# 3. Roll out the new image (control node)
kubectl set image deploy/backend-blue backend=192.168.221.1:8082/backend:v3
kubectl rollout status deploy/backend-blue
```

---

## 2. Blue/Green Deployment

Two backends run in parallel. The `backend-service` Selector decides which one
receives live traffic. Switching is instant and has **zero downtime**.

```bash
# Inspect which color is live
kubectl get svc backend-service -o jsonpath='{.spec.selector}'

# Deploy the new version to the IDLE color (e.g. green), then test it privately
kubectl set image deploy/backend-green backend=192.168.221.1:8082/backend:v3
kubectl rollout status deploy/backend-green

# Cut traffic over: point the Service at green
kubectl patch svc backend-service -p '{"spec":{"selector":{"app":"backend","version":"green"}}}'

# Roll back instantly if needed: point it back at blue
kubectl patch svc backend-service -p '{"spec":{"selector":{"app":"backend","version":"blue"}}}'
```

**Why:** the idle color is fully tested before it serves a single user; the switch is a label change, so rollback is one command.

---

## 3. Canary Deployment

Send a small % of traffic to the new version using ingress-nginx canary annotations.
A second Ingress (canary) shadows the main host and routes a weighted slice to
`backend-green-service`.

```yaml
# canary-ingress.yaml (key annotations)
metadata:
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "20"   # 20% to the new version
```

```bash
kubectl apply -f k8s/canary-ingress.yaml

# Gradually increase confidence: 20 → 50 → 100
kubectl annotate ingress burger-builder-canary \
  nginx.ingress.kubernetes.io/canary-weight="50" --overwrite

# Promote (canary becomes the only path) or remove the canary ingress to abort
kubectl delete ingress burger-builder-canary
```

**Why:** real production traffic validates the release on a small blast radius before full rollout.

---

## 4. Rollback

Every `kubectl set image` / `apply` creates a new ReplicaSet revision. Roll back
to the previous (or any) revision instantly.

```bash
# View revision history
kubectl rollout history deploy/backend-blue

# Roll back to the immediately previous revision
kubectl rollout undo deploy/backend-blue

# Roll back to a specific revision
kubectl rollout undo deploy/backend-blue --to-revision=2

# Confirm
kubectl rollout status deploy/backend-blue
```

**Why:** Kubernetes keeps old ReplicaSets, so recovery from a bad deploy is a single command — no rebuild required.

---

## 5. Scaling

```bash
# Manual scale
kubectl scale deploy/backend-blue --replicas=3

# Autoscale on CPU (requires metrics-server)
kubectl autoscale deploy/backend-blue --min=2 --max=5 --cpu-percent=70
```

---

## 6. Inspecting & Restarting Workloads

```bash
kubectl get pods -o wide                       # where pods run + status
kubectl describe pod <pod>                      # events (scheduling, pull, probes)
kubectl logs -f <pod>                           # live logs
kubectl logs <pod> --previous                   # logs from a crashed container
kubectl rollout restart deploy/backend-blue     # graceful rolling restart
```

---

## 7. Database Operations

```bash
# Shell into the DB pod
kubectl exec -it deploy/database -- psql -U postgres -d burgerbuilder

# Verify seed data
kubectl exec -it deploy/database -- \
  psql -U postgres -d burgerbuilder -c "SELECT count(*) FROM ingredients;"
```

Data survives pod restarts via the `database-pvc` (hostPath on `k8s-worker1`).

---

## 8. Routine Cluster Hygiene

```bash
# Remove all non-Running pods (Evicted/Error leftovers after DiskPressure)
kubectl delete pods -A --field-selector=status.phase=Failed

# Check node pressure (disk/memory) — root cause of most evictions
kubectl describe node k8s-worker1 | grep -A5 Conditions

# Disk usage on a node (run on the node itself)
df -h /
```
