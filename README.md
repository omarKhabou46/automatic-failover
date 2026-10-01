# PostgreSQL High Availability with Patroni, etcd, HAProxy and Docker

## 📌 Description

Ce projet met en place une architecture **PostgreSQL High Availability (HA)** basée sur :

- **PostgreSQL 16** — système de gestion de base de données
- **Patroni** — gestion du cluster PostgreSQL et failover automatique
- **etcd** — Distributed Configuration Store (DCS) utilisé pour l'élection du leader
- **HAProxy** — point d'accès unique pour les applications clientes
- **Docker / Docker Compose** — isolation et orchestration des services

L'objectif est de permettre à une application de continuer à accéder à PostgreSQL même lorsqu'un serveur PostgreSQL primaire tombe en panne.

---

## 🏗️ Architecture

```text
                         Spring Boot
                              |
                              v
                        +-----------+
                        |  HAProxy  |
                        |  :5435    |
                        +-----------+
                              |
                +-------------+-------------+
                |                           |
                v                           v
       +------------------+        +------------------+
       | postgres-primary |        | postgres-replica |
       |    PostgreSQL    |        |    PostgreSQL    |
       |     Patroni      |        |     Patroni      |
       +------------------+        +------------------+
                |                           |
                +-------------+-------------+
                              |
                              v
                         PostgreSQL WAL

                         +---------+
                         |   etcd  |
                         |  :2379  |
                         +---------+
                              ^
                              |
                         Patroni DCS
```

### Rôle de chaque composant

| Composant | Rôle |
|---|---|
| PostgreSQL | Stockage et gestion des données |
| WAL | Réplication des modifications entre PostgreSQL |
| Patroni | Supervision, gestion du leader et failover |
| etcd | Stockage de l'état partagé et du leader |
| HAProxy | Routage des connexions vers le PostgreSQL primaire |
| Docker | Exécution isolée des services |

---

# 🚀 1. Fonctionnement général

Le cluster contient deux instances PostgreSQL :

```text
postgres-primary
postgres-replica
```

À un instant donné, une seule instance est **PRIMARY** et l'autre est **SECONDARY**.

Par exemple :

```text
postgres-primary
      |
      | PRIMARY
      |
      v
postgres-replica
      |
      | SECONDARY
      |
      v
      WAL
```

Les modifications effectuées sur le primaire sont transmises au secondaire grâce à la réplication PostgreSQL basée sur le **WAL (Write-Ahead Log)**.

---

# 🔄 2. Patroni

Patroni est exécuté dans chaque conteneur PostgreSQL.

Il surveille :

- PostgreSQL
- l'état du cluster
- le leader enregistré dans etcd
- la disponibilité des autres nœuds

Patroni utilise etcd pour savoir quel nœud possède actuellement le rôle de leader.

Exemple :

```text
etcd
 |
 +-- /service/postgres-ha/leader
             |
             v
      postgres-primary
```

Le nœud qui possède le **leader lock** est le PostgreSQL primaire.

---

# 🗄️ 3. etcd

`etcd` est utilisé comme **DCS (Distributed Configuration Store)**.

Il ne contient pas les données PostgreSQL.

Son rôle est notamment de stocker :

- le leader actuel
- l'état du cluster
- certaines informations de configuration
- le leader lock

Exemple :

```bash
docker exec etcd etcdctl get /service/postgres-ha/leader
```

Résultat possible :

```text
postgres-primary
```

Après un failover :

```text
postgres-replica
```

---

# 🔁 4. PostgreSQL WAL Replication

PostgreSQL utilise le **WAL (Write-Ahead Log)** pour transmettre les modifications du primaire vers le secondaire.

Le flux est conceptuellement :

```text
Application
     |
     v
PRIMARY
     |
     | WAL
     v
SECONDARY
```

Sur le secondaire, on peut vérifier que PostgreSQL est bien en mode recovery :

```sql
SELECT pg_is_in_recovery();
```

Résultat :

```text
t
```

signifie que le serveur est secondaire.

