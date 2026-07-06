# Deployment Runbook

How to build and deploy the entire platform from a clean cluster.
All `kubectl` commands run on **`k8s-control`** only.

> Conventions: `<NEXUS_USER>` / `<NEXUS_PASSWORD>` and `<DB_PASSWORD>` are placeholders.
> Never commit real credentials.

---

## 0. Prerequisites

- Kubernetes cluster up (`kubectl get nodes` shows `control` + `worker1` Ready).
- Nexus registry reachable at `192.168.221.1:8082`.
- containerd on each node configured to trust the insecure registry
  (`/etc/containerd/certs.d/192.168.221.1:8082/hosts.toml`).

---

## 1. Build & Push Images (Windows host, Docker Desktop)

```bash
# Backend
cd backend
docker build -t 127.0.0.1:8082/backend:v2 .
docker push 127.0.0.1:8082/backend:v2

# Frontend
cd ../frontend
docker build -t 127.0.0.1:8082/frontend:v3 .
docker push 127.0.0.1:8082/frontend:v3
```

> Build/push uses `127.0.0.1:8082`; the cluster pulls the same images via `192.168.221.1:8082`.

---

## 2. Create Secrets & Config (one time)

```bash
# Image pull secret (lets Pods pull from Nexus)
kubectl create secret docker-registry nexus-registry-secret \
  --docker-server=192.168.221.1:8082 \
  --docker-username=<NEXUS_USER> \
  --docker-password=<NEXUS_PASSWORD>

# Database credentials
kubectl create secret generic db-credentials \
  --from-literal=POSTGRES_USER=postgres \
  --from-literal=POSTGRES_PASSWORD=<DB_PASSWORD> \
  --from-literal=POSTGRES_DB=burgerbuilder

# DB init scripts (schema + seed data)
kubectl create configmap db-init-scripts \
  --from-file=01-schema.sql=database/schema.sql \
  --from-file=02-data.sql=database/data.sql
```

---

## 3. Deploy in Order

Dependencies first (storage → database → backend → frontend → routing).

```bash
# 3.1 Persistent storage for the DB
kubectl apply -f k8s/database-pv.yaml
kubectl apply -f k8s/database-pvc.yaml

# 3.2 Database
kubectl apply -f k8s/database-deployment.yaml
kubectl apply -f k8s/database-service.yaml
kubectl rollout status deploy/database          # wait until Ready

# 3.3 Backend (blue + green) + service
kubectl apply -f k8s/backend-blue-deployment.yaml
kubectl apply -f k8s/backend-green-deployment.yaml
kubectl apply -f k8s/backend-service.yaml
kubectl rollout status deploy/backend-blue

# 3.4 Frontend
kubectl apply -f k8s/frontend-deployment.yaml
kubectl apply -f k8s/frontend-service.yaml
kubectl rollout status deploy/frontend

# 3.5 Ingress routing
kubectl apply -f k8s/ingress.yaml
```

---

## 4. Deploy Logging (ELK)

```bash
kubectl apply -f elasticsearch.yaml     # namespace: logging
kubectl apply -f kibana.yaml
kubectl apply -f filebeat.yaml          # DaemonSet on every node
kubectl get pods -n logging             # all Running
```

---

## 5. Deploy Monitoring (Prometheus + Grafana)

```bash
kubectl apply -f prometheus.yaml        # creates namespace: monitoring
kubectl apply -f grafana.yaml
kubectl get pods -n monitoring          # prometheus + grafana Running
```

Grafana's Prometheus datasource is **auto-provisioned** via ConfigMap `grafana-datasources`,
so it is connected on first start. Dashboard import: see [MONITORING.md](MONITORING.md).

---

## 6. Verify the Deployment

```bash
kubectl get pods -A                     # everything Running
kubectl get ingress                     # address assigned

# Backend health & metrics from inside the cluster
kubectl run tmp --rm -it --image=busybox --restart=Never -- \
  wget -qO- http://backend-service:8080/actuator/health
```

Expected: `{"status":"UP"}` and the app reachable through the ingress / NodePort.

---

## 7. Access URLs (via port-forward, run on control node)

| Service | Command | URL |
|---------|---------|-----|
| Kibana | `kubectl port-forward -n logging svc/kibana-service 5601:5601 --address 0.0.0.0` | `http://192.168.221.128:5601` |
| Prometheus | `kubectl port-forward -n monitoring svc/prometheus-service 9090:9090 --address 0.0.0.0` | `http://192.168.221.128:9090` |
| Grafana | `kubectl port-forward -n monitoring svc/grafana-service 3000:3000 --address 0.0.0.0` | `http://192.168.221.128:3000` |

> `--address 0.0.0.0` is required so the Windows host can reach the forwarded port.
