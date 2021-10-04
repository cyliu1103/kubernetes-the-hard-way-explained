# Preface
The purpose of this repo is to detailed explain steps in the Kubernetes the Hard Way AWS ([KTHW-AWS](https://github.com/prabhatsharma/kubernetes-the-hard-way-aws) It tries to explain questions like "why do I need to run this command?", "what are these certificates for?"

KTHW-AWS is a guide inspired by the famous Kubernetes the Hard Way (KTHW) with adoption of AWS specifications. It's generally easy to follow either of these 2 guides to setup a multi-cluster Kubernetes services on Cloud providers. But it requires bit of explanation and research of what actually happened when you start a service, or why do I need this certificate to be created. 

I'll skip Chapter 1 to 3 and directly from [Chapter 4 Provisioning a CA and Generating TLS Certificates](https://github.com/prabhatsharma/kubernetes-the-hard-way-aws/blob/master/docs/04-certificate-authority.md). But if you never worked with Kubernetes before, you likely will miss some client tools. So do go through the original KTHW-AWS

Please also keep in mind of below diagram. It shows the communications between Kubernetes components, including:

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

