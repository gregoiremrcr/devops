# Terraform — Fondamentaux

## Objectif
Écrire votre première configuration Terraform, comprendre le cycle plan/apply et gérer le state.

## Consignes

### 1. Premier projet Terraform

Créer `infra/terraform/main.tf` :

```hcl
# Configuration Terraform
terraform {
  required_version = ">= 1.6"
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 3.0"
    }
  }
}

# Provider Docker (local)
provider "docker" {}

# --- Réseau ---
resource "docker_network" "app_network" {
  name = "devops-network"
}

# --- Image Nginx ---
resource "docker_image" "nginx" {
  name         = "nginx:alpine"
  keep_locally = true
}

# --- Conteneur Nginx ---
resource "docker_container" "web" {
  name  = "devops-web"
  image = docker_image.nginx.image_id

  ports {
    internal = 80
    external = 8080
  }

  networks_advanced {
    name = docker_network.app_network.name
  }

  env = [
    "NGINX_HOST=localhost",
    "NGINX_PORT=80"
  ]
}

# --- Outputs ---
output "web_url" {
  value       = "http://localhost:${docker_container.web.ports[0].external}"
  description = "URL du serveur web"
}

output "container_id" {
  value = docker_container.web.id
}
```

### 2. Cycle Terraform

```bash
cd infra/terraform

# Initialiser (télécharge le provider)
terraform init

# Voir ce qui va être créé
terraform plan

# Appliquer
terraform apply

# Vérifier
curl http://localhost:8080
docker ps

# Voir le state
terraform show
terraform state list
```

### 3. Variables

Créer `infra/terraform/variables.tf` :

```hcl
variable "app_name" {
  description = "Nom de l'application"
  type        = string
  default     = "devops-app"
}

variable "web_port" {
  description = "Port externe du serveur web"
  type        = number
  default     = 8080
}

variable "environment" {
  description = "Environnement (dev, staging, prod)"
  type        = string
  default     = "dev"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "L'environnement doit être dev, staging ou prod."
  }
}
```

Modifier `main.tf` pour utiliser les variables :

```hcl
resource "docker_container" "web" {
  name  = "${var.app_name}-${var.environment}"
  image = docker_image.nginx.image_id

  ports {
    internal = 80
    external = var.web_port
  }
  # ...
}
```

### 4. Fichier de variables

Créer `infra/terraform/dev.tfvars` :

```hcl
app_name    = "devops-app"
web_port    = 8080
environment = "dev"
```

Créer `infra/terraform/prod.tfvars` :

```hcl
app_name    = "devops-app"
web_port    = 80
environment = "prod"
```

```bash
# Appliquer avec un fichier de variables
terraform plan -var-file="dev.tfvars"
terraform apply -var-file="dev.tfvars"
```

### 5. Data sources et locals

Ajouter à `main.tf` :

```hcl
locals {
  common_tags = {
    project     = var.app_name
    environment = var.environment
    managed_by  = "terraform"
  }

  container_name = "${var.app_name}-${var.environment}"
}

data "docker_network" "bridge" {
  name = "bridge"
}
```

### 6. Nettoyage

```bash
# Détruire toute l'infra
terraform destroy

# Vérifier que tout est propre
docker ps
docker network ls
```

## Livrable
- Configuration Terraform fonctionnelle avec provider Docker
- Variables et fichiers tfvars pour dev et prod
- Outputs affichant l'URL et l'ID du conteneur
- Cycle complet init → plan → apply → destroy maîtrisé

## Aide

### Commandes utiles
```bash
terraform fmt          # Formater le code
terraform validate     # Valider la syntaxe
terraform plan -out=tfplan  # Sauvegarder le plan
terraform apply tfplan      # Appliquer un plan sauvegardé
```

### Erreurs courantes
- **Provider not found** : relancer `terraform init`
- **Port already in use** : changer `web_port` ou arrêter le conteneur existant
- **State lock** : un autre process Terraform tourne — attendre ou supprimer le lock
