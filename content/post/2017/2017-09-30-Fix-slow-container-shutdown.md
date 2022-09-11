---
date: 2017-10-30
title: 'Fix slow container shutdown'
slug: docker-containers-slow-shutdown
toc: false
tags:
  - Docker
---

Today I was working in a dev environment where everything was done within containers and docker-compose was used to define the services. Usually, when I work on my own projects in this way I don't have to rebuild containers every time I make changes to my code to reflect those changes but in this special case, there was no getting around it. Every time I made changes to the code I had to rebuild the container and run it again. The annoying part was that it took such a long time to re-up the container because the stop process took such a long time. I investigated it and it seems I have been running my apps in the container as the PID 1 process. How does this happen?

There are two ways of running your containers. With CMD or ENTRYPOINT or a combination of both. For example


```dockerfile
FROM alpine:3.6

CMD ["ping"]
```

and

<!--more-->

```dockerfile
FROM alpine:3.6
ENTRYPOINT ["ping"]
CMD ["8.8.8.8"]
```

are two ways of defining how your container should later be run. 


> CMD lets the user that runs the container overwrite the command where ENTRYPOINT does not give this freedom thus making the container a ping only app.


In both cases running the containers, the ping application is run as PID 1 and comes with a bunch of responsibilities and [edge cases](https://engineeringblog.yelp.com/2016/01/dumb-init-an-init-for-docker.html) that we have not anticipated - notably the handling of SIGTERMs.


Lets have a look at the container

```bash
docker run --rm -it --name pinger donchev7/alpine-pinger
```

```bash
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: seq=0 ttl=59 time=10.643 ms
64 bytes from 8.8.8.8: seq=1 ttl=59 time=9.861 ms
64 bytes from 8.8.8.8: seq=2 ttl=59 time=12.448 ms
```

The *donchev7/alpine-pinger* image is only 3.97MB, you should have no problem running this container in seconds.

While the container is running issue this command in another terminal window

```basb
docker exec pinger ps aux
```

you should see the following

```bash
docker exec pinger ps aux                               
PID   USER     TIME   COMMAND
1     root     0:00   ping 8.8.8.8
7     root     0:00   ps aux
```

There it is, ping is PID 1


### Connecting the dots

When we issue a docker stop command the docker daemon sends a SIGTERM signal to stop our pinger but since it's PID 1 nothing happens and after a while (10s) the docker daemon loses its patience with the container and sends a SIGKILL. Why does this happen? Obviously, because our application running in the container wasn't programmed to listen to SIGTERM.

While still having your pinger container running run the following

```bash
time docker stop pinger
```

On my computer it took 10,577 seconds to stop the container. That's crazy, containers should be lighting fast :)

### Solutions

- Add STOPSIGNAL SIGINT  to your Dockerfile (this is a quick & dirty solution and only takes care of one edge case and shouldn't actually be used

- Avoid being PID 1 when you run ad-hoc docker run commands you can pass the--init flagand your app won't become PID 1

Let's test the second solution out. Run the pinger again this time passing the --init flag like so

```bash
docker run --rm -it --name pinger --init donchev7/alpine-pinger
```

and then

```bash
time docker stop pinger
```

Now it takes around 500ms to stop the running container!


### Bonus

If you are using docker-compose and a YML version 2.2 or 2.3 you can do the same

```yaml
version: '2.3'
services:
  pinger:
    image: donchev7/alpine-pinger
    init: true
```

save this in a docker-compose.yml file and your docker-compose stop command should be lightning fast!

<br />
<br />