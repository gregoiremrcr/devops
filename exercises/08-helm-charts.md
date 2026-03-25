# Helm & Kustomize

## Objectif
Packager l'application avec Helm, comprendre les charts, values et releases. Découvrir Kustomize comme alternative.

## Consignes

### 1. Créer un chart Helm

```bash
helm create devops-app-chart
```

Structure générée :
```
devops-app-chart/
  Chart.yaml
  values.yaml
  templates/
    deployment.yaml
    service.yaml
    ingress.yaml
    hpa.yaml
    _helpers.tpl
    NOTES.txt
  charts/        (dépendances)
```

### 2. Personnaliser le Chart

**devops-app-chart/Chart.yaml** :
```yaml
apiVersion: v2
name: devops-app
description: Application de démo DevOps
type: application
version: 0.1.0
appVersion: "1.0.0"
maintainers:
  - name: DevOps Team
```

### 3. Values par défaut

**devops-app-chart/values.yaml** :
```yaml
# -- Nombre de réplicas
replicaCount: 3

image:
  repository: devops-app
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  host: devops.local

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 250m
    memory: 256Mi

# -- Configuration applicative
config:
  nodeEnv: production
  logLevel: info
  appPort: "3000"

# -- Base de données
postgresql:
  enabled: true
  auth:
    database: appdb
    username: appuser
    # Le password doit être fourni via --set ou un secret externe
  primary:
    persistence:
      size: 1Gi

# -- Probes
probes:
  readiness:
    path: /
    initialDelaySeconds: 5
    periodSeconds: 10
  liveness:
    path: /
    initialDelaySeconds: 15
    periodSeconds: 20

# -- Autoscaling
autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

### 4. Template Deployment

**devops-app-chart/templates/deployment.yaml** :
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "devops-app.fullname" . }}
  labels:
    {{- include "devops-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "devops-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "devops-app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ int .Values.config.appPort }}
          env:
            - name: NODE_ENV
              value: {{ .Values.config.nodeEnv | quote }}
            - name: LOG_LEVEL
              value: {{ .Values.config.logLevel | quote }}
            - name: APP_PORT
              value: {{ .Values.config.appPort | quote }}
          readinessProbe:
            httpGet:
              path: {{ .Values.probes.readiness.path }}
              port: {{ int .Values.config.appPort }}
            initialDelaySeconds: {{ .Values.probes.readiness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.readiness.periodSeconds }}
          livenessProbe:
            httpGet:
              path: {{ .Values.probes.liveness.path }}
              port: {{ int .Values.config.appPort }}
            initialDelaySeconds: {{ .Values.probes.liveness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.liveness.periodSeconds }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

### 5. Values par environnement

**values-dev.yaml** :
```yaml
replicaCount: 1
config:
  nodeEnv: development
  logLevel: debug
resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 100m
    memory: 128Mi
autoscaling:
  enabled: false
```

**values-prod.yaml** :
```yaml
replicaCount: 5
config:
  nodeEnv: production
  logLevel: warn
resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
```

### 6. Installer et gérer

```bash
# Valider le chart
helm lint devops-app-chart/

# Template (voir le YAML généré sans installer)
helm template devops-app devops-app-chart/ -f values-dev.yaml

# Installer en dev
helm install devops-app devops-app-chart/ \
  -f values-dev.yaml \
  -n devops-training \
  --set postgresql.auth.password=secret123

# Lister les releases
helm list -n devops-training

# Mettre à jour
helm upgrade devops-app devops-app-chart/ \
  -f values-prod.yaml \
  -n devops-training \
  --set postgresql.auth.password=secret123

# Rollback
helm rollback devops-app 1 -n devops-training

# Historique
helm history devops-app -n devops-training
```

### 7. Kustomize (alternative)

Créer la structure Kustomize :
```
k8s/
  base/
    kustomization.yaml
    deployment.yaml
    service.yaml
  overlays/
    dev/
      kustomization.yaml
      replicas-patch.yaml
    prod/
      kustomization.yaml
      replicas-patch.yaml
```

**k8s/base/kustomization.yaml** :
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
commonLabels:
  app: devops-app
```

**k8s/overlays/dev/kustomization.yaml** :
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
namePrefix: dev-
patches:
  - path: replicas-patch.yaml
```

**k8s/overlays/dev/replicas-patch.yaml** :
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: devops-app
spec:
  replicas: 1
```

```bash
# Aperçu
kubectl kustomize k8s/overlays/dev/

# Appliquer
kubectl apply -k k8s/overlays/dev/
```

## Livrable
- Chart Helm fonctionnel avec values pour dev et prod
- Installation et upgrade réussis
- Comparaison Helm vs Kustomize

## Aide

### Helm debug
```bash
helm template my-release ./chart --debug
helm install my-release ./chart --dry-run --debug
```
