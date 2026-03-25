# Mise en place de l'environnement

## Objectif
Installer et vérifier tous les outils nécessaires pour les 3 jours de formation.

## Pré-requis système
- Un terminal (bash/zsh)
- Git installé
- Docker Desktop ou Docker Engine installé et lancé
- Un compte GitHub
- Sous Windows : **WSL2 + Ubuntu recommandé** pour tous les labs CLI

## Consignes

### 1. Vérifier les outils de base

```bash
# Git
git --version
# attendu : >= 2.30

# Docker
docker --version
docker compose version
# attendu : Docker >= 24, Compose >= 2.20

# kubectl
kubectl version --client
# attendu : >= 1.28

# Terraform
terraform --version
# attendu : >= 1.6

# Ansible
ansible --version
# attendu : >= 2.15

# Helm
helm version --short
# attendu : >= 3.14

# ArgoCD CLI
argocd version --client
# attendu : client disponible
```

### 2. Installer les outils manquants

#### macOS (Homebrew)
```bash
brew install git terraform ansible kubectl helm argocd
brew install --cask docker
```

#### Ubuntu/Debian
```bash
# Terraform
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common
wget -O- https://apt.releases.hashicorp.com/gpg | \
  gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

# Ansible
sudo apt install ansible

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# ArgoCD CLI
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd && sudo mv argocd /usr/local/bin/
```

#### Windows (avec Chocolatey)
```powershell
choco install git terraform kubectl kubernetes-helm kind
wsl --install -d Ubuntu
```

Puis dans **Ubuntu via WSL2** :

```bash
sudo apt-get update
sudo apt-get install -y ansible
```

> **Important** : utilisez Ansible depuis Linux/macOS/WSL2. Ne partez pas sur un control node Windows natif pour cette formation.

### 3. Cluster Kubernetes local

Nous utiliserons **minikube** ou **kind** pour le Jour 2 :

```bash
# Option A : minikube
brew install minikube   # ou choco install minikube
minikube start --cpus=2 --memory=4096

# Option B : kind (Kubernetes IN Docker)
brew install kind       # ou go install sigs.k8s.io/kind@latest
kind create cluster --name devops-training
```

Pour les labs utilisant un **Ingress**, deux options :

```bash
# Minikube : addon intégré
minikube addons enable ingress

# Kind : rester sur port-forward tant qu'aucun ingress controller n'est installé
```

### 4. Vérification finale

```bash
# Docker fonctionne
docker run --rm hello-world

# kubectl connecté au cluster
kubectl cluster-info
kubectl get nodes

# Terraform init fonctionne
mkdir -p /tmp/tf-test && cd /tmp/tf-test
echo 'output "hello" { value = "world" }' > main.tf
terraform init && terraform apply -auto-approve
cd - && rm -rf /tmp/tf-test
```

### 5. Cloner le repo de formation

```bash
git clone <url-du-repo> devops-training
cd devops-training
cp -R starter-code/devops-app app
```

## Livrable
- Tous les outils installés et fonctionnels
- Cluster K8s local opérationnel
- Repo cloné
- Starter applicatif copié dans `app/`

## Aide

Si Docker Desktop ne démarre pas :
- macOS : vérifier les permissions dans Préférences Système > Sécurité
- Windows : activer WSL2 et Hyper-V
- Linux : ajouter votre user au groupe docker : `sudo usermod -aG docker $USER`

Si minikube échoue, utiliser kind comme alternative (plus léger).
