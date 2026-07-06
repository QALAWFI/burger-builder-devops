# Troubleshooting Runbook

Real issues encountered while operating this cluster, with root cause and the
verified fix. Each entry is written so it can be applied directly during an incident.

---

## Image pull on worker

**Symptom:** App pods stuck in `ImagePullBackOff` / `ErrImagePull`, especially after a
node restart or disk cleanup.

**Root cause:** Nexus is an **insecure HTTP** registry. After Kubernetes' image garbage
collection deletes images under disk pressure, containerd does not reliably re-pull from
the HTTP registry on demand.

**Fix — pre-pull the image on the worker** (run on `k8s-worker1`):
```bash
sudo ctr -n k8s.io images pull --hosts-dir /etc/containerd/certs.d \
  -u <NEXUS_USER>:<NEXUS_PASSWORD> 192.168.221.1:8082/backend:v2

sudo ctr -n k8s.io images pull --hosts-dir /etc/containerd/certs.d \
  -u <NEXUS_USER>:<NEXUS_PASSWORD> 192.168.221.1:8082/frontend:v3
```
Then let the ReplicaSet recreate the pods:
```bash
kubectl delete pod -l app=backend
kubectl delete pod -l app=frontend
```

> **Note:** public images (postgres, docker.elastic.co, prom/prometheus, grafana/grafana)
> pull automatically — only Nexus images need this manual step.

---

## DiskPressure → evictions & dead pods

**Symptom:** Many pods in `Evicted`, `ContainerStatusUnknown`, or `Error`; images
disappearing; nodes flapping `Ready` → `NotReady`.

**Root cause:** Worker node ran low on disk. Kubelet responds with **Image GC** (deletes
images) and **eviction** (kills pods) to reclaim space.

**Fix:**
```bash
# 1. Confirm the pressure
kubectl describe node k8s-worker1 | grep -A5 Conditions

# 2. Clean up dead pods (keeps Running ones)
kubectl delete pods -A --field-selector=status.phase=Failed
kubectl delete pods -n ingress-nginx --field-selector=status.phase!=Running

# 3. Free disk on the node (run on k8s-worker1)
df -h /
sudo crictl rmi --prune          # remove unused images
```
**Permanent fix applied:** expanded the worker VM disk (→ 40 GB) and RAM so GC/eviction
no longer triggers under normal load.

---

## kubectl: connection refused on worker

**Symptom:** `The connection to the server localhost:8080 was refused` when running
`kubectl` on the worker node.

**Root cause:** `kubectl` is only configured (kubeconfig) on **`k8s-control`**. The worker
has no admin config, so it tries the default `localhost:8080`.

**Fix:** Always run `kubectl` on `k8s-control`. The worker is only for `ctr`/`crictl`/node-level
commands.

---

## Port not reachable from Windows host

**Symptom:** `Test-NetConnection 192.168.221.128 -Port 5601` → `TcpTestSucceeded : False`;
Kibana/Grafana won't open in the browser.

**Root cause:** the `port-forward` wasn't running, or was bound to `127.0.0.1` only.

**Fix:** run port-forward with `--address 0.0.0.0` and keep the terminal open:
```bash
kubectl port-forward -n logging svc/kibana-service 5601:5601 --address 0.0.0.0
```
`port-forward` is foreground and dies when the terminal closes — use a dedicated window
(or a NodePort Service for a permanent path).

---

## Multi-line YAML corrupted when pasted into terminal

**Symptom:** Heredoc (`cat <<EOF`) paste of large YAML produces garbled content
(e.g. `path: /var/logyOrCreate-dataat/data`).

**Root cause:** terminal line-wrapping / bracketed-paste mangles long multi-line input.

**Fix (reliable):** author the file on the Windows host, then copy it to the control node:
```bash
scp filebeat.yaml user@192.168.221.128:~/filebeat.yaml
kubectl apply -f ~/filebeat.yaml
```
Avoid pasting large heredocs directly into an interactive shell.

---

## Prometheus target DOWN

**Symptom:** Prometheus → Targets shows `burger-backend` as **DOWN**.

**Checklist:**
```bash
# 1. Is the metrics endpoint live?
kubectl run tmp --rm -it --image=busybox --restart=Never -- \
  wget -qO- http://backend-service.default.svc.cluster.local:8080/actuator/prometheus

# 2. Does the Service have endpoints (pods Ready)?
kubectl get endpoints backend-service

# 3. Is the scrape path/port correct in prometheus.yml?
#    metrics_path: /actuator/prometheus  ·  port 8080
```
Most often the cause is the actuator endpoint not exposed — confirm
`management.endpoints.web.exposure.include` contains `prometheus` and the image was rebuilt.

---

## Stuck bash continuation prompt `>`

**Symptom:** terminal shows a `>` prompt and ignores commands.

**Root cause:** an incomplete command (unclosed quote, or a wget/command pasted across
lines) left the shell waiting for more input.

**Fix:** press `Ctrl+C` to abort, then re-enter the command on a single line.

---

## Quick diagnostic cheat-sheet

```bash
kubectl get pods -A -o wide                 # cluster-wide health + placement
kubectl describe pod <pod>                   # events: scheduling, pulls, probes, OOM
kubectl logs <pod> --previous                # why a crashed container died
kubectl get events -A --sort-by=.lastTimestamp | tail -30
kubectl top nodes                            # CPU/mem pressure (needs metrics-server)
kubectl rollout status deploy/<name>         # is a rollout stuck?
```
