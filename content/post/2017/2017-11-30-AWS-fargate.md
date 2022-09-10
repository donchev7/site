---
date: 2017-11-30
dateName: '30 November 2017'
title: 'AWS Fargate'
slug: aws-fargate
tags:
  - AWS
  - Cloud
  - Docker
---

In re:Invent 2017 AWS announced AWS Fargate among many other new services. AWS describes Fargate as

> A technology that allows you to use containers as a fundamental compute primitive without having to manage the underlying instances

<br />

![fargate](https://d2908q01vomqb2.cloudfront.net/da4b9237bacccdf19c0760cab7aec4a8359010b0/2017/11/29/Picture1.png)*Image Source and Credits ([AWS Blog](https://aws.amazon.com/blogs/aws/aws-fargate/))*


I have experience with ECS and Kubernetes and the underlying machines haven't posed a problem for me to manage. In Kubernetes draining a node for maintenance or update is achieved via the CLI

```
kubectl drain <node name>
```

====


This command will tell the kubernetes cluster to mark the node as unschedulable and prevent new pods from arriving. It will also evict running pods and move them to other nodes (except those are [static pods](https://kubernetes.io/docs/tasks/administer-cluster/static-pod/) which I never use).

After the node is drained you could perform OS upgrades or security patch installations and return the node to the cluster via 

```
kubectl uncordon <node name>
```

With AWS ECS its a little more involved but still not that difficult. Usually, you would run your ECS cluster with a launch configuration and auto scaling group and the services would be made accessible through a load balancer (AWS ELB). If you wanted to update the underlying ec2 instance to use a newer version of the AWS ECS optimized AMI you would use terraform or CloudFormation to update your launch configuration group and configure it to create new instances before deleting old ones.


<br />

With AWS Fargate obviously this burden shifts to AWS and you can focus on your containers (Business Logic) but I have a couple of questions that I haven't been able to find answers to:


1. If a container of some customer gets hacked and the hacker obtains escalated host rights what would stop him to shutdown my containers? Or worse break into my container and stealing sensitive data?
   - In my ECS clusters I use [Sysdig](https://www.sysdig.org/falco/) to detect this kind of problems but what does AWS use to keep my containers safe?

2. The pricing is ambiguous. As of this writing, the [official pricing page](https://aws.amazon.com/fargate/pricing/) states both that the pricing is *calculated based on the vCPU and memory resources **used** and that pricing is based on **requested** vCPU and memory resources*. Which one is it? Used or requested? People experienced with Kubernetes know the difference between kubernetes limits vs requests so how are we to interpret these pricings? I guess there is only one way to find out. I have to test it myself :)


Although there are still some questions open with AWS Fargate I still think this service might get popular with companies that use containers. I don't see companies abandoning AWS ECS totally and switch to Fargate but I can see its use case in CI where I could have different test tasks (Integration, End-End) and perform the tests on Fargate without much Terraform/CloudFormation boilerplate code.


On another note, you should check Google's recent [announcement](https://cloudplatform.googleblog.com/2017/11/Cutting-Cluster-Management-Fees-on-Google-Kubernetes-Engine.html) to cut kubernetes management fees to nil. Now that might pursue me away from AWS at least temporarily :)



