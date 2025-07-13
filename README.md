# Déploiement du projet *Projet-pro* avec Ansible

## Prérequis

- **5 machines virtuelles** configurées en rocky 9
- Un **nom de domaine** pointant vers l'IP du reverse proxy
- **Docker installé** sur la machine de gestion (et exécutable sans `sudo`)

---

## Variables

Toutes les variables sont regroupées dans le dossier `vars` de chaque rôle Ansible. Il faut les adapters pour ton environnement :

| Rôle              | Variables à modifier                                               |
|-------------------|--------------------------------------------------------------------|
| `users`           | `ssh_key`, `user_name`                                             |
| `nginx`           | `email`, `domains`                                                 |
| `keycloak`        | `grafana_root_url`, `gitlab_root_url`,`keycloak_url`               |
| `grafana-prom`    | `keycloak_url`, `keycloak_container`                               |
| `gitlab`          | `external_url`                                                     |
| `nfs-client`      | `nfs_ip_server`                                                    |
| `nfs`             | `nfs_ip_client1`, `nfs_ip_client2`                                 |

---

## Configuration de l’inventaire

Dans le fichier `inventory/hosts.yml`, remplace les IPs et utilisateurs SSH pour chaque groupe de machines :

```yaml
[gitlab]
192.168.X.X ansible_user=...

[keycloak]
192.168.X.X ansible_user=...

...
```
## Start:
Une fois les vms up et les variables changer, à la racine du dossier:

```
ansible-playbook -i inventory/hosts.yml playbook.yml -v

```

Lorsque tout est up, il est possible de créé un user sur keycloak pour la connexion en SSO

Ensuite récuperer le token de registration pour le runner dans la partie admin de gitlab, puis changer la varible token_registration et changer la variable external_url aussi dans le rôle **runner-ci**.

### Lancer maintenant:
```
ansible-playbook -i inventory/hosts.yml playbook2.yml -v
```

# Indicateurs et Sauvegardes de l'Infrastructure

---

## 1. Indicateurs de supervision

### 1.1. Ports à surveiller

| Composant     | Ports         | Protocole |
|---------------|---------------|-----------|
| GitLab        | 80, 443, 22   | TCP       |
| Grafana       | 3000          | TCP       |
| Prometheus    | 9090          | TCP       |
| Node Exporter | 9100          | TCP       |
| cAdvisor      | 8080          | TCP       |
| PostgreSQL    | 5432          | TCP       |
| Redis         | 6379          | TCP       |
| Keycloak      | 8080, 8443    | TCP       |
| NGINX         | 80, 443       | TCP       |

---

### 1.2. Services à surveiller

- **Tous les serveurs** :  
  `node_exporter`, `ssh`, `nfs-utils`, `docker`, `systemd`

- **GitLab** :  
  `gitlab-runsvdir`, `gitlab-workhorse`, `gitlab-sidekiq`, `gitlab-puma`, `gitlab-gitaly`

- **Base de données** :  
  `postgresql`, `nfs-kernel-server`

- **Redis** :  
  `redis-server`

- **Supervision** :  
  `prometheus`, `grafana-server`, `cadvisor`

- **SSO** :  
  `keycloak`

- **Proxy** :  
  `nginx`

---

### 1.3. Processus critiques à surveiller

- `gitlab-runner`  
- `java` (Keycloak fonctionne sur JVM)  
- `prometheus`  
- `grafana-server`  
- `redis-server`  
- `postgres`  
- `cadvisor`  
- `nginx`  
- `node_exporter`

---

### 1.4. Adresses IP à surveiller

- 51.38.234.58  
- 51.38.232.14  
- 54.36.101.87  
- 152.228.172.167  
- 51.38.239.212  

---

## 2. Indicateurs de sauvegarde

### 2.1. Fichiers de configuration à sauvegarder

| Composant   | Chemins |
|-------------|---------|
| GitLab      | `/etc/gitlab/`, `/var/opt/gitlab/gitlab.rb` |
| PostgreSQL  | `/var/lib/pgsql/data/postgresql.conf` |
| Redis       | `/etc/redis/redis.conf` |
| Prometheus  | `/etc/prometheus/prometheus.yml` |
| Grafana     | `/etc/grafana/grafana.ini` |
| Keycloak    | `/opt/keycloak/standalone/configuration/standalone.xml` |
| NGINX       | `/etc/nginx/nginx.conf`, `/etc/nginx/sites-available/*`, `/etc/nginx/sites-enabled/*` |

---

### 2.2. Dossiers de données à sauvegarder

| Composant     | Dossiers |
|---------------|----------|
| GitLab        | `/var/opt/gitlab/`, `/var/log/gitlab/`, `/var/opt/gitlab/backups/`,  |
| PostgreSQL    | `/var/lib/pgsql/data/` |
| Redis         | `/var/lib/redis/` |
| Prometheus    | `/var/lib/prometheus/` |
| Grafana       | `/var/lib/grafana/`, `/etc/grafana/provisioning/`, `/var/lib/grafana/dashboards` |
| Keycloak      | `/var/lib/postgresql/data` |
| NFS Server    | `/srv/nfs/` |
| NFS Client    | `/srv/gitlab-nfs/repositories`,`/srv/gitlab-nfs/uploads`,`/srv/gitlab-nfs/shared`,`/srv/gitlab-nfs/gitlab-ci/builds` |

---

### 2.3. Sauvegarde des bases de données

#### PostgreSQL

- **Nom de la base** : `gitlabhq_production`
- **Méthode de connexion** :
  ```bash
  psql -U gitlab -h localhost -d gitlabhq_production
  ```
- **Méthode de sauvegarde** :
  ```bash
  pg_dump gitlabhq_production > backup.sql
  ```
  ou via :
  ```bash
  gitlab-backup create
  ```

#### Redis

- **Connexion** :
  ```bash
  redis-cli
  ```
- **Fichier à sauvegarder** :
  - `dump.rdb`

#### Keycloak

- Sauvegarder les fichiers XML/JSON des configurations et exports internes.

#### Grafana

- Sauvegarder :
  - Dashboards (format JSON)
  - Datasources
  - Utilisateurs
