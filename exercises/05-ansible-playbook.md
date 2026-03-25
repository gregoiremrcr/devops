# Ansible — Playbooks & Roles

## Objectif
Écrire des playbooks Ansible pour configurer des serveurs, comprendre l'idempotence et structurer le code en roles.

## Contexte
Nous utiliserons des conteneurs Docker comme "serveurs cibles" pour simuler un environnement multi-machines sans avoir besoin de VMs.

## Consignes

### 1. Infrastructure de test

Créer `infra/ansible/docker-compose.yml` :

```yaml
services:
  node1:
    image: ubuntu:22.04
    container_name: ansible-node1
    command: sleep infinity
    networks:
      - ansible-net

  node2:
    image: ubuntu:22.04
    container_name: ansible-node2
    command: sleep infinity
    networks:
      - ansible-net

networks:
  ansible-net:
    name: ansible-net
```

```bash
cd infra/ansible
docker compose up -d
```

### 2. Inventaire

Créer `infra/ansible/inventory.yml` :

```yaml
all:
  vars:
    ansible_connection: docker
  children:
    webservers:
      hosts:
        ansible-node1:
    appservers:
      hosts:
        ansible-node2:
```

Vérifier la connexion :
```bash
ansible all -i inventory.yml -m ping
```

### 3. Premier playbook

Créer `infra/ansible/playbook-base.yml` :

```yaml
---
- name: Configuration de base des serveurs
  hosts: all
  become: true
  vars:
    packages:
      - curl
      - wget
      - vim
      - htop
      - net-tools

  tasks:
    - name: Mettre à jour le cache apt
      apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Installer les paquets de base
      apt:
        name: "{{ packages }}"
        state: present

    - name: Créer le répertoire d'application
      file:
        path: /opt/app
        state: directory
        owner: root
        group: root
        mode: '0755'

    - name: Copier le fichier de configuration
      copy:
        content: |
          # Configuration de l'application
          APP_ENV=production
          APP_LOG_LEVEL=info
          APP_PORT=3000
        dest: /opt/app/.env
        mode: '0600'

    - name: Afficher les infos système
      debug:
        msg: |
          Hostname: {{ ansible_hostname }}
          OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
```

```bash
ansible-playbook -i inventory.yml playbook-base.yml
```

### 4. Playbook avec handlers

Créer `infra/ansible/playbook-nginx.yml` :

```yaml
---
- name: Installer et configurer Nginx
  hosts: webservers
  become: true

  handlers:
    - name: restart nginx
      shell: nginx -g 'daemon off;' &
      async: 10
      poll: 0

  tasks:
    - name: Installer Nginx
      apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Configurer le site
      copy:
        content: |
          server {
              listen 80 default_server;
              root /var/www/html;
              index index.html;

              location / {
                  try_files $uri $uri/ =404;
              }

              location /health {
                  return 200 '{"status": "ok"}';
                  add_header Content-Type application/json;
              }
          }
        dest: /etc/nginx/sites-available/default
      notify: restart nginx

    - name: Déployer la page d'accueil
      copy:
        content: |
          <!DOCTYPE html>
          <html>
          <head><title>DevOps Training</title></head>
          <body>
            <h1>Serveur configuré par Ansible</h1>
            <p>Host: {{ ansible_hostname }}</p>
          </body>
          </html>
        dest: /var/www/html/index.html
```

### 5. Structure en roles

```bash
mkdir -p roles/{base,nginx,app}/{tasks,handlers,templates,files,vars,defaults}
```

**roles/base/tasks/main.yml** :
```yaml
---
- name: Mettre à jour le cache apt
  apt:
    update_cache: yes
    cache_valid_time: 3600

- name: Installer les paquets de base
  apt:
    name: "{{ base_packages }}"
    state: present
```

**roles/base/defaults/main.yml** :
```yaml
---
base_packages:
  - curl
  - wget
  - vim
  - htop
  - net-tools
```

**roles/nginx/tasks/main.yml** :
```yaml
---
- name: Installer Nginx
  apt:
    name: nginx
    state: present

- name: Configurer le site
  template:
    src: default.conf.j2
    dest: /etc/nginx/sites-available/default
  notify: restart nginx

- name: Déployer la page d'accueil
  template:
    src: index.html.j2
    dest: /var/www/html/index.html
```

**roles/nginx/templates/default.conf.j2** :
```
server {
    listen {{ nginx_port }} default_server;
    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location /health {
        return 200 '{"status": "ok", "host": "{{ ansible_hostname }}"}';
        add_header Content-Type application/json;
    }
}
```

**roles/nginx/handlers/main.yml** :
```yaml
---
- name: restart nginx
  shell: nginx -g 'daemon off;' &
  async: 10
  poll: 0
```

### 6. Playbook principal avec roles

Créer `infra/ansible/site.yml` :

```yaml
---
- name: Configuration commune
  hosts: all
  become: true
  roles:
    - base

- name: Serveurs web
  hosts: webservers
  become: true
  roles:
    - nginx

- name: Serveurs d'application
  hosts: appservers
  become: true
  roles:
    - app
```

### 7. Démontrer l'idempotence

```bash
# Première exécution : tout change
ansible-playbook -i inventory.yml site.yml

# Deuxième exécution : rien ne change (idempotent !)
ansible-playbook -i inventory.yml site.yml
# → changed=0
```

## Livrable
- Inventaire avec 2 groupes de hosts
- Playbook de base avec variables et handlers
- Au moins 2 roles structurés (base + nginx)
- Démonstration de l'idempotence (2 runs identiques)

## Aide

### Commandes utiles
```bash
# Vérifier la syntaxe
ansible-playbook --syntax-check playbook.yml

# Dry run (check mode)
ansible-playbook -i inventory.yml playbook.yml --check

# Limiter à un host
ansible-playbook -i inventory.yml site.yml --limit ansible-node1

# Verbose
ansible-playbook -i inventory.yml site.yml -vvv

# Lister les tâches
ansible-playbook -i inventory.yml site.yml --list-tasks
```

### Nettoyage
```bash
docker compose down
```
