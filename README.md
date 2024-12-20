# Test C3++ EA over local ansible based on docker containers

This project is based on https://github.com/rjmfernandes/local-ansible-cp 

- [Test C3++ EA over local ansible based on docker containers](#test-c3-ea-over-local-ansible-based-on-docker-containers)
  - [Disclaimer](#disclaimer)
  - [Build the base image](#build-the-base-image)
  - [Build the C3++ image](#build-the-c3-image)
  - [Ansible start](#ansible-start)
  - [Test C3++](#test-c3)
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

## Build the C3++ image

```bash
cd docker
docker build . -t c3pp
cd ..
```

## Ansible start

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

In case you have an error regarding the custom java path edit `hosts.yml` and change it probably to something like the following and try again:

```
custom_java_path: /usr/lib/jvm/java-17-openjdk-amd64
```

You should be able to list the topics:

```shell
kafka-topics --bootstrap-server localhost:9092 --list
```

And also access Control Center at: http://localhost:9022/clusters

## Test C3++

Now first lets install `curl` on each broker:

```shell
docker compose exec kafka1 sudo apt-get install -y curl
docker compose exec kafka2 sudo apt-get install -y curl
docker compose exec kafka3 sudo apt-get install -y curl
```

Confirm they can access the Next Gen Control Center:

```shell
docker compose exec kafka1 curl http://c3pp:9090/-/healthy
docker compose exec kafka2 curl http://c3pp:9090/-/healthy
docker compose exec kafka3 curl http://c3pp:9090/-/healthy
```

And access C3++ over http://localhost:9021/

## Cleanup

```bash
cd ..
docker compose down -v
rm -fr cp-ansible
```