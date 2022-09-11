---
date: 2019-08-08
title: 'Prometheus with Azure Monitor'
slug: prometheus-with-azure-monitor-for-containers
toc: false
tags:
  - Cloud
  - Azure
---

I have been using Azure AKS for quite some time now and haven't had many problems with it. I run my own observability stack (Prometheus + Grafana) and logging stack (EFK) on AKS myself. I recently noticed on Azures [blog](https://azure.microsoft.com/en-us/blog/azure-monitor-for-containers-with-prometheus-now-in-preview/) that I can ditch my Prometheus from inside kubernetes and have azure take care of scraping and storing the application metrics. If you have been using Prometheus with AKS than there is not much that will change as a developer. If you, however, have been using Prometheus for DevOps related work and are quite good at writing PromQL than embrace yourself for KQL (Kusto Query Language)

```
<Queries>
InsightsMetrics
| where Name == "user_management"
| extend dimensions=parse_json(Tags)
| where request_status == "fail"
| where TimeGenerated > todatetime('2019-08-02T09:40:00.000')
| where TimeGenerated < todatetime('2019-08-02T09:54:00.000')
| project request_status, Num, TimeGenerated | render timechart
```

====

<br />

Note that this feature is still in preview and might change in the future. For all I know maybe Microsoft decides to support PromQL as well as KQL :)


### Implementation

In order to try this feature, you'll need an AKS cluster with OMS agent running version **ciprod07092019**   or above. Check if you have OMS agent in your AKS cluster and the version by running the following command

```
k -n kube-system get ds omsagent -o yaml | grep image:
image: mcr.microsoft.com/azuremonitor/containerinsights/ciprod:ciprod07092019
```

if you get an output with something similar to the above you are good to go. If not, you will need to enable OMS agent on your AKS cluster:

```
az aks enable-addons -a monitoring -n MyExistingManagedCluster -g MyExistingManagedClusterRG
```

I will write a follow-up post where I go into details of how to setup a production-grade AKS cluster with terraform. So for this post I assume you have a running AKS cluster.


### Activation

Now that we have verified that the OMS agent is running we can apply the configuration and have the OMS agent scrape our services:

```
kubectl apply -f https://raw.githubusercontent.com/microsoft/OMS-docker/ci_feature_prod/Kubernetes/container-azm-ms-agentconfig.yaml
```

The OMS agent should restart automatically by running the above command. Also, make sure you have enabled Prometheus metrics by annotating your services:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: user-management
  namespace: user-domain
  labels:
    app: user-management
  annotations:
    prometheus.io/scrape: 'true'
    prometheus.io/path: /metrics
    prometheus.io/port: '8080'
```

<br />

That should be all. After a minute you can query the service metrics from within azures portal using KQL :)

