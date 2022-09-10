---
date: 2018-01-20
dateName: '20 January 2018'
title: 'Docker for developers'
slug: docker-for-developers
tags:
  - Docker
---

With the rise of microservices, the complexity of dev environments has increased. Technology stacks can be used independently and thus the choices of programming languages have increased. Companies used to be proficient in one language and implement their solutions using the tools/libraries available to them in the specific language in a monolithic way. Today with containers it's totally different. If you are a Ruby on Rails shop but need to implement a real-time service you might dip into NodeJS. Or you are a PHP shop and need to implement a high traffic service you might look into Golang. Or you have been using python2 but want the new service to use asyncio (python3). Now imagine the time it will take onboarding developers and setting up their dev environments. That can be quite time-consuming for the developer and for you! So what is the solution? Docker of course :)

Usually, docker is utilized for CI (building / testing / QA..etc) but I think that every project should begin with a Dockerfile and a docker-compose file. That way when somebody new comes and joins the team he would just need to do a docker-compose up to start his work. NodeJS 6 or 8? No problem just define it in the Dockerfile and the developer doesn't have to bother with installing anything.

For optimum results, the developer has to know some useful docker commands. In this post, I intend to go over some of the good to know docker commands that I have found myself using on a regular basis while using docker as my development environment.


![docker_for_devs](dockerwareenv.png)

====

# Spawning a shell inside a container

So you have your containers up and running with **docker-compose up** but need to drop into the postgreSQL container to issue some **psql** commands. If your postgreSQL container name is **db** you can accomplish this with

```bash
docker-compose exec db /bin/bash
```

this works only if the container **db** is running. To view running containers, do

```
docker container ls
```

if you have an image you pulled from the docker hub and want to take a look at the filesystem of this image, you can

```
docker run --rm -i -t --name db postgres:10 /bin/bash
```

- --rm remove the container after exit
- -i keep STDIN open
- -t allocate a pseudo-tty
- --name name the container

When you run the above command in your shell you will notice that after pulling the image docker starts to execute a bunch of different commands and only in the end of it all you'll get a bash shell. The reason for this behavior is because the image (postgres:10) was built with an entrypoint script. The entrypoint script allows the creators of the postgres image to configure a container to run as an executable. But what if you don't want this behavior, what if you just want to look at the files inside the image and the configuration of postgres? Don't worry you can overwrite this behavior:


```
docker run --rm -i -t --name db --entrypoint /bin/bash postgres:10
```

the **--entrypoint**  argument in the docker run command overwrites the entrypoint script used in the official image. 


### Copying a file from a docker container

There might be some files in your running docker container that you might want to keep. For example, you might decide to keep the **postgresql.conf** file on your laptop for later reference maybe? Executing a shell inside the postgres container everytime you want to look/change the postgresql.conf isn't fun so why not just copy it to your laptop?

```
docker cp db:/var/lib/postgresql/data/postgresql.conf .
```

This command will copy the config file from the running container to your current directory. Neat, right? :)


### Debugging docker builds

In a previous [post](https://donchev.is/post/building-and-shipping-docker-images) of mine. I mentioned that when I build docker images I sometimes like to start from the shell and issue the build commands manually and afterward save the running container to an image via [docker commit](https://docs.docker.com/engine/reference/commandline/commit/). What I have been doing recently is slightly different but still the same principle. I start with a Dockerfile that I build and if the build fails I will drop into the container to view the filesystem and correct the error and commit the container to an image. An example might be suitable here, imagine the following Dockerfile:

```
FROM python:alpine3.7

RUN pip install psycopg2
CMD ["python"]
```

building this Dockerfile will fail

```
Sending build context to Docker daemon  2.048kB
Step 1/3 : FROM python:alpine3.7
 ---> 4b00a94b6f26
Step 2/3 : RUN pip install psycopg2
 ---> Running in 8a2acbca6301
Collecting psycopg2
  Downloading psycopg2-2.7.3.2.tar.gz (425kB)
    Complete output from command python setup.py egg_info:
    running egg_info
    creating pip-egg-info/psycopg2.egg-info
    writing pip-egg-info/psycopg2.egg-info/PKG-INFO
    writing dependency_links to pip-egg-info/psycopg2.egg-info/dependency_links.txt
    writing top-level names to pip-egg-info/psycopg2.egg-info/top_level.txt
    writing manifest file 'pip-egg-info/psycopg2.egg-info/SOURCES.txt'
    Error: pg_config executable not found.
    
    Please add the directory containing pg_config to the PATH
    or specify the full executable path with the option:
    
        python setup.py build_ext --pg-config /path/to/pg_config build ...
    
    or with the pg_config option in 'setup.cfg'.
    
    ----------------------------------------
Command "python setup.py egg_info" failed with error code 1 in /tmp/pip-build-n3mrzydh/psycopg2/
The command '/bin/sh -c pip install psycopg2' returned a non-zero code: 1
```

The error is because pg_config wasn't found. We can have a look at the filesystem of this build with:

```
docker run --rm -i -t 4b00a94b6f26 /bin/sh
```

4b00a94b6f26 is the filesystem layer just before the error occured. By opening a shell in this layer we can examine the filesystem and look for clues if pg_config is installed:

```
~$ find / -name pg_config
~$ 
~$ apk add postgresql-dev 
(1/9) Installing libressl2.6-libtls (2.6.3-r0)
(2/9) Installing pkgconf (1.3.10-r0)
(3/9) Installing libressl-dev (2.6.3-r0)
(4/9) Installing db (5.3.28-r0)
(5/9) Installing libsasl (2.1.26-r11)
(6/9) Installing libldap (2.4.45-r3)
(7/9) Installing libpq (10.1-r1)
(8/9) Installing postgresql-libs (10.1-r1)
(9/9) Installing postgresql-dev (10.1-r1)
Executing busybox-1.27.2-r7.trigger OK: 56 MiB in 44 packages 
~$ find / -name pg_config
/usr/bin/pg_config
```


### Add dependencies before copying your code

Docker uses caches when building / rebuilding images. You should utilize the caching mechanism as much as possible. Instead of doing:

```
FROM python:alpine3.7
COPY . .
RUN pip install -r requirements.txt
```

DO THIS:

```
FROM python:alpine3.7
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
```

The latter will create 3 layers. If your dependencies don't change between builds then the first two layers will be executed from cache! Although I used python and pip as an example remember that this is also valid for npm, maven builds, ruby gems..etc..

This post is already quite long. In a future post, I will discuss two more topics, namely gotchas and caveats around multi-stage builds and permission problems around host mounted volumes. 


