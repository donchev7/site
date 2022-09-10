---
date: 2018-02-02
dateName: '02 February 2018'
title: 'Docker for developers: part 2'
slug: docker-for-developers-part2
tags:
  - Docker
---

In a previous [post](https://donchev.is/post/docker-for-developers), I wrote about some useful commands developers should know when using docker as a development environment. In this post, I'll add two more topics, namely multi-stage builds and permission problems with host mounted volumes. Let's start with docker multi-stage builds.

### Docker multi-stage builds

In docker version 17.05 multi-stage builds were introduced which simplified life for us who used docker in production and testing. Back when applications, where deployed to production servers over SCP or similar, production servers were kept lean. Only packages related to the apps where added to the OS and nothing else. Keeping the production OS lean was a way of reducing the attacker surface and thus smaller probabilities of vulnerabilities.

Similar to that logic when you deploy containers to production you want the images to be as lean as possible. Build-essentials in the container? No way.

However, some packages require that you have compilers installed when doing package installs. The way to solve this puzzle prior to docker multi-stage builds was to have two separate Dockerfiles. One for building the binaries and one for running the actual application. With docker multi-stage builds you can now combine the two dockerfiles in one and you will end up with a lean image for production. An example might be useful to show this

====

Clone my [GitHub repo](https://github.com/donchev7/docker-multistage) to participate or just look at the code in the repo and read along to better understand the concept.

The repo consists of a fairly simple JAVA app and one test. I'll use Maven for building the app, for dependencies take a look at the pom.xml file.

Now, let us take a look at the Dockerfile in the repo:

```
FROM maven:3.5-jdk-8 AS builder

WORKDIR /opt/hello_world

COPY pom.xml .
RUN mvn install

COPY . .
RUN mvn package

FROM openjdk:8-jre-alpine3.7
WORKDIR /usr/src/myapp
COPY --from=builder /opt/hello_world/target .

RUN addgroup -S java && adduser -S -g java appuser
USER appuser

CMD ["java", "-jar", "hello-world-app-0.1.0.jar"]
```

We start by declaring the builder image, in this case, the maven:3.5-jdk-8 image. This image will be used to build the code and as such only used as an intermediate step. After maven has done installing dependencies and packaging the app in a jar file this image will exit and we move on to the actual image that we will use in production for running our app. This production image copies the jar file from the previous image (COPY --from=builder /opt/hello_world/target .) and runs our app.

When we build the image:

```
docker build -t simpleapp .
```

we can check the size of the image:

```
docker image ls | grep simpleapp | awk '{ print $8 "         " }'
83MB
```

The image is only 83MB!

If you change the source code in `src/main/java/hello/Greeter.java` from "Hello World!" to "Hello World!!!!!" and rebuild the image, the rebuilding process only takes 3 secs. Can you spot why it is faster this time? ***Cache*** If no, you should re-read my previous [post](https://donchev.is/post/docker-for-developers) :)


### Docker multi-stage builds caveats

If we change the source code a couple of times and build the image after each change we will get many orphaned (dangling) images. For example, I changed the Greeter.java function three times and build the corresponding image three times. If I list the images on my laptop I get:

```
docker images
REPOSITORY            TAG               IMAGE ID            CREATED             SIZE
simpleapp             latest            057e18566a5c        About an hour ago   83MB
none                  none              396fd17d8ba6        About an hour ago   765MB
none                  none              899f7cde139c        About an hour ago   83MB
none                  none              d28bf47eed96        About an hour ago   765MB
none                  none              c2ebfd088f1d        About an hour ago   83MB
none                  none              8e88d370e323        About an hour ago   765MB`
```

I can see that I have 5 dangling images (maked none). I need to remove them manually and the caveat with removing them manually is that caching stops to work and the next build will take 3-4 minutes :/

Something you should definitely keep in mind when using multi-stage builds.


### Docker volumes and file permission

Having worked with docker in many development environments I have encountered many problems with user privileges and the file system. Usually, the problems are caused because the developers are sharing code folders from the hosts to the container. The main problems I have seen can be categorized as two:


1. When writing to a docker volume you won't be able to access the files that the container has written because the process in the container usually runs as root.

2. You shouldn't run the process inside as root but even when you hard-code a user it still won't match the user in your team/jenkins/staging.


Docker provides a --user flag with it's run command. You can switch to a specified UID during container start like so:

```
docker run --rm -i -t --user $( id -u $USER ):$( id -g $USER ) simpleapp /bin/sh
```

This approach works 80% of the time but has some drawbacks. The biggest drawback being that the UID is not present in /etc/passwd file. I know the linux filesystem only cares about the UID and not the username BUT some applications will refuse to start if the user is not present in /etc/passwd

You will get an error like the following:

```
docker: Error response from daemon: linux spec user: unable to find user uidn6484: no matching entries in passwd file.
```

An improved approach would be to mount the /etc/passwd and /etc/group files into your container! Like so:

```
docker run --rm - i -t -v /etc/passwd:/etc/passwd:ro -v /etc/group:/etc/group:ro  --user $( id -u $USER ):$( id -g $USER ) simpleapp /bin/sh
```

The same solution can be applied to docker-compose files! Now the whole team can work with the same containers!

