# Bootstrapping the Kubernetes Worker Nodes - Part 5 Kube-proxy
Kube-proxy is responsible to communicate with the master node and do the routing. It is a daemon running on each worker node. 

People sometimes get confused with kube-proxy and CNI. The biggest difference is that, Kube-proxy deals with service load balancing and routing to backed Pods via ClusterIP or NodePort, whereas CNI is responsible for a lower level - it sets up IP addresses (an overlay network that enables Pod communicating) and iptables rules to Pods. Image you have a service with one Pod. Kube-proxy is responsible to setup iptables rules so any request to the service's ClusterIP or NodePort can be forwarded to backed Pod; then CNI is responsible to make sure that on the host machine that the Pod is running, there's an iptables rule that can forward traffic to network bridge which has the Pods connect to it. 

Similar with Kubelet, Kubernetes Proxy also requires a `kubeconfig` file to connect to API Server. Besides, it requires a kube-proxy-config.yaml file that configure kube-proxy itself

```yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
```
There's only one configuration that we need to explain which is `mode`. `mode` specifies how to actually handle the connection forwarding. Kube-proxy can execute three mode: userspace, iptables and IPVS. 

### Userspace

Userspace mode is the legacy mode and it's not recommended anymore. 

### Iptables

In iptables mode, each time a service is modified or the underlying Pods changed (i.e., the endpoints are modified), kube-proxy will update iptables rules on each node of the cluster to reflect the change. The iptables mode is the default mode of kube-proxy. It's simple and efficiency. However, since iptables mode utilise the iptables chain, and the chain can become extremely large if number of service is huge, it can cost lots of CPU resources. 

Image we have Pod A with IP `172.17.0.5` wants to access service ClusterIP `10.110.175.126` which has 3 Pods backed. Their IPs are `172.17.0.5`, `172.17.0.6`, `172.17.0.7`

Let's examine the iptables rule for this traffic.

(1) It will firstly match OUTPUT chain (this is the default behaviour for all outbound traffic). It will match first rule of OUTPUT chain, which will be target `KUBE-SERVICES` chain
```bash
# iptables -t nat -L OUTPUT
Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
KUBE-SERVICES  all  --  anywhere             anywhere             /* kubernetes service portals */
DOCKER_OUTPUT  all  --  anywhere             host.minikube.internal
DOCKER     all  --  anywhere            !127.0.0.0/8          ADDRTYPE match dst-type LOCAL
```
(2) In `KUBE-SERVICES` chain, the target is `10.110.175.126` which will be targeted to `KUBE-SVC-V2OKYYMBY3REGZOG` chain

```bash

# iptables -t nat -L KUBE-SERVICES
Chain KUBE-SERVICES (2 references)
target     prot opt source               destination
KUBE-MARK-MASQ  tcp  -- !10.244.0.0/16        10.96.0.10           /* kube-system/kube-dns:metrics cluster IP */ tcp dpt:9153
KUBE-SVC-JD5MR3NA4I4DYORP  tcp  --  anywhere             10.96.0.10           /* kube-system/kube-dns:metrics cluster IP */ tcp dpt:9153
KUBE-MARK-MASQ  tcp  -- !10.244.0.0/16        10.110.175.126       /* default/nginx-service cluster IP */ tcp dpt:http
KUBE-SVC-V2OKYYMBY3REGZOG  tcp  --  anywhere             10.110.175.126       /* default/nginx-service cluster IP */ tcp dpt:http
KUBE-MARK-MASQ  tcp  -- !10.244.0.0/16        10.109.214.118       /* kube-system/metrics-server:https cluster IP */ tcp dpt:https
KUBE-SVC-Z4ANX4WAEWEBLCTM  tcp  --  anywhere             10.109.214.118       /* kube-system/metrics-server:https cluster IP */ tcp dpt:https
KUBE-MARK-MASQ  tcp  -- !10.244.0.0/16        10.96.0.1            /* default/kubernetes:https cluster IP */ tcp dpt:https
KUBE-SVC-NPX46M4PTMTKRN6Y  tcp  --  anywhere             10.96.0.1            /* default/kubernetes:https cluster IP */ tcp dpt:https
KUBE-MARK-MASQ  udp  -- !10.244.0.0/16        10.96.0.10           /* kube-system/kube-dns:dns cluster IP */ udp dpt:domain
KUBE-SVC-TCOU7JCQXEZGVUNU  udp  --  anywhere             10.96.0.10           /* kube-system/kube-dns:dns cluster IP */ udp dpt:domain
KUBE-MARK-MASQ  tcp  -- !10.244.0.0/16        10.96.0.10           /* kube-system/kube-dns:dns-tcp cluster IP */ tcp dpt:domain
KUBE-SVC-ERIFXISQEP7F7OF4  tcp  --  anywhere             10.96.0.10           /* kube-system/kube-dns:dns-tcp cluster IP */ tcp dpt:domain
KUBE-NODEPORTS  all  --  anywhere             anywhere             /* kubernetes service nodeports; NOTE: this must be the last rule in this chain */ ADDRTYPE match dst-type LOCAL
```

