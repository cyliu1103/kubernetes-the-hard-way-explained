# Bootstrapping the Kubernetes Control Plane - Part 1 API Server
Chapter 8 of this guide focuses on bootstrapping services running on Control Plane. I'll devided into 3 parts, with each part focus on 
- API server
- Controller manager
- Scheduler

I won't go through installations because it's relatively simple. You just need to download from official website, extract the binary files and make them executable. I'll jump to configuration directly.

## Configure the Kubernetes API Server 

The API server needs 3 sets of certificates. 2 of them are same as etcd:

- The server certificate and private key to provide service to clients (`kubernetes.pem` and `kubernetes-key.pem`)
- The CA that API Server can use to authenticate client (`ca.pem`)

The following certificate is specific in API Server:

- The Service Account certificate (`service-account.pem`) in order to authenticate other Service Account's JWT token

Also, API server needs the `encryption-config.yaml`, which contains an encryption key, to encrypt secrets stored in etcd.

Note in the guide, you also transfer `service-account-key.pem` and `ca-key.pem` to the API Server. This is not 100% accurate. The API Server doesn't need these 2 files. Instead, it is the controller manager, which runs on the same machine of the API Server that needs these 2 files. 

I'll explain the API Server service unit and each of the flags

```bash
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://10.0.1.10:2379,https://10.0.1.11:2379,https://10.0.1.12:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --kubelet-https=true \\
  --runtime-config='api/all=true' \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
```
- `--advertise-address`: the IP address to advertise to the members (i.e., other API Server member) of the cluster. Because this is the advertise address, it has to be reachable on other members so it's the IP address of the EC2 instance. 

- `--allow-privileged`: if set to true, allow privileged containers. Note a privileged container has the same resource limit as host machine (e.g., share same IPC, same network namespaces). Generally, it may not be required to run application containers in most cases, but is required if you run kube-proxy containers, for example.

- `--apiserver-count`: Specifies how many of API Server members is running in this cluster. Note this option needs to match across all API Servers in the cluster. In a scenario where you need to add a extra API Server instance in the cluster, spin up the new one with `apiserver-count` increased by 1, then modify existing API Servers `apiserver-count` and restart them

- `--audit-log-maxage=30` `--audit-log-maxbackup=3` `--audit-log-maxsize=100` `--audit-log-path` these are a number of audit log parameters. The API Server will create audit file under what specified in `--audit-log-path`. When hit max size in `--audit-log-maxsize`, API Server will rotate it and create a new log file. When API Server create new log file, it checks if there's any old log file that is older than `--audit-log-maxage` of days and delete the old logs.

- `--authorization-mode`: Specifies a list of authorization plugins. The allowed options are 
  - `AlwaysAllow`: allows all API requests, i.e., do not require authorization for API requests
  - `AlwaysDeny`: blocks all API requests
  - `ABAC`: Attribute-Based Access Control (ABAC) mode
  - `RBAC`: Role-based access control
  - `Node`: Node authorizer, which is a special authorization mode for kubelets' API requests 
  - `Webhook`: an HTTP callback mode that allows you to manage authorization using a remote REST endpoint. So effectively, Kubernetes API Server will query an external REST endpoint to decide if authorization is successful or not

- `--bind-address` is the IP address on which to listen for the connections port

- `--client-ca-file` when set, any requests with a client certificate signed by one of the CA in `--client-ca-file` is authenticated with the CN of the client certificate

- `--enable-admission-plugins` a list of admission controller plugins to enable. More detailed information can be found from [official doc](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/). Also, see [RedHat's Kubernetes admission controller best practices](https://cloud.redhat.com/blog/11-kubernetes-admission-controller-best-practices-for-security) about what should be enabled by default
    - `NamespaceLifecycle` ensures a `Namespace` that is terminating cannot have new objects created in it
    - `NodeRestriction` limits the `Node` and `Pod` objects a kubelet can modify. Remember we said in the Kubelete client certificate section that "Once authorised, the Node authoriser allows kubelet to perform a list of pre-defined API operations.". The pre-defined API operations is defined here. For detailed operations, check official doc. 
    - `LimitRanger` once enabled, it will observe any incoming request and ensure it doesn't violate any constraints in the `LimitRange` object in a `Namespace`. For example, with it enabled, you can enforce a default memory requests and limits for a namespace by creating a `LimitRanger` object. 
    - `ServiceAccount` implements automation for serviceAccounts. It's strongly recommend to turn on. With it turned on, when a Pod is created or modified, it does a series of actions to take care of the Service Account - Pods association. These actions can be found [here](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/#serviceaccount-admission-controller) 
    - `DefaultStorageClass` deals with `PersistentVolumeClaim`. When a `PersistentVolumeClaim` is created and no specific storage class it required, it will automatically adds a default storage class to them. This is also known as Dynamic provisioning. See [here](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#dynamic) for more information
    - `ResourceQuota` is similar with `LimitRanger` that deals with resources. The difference of `LimitRanger` and `ResourceQuota` is that, `LimitRanger` set limit for individual Pods and Containers whereas `ResourceQuota` sets limit of the total resource consumption of a namespace.

- `--etcd-cafile` `--etcd-certfile` `--etcd-keyfile` `--etcd-servers` all about the secure connection to etcd cluster. Note the list of strings in `--etcd-servers`. They're individual etcd server IP addresses. In reality, you may have a Load Balancer in front of the etcd cluster for. Then you can set `--etcd-servers=$LOAD_BALANCER_IP_ADDRESSS:2379` instead of the IP of individual etcd members

- `--event-ttl` specifies the amount of time to retain events. For example, one of your Pods were restarted but the `Events` when you run `kubectl describe pod pod-id` is `<none>`. You may want to increase the ttl to retain more events data.

- `--encryption-provider-config` specify the configuration for encryption to be used for storing secrets in etcd, which we created in Chapter 06

- `--kubelet-certificate-authority` `--kubelet-client-certificate` `--kubelet-client-key` `--kubelet-https` are configuration flags used by API Server (as client) to talk to kubelet (as server)

- `--runtime-config` specifies runtime versions. For example, `api/all=true` means you can use all versions of build-in APIs. Another example is if you set `api/ga=true` the API version has to be v[0-9]+. API versions such as `v1-beta1` is not allowed

- `--service-account-key-file` specifies the key file used to verify ServiceAccount tokens

- `--service-cluster-ip-range` and `--service-node-port-range` specifies an IP and port range from which to assign service cluster IPs. This is the IP range that a `Service` object will get `clusterIP` from. The `clusterIP` is one of the `ServiceTypes` . Check [here](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types) for more information

- `--tls-cert-file` and `--tls-private-key-file` are certificate and private keys that API Server used to provide service (i.e., Server certificate and key). In other words, if client connects to the API Server, the API Server will use `--tls-cert-file` to prove that it is the legitimate server

- `--v` log level
