# GitOps avec ArgoCD

## Objectif
Mettre en place un workflow GitOps avec ArgoCD : déploiement déclaratif, synchronisation automatique et gestion des environnements via Git.

## Contexte
GitOps = Git comme source de vérité unique pour l'état de l'infrastructure et des applications. ArgoCD surveille un repo Git et synchronise automatiquement le cluster Kubernetes.

## Consignes

### 1. Installer ArgoCD

```bash
# Créer le namespace
kubectl create namespace argocd

# Installer ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Attendre que les pods soient prêts
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=120s

# Récupérer le mot de passe admin
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo

# Port-forward pour accéder à l'UI
kubectl port-forward svc/argocd-server -n argocd 8443:443
# → Ouvrir https://localhost:8443
# Login : admin / <password récupéré>
```

### 2. Installer le CLI ArgoCD

```bash
# macOS
brew install argocd

# Linux
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd && sudo mv argocd /usr/local/bin/

# Se connecter
argocd login localhost:8443 --insecure
```

### 3. Choisir une stratégie d'image

ArgoCD **ne build pas d'image**. L'image référencée dans Git doit déjà être disponible dans une registry accessible par le cluster.

Stratégie recommandée pour le cours :

- Registry : `ghcr.io/<your-user>/devops-app`
- Tags : immuables (`sha-abc1234`, `1.0.0`, `1.0.1`)
- Mise à jour GitOps : la CI modifie seulement le tag dans l'overlay

### 4. Structure du repo GitOps

Créer un repo dédié (ou un dossier `gitops/`) :

```
gitops/
  apps/
    devops-app/
      base/
        deployment.yaml
        service.yaml
        kustomization.yaml
      overlays/
        dev/
          kustomization.yaml
          replicas-patch.yaml
        staging/
          kustomization.yaml
          replicas-patch.yaml
        prod/
          kustomization.yaml
          replicas-patch.yaml
  argocd/
    app-dev.yaml
    app-staging.yaml
    app-prod.yaml
    project.yaml
```

### 5. Manifestes de base

**gitops/apps/devops-app/base/deployment.yaml** :
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: devops-app
spec:
  replicas: 1
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
          image: ghcr.io/<your-user>/devops-app:1.0.0
          ports:
            - containerPort: 3000
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 250m
              memory: 256Mi
```

**gitops/apps/devops-app/base/service.yaml** :
```yaml
apiVersion: v1
kind: Service
metadata:
  name: devops-app-svc
spec:
  selector:
    app: devops-app
  ports:
    - port: 80
      targetPort: 3000
  type: ClusterIP
```

**gitops/apps/devops-app/base/kustomization.yaml** :
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

### 6. Overlays par environnement

**gitops/apps/devops-app/overlays/dev/kustomization.yaml** :
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
namePrefix: dev-
namespace: devops-dev
patches:
  - path: replicas-patch.yaml
images:
  - name: ghcr.io/<your-user>/devops-app
    newTag: sha-abc1234
```

**gitops/apps/devops-app/overlays/dev/replicas-patch.yaml** :
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: devops-app
spec:
  replicas: 1
```

**gitops/apps/devops-app/overlays/prod/kustomization.yaml** :
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
namePrefix: prod-
namespace: devops-prod
patches:
  - path: replicas-patch.yaml
images:
  - name: ghcr.io/<your-user>/devops-app
    newTag: 1.0.0
```

**gitops/apps/devops-app/overlays/prod/replicas-patch.yaml** :
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: devops-app
spec:
  replicas: 5
```

### 7. Application ArgoCD

**gitops/argocd/project.yaml** :
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: devops-training
  namespace: argocd
spec:
  description: "Projet de formation DevOps"
  sourceRepos:
    - 'https://github.com/<your-user>/devops-gitops.git'
  destinations:
    - namespace: devops-*
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
```

**gitops/argocd/app-dev.yaml** :
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: devops-app-dev
  namespace: argocd
spec:
  project: devops-training
  source:
    repoURL: https://github.com/<your-user>/devops-gitops.git
    targetRevision: main
    path: apps/devops-app/overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: devops-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### 8. Déployer avec ArgoCD

```bash
# Créer le projet
kubectl apply -f gitops/argocd/project.yaml

# Créer l'application dev
kubectl apply -f gitops/argocd/app-dev.yaml

# Vérifier le statut
argocd app list
argocd app get devops-app-dev

# Forcer une synchronisation
argocd app sync devops-app-dev

# Voir l'historique
argocd app history devops-app-dev
```

### 9. Workflow GitOps complet

```
1. Développeur pousse du code → CI build + test + scan
2. CI publie l'image dans la registry avec un tag immuable
3. CI met à jour le tag dans le repo GitOps
4. ArgoCD détecte le changement et sync automatiquement
5. Le cluster est mis à jour sans kubectl manuel

Code Repo          Registry           GitOps Repo        ArgoCD          Cluster
   │                  │                   │                 │               │
   ├─ push ──→ CI ───▶│                   │                 │               │
   │           │      └─ image sha ───▶  │                 │               │
   │           └───────── update tag ───▶│        detect ──┤               │
   │                                      │                 ├── sync ─────→ │
```

## Livrable
- ArgoCD installé et accessible
- Repo GitOps avec base + overlays (dev/staging/prod)
- Image poussée dans une registry accessible par le cluster
- Application ArgoCD avec sync automatique
- Démonstration : modifier le repo GitOps → ArgoCD sync automatiquement

## Aide

### ArgoCD UI
- Vert = Synced + Healthy
- Jaune = OutOfSync (changement non appliqué)
- Rouge = Degraded (pods en erreur)

### Rollback via ArgoCD
```bash
argocd app history devops-app-dev
argocd app rollback devops-app-dev <revision>
```

### Nettoyage
```bash
argocd app delete devops-app-dev
kubectl delete namespace argocd
```