(3) In `KUBE-SVC-V2OKYYMBY3REGZOG` chain, the target is load balanced to 3 target, `KUBE-SEP-C54WIGIB4NQVIFB3`, `KUBE-SEP-KN3IA7DQGTHQJWSD` and `KUBE-SEP-PIN45DA7MSW3VTQG`

```bash
# iptables -t nat -L KUBE-SVC-V2OKYYMBY3REGZOG
Chain KUBE-SVC-V2OKYYMBY3REGZOG (1 references)
target     prot opt source               destination
KUBE-SEP-C54WIGIB4NQVIFB3  all  --  anywhere             anywhere             /* default/nginx-service */ statistic mode random probability 0.33333333349
KUBE-SEP-KN3IA7DQGTHQJWSD  all  --  anywhere             anywhere             /* default/nginx-service */ statistic mode random probability 0.50000000000
KUBE-SEP-PIN45DA7MSW3VTQG  all  --  anywhere             anywhere             /* default/nginx-service */
```

(4) In each of these 3 chains, the rules are similar. Each chain is responsible to forward to each Pod's IP.

```bash
root@minikube:/home/docker# iptables -t nat -L KUBE-SEP-C54WIGIB4NQVIFB3
Chain KUBE-SEP-C54WIGIB4NQVIFB3 (1 references)
target     prot opt source               destination
KUBE-MARK-MASQ  all  --  172.17.0.5           anywhere             /* default/nginx-service */
DNAT       tcp  --  anywhere             anywhere             /* default/nginx-service */ tcp to:172.17.0.5:80
root@minikube:/home/docker# iptables -t nat -L KUBE-SEP-KN3IA7DQGTHQJWSD
Chain KUBE-SEP-KN3IA7DQGTHQJWSD (1 references)
target     prot opt source               destination
KUBE-MARK-MASQ  all  --  172.17.0.6           anywhere             /* default/nginx-service */
DNAT       tcp  --  anywhere             anywhere             /* default/nginx-service */ tcp to:172.17.0.6:80
root@minikube:/home/docker# iptables -t nat -L KUBE-SEP-PIN45DA7MSW3VTQG
Chain KUBE-SEP-PIN45DA7MSW3VTQG (1 references)
target     prot opt source               destination
KUBE-MARK-MASQ  all  --  172.17.0.7           anywhere             /* default/nginx-service */
DNAT       tcp  --  anywhere             anywhere             /* default/nginx-service */ tcp to:172.17.0.7:80
```
If we change the number of Pods to be 4 and examine the last chain, we'll find an extra rule is made by kube-proxy and the load balancing probability adjusted, too

```bash
# iptables -t nat -L KUBE-SVC-V2OKYYMBY3REGZOG
KUBE-SEP-C54WIGIB4NQVIFB3  all  --  anywhere             anywhere             /* default/nginx-service */ statistic mode random probability 0.25000000000
KUBE-SEP-KN3IA7DQGTHQJWSD  all  --  anywhere             anywhere             /* default/nginx-service */ statistic mode random probability 0.33333333349
KUBE-SEP-PIN45DA7MSW3VTQG  all  --  anywhere             anywhere             /* default/nginx-service */ statistic mode random probability 0.50000000000
KUBE-SEP-KD3I42GWNQOKYGXO  all  --  anywhere             anywhere             /* default/nginx-service */
```
### IPVS

IPVS also uses Netfilter, same as iptables. The major difference is that IPVS implements layer-4 load balancing whereas iptables can only work on layer 3

---
Once we setup `kube-proxy-config` YAML file, we can start kube-proxy

```bash
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

---
## Start the Worker Services

Let's wrap up the services running on Worker node

- containerd is the container runtime that will call lower-level `runc` or `runsc` to spin up containers. `runc` is for trusted workload whereas `runsc` is for untrusted
- kubelet is the agent for Kubernetes. It will register nodes to API server and maintain desired Pods status. Kubelet is the essential service and is also the target for intruders. Be mindful of configuring the authentication and authorisation methods
- kube-proxy is the networking component for Kubernetes. It maintains routing.

All of above services need to start on system booting, i.e., `sudo systemctl enable containerd kubelet kube-proxy`