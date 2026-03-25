# Terraform — Modules & Remote State

## Objectif
Créer des modules Terraform réutilisables et configurer un backend distant pour le state.

## Consignes

### 1. Structure modulaire

```
infra/terraform/
  modules/
    webapp/
      main.tf
      variables.tf
      outputs.tf
    database/
      main.tf
      variables.tf
      outputs.tf
  environments/
    dev/
      main.tf
      terraform.tfvars
    prod/
      main.tf
      terraform.tfvars
```

### 2. Module webapp

**modules/webapp/variables.tf** :
```hcl
variable "app_name" {
  type = string
}

variable "environment" {
  type = string
}

variable "image" {
  type    = string
  default = "nginx:alpine"
}

variable "port" {
  type    = number
  default = 80
}

variable "replicas" {
  type    = number
  default = 1
}

variable "network_id" {
  type = string
}
```

**modules/webapp/main.tf** :
```hcl
resource "docker_image" "app" {
  name         = var.image
  keep_locally = true
}

resource "docker_container" "app" {
  count = var.replicas
  name  = "${var.app_name}-${var.environment}-${count.index}"
  image = docker_image.app.image_id

  ports {
    internal = 80
    external = var.port + count.index
  }

  networks_advanced {
    name = var.network_id
  }

  labels {
    label = "app"
    value = var.app_name
  }

  labels {
    label = "env"
    value = var.environment
  }
}
```

**modules/webapp/outputs.tf** :
```hcl
output "container_ids" {
  value = docker_container.app[*].id
}

output "urls" {
  value = [
    for c in docker_container.app :
    "http://localhost:${c.ports[0].external}"
  ]
}
```

### 3. Module database

**modules/database/main.tf** :
```hcl
resource "docker_image" "db" {
  name         = "postgres:16-alpine"
  keep_locally = true
}

resource "docker_container" "db" {
  name  = "${var.app_name}-db-${var.environment}"
  image = docker_image.db.image_id

  env = [
    "POSTGRES_DB=${var.db_name}",
    "POSTGRES_USER=${var.db_user}",
    "POSTGRES_PASSWORD=${var.db_password}",
  ]

  ports {
    internal = 5432
    external = var.db_port
  }

  networks_advanced {
    name = var.network_id
  }

  volumes {
    host_path      = "/tmp/${var.app_name}-db-${var.environment}"
    container_path = "/var/lib/postgresql/data"
  }
}
```

**modules/database/variables.tf** :
```hcl
variable "app_name" { type = string }
variable "environment" { type = string }
variable "network_id" { type = string }
variable "db_name" {
  type    = string
  default = "appdb"
}
variable "db_user" {
  type    = string
  default = "appuser"
}
variable "db_password" {
  type      = string
  sensitive = true
}
variable "db_port" {
  type    = number
  default = 5432
}
```

**modules/database/outputs.tf** :
```hcl
output "connection_string" {
  value     = "postgresql://${var.db_user}:${var.db_password}@localhost:${var.db_port}/${var.db_name}"
  sensitive = true
}

output "container_id" {
  value = docker_container.db.id
}
```

### 4. Environnement dev

**environments/dev/main.tf** :
```hcl
terraform {
  required_version = ">= 1.6"
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 3.0"
    }
  }
}

provider "docker" {}

resource "docker_network" "main" {
  name = "devops-${var.environment}"
}

module "webapp" {
  source      = "../../modules/webapp"
  app_name    = var.app_name
  environment = var.environment
  port        = var.web_port
  replicas    = var.web_replicas
  network_id  = docker_network.main.name
}

module "database" {
  source      = "../../modules/database"
  app_name    = var.app_name
  environment = var.environment
  db_password = var.db_password
  db_port     = var.db_port
  network_id  = docker_network.main.name
}

variable "app_name" { type = string }
variable "environment" { type = string }
variable "web_port" { type = number }
variable "web_replicas" { type = number }
variable "db_password" {
  type      = string
  sensitive = true
}
variable "db_port" { type = number }

output "web_urls" {
  value = module.webapp.urls
}

output "db_connection" {
  value     = module.database.connection_string
  sensitive = true
}
```

**environments/dev/terraform.tfvars** :
```hcl
app_name     = "devops-app"
environment  = "dev"
web_port     = 8080
web_replicas = 2
db_port      = 5432
```

### 5. Workspaces (bonus)

```bash
cd environments/dev

# Créer des workspaces
terraform workspace new dev
terraform workspace new staging

# Lister
terraform workspace list

# Basculer
terraform workspace select dev
terraform apply -var-file="terraform.tfvars" -var="db_password=secret123"

# Le state est isolé par workspace
terraform workspace select staging
terraform apply -var-file="terraform.tfvars" -var="db_password=secret456"
```

## Livrable
- 2 modules Terraform (webapp + database)
- 1 environnement dev fonctionnel utilisant les modules
- Variables sensibles gérées correctement
- Cycle complet apply → verify → destroy

## Aide

### Passer le mot de passe sans le commiter
```bash
# Via variable d'environnement
export TF_VAR_db_password="mon-secret"
terraform apply -var-file="terraform.tfvars"

# Ou via un fichier .auto.tfvars (dans .gitignore !)
echo 'db_password = "mon-secret"' > secret.auto.tfvars
```

### Visualiser les dépendances
```bash
terraform graph | dot -Tpng > graph.png
```
