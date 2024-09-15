---
date: 2024-07-19
image: 'post/2024/sealed-secrets-ui.jpg'
title: 'Sealed Secrets UI'
slug: 'sealed-secrets-ui'
toc: true
tags:
  - DevOps
  - Kubernetes
---

Kubernetes is a powerful orchestration platform for microservices or for that matter any kind of workloads. Yet, managing sensitive data such as passwords, API keys, and credentials in Kubernetes can be frustrating. 
Tools such as [Vault](https://www.vaultproject.io/), [Secret Store CSI](https://secrets-store-csi-driver.sigs.k8s.io/) and [External Secrets Operator](https://external-secrets.io/latest/) exist for this purpose. But there is a simpler way to manage secrets in Kubernetes especially if you are already doing GitOps. [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) by Bitnami is one of my favorite tools. I can store my secrets **encrypted** in Git and have them **decrypted** in my Kubernetes cluster.

However, managing Sealed Secrets can be a bit challenging especially for beginners. The CLI tool is great but it can be a bit intimidating and don't get me started with updating / merging secrets into existing Sealed Secrets.

This challenge is precisely what inspired us to create and now open-source the [Sealed Secrets UI](https://github.com/alpheya/sealed-secrets-ui).

## Why Sealed Secrets UI?
Sealed Secrets UI helps creating and updating sealed secrets in Kubernetes. Developed to integrate seamlessly with the Bitnami Sealed Secrets controller, this tool significantly reduces the complexity and potential for errors when managing secrets.

You can now go from this:

```bash
kubectl create secret generic awesome-secret \                                                                                                             
--from-literal=username=someUser \
--from-literal=password=somePassword \
--namespace=dev \
--dry-run=client \
-o yaml | kubeseal -o yaml
```

or this
  
```bash
echo -n PutPasswordHere | kubectl create secret generic awesome-secret --dry-run=client --from-file=apiKey=/dev/stdin -o yaml \
  | kubeseal --merge-into path/to/existing-sealed-secret.yaml
```

to this

![Sealed_Secrets_UI](post/2024/sealed-secrets-ui.png)


For a better understanding of how and why check out this cool **video** by **[kubesimplify](https://www.youtube.com/@kubesimplify)**!

{{< youtube E0F4usFauvc >}}


He did a great job explaining the Sealed Secrets UI and how it can be used to manage secrets in Kubernetes. At least a lot better than I can do here in writing 😅


## Features and Benefits

- Simple Web Interface: Instead of juggling kubectl commands, users can create and update sealed secrets directly through a user-friendly dashboard.
- Seamless Integration: The UI fetches the necessary public keys directly from the Sealed Secrets controller without manual intervention, ensuring that secrets are always encrypted with the correct keys.
- Intelligent Management: The UI intelligently checks for existing secrets and updates them accordingly.
- Environment Customization: Users can customize settings through environment variables to fit different deployments, such as specifying namespaces or controller names.

## Getting Started

The prerequisites for using Sealed Secrets UI are minimal:

The Kubernetes cluster must have the Sealed Secrets controller installed. The Sealed Secrets Helm chart is recommended for ease of installation.

### Quick Configuration
Setting up the Sealed Secrets UI involves minor configuration, primarily related to pointing it at the right namespace and controller, if defaults are not used:

```
SEALED_SECRETS_CONTROLLER_NAMESPACE: kube-system (default)
SEALED_SECRETS_CONTROLLER_NAME: sealed-secrets-controller (default)
```

### Deployment
Deploying the Sealed Secrets UI is straightforward with the provided Kubernetes manifests. Here’s an example of how to deploy it within your cluster:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sealed-secrets-ui
  namespace: kube-system
...
```
For complete deployment instructions, refer to the deployment section in our [repository](https://github.com/alpheya/sealed-secrets-ui)


As always, if you have any questions or remarks, feel free to ping me on twitter [@bobby_donchev](https://twitter.com/bobby_donchev).