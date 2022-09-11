---
date: 2017-02-23
title: 'Awesome Docker'
slug: awesome-docker
toc: false
tags:
  - Docker
---

Docker is a disrupting technology for software delivery, infrastructure, and development. Just look at the adoption numbers (courtesy to datadog)

![docker_adoption](post/2017/docker-2017-1_v3.png)

<!--more-->

But let us step back for a second and answer the common question, What is Docker?

Docker started as an application build and deployment management system with the idea of having  homogeneous application containers. Similarly to the shipping container industry, loading and unloading cargo ships full of containers are made possible because of the **homogeneous** container sizes. Imagine what would happen if the containers on the ships were all in different sizes and shapes, loading, unloading, and stacking would be a nightmare. And exactly this problem has also existed in the software development and operations space for more than a decade. Usually, a developer would write code and a complementary document detailing how to install, configure and run his code. Operations would pick up a ticket and try to replicate these steps in a production environment with the obvious problems. Docker solves this problem by letting the developer package up his application with all of the dependencies it needs and ship it out as one package in the form of an image.


# What are docker images?

Docker images are indexed snapshots of the entire filesystem a container is meant to run. Every time the developer introduces a change in his code a new version of the image is automatically created and assigned a unique ID. 


# Why are docker images revolutionary?

- Every production deployment is re-buildable making rollbacks easy
- Is one application using Java 7 and the other Java 8? No problem with docker images
- Need 2 isolated instances of MySQL on the same host? Yeah no problem with docker images

Not let us look at a practical example to see docker and it benefits in action. We will go for a WordPress deployment with ELK monitoring. Think for a second from an operational standpoint what would be needed to realize this


1. Install PHP, Apache, MySQL
2. Create a database
3. Setup WordPress and configure it
4. Copy source files to Apache
5. Install ELK
6. Configure ELK
7. Configure Logstash to read Apache logs

I am probably forgetting something but the steps are quite many! Whit docker you could use docker-compose to bundle and configure every service and  just run it. Here an example docker-compose.yml file for spinning up a WordPress site with ELK:


```yaml
version: '2'

services:
  elasticsearch:
    image: elasticsearch:5.3.1
    restart: always
    ports:
      - "9200:9200"
      - "9300:9300"
    volumes:
      - data:/usr/share/elasticsearch/data<
  kibana:
    image: kibana:5.3.1
    restart: always
    links:
      - elasticsearch
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch
      - logstash
  logstash:
    image: logstash:5.3.1
    restart: always
    command: logstash -e 'input { stdin {  } udp { port => 5050 type => syslog } tcp { port => 5050 type => syslog } } output { elasticsearch { hosts => [ 'elasticsearch' ] index => "logstash-%{+YYYY.MM.dd}" } stdout { } }'
    ports:
      - "5050:5050/udp"
      - "5050:5050/tcp"
  db:
    image: mysql:5.7
    volumes:
      - db_data:/var/lib/mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: evil_password
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: sentencepress

  wordpress:
    depends_on:
      - db
    image: wordpress:latest
    ports:
      - "80:80"
    restart: always
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: sentencepress
      SYSLOGSERVER: "udp://logstash:5050"
    logging:
      driver: syslog
      options:
        syslog-address: "udp://logstash:5050"
       
volumes:
  data:
  db_data:
```

save this to a docker-compose.yml file in a directory and inside the directory run

```bash
docker-compose up
```

You can view your WordPress site by opening your browser and navigate to http://localhost

Kibana is running on port 5601, http://localhost:5601

Now I ask you, would you prefer a document with 300-600 words and pictures on how to install and configure our solution or would you rather receive a docker-compose.yml file and run one line in your command prompt? You decide!

Need professional help with setting up an ELK cluster with redundancy? Send me an email and I'll gladly help out!

