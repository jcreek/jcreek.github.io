---
tags:
  - deploying
  - docker
---

# Docker Compose for Elasticsearch and Kibana 7.9

_2021-07-13_

In this post, I will cover how you can set up Elasticsearch and Kibana with a single `docker-compose.yml` file.

## Set up and install ELK stack on Ubuntu server

You must run `sudo sysctl -w vm.max_map_count=262144` to get Elasticsearch to work. To make this permanent, run `sudo nano /etc/sysctl.conf` and add `vm.max_map_count=262144` to the end of the file on a new line, then save and exit.

Go to the directory with the files in. Make sure you've used SFTP to put the `docker-compose.yml` file in there first. For example you may need to use this command `cd /home/administrator/docker/elk-stack`

Install it all with `docker-compose up -d` and you're finished! It really is that simple.

The compose file is as below.

```docker
version: '2.2'

services:

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.9.3
    container_name: elasticsearch
    environment:
      - node.name=elasticsearch
      - discovery.seed_hosts=elasticsearch
      - cluster.initial_master_nodes=elasticsearch
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms2g -Xmx2g"
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - esdata1:/usr/share/elasticsearch/data
    ports:
      - 9200:9200

  kibana:
    image: docker.elastic.co/kibana/kibana:7.9.3
    container_name: kibana
    environment:
      ELASTICSEARCH_URL: "http://elasticsearch:9200"
      ELASTICSEARCH_SHARDTIMEOUT: 300000
    ports:
      - 5601:5601
    depends_on:
      - elasticsearch

volumes:
  esdata1:
    driver: local
```
