# Alta disponibilidad con PostgreSQL
## Santino Colmena

---

## 1. ¿Qué se pretende hacer con estos ficheros?

Con los ficheros proporcionados se pretende construir un entorno de alta disponibilidad donde:

- hay dos nodos PostgreSQL
- uno actúa como **Leader**
- el otro actúa como **Replica**
- Patroni se encarga de la gestión del clúster y del failover
- etcd guarda el estado del clúster
- HAProxy redirige las conexiones al nodo que en ese momento es el primario

---

## 2. Tecnologías utilizadas y para qué sirve cada una

### PostgreSQL
Es el sistema gestor de bases de datos que se quiere ejecutar en alta disponibilidad.

### Patroni
Es la herramienta que administra el clúster PostgreSQL. Se encarga de:
- elegir el nodo líder
- gestionar la replicación
- realizar el failover automático si cae el nodo principal

### etcd
Es un almacén clave-valor distribuido. Patroni lo usa para guardar el estado del clúster y coordinar qué nodo debe ser el líder.

### HAProxy
Es un proxy/balanceador. En esta práctica ofrece:
- acceso al servicio PostgreSQL desde un único puerto (`5432`)
- una interfaz web de estadísticas en el puerto `7000`

### Docker
Permite ejecutar cada componente dentro de un contenedor.

### Docker Compose
Permite levantar todos los contenedores del entorno con un solo comando.

---

## 3. Archivos del proyecto

### docker-compose.yml

```yml
networks:
  ha-net:
    driver: bridge

services:
  # Etcd
  etcd:
    image: quay.io/coreos/etcd:v3.5.9
    container_name: etcd
    networks:
      - ha-net
    environment:
      - ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379
      - ETCD_ADVERTISE_CLIENT_URLS=http://etcd:2379
      - ETCD_ENABLE_V2=true

  # Node 1
  pg-1:
    build: .
    container_name: pg-1
    networks:
      - ha-net
    volumes:
      - ./patroni.yml:/patroni.yml:ro
      - pg_1_data:/var/lib/postgresql/data
    environment:
      - PATRONI_NAME=pg-1
      - PATRONI_RESTAPI_CONNECT_ADDRESS=pg-1:8008
      - PATRONI_POSTGRESQL_CONNECT_ADDRESS=pg-1:5432
    depends_on:
      - etcd

  # Node 2
  pg-2:
    build: .
    container_name: pg-2
    networks:
      - ha-net
    volumes:
      - ./patroni.yml:/patroni.yml:ro
      - pg_2_data:/var/lib/postgresql/data
    environment:
      - PATRONI_NAME=pg-2
      - PATRONI_RESTAPI_CONNECT_ADDRESS=pg-2:8008
      - PATRONI_POSTGRESQL_CONNECT_ADDRESS=pg-2:5432
    depends_on:
      - etcd

  # HAproxy
  haproxy:
    image: haproxy:alpine
    container_name: haproxy
    ports:
      - "5432:5432"
      - "7000:7000"
    networks:
      - ha-net
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
    depends_on:
      - pg-1
      - pg-2

volumes:
  pg_1_data:
  pg_2_data:
```

### Dockerfile

```
FROM postgres:15-bullseye

USER root

RUN apt-get update && \
    apt-get install -y patroni python3-etcd curl && \
    rm -rf /var/lib/apt/lists/*

USER postgres

CMD ["patroni", "/patroni.yml"]
```

### haproxy.cfg

```cfg
global
    maxconn 100

defaults
    log global
    mode tcp
    retries 2
    timeout client 30m
    timeout connect 4s
    timeout server 30m
    timeout check 5s

listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /

listen postgres
    bind *:5432
    option httpchk GET /primary
    http-check expect status 200

    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions

    server pg-1 pg-1:5432 maxconn 100 check port 8008
    server pg-2 pg-2:5432 maxconn 100 check port 8008
```

### patroni.yml

```yml
scope: mi_cluster_ha
namespace: /db/
restapi:
  listen: 0.0.0.0:8008
etcd:
  host: etcd:2379
bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
  initdb:
    - auth-host: scram-sha-256
    - auth-local: trust
    - encoding: UTF8
  users:
    admin:
      password: ScolmenaAdmin
      options:
        - createrole
        - createdb
postgresql:
  listen: 0.0.0.0:5432
  data_dir: /var/lib/postgresql/data/patroni
  pg_hba:
    - host replication replicator all scram-sha-256
    - host all all all scram-sha-256
  authentication:
    replication:
      username: replicator
      password: "ScolmenaRepl!"
    superuser:
      username: postgres
      password: "ScolmenaPg!"
```

