FROM my-geerlingguy-docker-ubuntu-ansible
LABEL maintainer="Rui Fernandes"

RUN apt-get update \
    && apt-get install -y wget \
    && apt-get install -y gpg-agent 

RUN export BASE_URL=http://confluent-platform-hotfixes-891377121322-us-west-2.s3-website-us-west-2.amazonaws.com/7.7.1-cp1/deb/7.7/ \
    && sudo apt-get update \
    && wget ${BASE_URL}archive.key \
    && sudo apt-key add archive.key \
    && sudo add-apt-repository -y "deb ${BASE_URL} stable main" 

RUN sudo apt-get update
RUN sudo apt install -y confluent-cli 
RUN sudo apt install -y confluent-common
RUN sudo apt install -y confluent-rest-utils
RUN sudo apt install -y confluent-ce-kafka-http-server 
RUN sudo apt install -y confluent-telemetry
RUN sudo apt install -y confluent-control-center-fe 
RUN sudo apt install -y confluent-control-center
RUN sudo apt install -y confluent-hub-client
RUN sudo apt install -y confluent-kafka-connect-replicator 
RUN sudo apt install -y confluent-kafka-mqtt 
RUN sudo apt install -y confluent-kafka-rest
RUN sudo apt install -y confluent-ksqldb 
RUN sudo apt install -y confluent-metadata-service 
RUN sudo apt install -y confluent-rebalancer
RUN sudo apt install -y confluent-schema-registry
RUN sudo apt install -y confluent-schema-registry-plugins
RUN sudo apt install -y confluent-server-rest
RUN sudo apt install -y confluent-server 

RUN sudo apt-get update \
    && sudo apt install -y confluent-platform

RUN sudo apt-get update \
    && sudo apt install -y confluent-security

COPY ./control-center-production.properties /etc/confluent-control-center/control-center-production.properties

RUN chown -c cp-control-center /etc/confluent-control-center/trigger_rules-generated.yml \
    && chown -c cp-control-center /etc/confluent-control-center/alertmanager-generated.yml

RUN systemctl enable prometheus \
    && systemctl enable alertmanager \
    && systemctl enable confluent-control-center

ENTRYPOINT ["/lib/systemd/systemd"]