Sur le primaire :

```text
f
```

signifie qu'il est primaire.

---

# 🛠️ 5. Configuration Docker

Les principaux services sont définis dans `docker-compose.yml`.

Exemple :

```yaml
services:

  postgres-primary:
    image: postgres-patroni:16
    container_name: postgres-primary
    environment:
      POSTGRES_DB: sales
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5433:5432"
    volumes:
      - postgres_primary_data:/var/lib/postgresql/data
      - ./patroni-primary.yml:/etc/patroni/patroni.yml:ro
    command:
      - patroni
      - /etc/patroni/patroni.yml

  postgres-replica:
    image: postgres-patroni:16
    container_name: postgres-replica
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5434:5432"
      - "8009:8008"
    volumes:
      - postgres_replica_data:/var/lib/postgresql/data
      - ./patroni-replica.yml:/etc/patroni/patroni.yml:ro
    command:
      - patroni
      - /etc/patroni/patroni.yml

  etcd:
    image: quay.io/coreos/etcd:v3.5.15
    container_name: etcd
    command:
      - etcd
      - --name=etcd
      - --advertise-client-urls=http://etcd:2379
      - --listen-client-urls=http://0.0.0.0:2379
      - --initial-advertise-peer-urls=http://etcd:2380
      - --listen-peer-urls=http://0.0.0.0:2380
      - --initial-cluster=etcd=http://etcd:2380
      - --initial-cluster-state=new
      - --initial-cluster-token=postgres-ha

  haproxy:
    image: haproxy:3.0
    container_name: haproxy
    ports:
      - "5435:5432"
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
```

---

# 🐘 6. Image PostgreSQL + Patroni

Une image Docker personnalisée est utilisée afin d'installer Patroni :

```dockerfile
FROM postgres:16

RUN apt-get update \
    && apt-get install -y python3 python3-pip python3-venv \
    && python3 -m venv /opt/patroni \
    && /opt/patroni/bin/pip install --no-cache-dir patroni[etcd3] psycopg[binary] \
    && rm -rf /var/lib/apt/lists/*

ENV PATH="/opt/patroni/bin:$PATH"

USER postgres
```

Construction de l'image :

```bash
docker build -t postgres-patroni:16 .
```

Le `USER postgres` est important car PostgreSQL ne doit pas être exécuté avec l'utilisateur `root`.

---

# ⚙️ 7. Configuration Patroni

Exemple simplifié de `patroni-primary.yml` :

```yaml
scope: postgres-ha
name: postgres-primary

restapi:
  listen: 0.0.0.0:8008
  connect_address: postgres-primary:8008

etcd3:
  hosts: etcd:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10

    postgresql:
      use_pg_rewind: true
      use_slots: true

  initdb:
    - encoding: UTF8
    - data-checksums

postgresql:
  listen: 0.0.0.0:5432
  connect_address: postgres-primary:5432

  data_dir: /var/lib/postgresql/data

  authentication:
    superuser:
      username: postgres
      password: postgres

    replication:
      username: replicator
      password: replica_password

  parameters:
    wal_level: replica
    max_wal_senders: 10
    max_replication_slots: 10
```

La configuration du replica est similaire, mais son nom et son adresse changent :

```yaml
name: postgres-replica
```

et :

```yaml
connect_address: postgres-replica:8008
```

---

# ❤️ 8. Patroni REST API

Patroni expose une API HTTP permettant notamment de vérifier le rôle du serveur.

Par exemple :

```bash
docker run --rm \
  --network postgres-ha_default \
  curlimages/curl:latest \
  -i http://postgres-replica:8008/primary
```

Si le serveur est primaire, on obtient un HTTP `200`.

Exemple :

```json
{
  "state": "running",
  "role": "primary",
  "patroni": {
    "version": "4.1.5",
    "scope": "postgres-ha",
    "name": "postgres-replica"
  }
}
```

L'endpoint `/primary` est particulièrement utile pour HAProxy.

---

# 🔀 9. Failover automatique

