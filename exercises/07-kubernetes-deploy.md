# Kubernetes — Déployer une application

## Objectif
Déployer une application multi-tiers sur Kubernetes en utilisant des manifestes YAML : Deployments, Services, ConfigMaps, Secrets et, si le cluster est prêt, Ingress.

## Pré-requis
- Cluster K8s local opérationnel (minikube ou kind)
- kubectl configuré
- Image locale `devops-app:1.0.0` construite depuis le starter du Jour 1

## Consignes

### 1. Namespace dédié

```bash
kubectl create namespace devops-training
kubectl config set-context --current --namespace=devops-training
```

### 2. ConfigMap

Créer `k8s/configmap.yaml` :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: devops-training
data:
  NODE_ENV: "production"
  APP_PORT: "3000"
  DB_HOST: "postgres-svc"
  DB_PORT: "5432"
  DB_NAME: "appdb"
  LOG_LEVEL: "info"
```

### 3. Secret

```bash
# Créer le secret en ligne de commande (pas en YAML pour ne pas commiter les secrets !)
kubectl create secret generic app-secrets \
  --namespace=devops-training \
  --from-literal=DB_USER=appuser \
  --from-literal=DB_PASSWORD=supersecret123
```

### 4. Deployment PostgreSQL

Créer `k8s/postgres.yaml` :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: devops-training
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: devops-training
  labels:
    app: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: DB_NAME
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: DB_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: DB_PASSWORD
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data
          readinessProbe:
            exec:
              command:
                - pg_isready
                - -U
                - appuser
            initialDelaySeconds: 5
            periodSeconds: 5
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
      volumes:
        - name: postgres-storage
          persistentVolumeClaim:
            claimName: postgres-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-svc
  namespace: devops-training
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
  type: ClusterIP
```

### 5. Deployment Application

Créer `k8s/app.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: devops-app
  namespace: devops-training
  labels:
    app: devops-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: devops-app
  template:
    metadata:
      labels:
        app: devops-app
    spec:
      containers:
        - name: app
          image: devops-app:1.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: app-config
            - secretRef:
                name: app-secrets
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 15
            periodSeconds: 20
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 250m
              memory: 256Mi
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
---
apiVersion: v1
kind: Service
metadata:
  name: devops-app-svc
  namespace: devops-training
spec:
  selector:
    app: devops-app
  ports:
    - port: 80
      targetPort: 3000
  type: ClusterIP
```

### 5bis. Ingress (optionnel)

Ajoutez un Ingress **uniquement** si votre cluster dispose déjà d'un ingress controller.

Créer `k8s/ingress.yaml` :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: devops-app-ingress
  namespace: devops-training
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: devops.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: devops-app-svc
                port:
                  number: 80
```

### 6. Déployer

```bash
# Avec minikube : charger l'image locale
minikube image load devops-app:1.0.0

# Avec kind : charger l'image dans le cluster
kind load docker-image devops-app:1.0.0 --name devops-training

# Appliquer dans l'ordre
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/postgres.yaml
kubectl apply -f k8s/app.yaml

# Optionnel : seulement si ingress controller installé
kubectl apply -f k8s/ingress.yaml

# Ou tout d'un coup
kubectl apply -f k8s/
```

### 7. Vérifier

```bash
# Pods
kubectl get pods -w

# Services
kubectl get svc

# Logs
kubectl logs -l app=devops-app --tail=50

# Décrire un pod en erreur
kubectl describe pod <pod-name>

# Accéder au service (minikube)
minikube service devops-app-svc -n devops-training

# Ou port-forward
kubectl port-forward svc/devops-app-svc 8080:80
curl http://localhost:8080
```

Si vous avez installé un ingress controller :

```bash
echo "127.0.0.1 devops.local" | sudo tee -a /etc/hosts
curl http://devops.local
```

### 8. Scaling et rolling update

```bash
# Scaler
kubectl scale deployment devops-app --replicas=5
kubectl get pods -w

# Rolling update (changer l'image)
kubectl set image deployment/devops-app app=devops-app:1.0.1
kubectl rollout status deployment/devops-app

# Rollback
kubectl rollout undo deployment/devops-app
kubectl rollout history deployment/devops-app
```

## Livrable
- Namespace dédié avec ConfigMap et Secret
- Deployment PostgreSQL avec PVC et readiness probe
- Deployment application avec 3 replicas, probes et resource limits
- Service ClusterIP
- Ingress optionnel si le cluster le supporte
- Démonstration de scaling et rolling update

## Aide

### Commandes essentielles
```bash
kubectl get all                    # Vue globale
kubectl logs -f <pod>              # Logs en direct
kubectl exec -it <pod> -- sh       # Shell dans un pod
kubectl top pods                   # Ressources (si metrics-server)
kubectl get events --sort-by=.metadata.creationTimestamp
```

### Nettoyage
```bash
kubectl delete namespace devops-training
```
