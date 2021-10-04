# Bootstrapping the Kubernetes Control Plane - Part 2 Controller Manager

In previous chapter, we discussed the API Server service unit and the meaning of each flags. This chapter we will focus on Controller Manager.

Controller Manager runs an infinite loop. It watches the state of all resources and make changes to ensure current state meets the desire state. 

Let's examine the flags of Controller Manager

```bash
ExecStart=/usr/local/bin/kube-controller-manager \\
  --bind-address=0.0.0.0 \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
```
- `--bind-address` the IP address on which to listen

- `--cluster-cidr` specifies the CIDR range for Pods in cluster. Note this should not collide with `--service-cluster-ip-range` (which is the ClusterIP range for Services) in API Server configuration. 

- `--cluster-name` is the name prefix of the cluster

- `--cluster-signing-cert-file` and `--cluster-signing-key` specifies the CA that Controller Manager can use to sign cluster-scroped certificates. Remember we talked about that Kubernetes provides a `certificate.k8s.io` API? While the API Server is responsible to receive these certificate signing request and authenticates those requests, the controller manager signs the actual certificate. It will use the CA file and CA key specified in these 2 flags to issue certificates

- `--kubeconfig` kubeconfig file location. The `kube-controller-manager.kubeconfig` file was created previously and has CA, server certificate and server key embedded to it. One thing to noticed is that it is [convention](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/) to use `kubeconfig` configuration file to store cluster access and authentication information in Kubernetes cluster. Later you'll see that other services, such as kube-proxy and kubelet also use `kubeconfig`

- `--leader-elect` whether to use leader election or no. Note both Controller Manager and Scheduler optionally use leader elect. If this flag is enabled, when start up, the Controller Manager instances will compete to create a service endpoint called `kube-controller-manager`. The instance that firstly creates this endpoint will add an annotation `control-plane.alpha.kubernetes.io/leader` to itself in order to expose leadership to followers. The followers, on the other hand, will add ``control-plane.alpha.kubernetes.io/leader:{"holderIdentity":"identity_string"}` in their own annotations to specify the leader identity.

- `--root-ca-file` and `service-account-private-key-file` are both for issuing ServerAccount JWT tokens. If `--root-ca-file` is set, the root CA will be included in SA's token secret. And `service-account-private-key-file` provides the private key file to sign service account tokens

- `--service-cluster-ip-range` specifies the service IP range. This must be identical with the value specified in `--service-cluster-ip-range` flag of API Server

- `--use-service-account-credentials` when enabled, each controller will use a separate service account in kube-system namespace, corresponding roles for each controller. For example, a service account for deployment-controller will be created, corresponding to `system:controller:deployment-controller` role. If this set to false, it runs all control loops using its own credential and you need to grant all the relevant roles

- `--v` log level
