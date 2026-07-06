# Monitoring & Logging Runbook

The platform has two complementary observability stacks:

| Concern | Stack | Question it answers |
|---------|-------|---------------------|
| **Logs** (text) | ELK — Elasticsearch + Kibana + Filebeat | *What happened? Why did it fail?* |
| **Metrics** (numbers) | Prometheus + Grafana | *How is it performing? How much is it using?* |

A key architectural difference:
- **Filebeat → Elasticsearch is PUSH** (the agent ships logs out).
- **Prometheus → backend is PULL** (Prometheus scrapes `/actuator/prometheus`).

---

## 1. Logging — ELK Stack

### Components
| Component | Role | Namespace |
|-----------|------|-----------|
| Elasticsearch | Stores & indexes log documents | `logging` |
| Kibana | Web UI to search/visualize | `logging` |
| Filebeat | DaemonSet on every node; tails `/var/log/containers/*.log` | `logging` |

Filebeat enriches each line with Kubernetes metadata (pod, namespace, container)
via the `add_kubernetes_metadata` processor, then ships to `elasticsearch-service:9200`.

### Access Kibana
```bash
kubectl port-forward -n logging svc/kibana-service 5601:5601 --address 0.0.0.0
# → http://192.168.221.128:5601
```

### First-time setup
1. **Stack Management → Data Views → Create data view**
2. Index pattern: `filebeat-*`  → Timestamp field: `@timestamp`
3. Open **Discover** to search logs.

### Useful KQL filters (Discover search bar)
```text
kubernetes.namespace : "default"
kubernetes.pod.name : "backend-blue*"
message : "ERROR"
kubernetes.container.name : "backend" and message : "Exception"
```

### Health checks
```bash
# Documents indexed (proves the pipeline works)
kubectl exec -n logging deploy/elasticsearch -- \
  curl -s localhost:9200/_cat/indices/filebeat-*?v

# Filebeat running on every node?
kubectl get pods -n logging -o wide -l k8s-app=filebeat
```

---

## 2. Metrics — Prometheus + Grafana

### How metrics are produced
The Spring Boot backend exposes metrics through **Micrometer**:
- `pom.xml` includes `micrometer-registry-prometheus`.
- `application.properties` exposes the endpoint and tags every metric:
  ```properties
  management.endpoints.web.exposure.include=health,info,prometheus,metrics
  management.metrics.tags.application=burger-builder-backend
  ```
- Result: `GET /actuator/prometheus` returns JVM, HTTP, HikariCP, Tomcat metrics.

### How Prometheus collects them
`prometheus.yml` scrapes the backend every 15s via cluster DNS:
```yaml
scrape_configs:
  - job_name: 'burger-backend'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['backend-service.default.svc.cluster.local:8080']
```

### Access Prometheus
```bash
kubectl port-forward -n monitoring svc/prometheus-service 9090:9090 --address 0.0.0.0
# → http://192.168.221.128:9090
```
- **Status → Targets**: both `prometheus` and `burger-backend` must show **UP**.
- Try a query (PromGQL): `jvm_memory_used_bytes{application="burger-builder-backend"}`

### Access Grafana
```bash
kubectl port-forward -n monitoring svc/grafana-service 3000:3000 --address 0.0.0.0
# → http://192.168.221.128:3000   (admin / admin)
```

The Prometheus datasource is auto-provisioned (ConfigMap `grafana-datasources`),
pointing at `http://prometheus-service:9090`.

### Import the JVM dashboard
1. Open `http://192.168.221.128:3000/dashboard/import`
2. Enter dashboard ID **`4701`** (JVM / Micrometer) → **Load**
3. Select **Prometheus** as the data source → **Import**

The dashboard shows: Uptime, Heap/Non-Heap usage, JVM threads, GC pauses,
HTTP request rate & duration, and connection-pool stats.

### Generate traffic for a live demo
```bash
# From any node — hammer an endpoint so the Rate/Duration panels move
for i in $(seq 1 200); do \
  curl -s http://<INGRESS_OR_NODEPORT>/api/ingredients > /dev/null; done
```

---

## 3. What to check during an incident

| Symptom | Where to look |
|---------|---------------|
| App returning errors | Kibana Discover → filter `message : "ERROR"` |
| High latency | Grafana → HTTP Duration panel |
| Memory growth / OOM | Grafana → JVM Heap; `kubectl describe pod` for OOMKilled |
| Backend not scraped | Prometheus → Targets (DOWN) → check `backend-service` endpoints |
| Pod crash looping | `kubectl logs <pod> --previous` + `kubectl describe pod` |
