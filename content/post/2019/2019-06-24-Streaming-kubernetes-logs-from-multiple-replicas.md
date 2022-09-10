---
date: 2019-06-24
dateName: '24 June 2019'
title: 'Streaming kubernetes logs from multiple replicas'
slug: kubernetes-logs
tags:
  - Kubernetes
---

I have been really busy these days with kubernetes. Not only managing it but also developing microservices on top of it. One thing that has been particularly annoying is viewing logs in the command line from multiple replicas. First I installed [stern](https://github.com/wercker/stern) and then I tried [kubetail](https://github.com/johanhaleby/kubetail). Both are fine but require me to remember some commands on top of all the commands I have to remember using kubectl. 

I was reading this [document](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#logs) the other day and noticed the paragraph:

```
Begin streaming the logs from all containers in pods defined by label app=nginx

kubectl logs -f -l app=nginx --all-containers=true
```

I am not sure when this feature was added but I wanted to try it out and see if it was working the way I expected it. So I tried it out on one of the microservices I am working on.

I have a microservice called user-information it is of kind deployment and has the replica count set to 3. I want to stream all pod logs of this service so I tried:

```
kubectl logs -f --namespace user-information -l app=user-information
```

and see there all logs from all pods displayed in my console! Nice. No more external tools, just plain kubectl  and grep

====

### Bonus

If your services are structured and all of them have labels (which they should) you can further simplify the above command by adding a function to your ~/.functions file. Open the ~/.functionsfile with your preferred editor and paste the following block inside it:


```
logs() {
  kubectl logs -f -n $1 -l app=$1 --max-log-requests 5 --all-containers=true --timestamps=true
}
```

in a new terminal I can then simply view the logs with:

```
logs user-information
```


### Caveats

Unfortunately, this works only on deployments / statefulSets with replication count <= 5. If you try the above and encounter this

```
error: you are attempting to follow 6 log streams, but maximum allowed concurrency is 5
```

You might need to resort to something else then kubectl.