L'un des objectifs principaux du projet est de tester le **failover automatique**.

Supposons :

```text
PRIMARY
postgres-primary

SECONDARY
postgres-replica
```

Si le primaire tombe :

```bash
docker stop postgres-primary
```

Patroni détecte que le leader n'est plus disponible.

Le processus devient alors :

```text
postgres-primary
       X
       |
       v
     FAIL
       |
       v
Patroni + etcd
       |
       v
postgres-replica
       |
       v
    PRIMARY
```

Le replica est automatiquement promu en primaire.

On peut vérifier :

```sql
SELECT pg_is_in_recovery();
```

Résultat :

```text
f
```

Le serveur est maintenant primaire.

---

# 🔍 10. Vérification de la réplication

Sur le secondaire :

```sql
SELECT
    status,
    sender_host,
    sender_port
FROM pg_stat_wal_receiver;
```

Exemple :

```text
status     | sender_host       | sender_port
-----------+-------------------+------------
streaming  | postgres-primary  | 5432
```

`streaming` indique que le secondaire reçoit actuellement le WAL depuis le primaire.

---

# 🌐 11. HAProxy

HAProxy fournit un **point d'entrée unique** pour l'application.

Sans HAProxy :

```text
Spring Boot
    |
    +--> postgres-primary
```

Si `postgres-primary` tombe, l'application doit connaître le nouveau primaire.

Avec HAProxy :

```text
Spring Boot
     |
     v
  HAProxy
     |
     +----> PRIMARY
     |
     +----> SECONDARY
```

L'application n'a donc qu'une seule adresse à utiliser.

---

# ❤️‍🩹 12. Health Check HAProxy + Patroni

HAProxy utilise l'API REST de Patroni pour déterminer quel serveur est actuellement primaire.

Configuration :

```cfg
global
    log stdout format raw local0

defaults
    log global
    mode tcp
    timeout connect 5s
    timeout client 30s
    timeout server 30s

frontend postgres
    bind *:5432
    default_backend postgres-primary

backend postgres-primary
    option httpchk GET /primary
    http-check expect status 200

    server postgres-primary postgres-primary:5432 check port 8008
    server postgres-replica postgres-replica:5432 check port 8008
```

Le fonctionnement est :

```text
HAProxy
   |
   | HTTP GET /primary
   |
   +----------------------+
   |                      |
   v                      v
primary:8008         replica:8008
   |                      |
  503                    200
   |                      |
 DOWN                     UP
                          |
                          v
                     PostgreSQL
```

HAProxy envoie ensuite les connexions PostgreSQL vers le serveur marqué `UP`.

---

# 🔌 13. Connexion à HAProxy

Le port PostgreSQL interne de HAProxy est :

```text
5432
```

Mais il est exposé sur le port `5435` de la machine :

```yaml
ports:
  - "5435:5432"
```

Donc :

```text
Host
localhost:5435
       |
       v
Docker
HAProxy:5432
       |
       v
PostgreSQL PRIMARY
```

Test :

```bash
psql -h localhost -p 5435 -U postgres -d sales
```

L'application Spring Boot peut utiliser :

```properties
spring.datasource.url=jdbc:postgresql://localhost:5435/sales
spring.datasource.username=postgres
spring.datasource.password=postgres
```

---

# 🧪 14. Scénario de test du Failover

## Étape 1 — Vérifier le primaire

```sql
SELECT pg_is_in_recovery();
```

Résultat attendu :

```text
f
```

---

## Étape 2 — Vérifier le secondaire

```sql
SELECT pg_is_in_recovery();
```

Résultat attendu :

```text
t
```

---

## Étape 3 — Vérifier le leader dans etcd

```bash
docker exec etcd etcdctl get /service/postgres-ha/leader
```

---

## Étape 4 — Arrêter le primaire

```bash
docker stop postgres-primary
```

---

## Étape 5 — Vérifier le nouveau primaire

Sur `postgres-replica` :

```sql
SELECT pg_is_in_recovery();
```

