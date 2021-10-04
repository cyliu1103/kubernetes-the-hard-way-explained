# Kubernetes the Hard Way AWS Explained

## Table of Contents
[Chapter 04: Provisioning a CA and Generating TLS Certificates](https://github.com/cyliu1103/kubernetes-the-hard-way-explained/blob/master/04-certificate-authority.md)

[Chapter 05: Generating Kubernetes Configuration Files for Authentication](https://github.com/cyliu1103/kubernetes-the-hard-way-explained/blob/master/05-kubernetes-configuration-files.md)

[Chapter 06: Data Encryption Keys](https://github.com/cyliu1103/kubernetes-the-hard-way-explained/blob/master/06-data-encryption-keys.md)

[Chapter 07: Bootstrapping the etcd Cluster](https://github.com/cyliu1103/kubernetes-the-hard-way-explained/blob/master/07-bootstrapping-etcd.md)

[Chapter 08 Part 1: Bootstrapping the Kubernetes Control Plane - API Server](https://github.com/cyliu1103/kubernetes-the-hard-way-explained/blob/master/08-01-api-server.md)

[Chapter 08 Part 2: Bootstrapping the Kubernetes Control Plane - Controller Manager](https://github.com/cyliu1103/kubernetes-the-hard-way-explained/blob/master/08-02-controller-manager.md)

[Chapter 08 Part 3: Bootstrapping the Kubernetes Control Plane - Scheduler and RBAC for Kubelet Authorization](https://github.com/cyliu1103/kubernetes-the-hard-way-explained/blob/master/08-03-scheduler-rbac.md)

[Chapter 09 Part 1: Bootstrapping the Kubernetes Worker Nodes - Overview](https://github.com/cyliu1103/kubernetes-the-hard-way-explained/blob/master/09-01-overview.md)

[Chapter 09 Part 2: Bootstrapping the Kubernetes Worker Nodes - CNI](https://github.com/cyliu1103/kubernetes-the-hard-way-explained/blob/master/09-02-cni.md)

[Chapter 09 Part 3: Bootstrapping the Kubernetes Worker Nodes - Containerd](https://github.com/cyliu1103/kubernetes-the-hard-way-explained/blob/master/09-03-containerd.md)

[Chapter 09 Part 4: Bootstrapping the Kubernetes Worker Nodes - Kubelet](https://github.com/cyliu1103/kubernetes-the-hard-way-explained/blob/master/09-04-kubelet.md)

[Chapter 09 Part 5: Bootstrapping the Kubernetes Worker Nodes - Kube-proxy](https://github.com/cyliu1103/kubernetes-the-hard-way-explained/blob/master/09-05-kube-proxy.md)

[Chapter 10: Configure Kubectl for Remote Access](https://github.com/cyliu1103/kubernetes-the-hard-way-explained/blob/master/10-configuring-kubectl.md)

[Chapter 11: Pod Network Routes](https://github.com/cyliu1103/kubernetes-the-hard-way-explained/blob/master/11-pod-network-routes.md)

[Chapter 12: Deploying the DNS Cluster Add-on](https://github.com/cyliu1103/kubernetes-the-hard-way-explained/blob/master/12-dns-addon.md)

[Chapter 13: Smoke Test](https://github.com/cyliu1103/kubernetes-the-hard-way-explained/blob/master/13-smoke-test.md)

[Chapter 14: Cleanup](https://github.com/cyliu1103/kubernetes-the-hard-way-explained/blob/master/14-cleanup.md)

---
The purpose of this repo is to detailed explain steps in the Kubernetes the Hard Way AWS ([KTHW-AWS](https://github.com/prabhatsharma/kubernetes-the-hard-way-aws)) It tries to answer questions like "why do I need to run this command?", "what are these certificates for?"

KTHW-AWS is a guide inspired by the famous Kubernetes the Hard Way (KTHW) with adoption of AWS specifications. It's generally easy to follow either of these 2 guides to setup a multi-cluster Kubernetes services on Cloud providers. But it requires bit of explanation and research of what actually happened when you start a service, or why do I need this certificate to be created. 

I'll skip Chapter 1 to 3 as they are some basic setups on AWS and client tools. But if you never worked with Kubernetes before, you likely will miss some client tools. So do go through the original KTHW-AWS

Understanding [below diagram](https://kubernetes.io/docs/concepts/overview/components/) is crucial to understand Kubernetes the Hard Way AWS. It shows the communications between Kubernetes components that we will be dealing with in following chapters, including:

(1) etcd node talk to peers

(2) API Server talks to etcd

(3) Controller Manager talks to API server

(4) Scheduler talks to API server

(5) Cloud Controller Manager talks to API Server

(6) Kubelet talks to API Server

(7) API Server talks to Kubelet

(8) Kubeproxy talks to API server

(9) Clients (such as `kubectl`) talks to API Server (Not shown in this diagram)

![Kubernetes Components, taken from Official [Document](https://kubernetes.io/docs/concepts/overview/components/)](components-of-kubernetes.svg "Title")

---

