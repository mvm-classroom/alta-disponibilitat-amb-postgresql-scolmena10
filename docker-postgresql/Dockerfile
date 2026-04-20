FROM postgres:15-bullseye

USER root

RUN apt-get update && \
    apt-get install -y patroni python3-etcd curl && \
    rm -rf /var/lib/apt/lists/*

USER postgres

CMD ["patroni", "/patroni.yml"]