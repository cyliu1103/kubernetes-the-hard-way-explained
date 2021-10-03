# Provisioning a CA and Generating TLS Certificates
## Certificate Authority
Kubernetes requires PKI certificates for authentication over TLS. This official [doc](https://kubernetes.io/docs/setup/best-practices/certificates/) includes all certificates that we need to provision a Kubernetes cluster. But the first thing we need to do is to have a Certificate Authority (CA) that can sign these certificates.

There're multiple ways to create a CA. `cfssl`, which is used in this guide, is one of these tools developed by Cloudflare.

The usage and configuration examples of `cfssl` can be found in its [Official Github repo](https://github.com/cloudflare/cfssl). In this guide, `ca-config.json` and `ca-csr.json` are both configuration files taken by `cfssl` command line tool to generate a self-signed CA. These 2 files define some parameters of the CA. Particularly,

- The CA has an expiry of 5 years (`"expiry": "8760h"`)
- The CA key usages (`usages` section. For a full list of signing profile usages, see cfssl [doc](https://github.com/cloudflare/cfssl/blob/master/doc/cmd/cfssl.txt))
- The CA certificate Common Name (CN) is `kubernetes`
- The CA private key use RSA algorithm and the key length is 2048 bit
- Some Issuer Name filed such as Country is US, Locality is Portland, etc

Once you run the command, it will generate 2 files  `ca-key.pem` and `ca.pem` . They are CA key and CA certificate. These 2 files will be used almost in every components of this Kubernetes Cluster as the root CA.

### A Side Note
In this guide, only 1 root CA is provisioned. However, it is possible to use multiple CAs inside a Kubernetes Cluster. For example, you can have a dedicated root CA for etcd so etcd peers can use that certificate to authenticate each other. If you do such way, the etcd bootstrap command will be
```
/usr/local/bin etcd \
--trusted-ca-file=/etc/etcd/ca.pem \
--peer-trusted-ca-file=/etc/etcd/<etcd-dedicated-ca.pem>
<..other arguments omitted..>
```
The `--trusted-ca-file` is used to authenticate any clients connected to the etcd service (e.g., API server). The `--peer-trusted-ca-file` is to authenticate etcd peers, as etcd is a distributed storage and each etcd node need to authenticate and connect to other peers. 

But you should consider to run only 1 CA in a Kubernetes cluster for easier management. Imagine if you have 4 CAs and they have different expiry date, it is extremely hard to tell which CA and the certificate it signed expired.

---
## Client and Server Certificates

### The Admin Client Certificate

This is the certificate that will be used by the `admin` RBAC user to interact with the Kubernetes Cluster. The certificate signing request file `admin-csr.json` is similar with what is used in CA `ca-csr.json`. The difference is that instead of creating a self-signed certificate, it generate a certificate and ask CA to sign it (see the  `cfssl gencert` command)

The output `admin-key.pem` and `admin.pem` will be used in [`kubectl` configuration section](https://github.com/prabhatsharma/kubernetes-the-hard-way-aws/blob/master/docs/10-configuring-kubectl.md#the-admin-kubernetes-configuration-file) in Chapter 10

---
### The Kubelet Client Certificates

This is the certificate that will be used by `kubelet` on each worker node to communicate with API server. Have a look at the Kubernetes Components diagram at the beginning if you can't remember.

As stated in the guide
> Kubernetes uses a special-purpose authorization mode called Node Authorizer, that specifically authorizes API requests made by Kubelets.

What does this mean?

Basically, the Node Authorisation is a special authorization mode in Kubernetes that is only used to authorise API calls by kubelet. API calls by other client, such as `kubectl` CLI, will not be authorized by Node Authorisation. Once authorised, the Node authoriser allows kubelet to perform a list of pre-defined API operations. 

Then the question comes to, how does the Node Authorisation know this API request comes from kubelet?

Well, the TL;DR version is, when kubelet starts up, it will search for API Server URL in its `kubeconfig` configuration file and try to connect to it. If connection is successful, the API server will treat it as a node and start assigning Pods to it. 

Here's a long version of explanation.

When a worker node starts up, it won't be automatically connect to the API server. Then the kubelet service starts. The kubelet service usually starts with a number of arguments. You can find such an example from Chapter 09, under the `/etc/systemd/system/kubelet.service` file. Note the kubelet service contains 2 arguments
```shell
ExecStart=/usr/local/bin/kubelet \
  --config=/var/lib/kubelet/kubelet-config.yaml \
  --kubeconfig=/var/lib/kubelet/kubeconfig \
<..other arguments omitted..>
```
The `kubeconfig` file contains API Server information. A sample looks like this. Kubelet will make connection request to the server URL
```YAML
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <ca.pem content>
	server: https://kube-loadbalancer.aws.com:443
  name: kubernetes-the-hard-way
contexts: null
current-context: ""
kind: Config
preferences: {}
users: null
```
If the connection is successful, API Server will treat the kubelet as a node and start assigning Pods to it. 

But this requires `kubeconfig` file to be pre-configured prior worker nodes bootstrap. What's more, if we have a detailed look at the other configuration files, you will find out that the `/var/lib/kubelet/kubelet-config.yaml` also requires pre-configured, because it contains worker node's own TLS certificate. That means if I want to provision a worker node, not only do I need to ship the kubeconfig file but also setup TLS keys for it. Wouldn't it be nice if the API Server can automatically sign certificate for these worker nodes?

It can. From Kubernetes version 1.4, a new API `certificates.k8s.io` is introduced to allow kubelet to create CSR and API Server to sign it. But as it's beyond this guide. The details and documents can be found in TLS bootstrapping and Authenticating with Bootstrap Tokens [official document](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/)

---
### The Controller Manager Client Certificate
This is self-explanatory. These private key and certificate file are for Kubernetes Controller Manager. The Controller Manager is a single process but running different controllers, such as Job controller who watches for `Job` object, etc. 

The output `kube-controller-manager-key.pem` and `kube-controller-manager.pem` will be embedded to the `kube-controller-manager.kubeconfig` file in [Chapter 05](https://github.com/prabhatsharma/kubernetes-the-hard-way-aws/blob/master/docs/05-kubernetes-configuration-files.md#the-kube-controller-manager-kubernetes-configuration-file), which will subsequently be used in [Chapter 08](https://github.com/prabhatsharma/kubernetes-the-hard-way-aws/blob/master/docs/08-bootstrapping-kubernetes-controllers.md#configure-the-kubernetes-controller-manager) as an argument to bootstrap the kube-controller-manager service

---
### The Kube Proxy Client Certificate

Similar with the Controller Manager, they are used in kube-proxy service on each worker node to provide networking functionality. 

The output `kube-proxy-key.pem` and `kube-proxy.pem` will be embedded to the `kube-proxy.kubeconfig` file in [Chapter 05](https://github.com/prabhatsharma/kubernetes-the-hard-way-aws/blob/master/docs/05-kubernetes-configuration-files.md#the-kube-controller-manager-kubernetes-configuration-file), which will subsequently be used in [Chapter 09](https://github.com/prabhatsharma/kubernetes-the-hard-way-aws/blob/master/docs/09-bootstrapping-kubernetes-workers.md#configure-the-kubernetes-proxy) as an argument to bootstrap the kube-proxy service

---
### The Scheduler Client Certificate

Similar with above, these 2 output files are for Scheduler, on Control Plane. The output `kube-scheduler-key.pem` and `kube-scheduler.pem` will be embedded to the `kube-scheduler.kubeconfig` file in [Chapter 05](https://github.com/prabhatsharma/kubernetes-the-hard-way-aws/blob/master/docs/05-kubernetes-configuration-files.md#the-kube-scheduler-kubernetes-configuration-file), which will subsequently be used in [Chapter 08](https://github.com/prabhatsharma/kubernetes-the-hard-way-aws/blob/master/docs/08-bootstrapping-kubernetes-controllers.md#configure-the-kubernetes-scheduler) as an argument to bootstrap the kube-scheduler service

---
### The Kubernetes API Server Certificate

These 2 output files are for API Server, on Control Plane. Pay special attention to the `-hostname` field when create CSR.
```shell
-hostname=10.32.0.1,10.0.1.10,10.0.1.11,10.0.1.12,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,kubernetes.default
```
This `-hostname` filed will be injected to API Server certificate's Subject Alternative Names (SANs), which restricts the IP address that a client wants to connect to. For example, if I were to miss the `${KUBERNETES_PUBLIC_ADDRESS}` in `-hostname`, later I would encounter certificate error if I use `kubectl` to connect to the Kubernetes cluster's public IP endpoint. 

The listed hostnames are:
- `10.32.0.1` - the IP address that will be resolved cluster-wide (note the cluster IP range is `10.32.0.1/16`). This IP is used inside cluster
- `10.0.1.10` - the IP address of controller-0. This IP is used inside cluster
- `10.0.1.11` - the IP address of controller-1. This IP is used inside cluster
- `10.0.1.12` - the IP address of controller-2. This IP is used inside cluster
- `${KUBERNETES_PUBLIC_ADDRESS}` - the public IP address. This IP is used when outside clients (such as `kubectl`) connect to API Server
- `127.0.0.1` - the localhost address. This is used when, for example, Controller Manager calls the API Server's public endpoint
- `kubernetes.default` - the DNS name that is resolved inside the Pod of default namespace

If you look detailed to the `CSR.json` file of each components, you'll find they have set Issuer `Organisation` field differently:

```json
kube-scheduler-csr.json:
"O": "system:kube-scheduler",
kube-proxy-csr.json:
"O": "system:node-proxier"
kube-controller-manager-csr.json:
"O": "system:kube-controller-manager"
worker-0.csr.json:
"O": "system:nodes"
```

Except for the last `worker-0.csr.json` whose Organisation field must be `system:nodes` in order for Node Authorisation to authorize it, I don't seem to find any significance of using these particular RBAC groups.

---
## The Service Account key Pair
The implementation of Kubernetes Service Account (SA) differs from all above components. Instead of providing a signed certificate to each SA, Kubernetes leverage JWT for authorization. In this section, the `service-account-key.pem` and `service-account.pem` are private and public keys that are used to generate and authenticate JWT tokens. Specifically, the public key will be distributed to API Server in order to authenticate the JWT token whereas the private key will be distributed to Controller Manager to generate the JTW token, as mentioned in the [official document](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/):
> You must pass a service account private key file to the token controller in the kube-controller-manager using the --service-account-private-key-file flag. The private key is used to sign generated service account tokens. Similarly, you must pass the corresponding public key to the kube-apiserver using the --service-account-key-file flag. The public key will be used to verify the tokens during authentication.

You can also observe such behaviour in Chapter 08, under [Configure the Kubernetes API Server](https://github.com/prabhatsharma/kubernetes-the-hard-way-aws/blob/master/docs/08-bootstrapping-kubernetes-controllers.md#configure-the-kubernetes-api-server) and [Configure the Kubernetes Controller Manager](https://github.com/prabhatsharma/kubernetes-the-hard-way-aws/blob/master/docs/08-bootstrapping-kubernetes-controllers.md#configure-the-kubernetes-controller-manager) section

---
## Distribute the Client and Server Certificates
The certificates required to setup a Kubernetes cluster are completed. Below table is a summaries
| Certificate Name | Purpose | Service | Control Plane / Worker Node |
| ---------------- | ------- | ------- | ---------------------------- |
| `ca-key.pem` and `ca.pem` | Root CA cert and private key to sign other certificates | Almost all services | Control Plane and Node |
| `admin-key.pem` and `admin.pem` | Admin RBAC user to interact with API Server | N.A., used by client tool such as `kubectl` | NA |
| `worker-[0,1,2]-key.pem` and `worker-[0,1,2].pem` | Kubelet client certificate to authenticate against API Server | kubelet | Worker Node |
| `kube-controller-manager-key.pem` and `kube-controller-manager.pem` | Controller Manager certificate to authenticate against API Server | Controller Manager | Control Plane | 
| `kube-proxy-key.pem` and `kube-proxy.pem` | Kube proxy certificate to authenticate against Server | Kube-proxy | Worker Node |
| `kube-scheduler-key.pem` and `kube-scheduler.pem` | Scheduler certificate to authenticate against API Server | Kube scheduler | Control Plane |
| `kubernetes-key.pem` and `kubernetes.pem` | API Server certificate | API Server, etcd | Control Plane |
| `service-account-key.pem` and `service-account.pem` | Private key used by Controller Manager to generate SA JWT token; public key used by API Server to authenticate JWK token | Controller Manager, API Server | Control Plane

Note some certificates are embedded to kubeconfig files so they don't need to be directly distributed. These certificates are `kube-proxy`, `kube-controller-manager`, `kube-scheduler` and `kubelet`

### A side note of mTLS
It is extremely important to keep in mind that all components in Kubernetes communications are mutual TLS (mTLS), meaning both Client and Server need to authenticate each other. The major difference between normal TLS and mTLS is that for mTLS, not only does client authenticates server, but also the server needs to authenticate client. That's why for example, when API Server, as a client, connects to etcd (server), the API Server not only needs to have a CA that can authenticate etcd is the genuine server but also also need to have:

(1) A TLS certificate signed by a CA that is trusted by etcd

(2) The private key associated with the TLS certificate so API Server can prove to etcd that it owns the TLS certificate (i.e., API Server can decrypt the information that etcd encrypted with API Server's TLS certificate)

Let's examine the connection procedure:
- Stage 1: etcd server needs to prove its legitimacy to API Server
  - API Server wants to connect to etcd
  - etcd sends its certificate (in etcd's `--cert-file`) to prove to API Server that it's legitimate 
  - API Server gets etcd's certificate, then it asks its own CA (in API Server's `--etcd-cafile`) if the certificate is valid
- Stage 2: API Server needs to prove its legitimacy to etcd
  - API Server sends its certificate (in API Server's `--etcd-certfile`)
  - etcd validate this certificate against CA (in etcd's `--trusted-ca-file`)

The API Server and etcd must go through above 2 stages before they exchange sensitive information. They will start exchanging information by:
- API Server sends information to etcd
    - API Server encrypts with etcd's certificate
    - etcd decrypt with etcd's private key
- etcd sends information to API Server
    - etcd encrypts with API Server's certificate
    - API Server decrypt with API Server's private key
