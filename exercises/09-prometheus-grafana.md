# Prometheus & Grafana — Monitoring

## Objectif
Mettre en place une stack de monitoring complète avec Prometheus pour la collecte de métriques, AlertManager pour les alertes et Grafana pour la visualisation.

## Consignes

> Cette séquence suppose que vous utilisez toujours l'application fil rouge du Jour 1. Elle doit déjà exposer `/metrics`.

### 1. Stack monitoring avec Docker Compose

Créer `monitoring/docker-compose.yml` :

```yaml
services:
  prometheus:
    image: prom/prometheus:v3.9.1
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/alerts.yml:/etc/prometheus/alerts.yml:ro
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=7d'
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:12.3.1
    container_name: grafana
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=${GRAFANA_ADMIN_USER:-admin}
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_ADMIN_PASSWORD:-devops-training-local}
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
      - ./grafana/dashboards:/var/lib/grafana/dashboards:ro
    depends_on:
      - prometheus
    networks:
      - monitoring

  alertmanager:
    image: prom/alertmanager:v0.30.0
    container_name: alertmanager
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
    networks:
      - monitoring

  node-exporter:
    image: prom/node-exporter:v1.10.2
    container_name: node-exporter
    ports:
      - "9100:9100"
    networks:
      - monitoring

  webhook-mock:
    image: node:20-alpine
    container_name: webhook-mock
    command:
      - node
      - -e
      - |
        require('http')
          .createServer((req, res) => {
            let body = '';
            req.on('data', (chunk) => { body += chunk; });
            req.on('end', () => {
              console.log(req.method, req.url, body);
              res.writeHead(200);
              res.end('ok');
            });
          })
          .listen(5001, '0.0.0.0');
    ports:
      - "5001:5001"
    networks:
      - monitoring

  app:
    build:
      context: ../app
      target: production
    container_name: monitored-app
    ports:
      - "3000:3000"
    networks:
      - monitoring

volumes:
  prometheus-data:
  grafana-data:

networks:
  monitoring:
    driver: bridge
```

### 2. Configuration Prometheus

Créer `monitoring/prometheus/prometheus.yml` :

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

rule_files:
  - "alerts.yml"

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node"
    static_configs:
      - targets: ["node-exporter:9100"]

  - job_name: "app"
    metrics_path: "/metrics"
    static_configs:
      - targets: ["app:3000"]
    scrape_interval: 5s
```

### 3. Règles d'alertes

Créer `monitoring/prometheus/alerts.yml` :

```yaml
groups:
  - name: app-alerts
    rules:
      - alert: AppDown
        expr: up{job="app"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Application is down"
          description: "L'application {{ $labels.instance }} est down depuis plus d'1 minute."

      - alert: HighLatency
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{job="app"}[5m])) > 0.5
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High latency detected"
          description: "Le P95 de latence dépasse 500ms sur {{ $labels.instance }}."

      - alert: HighErrorRate
        expr: sum(rate(http_requests_total{job="app",status=~"5.."}[5m])) / sum(rate(http_requests_total{job="app"}[5m])) > 0.05
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High error rate"
          description: "Plus de 5% de réponses 5xx sur {{ $labels.instance }}."

  - name: infra-alerts
    rules:
      - alert: HighCPU
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage"
          description: "CPU > 80% sur {{ $labels.instance }} depuis 5 min."

      - alert: DiskAlmostFull
        expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100 < 15
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disk almost full"
          description: "Moins de 15% d'espace disque disponible sur {{ $labels.instance }}."
```

### 4. Configuration AlertManager

Créer `monitoring/alertmanager/alertmanager.yml` :

```yaml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname', 'severity']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'default'
  routes:
    - matchers:
        - severity="critical"
      receiver: 'critical'

receivers:
  - name: 'default'
    webhook_configs:
      - url: 'http://webhook-mock:5001/webhook'
        send_resolved: true

  - name: 'critical'
    webhook_configs:
      - url: 'http://webhook-mock:5001/webhook'
        send_resolved: true
```

### 5. Provisioning Grafana

**monitoring/grafana/provisioning/datasources/prometheus.yml** :
```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
```

**monitoring/grafana/provisioning/dashboards/default.yml** :
```yaml
apiVersion: 1
providers:
  - name: 'default'
    orgId: 1
    folder: 'DevOps Training'
    type: file
    disableDeletion: false
    editable: true
    options:
      path: /var/lib/grafana/dashboards
      foldersFromFilesStructure: true
```

### 6. Dashboard Grafana (as code)

Créer `monitoring/grafana/dashboards/app-overview.json` :

```json
{
  "dashboard": {
    "title": "DevOps App Overview",
    "tags": ["devops", "training"],
    "timezone": "browser",
    "panels": [
      {
        "title": "Request Rate",
        "type": "timeseries",
        "gridPos": { "h": 8, "w": 12, "x": 0, "y": 0 },
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{method}} {{path}} {{status}}"
          }
        ]
      },
      {
        "title": "Latency P95",
        "type": "timeseries",
        "gridPos": { "h": 8, "w": 12, "x": 12, "y": 0 },
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "P95"
          }
        ]
      },
      {
        "title": "Error Rate",
        "type": "stat",
        "gridPos": { "h": 4, "w": 6, "x": 0, "y": 8 },
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) / sum(rate(http_requests_total[5m])) * 100"
          }
        ]
      },
      {
        "title": "Active Instances",
        "type": "stat",
        "gridPos": { "h": 4, "w": 6, "x": 6, "y": 8 },
        "targets": [
          {
            "expr": "count(up{job=\"app\"} == 1)"
          }
        ]
      }
    ]
  }
}
```

### 7. Requêtes PromQL utiles

```promql
# Taux de requêtes par seconde
rate(http_requests_total[5m])

# Latence P50, P90, P99
histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m]))
histogram_quantile(0.90, rate(http_request_duration_seconds_bucket[5m]))
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# Taux d'erreur en pourcentage
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) * 100

# CPU usage par instance
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Mémoire utilisée
node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes

# Espace disque disponible en %
(node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100
```

### 8. Lancer et explorer

```bash
cd monitoring

# Créer les répertoires
mkdir -p prometheus grafana/provisioning/{datasources,dashboards} grafana/dashboards alertmanager

# Credentials Grafana locales (démo uniquement)
cat <<'EOF' > .env
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=devops-training-local
EOF

# Lancer la stack
docker compose up -d

# Vérifier
docker compose ps
docker compose logs -f webhook-mock

# Ouvrir dans le navigateur
# Prometheus   : http://localhost:9090
# Grafana      : http://localhost:3001 (admin / devops-training-local)
# AlertManager : http://localhost:9093
```

## Livrable
- Stack Prometheus + Grafana + AlertManager fonctionnelle
- 5 règles d'alertes configurées
- 1 dashboard Grafana provisionné automatiquement
- Au moins 3 requêtes PromQL maîtrisées
- Vérification qu'AlertManager envoie bien ses webhooks au service `webhook-mock`

## Aide

### Vérifier que Prometheus scrape bien
- Aller sur http://localhost:9090/targets
- Tous les targets doivent être en état "UP"

### Grafana — importer un dashboard communautaire
1. Aller dans Dashboards > Import
2. Entrer l'ID : `1860` (Node Exporter Full)
3. Sélectionner la datasource Prometheus
