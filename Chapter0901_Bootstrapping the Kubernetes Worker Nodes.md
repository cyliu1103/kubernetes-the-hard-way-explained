# Bootstrapping the Kubernetes Worker Nodes - Part 1 Overview

The worker nodes are the ones that run actual application Pods. In this guide we have provisioned 3 worker nodes. Their configuration are almost identical.

Kubernetes the Hard Way AWS mentioned that
>The following components will be installed on each node: runc, gVisor, container networking plugins, containerd, kubelet, and kube-proxy.

I'll briefly explain what are these

- `runc`

It is a OCI compatible tool to actually spawn and run containers. 

- `containerd`

It is a CRI compliant runtime. `containerd` will call `runc` to spin up containers

- `gVisor` (or `runsc`)

It is an alternative to `runc` with additional security features. It's a container sandbox developed by Google. This [article](https://cloud.google.com/blog/products/containers-kubernetes/how-gvisor-protects-google-cloud-services-from-cve-2020-14386) explained the gVisor and why it is a secured sandbox.

- Container network plugins (`cni`)

CNI is responsible for Pod networking. The reason of having CNI is that, CRI, or container runtime interface doesn't provide any networking implementations. To make Pods talk with each other, you need to have CNI. When CRI spin up containers, CRI also calls CNI to include these containers in network

- kubelet

An agent runs on each node

- kube-proxy

A network proxy that runs on each node to maintain network rules 

It is not uncommon to confuse with `runc`, `containerd`, `OCI` and `CRI`. Have a look at this [article](https://www.tutorialworks.com/difference-docker-containerd-runc-crio-oci/) which explained these terminologies quite well.

In next 4 parts, I'll explain them in detail.