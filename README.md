# A Step by Step Guide to Logging with ElasticSearch, Kibana, ASP.NET Core 3.1 and Docker
### Published March 28, 2020
### Author: TONMOY RUDRA

## Docker 
### Download & Installtion
* Download Docker Desktop Version From [Here](https://docs.docker.com/docker-for-windows/install/) & install it as like as normal software installation process.
* Run it to double click on `Docker Desktop`.

### Create a docker-compose file
* Create a folder name `docker` on your project directory
* Then, Create a `docker-composer.yml` file on `docker` folder and Copy and Paste the below code.
```
version: '3.1'

services:

  elasticsearch:
   container_name: elasticsearch
   image: docker.elastic.co/elasticsearch/elasticsearch:7.9.2
   ports:
    - 9200:9200
   volumes:
    - elasticsearch-data:/usr/share/elasticsearch/data
   environment:
    - xpack.monitoring.enabled=true
    - xpack.watcher.enabled=false
    - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    - discovery.type=single-node
   networks:
    - elastic

  kibana:
   container_name: kibana
   image: docker.elastic.co/kibana/kibana:7.9.2
   ports:
    - 5601:5601
   depends_on:
    - elasticsearch
   environment:
    - ELASTICSEARCH_URL=http://localhost:9200
   networks:
    - elastic
  
networks:
  elastic:
    driver: bridge

volumes:
  elasticsearch-data:
```

