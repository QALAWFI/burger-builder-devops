# Burger Builder — DevOps Platform Documentation

End-to-end DevOps implementation for the **Burger Builder** full-stack application
(React/TypeScript frontend, Spring Boot/Java 21 backend, PostgreSQL database) running
on a self-managed Kubernetes cluster.

This folder is the operational source of truth. Start here, then jump to the runbook you need.

| Document | Purpose |
|----------|---------|
| [README.md](README.md) | Architecture, components, cluster topology (this file) |
| [DEPLOYMENT.md](DEPLOYMENT.md) | Build & deploy every component from scratch |
| [OPERATIONS.md](OPERATIONS.md) | Day-2 ops: Blue/Green, Canary, Rollback, scaling, image workflow |
| [MONITORING.md](MONITORING.md) | Access logs (ELK) and metrics (Prometheus/Grafana) |
| [TROUBLESHOOTING.md](TROUBLESHOOTING.md) | Known issues and verified fixes |

---

## 1. Architecture Overview

```
                         ┌──────────────────────────── Windows Host ───────────────────────────┐
                         │                                                                       │
                         │   Docker Desktop                     VMware Workstation               │
                         │   ┌───────────────┐    ┌─────────────────────────────────────────┐   │
   Developer ── git ───► │   │ image build   │    │   Kubernetes Cluster (kubeadm)           │   │
                         │   │ + Nexus       │    │                                          │   │
                         │   │ registry      │◄───┤  ┌────────────┐      ┌────────────────┐  │   │
                         │   │ :8081 / :8082 │    │  │ k8s-control │      │ k8s-worker1    │  │   │
                         │   └───────────────┘    │  │ control     │◄────►│ workloads run  │  │   │
                         │                        │  │ plane       │      │ here           │  │   │
                         │                        │  └────────────┘      └────────────────┘  │   │
                         │                        └─────────────────────────────────────────┘   │
                         └───────────────────────────────────────────────────────────────────────┘
```

The application is delivered through a full pipeline:

```
Code ─► Git/GitLab (GitFlow) ─► GitLab CI (test+build) ─► Nexus (registry)
     ─► Kubernetes (deploy) ─► Blue/Green · Canary · Rollback
     ─► ELK (logs) + Prometheus/Grafana (metrics)
```

---

## 2. Cluster Topology

| Node | Role | IP | Notes |
|------|------|----|-------|
| `k8s-control` | control-plane | `192.168.221.128` | API server, etcd, scheduler, controller-manager. `kubectl` runs **here only**. |
| `k8s-worker1` | worker | `192.168.221.129` | Runs all application + monitoring workloads. |
| Windows host | infra | `192.168.221.1` | Docker Desktop + Nexus registry (`:8082`). |

- **Container runtime:** containerd
- **CNI (networking):** Flannel (`10.244.0.0/16` pod network)
- **Ingress controller:** ingress-nginx (baremetal)
- **Kubernetes version:** v1.31.x

---

## 3. Namespaces

| Namespace | Contents |
|-----------|----------|
| `default` | Application: `backend-blue`, `backend-green`, `frontend`, `database` |
| `logging` | ELK stack: `elasticsearch`, `kibana`, `filebeat` (DaemonSet) |
| `monitoring` | `prometheus`, `grafana` |
| `ingress-nginx` | Ingress controller |
| `kube-system` / `kube-flannel` | Cluster core + networking |

---

## 4. Application Components

| Component | Image | Kind | Service |
|-----------|-------|------|---------|
| Backend (blue) | `192.168.221.1:8082/backend:v2` | Deployment | `backend-service:8080` |
| Backend (green) | `192.168.221.1:8082/backend:v2` | Deployment | `backend-green-service:8080` |
| Frontend | `192.168.221.1:8082/frontend:v3` | Deployment | `frontend-service:80` |
| Database | `postgres:16-alpine` | Deployment | `database-service:5432` |

- **Backend** runs Spring Boot with profile `docker`, exposes metrics at `/actuator/prometheus`.
- **Database** persists to a `hostPath` PersistentVolume on `k8s-worker1` (`/mnt/data/postgres`, 1Gi).
- Routing via Ingress `burger-builder-ingress`: `/api` → backend, `/` → frontend.

---

## 5. Configuration & Secrets

| Object | Type | Purpose |
|--------|------|---------|
| `nexus-registry-secret` | docker-registry Secret | Pull images from Nexus |
| `db-credentials` | generic Secret | `POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB` |
| `db-init-scripts` | ConfigMap | Ordered SQL init (`01-schema.sql`, `02-data.sql`) |
| `database-pv` / `database-pvc` | PV/PVC | Persistent storage for PostgreSQL |

> **Secrets are never committed to Git.** They are created imperatively on the cluster
> (see [DEPLOYMENT.md](DEPLOYMENT.md)). Manifests reference them via `secretKeyRef` / `imagePullSecrets`.

---

## 6. Image Delivery Model (important)

Because Nexus runs as an **insecure HTTP registry** reachable at two addresses:

- **Build & push** from the Windows host uses `127.0.0.1:8082` (Docker Desktop network).
- **Cluster pulls** reference `192.168.221.1:8082` (host-only network address).

Both addresses point to the same Nexus; the repository path (`backend:v2`) is identical.
See [TROUBLESHOOTING.md](TROUBLESHOOTING.md#image-pull-on-worker) for the containerd pull workaround.
