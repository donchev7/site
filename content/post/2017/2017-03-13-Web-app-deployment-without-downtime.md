---
date: 2017-03-13
title: 'Web app deployment without downtime'
slug: webapp-deployment-without-downtime
toc: false
tags:
  - Docker
---

Hi there today I'll show you how I deploy my site to production without incurring downtime. My site isn't big and is deployed on a single server where I have NGINX, HAproxy and redis running. Here a diagram that shows the setup

![diagram](post/2017/diagram.jpg)

<!--more-->


All services run in separate Docker containers. Why? Unless you have been living on Mars the past years you know why containers are awesome if not go on and read my post on docker containers.

On an architectural note the reason I am using NGINX and HAproxy is that I let NGINX handle compression, caching, SSL management and zipping static files and I use HAproxy in order to not incur downtime when I deploy a new version of my site. Only during deployments I spin up an additional instance of the web app and HAproxy automatically routes traffic between the instances. The routing decision is based on RR (Round-Robin) but can be configured in the docker-compose file to any load balancing algorithm HAproxy supports (weighted round robin, dynamic round robin, least connection ...etc..)

Let's now look at an example application to see how it all works


If you want to see this in action yourself you can clone my repo:

```bash
git clone https://github.com/donchev7/webapp-sample
docker build -t webapp-sample .
```

For this example to work properly on your computer it is important to tag the docker build as "webapp-sample" if you wish to use a different name update the docker-compose.yml file.

After building the Dockerfile do a

```bash
docker-compose up -d
```

This will bring up the environment. NGINX is running on port 80, if you do a quick

```bash
curl http://localhost
```

you should see the webapp responding with "Hello beautiful world!". Now imagine you make a change to the webapp and want the changes to be reflected on your website you would need to do a

```bash
docker build -t webapp-sample .
docker-compose up -d
```

That of course introduces service downtime to your website because with the  docker-compose up command docker recreates your container from the new image and in the process takes your container down for a few seconds.


## Now lets see how to manage this without downtime

Open app.py and make a change to the function route

```python
def route(app):
    """
    Add some routes to our test web app
    
    :param app: Flask application instance
    :return: Flask app
    """
    @app.route('/')
    @app.route('/index') 
    @app.route('/home')
    def index():
        return "Hello beautiful changed world!" 

    return app
```

As we can see I changed the return statement. Now we save the app.py file and build a new image:

```bash
docker build -t webapp-sample .
```

Now that we have a new image we need the webapp container to switch to this new image without restarting the container. We do this with the help of HAproxy, just write:

```bash
docker-compose scale website=2
docker ps
```

Now we should have two containers running the webapp, webappsample_website_1 is running your old version and webappsample_website_2 is running your newer version (take a look at the image above). To switch to only having one version running (the newer) we would need to just get the name of the container and execute

```bash
docker stop webappsample_website_1
```

Now HAproxy gets notified that it is only linked to one webapp container and thus only routes traffic to your newer version of webapp.


## How does this work behind the curtains?

The HAproxy docker image is configured to automatically configure HAproxy depending on the linked containers it has. The solution also works on docker swarm and docker cloud. You can have a look at the source code [here](https://github.com/docker/dockercloud-haproxy/tree/master/haproxy).

