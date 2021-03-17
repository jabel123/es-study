# Docker로 ELK 설치하기

```
$ git clone https://github.com/deviantony/docker-elk.git
```

## elastic yml 설정
```
$ vi elasticsearch/config/elasticsearch.yml
```
```
---
## Default Elasticsearch configuration from Elasticsearch base image.
## https://github.com/elastic/elasticsearch/blob/master/distribution/docker/src/docker/config/el    asticsearch.yml
#
cluster.name: "docker-cluster"
network.host: 0.0.0.0
```

## kibana yml 설정
```
$ vi kibana/config/kibana.yml
```
```
ARG ELK_VERSION

# https://www.docker.elastic.co/
FROM docker.elastic.co/elasticsearch/elasticsearch:${ELK_VERSION}

# Add your elasticsearch plugins setup here
# Example: RUN elasticsearch-plugin install analysis-icu
RUN elasticsearch-plugin install analysis-nori
```

## logstash yml 설정
```
vi logstash/config/logstash.yml
```
```
---
## Default Kibana configuration from kibana-docker.
## https://github.com/elastic/kibana-docker/blob/master/.tedi/template/kibana.yml.j2
#
server.name: kibana
server.host: "0"
elasticsearch.hosts: [ "http://elasticsearch:9200" ]
xpack.monitoring.ui.container.elasticsearch.enabled: true
```