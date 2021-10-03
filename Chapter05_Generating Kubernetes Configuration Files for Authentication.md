# Generating Kubernetes Configuration Files for Authentication

## Client Authentication Configs

As mentioned in previous chapter, the configuration file (i.e. `kubeconfig`) for kube-proxy, kube-controller-manager, kube-scheduler and kubelet will have certificates embedded. So we need to generate these configuration files

### Kubernetes Public DNS Address

We have provisioned an ALB in [Chapter 03](https://github.com/prabhatsharma/kubernetes-the-hard-way-aws/blob/master/docs/03-compute-resources.md#kubernetes-public-access---create-a-network-load-balancer). Below command registered a target group of EC2 with IP addresses 10.0.1.10, 10.0.1.11, 10.0.1.12, which are 3 controller EC2 instances
```shell
aws elbv2 register-targets --target-group-arn ${TARGET_GROUP_ARN} --targets Id=10.0.1.1{0,1,2}
```
The public DNS address is the DNS of the load balancer. It enables high availability in the case of any single Controller failure

### The kubelet Kubernetes Configuration File

The KTHW guide has a long command. The main command is `kubectl`. You can find the reference doc from [here](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands). Let's go through each of them
```shell
for instance in worker-0 worker-1 worker-2; do
# First command
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:443 \
    --kubeconfig=${instance}.kubeconfig

# Second command
  kubectl config set-credentials system:node:${instance} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

# Third command
  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${instance} \
    --kubeconfig=${instance}.kubeconfig

# Forth command
  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done
```
It is a loop of worker-0, worker-1 and worker-2We will use `instance=worker-0` for illustration. Also let's assume `KUBERNETES_PUBLIC_ADDRESS=kthw-explained.com`
- First command: `set-cluster` sets a cluster entry in `kubeconfig`

```shell
kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:443 \
	--kubeconfig=${instance}.kubeconfig
```
Above command tells `kubectl` to generate a `worker-0.kubeconfig` file from scratch, with some details:

- Set the cluster entry name to be `kubernetes-the-hard-way`. This can be an arbitrary name
- The CA file is `ca.pem` , as generated in previous chapter
- Embed certificate authority data of the cluster
- Specifies the server address as `kthw-explained.com:443`

After this step, we should have a `worker-0.kubeconfig` file with below content
```yaml
# worker-0.kubeconfig after step 1
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <ca.pem content here>   # --certificate-authority=ca.pem --embed-certs=true
	server: https://kthw-explained.com:443  # --server=https://${KUBERNETES_PUBLIC_ADDRESS}:443
  name: kubernetes-the-hard-way # set-cluster kubernetes-the-hard-way
contexts: null
current-context: ""
kind: Config
preferences: {}
users: null
```
- Second command: `set-credentials` sets a user entry in `kubeconfig`

```bash
kubectl config set-credentials system:node:${instance} \
  --client-certificate=${instance}.pem \
  --client-key=${instance}-key.pem \
  --embed-certs=true \
  --kubeconfig=${instance}.kubeconfig
```
Above commands takes the `worker-0.kubeconfig` generated in previous command and adds some user data:

  - Use `worker-0.pem` and `worker-0-key.pem` (generated in previous chapter, in kubelet section) as client certificates and key
  - Embed certificates and secrets to `kubeconfig` file

After this step, the `worker-0.kubeconfig` file should look like this. 
```yaml
# worker-0.kubeconfig after step 2
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <ca.pem content here> 
	server: https://kthw-explained.com:443
  name: kubernetes-the-hard-way
contexts: null
current-context: ""
kind: Config
preferences: {}
users:
- name: system:node:worker1   # set-credentials system:node:${instance}
  user:
    client-certificate-data: <worker-0.pem content here>  # --client-certificate=${instance}.pem --embed-certs=true
		client-key-data: <worker-0-key.pem content here>  # --client-key=${instance}-key.pem --embed-certs=true
```
- Third command: set-context sets a kubeconfig context
```bash
# Third command
kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:node:${instance} \
  --kubeconfig=${instance}.kubeconfig
```
```yaml
# worker-0.kubeconfig after step 3
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <ca.pem content here> 
	server: https://kthw-explained.com:443
  name: kubernetes-the-hard-way
contexts:
- context:  # set-context default
    cluster: kubernetes-the-hard-way
    user: system:node:worker1
  name: default
current-context: ""
kind: Config
preferences: {}
users:
- name: system:node:worker1
  user:
    client-certificate-data: <worker-0.pem content here>
		client-key-data: <worker-0-key.pem content here>
```
- Forth command: `use-context` sets the current context in a kubeconfig file.
```bash
# Forth command
kubectl config use-context default --kubeconfig=${instance}.kubeconfig
```
A `kubeconfig` file can have multiple contexts whereas there's only 1 activated, specifying in `current-context` entry. The green highlight entry is added in this step. The name, `default` refers to the name of `context`
```yaml
# worker-0.kubeconfig after step 4
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <ca.pem content here> 
	server: https://kthw-explained.com:443
  name: kubernetes-the-hard-way
contexts:
- context:
    cluster: kubernetes-the-hard-way
    user: system:node:worker1
  name: default
current-context: default  # use-context default
kind: Config
preferences: {}
users:
- name: system:node:worker1
  user:
    client-certificate-data: <worker-0.pem content here>
	client-key-data: <worker-0-key.pem content here>
```
One caveat is that, in order to be authorized by the Node Authorizer, the kubelet kubeconfig file must use `system:node:<nodeName>` as a username.

---
### The kube-proxy Kubernetes Configuration File

Similar with kubelet configuration file, kube-proxy configuration file also needs to embed server address, CA, client certificate and private key. The only difference is that you don't need to generate differenct kube-proxy configurations for each worker nodes. All kube-proxy services running on different nodes should run under same configuration.

The `kubeconfig` setups of Controller Manager, Scheduler and Admin user are the same. Note the `--server=https://127.0.0.1:6443` this is the API Server address and port

---
## Distribute the Kubernetes Configuration Files

In the end of this chapter, kubelet and kube-proxy `kubeconfig` files are distributed to Worker Nodes whereas Controller Manager and Scheduler `kubeconfig` files are copied to Controller Nodes. Copying Admin `kubeconfig` files to Controller Nodes are actually optional. However, you'll need the admin `kubeconfig` to run some test in later chapters
