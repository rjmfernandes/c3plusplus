# Test C3++ EA over local ansible based on docker containers

This project is based on https://github.com/rjmfernandes/local-ansible-cp 

- [Test C3++ EA over local ansible based on docker containers](#test-c3-ea-over-local-ansible-based-on-docker-containers)
  - [Disclaimer](#disclaimer)
  - [Build the base image](#build-the-base-image)
  - [Install C3++](#install-c3)
  - [Cleanup](#cleanup)

## Disclaimer

The code and/or instructions here available are **NOT** intended for production usage. 
It's only meant to serve as an example or reference and does not replace the need to follow actual and official documentation of referenced products.

## Build the base image

The image used for our instances comes from the work by Jeff Geerling https://github.com/geerlingguy/docker-ubuntu2204-ansible

We have added a couple of packages including java and changed the ubuntu version for compatibility with CP Ansible.

First you will need to create the image:

```bash
docker build . -t my-geerlingguy-docker-ubuntu-ansible
```

Clone locally the repository:

```shell
git clone --depth 1 --branch v7.7.1 https://github.com/confluentinc/cp-ansible
```

Inside the repository you need to copy the playbooks to root:

```bash
cd cp-ansible
cp -fr playbooks/* .
```

Copy hosts.yml to cp-ansible.

```bash
cp ../hosts.yml .
```

Edit the variables of hosts.yml as the example here. Pay attention to the following variables:

```yml
    ansible_connection: docker
    ansible_user: root
    ansible_become: true
    ssl_enabled: false
    confluent.platform.ssl_required: false
    ansible_python_interpreter: /usr/bin/python3
    custom_java_path: /usr/lib/jvm/java-1.17.0-openjdk-arm64
```

You will also want to make sure the server instances in hosts.yml match the ones defined in the docker-compose.yml file (just like the example here). Comment out either kafka_controller entry or the zookeeper one.

Finally run the docker-compose from the root of the project:

```bash
cd ..
docker compose up -d
```

You will need to map the host names on your `/etc/hosts` file:

```
127.0.0.1 zk1
127.0.0.1 zk2
127.0.0.1 zk3
127.0.0.1 kafka1
127.0.0.1 kafka2
127.0.0.1 kafka3
127.0.0.1 sr
127.0.0.1 cc
127.0.0.1 c3pp
```

Finally back to the cp-ansible cloned repository run:

```bash
cd cp-ansible
ansible-galaxy collection install git+https://github.com/confluentinc/cp-ansible.git,v7.7.1
ansible-playbook ./all.yml -i hosts.yml
```

You should be able to list the topics:

```shell
kafka-topics --bootstrap-server localhost:9092 --list
```

And also access Control Center at: http://localhost:9022/clusters

## Install C3++

```shell
docker compose exec -it c3pp bash
```

Once in the c3++ container shell execute:

```shell
sudo apt-get install wget
sudo apt-get install gpg-agent
sudo apt-get install vim
```

And after:

```shell
export BASE_URL=http://confluent-platform-hotfixes-891377121322-us-west-2.s3-website-us-west-2.amazonaws.com/7.7.1-cp1/deb/7.7/
sudo apt-get update
wget ${BASE_URL}archive.key
sudo apt-key add archive.key
sudo add-apt-repository -y "deb ${BASE_URL} stable main"
sudo apt-get update 
sudo apt install -y confluent-platform confluent-security
```

If in parallel we connect in another shell to cc:

```shell
docker compose exec cc cat /etc/confluent-control-center/control-center-production.properties
```

We should get:

```
# Written by Ansible
bootstrap.servers=kafka1:9092,kafka2:9092,kafka3:9092
confluent.controlcenter.command.topic.replication=3
confluent.controlcenter.data.dir=/var/lib/confluent/control-center
confluent.controlcenter.internal.topics.replication=3
confluent.controlcenter.rest.listeners=http://0.0.0.0:9021
confluent.controlcenter.schema.registry.url=http://sr:8081
confluent.controlcenter.streams.cprest.url=http://kafka1:8090,http://kafka2:8090,http://kafka3:8090
confluent.controlcenter.streams.num.stream.threads=12
confluent.controlcenter.streams.security.protocol=PLAINTEXT
confluent.metrics.topic.replication=3
confluent.monitoring.interceptor.security.protocol=PLAINTEXT
confluent.monitoring.interceptor.topic.replication=3
```

Lets add to this the following under same file `/etc/confluent-control-center/control-center-production.properties` on `c3pp`:

```shell
confluent.controlcenter.id=10
confluent.controlcenter.prometheus.enable=true
confluent.controlcenter.prometheus.url=http://localhost:9090
confluent.controlcenter.prometheus.rules.file=/etc/confluent-control-center/trigger_rules-generated.yml
confluent.controlcenter.alertmanager.config.file=/etc/confluent-control-center/alertmanager-generated.yml
```

Change ownership of configuration files:

```shell
chown -c cp-control-center /etc/confluent-control-center/trigger_rules-generated.yml
chown -c cp-control-center /etc/confluent-control-center/alertmanager-generated.yml
```

Start Next Gen Control Center services:

```shell
systemctl enable prometheus
systemctl start prometheus

systemctl enable alertmanager
systemctl start alertmanager

systemctl enable confluent-control-center
systemctl start confluent-control-center
```

Now first lets install `curl` on each broker (and also vim for easy editing after of configuration files):

```shell
docker compose exec kafka1 sudo apt-get install -y curl
docker compose exec kafka2 sudo apt-get install -y curl
docker compose exec kafka3 sudo apt-get install -y curl
docker compose exec kafka1 sudo apt-get install -y vim
docker compose exec kafka2 sudo apt-get install -y vim
docker compose exec kafka3 sudo apt-get install -y vim
```

Login to each broker and confirm they can access the Next Gen Control Center:

```shell
docker compose exec kafka1 curl http://c3pp:9090/-/healthy
docker compose exec kafka2 curl http://c3pp:9090/-/healthy
docker compose exec kafka3 curl http://c3pp:9090/-/healthy
```

After on each Kafka broker update `/etc/kafka/server.properties`:

```shell
metric.reporters=io.confluent.telemetry.reporter.TelemetryReporter,io.confluent.metrics.reporter.ConfluentMetricsReporter 
confluent.telemetry.exporter._c3plusplus.type=http
confluent.telemetry.exporter._c3plusplus.enabled=true
confluent.telemetry.exporter._c3plusplus.metrics.include=io.confluent.kafka.server.server.broker.state|io.confluent.kafka.server.replica.manager.leader.count|io.confluent.kafka.server.request.queue.size|io.confluent.kafka.server.broker.topic.failed.produce.requests.rate.1.min|io.confluent.kafka.server.tier.archiver.total.lag|io.confluent.kafka.server.request.total.time.ms.p99|io.confluent.kafka.server.broker.topic.failed.fetch.requests.rate.1.min|io.confluent.kafka.server.log.total.size|io.confluent.kafka.server.broker.topic.total.fetch.requests.rate.1.min|io.confluent.kafka.server.partition.caught.up.replicas.count|io.confluent.kafka.server.partition.observer.replicas.count|io.confluent.kafka.server.tier.tasks.num.partitions.in.error|io.confluent.kafka.server.broker.topic.bytes.out.rate.1.min|io.confluent.kafka.server.request.total.time.ms.p95|io.confluent.kafka.server.controller.active.controller.count|io.confluent.kafka.server.session.expire.listener.zookeeper.disconnects.total|io.confluent.kafka.server.request.total.time.ms.p999|io.confluent.kafka.server.controller.active.broker.count|io.confluent.kafka.server.request.handler.pool.request.handler.avg.idle.percent.rate.1.min|io.confluent.kafka.server.session.expire.listener.zookeeper.disconnects.rate.1.min|io.confluent.kafka.server.controller.unclean.leader.elections.rate.1.min|io.confluent.kafka.server.replica.manager.partition.count|io.confluent.kafka.server.controller.unclean.leader.elections.total|io.confluent.kafka.server.partition.replicas.count|io.confluent.kafka.server.broker.topic.total.produce.requests.rate.1.min|io.confluent.kafka.server.controller.offline.partitions.count|io.confluent.kafka.server.socket.server.network.processor.avg.idle.percent|io.confluent.kafka.server.partition.under.replicated|io.confluent.kafka.server.log.log.start.offset|io.confluent.kafka.server.log.tier.size|io.confluent.kafka.server.log.size|io.confluent.kafka.server.tier.fetcher.bytes.fetched.total|io.confluent.kafka.server.request.total.time.ms.p50|io.confluent.kafka.server.tenant.consumer.lag.offsets|io.confluent.kafka.server.session.expire.listener.zookeeper.expires.rate.1.min|io.confluent.kafka.server.log.log.end.offset|io.confluent.kafka.server.log.num.log.segments|io.confluent.kafka.server.broker.topic.bytes.in.rate.1.min|io.confluent.kafka.server.partition.under.min.isr|io.confluent.kafka.server.partition.in.sync.replicas.count|io.confluent.telemetry.http.exporter.batches.dropped|io.confluent.telemetry.http.exporter.items.total|io.confluent.telemetry.http.exporter.items.succeeded|io.confluent.telemetry.http.exporter.send.time.total.millis
confluent.telemetry.exporter._c3plusplus.client.base.url=http://c3pp:9090/api/v1/otlp
confluent.telemetry.exporter._c3plusplus.client.compression=gzip
confluent.telemetry.exporter._c3plusplus.api.key=dummy
confluent.telemetry.exporter._c3plusplus.api.secret=dummy
confluent.telemetry.exporter._c3plusplus.buffer.pending.batches.max=80 
confluent.telemetry.exporter._c3plusplus.buffer.batch.items.max=4000 
confluent.telemetry.exporter._c3plusplus.buffer.inflight.submissions.max=10 
confluent.telemetry.metrics.collector.interval.ms=60000 
confluent.telemetry.remoteconfig._confluent.enabled=false
confluent.consumer.lag.emitter.enabled=true
```

Typically comment the original line configuring `metric.reporters` and add the configuration listed before to the end of the file.

Perform a rolling restart of the brokers (example bellow for the kafka1 broker):

```shell
docker compose exec kafka1 systemctl restart confluent-server
```

You can check with:

```shell
docker compose exec kafka1 tail -f /var/log/kafka/server.log
```

**(To be reviewed as per ansible documentation.)**

## Cleanup

```bash
cd ..
docker compose down -v
rm -fr cp-ansible
```