Résultat :

```text
f
```

Le replica est devenu PRIMARY.

---

## Étape 6 — Tester HAProxy

```bash
psql -h localhost -p 5435 -U postgres -d sales
```

La connexion doit toujours fonctionner.

L'application utilise toujours :

```text
localhost:5435
```

Elle n'a pas besoin de connaître le changement de serveur.

---

# 🔄 15. Architecture après Failover

Avant :

```text
             HAProxy
                |
        +-------+-------+
        |               |
        v               v
     PRIMARY         SECONDARY
   primary:5432     replica:5432
```

Après la panne :

```text
             HAProxy
                |
        +-------+-------+
        |               |
        X               |
        |               v
      DOWN            PRIMARY
                    replica:5432
```

HAProxy détecte automatiquement que `postgres-replica` est maintenant le primaire grâce à :

```text
GET /primary
```

---

# 🧠 16. Rôle des composants

### PostgreSQL

Gère les données et la réplication via WAL.

### WAL

Transporte les modifications du PostgreSQL primaire vers le secondaire.

### Patroni

Surveille PostgreSQL et gère :

- le leader
- la promotion
- le failover
- la configuration PostgreSQL
- la réplication

### etcd

Permet aux instances Patroni de partager l'état du cluster et de déterminer le leader.

### HAProxy

Fournit une adresse stable aux applications et dirige les connexions vers le primaire actuel.

### Docker

Permet d'exécuter chaque composant dans un environnement isolé.

---

# 📁 17. Structure du projet

Une structure possible :

```text
postgres-ha/
│
├── docker-compose.yml
├── Dockerfile
├── haproxy.cfg
│
├── patroni-primary.yml
├── patroni-replica.yml
│
└── README.md
```

---

# 🔐 18. Points importants

Ce projet est principalement un **laboratoire d'apprentissage**.

Pour un environnement de production, plusieurs éléments devraient être renforcés :

- utiliser plusieurs nœuds etcd pour éviter un point unique de panne
- utiliser des mots de passe sécurisés
- utiliser TLS
- sécuriser les endpoints Patroni
- mettre en place plusieurs replicas si nécessaire
- configurer correctement les sauvegardes
- surveiller PostgreSQL, Patroni, etcd et HAProxy
- définir une stratégie de restauration après sinistre

---

# 🎯 19. Objectifs pédagogiques

Ce projet permet de pratiquer concrètement :

- PostgreSQL Streaming Replication
- WAL
- PostgreSQL High Availability
- Patroni
- etcd
- Automatic Failover
- Leader Election
- HAProxy
- Docker
- Docker Compose
- Health Checks
- Spring Boot avec une base PostgreSQL HA

---

# 📚 20. Résumé du fonctionnement

Le fonctionnement global peut être résumé ainsi :

```text
                    Spring Boot
                         |
                         v
                    HAProxy:5435
                         |
                         v
                +----------------+
                |   Patroni HA   |
                +----------------+
                   /          \
                  /            \
                 v              v
        PostgreSQL PRIMARY   PostgreSQL SECONDARY
                 |              ^
                 |              |
                 +---- WAL -----+
                         |
                         v
                       etcd
                 Leader / Cluster State
```

En cas de panne du primaire :

```text
PRIMARY
   |
   X
   |
   v
Patroni détecte la panne
   |
   v
etcd permet l'élection
   |
   v
SECONDARY
   |
   v
PROMOTION
   |
   v
NEW PRIMARY
   |
   v
HAProxy détecte /primary
   |
   v
Spring Boot continue à utiliser
localhost:5435
```

---

## ✅ Résultat

Le projet met donc en place une architecture PostgreSQL avec :

```text
Docker
  +
PostgreSQL
  +
WAL Replication
  +
Patroni
  +
etcd
  +
HAProxy
  =
High Availability PostgreSQL
```

L'objectif principal est atteint : **le rôle PRIMARY peut changer automatiquement et l'application peut continuer à utiliser un point d'accès unique via HAProxy.**