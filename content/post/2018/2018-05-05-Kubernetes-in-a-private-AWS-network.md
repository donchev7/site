---
date: 2018-05-05
dateName: '05 May 2018'
title: 'Kubernetes in a private AWS network'
slug: kubernetes-in-a-private-network
tags:
  - AWS
  - Kubernetes
---


I have been working on setting up a kubernetes cluster on AWS. Usually, the setup isn't difficult and there are many tools that can assist, for example, [kops](https://github.com/kubernetes/kops), [spray](https://github.com/kubernetes-incubator/kubespray), [conjure-up](https://github.com/kubernetes/kops) and probably many others I am forgetting. The problem I had with these tools is that they are configured to create all the resources a kubernetes cluster might need. For example, trying to use kops to create a cluster in a private subnet [fails](https://github.com/kubernetes/kops/issues/780) if no Internet gateway exists, thus it will try to create an internet gateway. But what if you are in a corporate environment and no internet gateway has been provisioned? What if the internet breakout should go through your own data center? Now your options are limited to:


1. Setting up the cluster manually (hard way)

2. Dig deep into the kops / spray code and modify it to do what you need 


Obviously, both options are time-consuming. What if there is another way? The semi-automated way. Actually, the creators of kubernetes thought about the case where flexibility and customizability are needed. That's why they gave us [kubeadm](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/) currently my go-to tool for provisioning and managing kubernetes clusters. I call it the semi-automated way because I have to ssh into the master / nodes and issue kubeadm commands and write some config files but with a little bash scripting and terraform knowledge it's rather easy to automate everything.

====


I already wrote the automation for you and you can get started by cloning my [repo](https://github.com/donchev7/kubernetes-private-aws). I'll explain briefly in this post how you can use the code.

1. Replace the ssh keys in the ssh/ folder with your own or generate new ones.

2. Take a look at **input.tf** and change it to your liking. If you want to install Kubernetes 1.10.1, just change the version variable. Obviously here you will need to define the proxy you want your kubernetes cluster to run behind. Declare the proxy variable like: **http://PROXY:PORT**

3. Go into the state folder and provision the resources there, e.g. **cd state/ && terraform init && terraform plan / apply**

4. Go back and provision the kubernetes cluster with terraform, e.g. **terraform init && terraform plan && terraform apply**

5. The previous step should give you the master node IP, (actually the LB IP). Use the IP to configure **kubectl**


In the background what the code does is utilize [redsocks](https://github.com/darkk/redsocks) to forward all the traffic from the containers / host through your corporate proxy. Redsocks will not route internal cluster traffic through your proxy. The traffic rules are configured [here](https://github.com/donchev7/kubernetes-private-aws/blob/master/scripts/install-redsocks.sh) if you want to have a look.


Using redoscks is essential in this setup. If you decide not to use it you will need to configure every node and every application / container with proxy configurations. This will quickly lead to maintenance nightmares and hard to debug networking problems.


### Cavets

The code assumes you are provisioning a kubernetes cluster with Ubuntu 16.04 as host OS. Further, it assumes that you already have the networking provisioned (VPC, subnets...)

It can also be made better by adding additional checks. It would also be nice to have a choice of CNI, currently it supports only flannel.

Feel free to copy and edit as you like.


