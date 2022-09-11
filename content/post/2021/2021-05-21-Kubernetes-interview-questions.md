---
date: 2021-05-21
title: 'Kubernetes interview questions'
slug: kubernetes-interview-questions
toc: false
# image: post/2021/kubernetes-horizontal-color.png
tags:
  - Kubernetes
  - Docker
---

What have all recent projects (last 5 years) have had in common? 

**Kubernetes**. :)

I have been developing applications and leading teams that developed large systems around Kubernetes. Most of the time the developers didn't want to know much of the inner workings of kubernetes. However, I believe to be a competent developer, that wants to deliver features ontop of kubernetes you need to be familar with kubernetes concepts.

<br />

The last years I have interviewed 20+ developers. In this article I want to note down some of the questions I ask and what answers I look for.

<!--more-->

<br />

Usually I start with docker. I know I know. You might not agree with me. Why ask docker questions? But I believe it still shows the depth the individual has. So I might start with something simple such as

> What is the difference between a VM and docker?

I am fishing for **kernel** here but talk around cgroup, security (SELinux) are also fine.

<br />

> Whats the differnece between an image and a container?

Fishing for runtime / package but anything around  repository / docker hub or describing the **docker build** vs **docker run** commands is fine.

<br />

Now we can go with some icebreakers for kubernetes questions :)

> Whats the difference between the service types, ClusterIP, NodePort and LoadBalancer

There are many ways to answer this question. It should also lead to a discussion that I can use to gauge the depth of the knowledge. My experience is that people usually don't know much about NodePorts, so let me answer it here:

You want to use NodePort if you want to provide a port number in addition to the virtual IP provided by the ClusterIP type. This way you can access your backend Pods by using the workers (k8s nodes) IP and the port number.

<br />

> What happens when a readinessProbe fails?

Fishing for availability, k8s service and how it relates to deployments and the probes. 

Correct answer: excludes pods from the Service targets.

<br />

> Whats the difference between statefulsets and deployments?

I am looking not only for the stateful vs stateless discussion (scaling a deployment with a PVC is painfull). But also the networking advantages statefulsets bring to the table, such as the pods have stable network IDs and predictable names.

<br />

> Can I use a Role to grant access to all Pods in the cluster?

For this you would need a ClusterRole, so no.

<br />

I believe these questions to have low to medium difficuly. If you have an unicorn you can ask something more exotic such as:


> What is **metadata.finalizers** and how do they work?

Async pre-delete hook. Will be called whenever the resources that have them gets deleted.

<br />

> What happens to PVCs once a statefulset has been deleted?

Will not be autodeleted :)

<br />

> What happens when you only define **resources.limits.memory** on a deployment?

**resources.requests.memory** will be equal to the limits you have defined.

<br />

> What happens when a container consumes more memory than defined in **resources.requests.memory**

Becomes a target for eviction. If the pod does not have **resources.requests.memory** and **resources.limits.memory**, by defualt they are using more than they requested :)

<br />

> Does kubernetes evict Pods that run on a worker that is low in CPU?

No. Thus it is important to specify **resources.requests.cpu** in production.