## 4. Despliegue del entorno

### Instalar Docker y herramientas necesarias

```bash
sudo apt update
sudo apt install docker.io docker-compose-v2 postgresql-client -y
```

### Arrancar el clúster

```bash
santinoc@santinoc:~/Documents$ cd docker-postgresql/
santinoc@santinoc:~/Documents/docker-postgresql$ docker compose up -d --build
```

```
[+] Building 1.8s (10/10) FINISHED                                              
 => [internal] load local bake definitions                                 0.0s
 => => reading from stdin 1.01kB                                           0.0s
 => [pg-2 internal] load build definition from Dockerfile                  0.0s
 => => transferring dockerfile: 234B                                       0.0s
 => [pg-2 internal] load metadata for docker.io/library/postgres:15-bulls  1.0s
 => [pg-2 internal] load .dockerignore                                     0.0s
 => => transferring context: 2B                                            0.0s
 => [pg-2 1/2] FROM docker.io/library/postgres:15-bullseye@sha256:4ce70bc  0.0s
 => CACHED [pg-2 2/2] RUN apt-get update &&     apt-get install -y patron  0.0s
 => [pg-1] exporting to image                                              0.0s
 => => exporting layers                                                    0.0s
 => => writing image sha256:687a4e4ee735c4a61b6c5dc822a678663cdc163943998  0.0s
 => => naming to docker.io/library/docker-postgresql-pg-1                  0.0s
 => [pg-2] exporting to image                                              0.0s
 => => exporting layers                                                    0.0s
 => => writing image sha256:c87993bdb923fc0a8c6e953dae1def0dfb3f3759334f7  0.0s
 => => naming to docker.io/library/docker-postgresql-pg-2                  0.0s
 => [pg-1] resolving provenance for metadata file                          0.0s
 => [pg-2] resolving provenance for metadata file                          0.0s
[+] up 6/6
 ✔ Image docker-postgresql-pg-1 Built                                       2.1s
 ✔ Image docker-postgresql-pg-2 Built                                       2.1s
 ✔ Container etcd               Started                                     0.4s
 ✔ Container pg-1               Started                                     0.6s
 ✔ Container pg-2               Started                                     0.5s
 ✔ Container haproxy            Started                                     0.4s
```

### Comprobar que los contenedores están levantados

santinoc@santinoc:~/Documents/docker-postgresql$ docker ps
```
CONTAINER ID   IMAGE                        COMMAND                  CREATED       STATUS          PORTS                                                                                      NAMES
f3510aa1c45b   haproxy:alpine               "docker-entrypoint.s…"   4 weeks ago   Up 11 seconds   0.0.0.0:5432->5432/tcp, [::]:5432->5432/tcp, 0.0.0.0:7000->7000/tcp, [::]:7000->7000/tcp   haproxy
92d2552d3a66   docker-postgresql-pg-2       "docker-entrypoint.s…"   4 weeks ago   Up 12 seconds   5432/tcp                                                                                   pg-2
b06a9889188b   docker-postgresql-pg-1       "docker-entrypoint.s…"   4 weeks ago   Up 12 seconds   5432/tcp                                                                                   pg-1
59876fab8463   quay.io/coreos/etcd:v3.5.9   "/usr/local/bin/etcd"    4 weeks ago   Up 12 seconds   2379-2380/tcp                                                                              etcd
```
## 5. Comprobación del estado del clúster

Para ver el estado del clúster con Patroni:

santinoc@santinoc:~/Documents/docker-postgresql$ docker exec -it pg-1 patronictl -c patroni.yml list
```
+ Cluster: mi_cluster_ha (7615693839243325460) ----------+-----+------------+-----+
| Member | Host | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
+--------+------+---------+-----------+----+-------------+-----+------------+-----+
| pg-1   | pg-1 | Replica | streaming |  6 |   0/50002D0 |   0 |  0/50002D0 |   0 |
| pg-2   | pg-2 | Leader  | running   |  6 |             |     |            |     |
+--------+------+---------+-----------+----+-------------+-----+------------+-----+
```

## 6. Comprobación de HAProxy

HAProxy publica estadísticas en el puerto `7000`.

Se puede comprobar abriendo en el navegador:

```text
http://localhost:7000
```

<img width="1919" height="563" alt="imatge" src="https://github.com/user-attachments/assets/7da7eb6e-ce31-41d8-b557-082e6f027b14" />


