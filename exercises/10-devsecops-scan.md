# DevSecOps — Scan & Conformité

## Objectif
Intégrer la sécurité dans le pipeline CI/CD : scanner les images Docker, les dépendances, les fichiers IaC et appliquer des politiques de conformité.

## Consignes

### 1. Scanner les images Docker avec Trivy

```bash
# Installer Trivy
brew install trivy  # macOS
# ou
docker pull aquasec/trivy

# Scanner une image
trivy image devops-app:1.0.0

# Scanner uniquement les vulnérabilités critiques et hautes
trivy image --severity HIGH,CRITICAL devops-app:1.0.0

# Scanner et échouer si vulnérabilités critiques
trivy image --severity CRITICAL --exit-code 1 devops-app:1.0.0

# Format JSON pour CI
trivy image --format json --output results.json devops-app:1.0.0

# Scanner une image officielle
trivy image nginx:alpine
trivy image node:20-alpine
```

### 2. Scanner le filesystem (dépendances)

```bash
# Scanner les dépendances Node.js
trivy fs --scanners vuln app/

# Scanner le Dockerfile
trivy config app/Dockerfile

# Scanner les fichiers Terraform
trivy config infra/terraform/

# Scanner tout le repo
trivy repo .
```

### 3. Intégrer dans le pipeline CI/CD

Ajouter à `.github/workflows/ci.yml` :

```yaml
  security-scan:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Scanner les dépendances
      - name: Scan dependencies
        uses: aquasecurity/trivy-action@0.33.1
        with:
          scan-type: 'fs'
          scan-ref: 'app/'
          severity: 'HIGH,CRITICAL'
          exit-code: '1'
          format: 'table'

      # Scanner l'image Docker
      - name: Build image for scanning
        run: docker build -t devops-app:scan app/

      - name: Scan Docker image
        uses: aquasecurity/trivy-action@0.33.1
        with:
          image-ref: 'devops-app:scan'
          severity: 'HIGH,CRITICAL'
          exit-code: '1'
          format: 'sarif'
          output: 'trivy-results.sarif'

      # Upload SARIF pour GitHub Security tab
      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'

      # Scanner IaC (Terraform)
      - name: Scan IaC
        uses: aquasecurity/trivy-action@0.33.1
        with:
          scan-type: 'config'
          scan-ref: 'infra/'
          severity: 'HIGH,CRITICAL'
          exit-code: '0'  # Warning only pour IaC
```

> Même en formation, évitez `@master`. En production, pinnez les actions tierces à un **SHA complet**.

### 4. Scanner les secrets dans le code

```bash
# Installer gitleaks
brew install gitleaks  # macOS

# Scanner le repo
gitleaks detect --source . --verbose

# Scanner avant chaque commit (pre-commit hook)
gitleaks protect --source . --verbose
```

Ajouter au pipeline :
```yaml
      - name: Scan for secrets
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### 5. Politiques OPA (Open Policy Agent)

Créer `policies/dockerfile.rego` :

```rego
package dockerfile

# Interdire l'exécution en root
deny[msg] {
    input[i].Cmd == "user"
    val := input[i].Value
    val[_] == "root"
    msg = "Les conteneurs ne doivent pas tourner en root"
}

# Exiger un USER dans le Dockerfile
deny[msg] {
    not any_user
    msg = "Le Dockerfile doit contenir une instruction USER"
}

any_user {
    input[i].Cmd == "user"
}

# Interdire le tag :latest
deny[msg] {
    input[i].Cmd == "from"
    val := input[i].Value
    contains(val[0], ":latest")
    msg = sprintf("Éviter le tag :latest, utiliser un tag spécifique : %s", [val[0]])
}

# Exiger un HEALTHCHECK
deny[msg] {
    not any_healthcheck
    msg = "Le Dockerfile doit contenir un HEALTHCHECK"
}

any_healthcheck {
    input[i].Cmd == "healthcheck"
}
```

Créer `policies/k8s.rego` :

```rego
package kubernetes

# Interdire les conteneurs privilégiés
deny[msg] {
    input.kind == "Deployment"
    container := input.spec.template.spec.containers[_]
    container.securityContext.privileged == true
    msg = sprintf("Le conteneur %s ne doit pas être privilégié", [container.name])
}

# Exiger des resource limits
deny[msg] {
    input.kind == "Deployment"
    container := input.spec.template.spec.containers[_]
    not container.resources.limits
    msg = sprintf("Le conteneur %s doit avoir des resource limits", [container.name])
}

# Interdire les images sans tag ou avec :latest
deny[msg] {
    input.kind == "Deployment"
    container := input.spec.template.spec.containers[_]
    not contains(container.image, ":")
    msg = sprintf("L'image %s doit avoir un tag spécifique", [container.image])
}

deny[msg] {
    input.kind == "Deployment"
    container := input.spec.template.spec.containers[_]
    endswith(container.image, ":latest")
    msg = sprintf("L'image %s ne doit pas utiliser le tag :latest", [container.image])
}
```

### 6. Checklist sécurité

Créer `SECURITY-CHECKLIST.md` dans votre repo :

```markdown
# Checklist Sécurité DevOps

## Images Docker
- [ ] Base image minimale (alpine)
- [ ] Pas de tag :latest
- [ ] User non-root
- [ ] HEALTHCHECK présent
- [ ] Scan Trivy sans vulnérabilité critique
- [ ] .dockerignore complet

## Kubernetes
- [ ] Resource limits sur chaque conteneur
- [ ] Pas de conteneur privilégié
- [ ] NetworkPolicies en place
- [ ] Secrets via Secret Manager (pas en clair)
- [ ] RBAC configuré

## Pipeline
- [ ] Secrets dans GitHub Secrets (pas dans le code)
- [ ] Scan de dépendances automatique
- [ ] Scan d'image automatique
- [ ] Détection de secrets (gitleaks)
- [ ] Environnements avec protection

## Infrastructure
- [ ] State Terraform chiffré
- [ ] Pas de credentials en dur dans les fichiers IaC
- [ ] Principe du moindre privilège (IAM)
```

## Livrable
- Pipeline avec scan Trivy intégré (images + dépendances + IaC)
- Détection de secrets avec gitleaks
- Au moins 2 politiques OPA écrites
- Checklist sécurité complétée pour votre projet

## Aide

### Ignorer des vulnérabilités connues
Créer `.trivyignore` :
```
# CVE acceptée car pas exploitable dans notre contexte
CVE-2023-XXXXX

# Faux positif
CVE-2024-YYYYY
```

### Fixer les vulnérabilités courantes
```bash
# Mettre à jour les dépendances
cd app && npm audit fix

# Mettre à jour l'image de base
# Remplacer node:20-alpine par la dernière version patchée
```
