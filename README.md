# Alta disponibilidad con PostgreSQL
## Santino Colmena

---

santinoc@santinoc:~/Documents$ cd docker-postgresql/

santinoc@santinoc:~/Documents/docker-postgresql$ docker compose up -d --build
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
santinoc@santinoc:~/Documents/docker-postgresql$ docker ps
```
CONTAINER ID   IMAGE                        COMMAND                  CREATED       STATUS          PORTS                                                                                      NAMES
f3510aa1c45b   haproxy:alpine               "docker-entrypoint.s…"   4 weeks ago   Up 11 seconds   0.0.0.0:5432->5432/tcp, [::]:5432->5432/tcp, 0.0.0.0:7000->7000/tcp, [::]:7000->7000/tcp   haproxy
92d2552d3a66   docker-postgresql-pg-2       "docker-entrypoint.s…"   4 weeks ago   Up 12 seconds   5432/tcp                                                                                   pg-2
b06a9889188b   docker-postgresql-pg-1       "docker-entrypoint.s…"   4 weeks ago   Up 12 seconds   5432/tcp                                                                                   pg-1
59876fab8463   quay.io/coreos/etcd:v3.5.9   "/usr/local/bin/etcd"    4 weeks ago   Up 12 seconds   2379-2380/tcp                                                                              etcd
```

santinoc@santinoc:~/Documents/docker-postgresql$ docker exec -it pg-1 patronictl -c patroni.yml list
```
+ Cluster: mi_cluster_ha (7615693839243325460) ----------+-----+------------+-----+
| Member | Host | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
+--------+------+---------+-----------+----+-------------+-----+------------+-----+
| pg-1   | pg-1 | Replica | streaming |  6 |   0/50002D0 |   0 |  0/50002D0 |   0 |
| pg-2   | pg-2 | Leader  | running   |  6 |             |     |            |     |
+--------+------+---------+-----------+----+-------------+-----+------------+-----+
```

<img width="1919" height="563" alt="imatge" src="https://github.com/user-attachments/assets/7da7eb6e-ce31-41d8-b557-082e6f027b14" />


