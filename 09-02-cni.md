# Bootstrapping the Kubernetes Worker Nodes - Part 2 CNI

Kubernetes networking is a deep topic that I won't go too much detail here. There's a number of ways to implement Kubernetes networking, either manually, automatically via plugins or even write your own CNI plugin. But netherless, the network inside a Kubernetes cluster must satisfy 2 requirements, as mentioned in [official document](https://kubernetes.io/docs/concepts/cluster-administration/networking/):

>(1) pods on a node can communicate with all pods on all nodes without NAT
>
>(2) agents on a node (e.g. system daemons, kubelet) can communicate with all pods on that node

We use CNI with Bridge plugin in this guide. That means inside a worker node, we create a virtual network bridge (`br0`). When Pod is created, CNI will create a virtual ethernet device (aka veth). The veth is like a patch cable, with one interface (`veth`) connects to the Pod with an IP address assigned by CNI (its IP address range has been specified in CNI configuration file and will be recorded by CNI in a file), the other interface (`eth0`)connects to the network brdige `br0` with a layer 2 MAC address. By doing so, all Pods inside a worker node are in a flat network so they can directly talk with each other via Layer 2 broadcast

What if the Pod needs to talk to another Pod in a difference worker node? Let's image Pod A on Node A wants to send packet to Pod B on Node B. When Node A's `br0` interface receives the packet, it does Layer 2 broadcast, and fails, because there's no match MAC address on this bridge. Then `br0` will send this packet to Node A's root namespace, let's assume it's called `eth0-nodeA` to avoid confusion with `eth0` on Pods. Remember rule number 2? The worker node A should know where the destination Pod is so it forwards packet to node B's `eth0-nodeB` interface. Upon receive the packet, Node B will check its iptables rule (which is setup by CNI) and decide to forward to its bridge, then to Pod B. 

You may wonder how about replying packet from Pod B to A? Well Pods inside a Cluster should have their own unique IP address, and NAT isn't required. So Pod B sending packet back to Pod A is same as Pod A sends to Pod B, it will go through 

```
Pod B's eth0 -> veth -> Node B's br0 -> Node B's eth0-nodeB -> Layer 3 routing devices -> Node A's eth0-nodeA -> iptables match then sends to br0 -> veth -> Pod A's eth0
```
One thing we need to specifically configure is that the Pod CIDR range of the Node. This is required for Pod-to-Pod communication across different Nodes . For detailed explanation, check this [blog](https://sookocheff.com/post/kubernetes/understanding-kubernetes-networking-model/#kubernetes-basic), especially chapter 4.2 Pod-to-Pod, across Nodes. 

The Pod CIDR for each worker node has been set in user data when create these EC2 instances. The Node-Pod IP address mappings are shown in below table
| Node Name | Node IP | Pod CIDR |
| --------- | ------- | -------- | 
| workder-0 | `10.0.1.20` | `10.200.0.0/24` |
| workder-1 | `10.0.1.21` | `10.200.1.0/24` |
| workder-2 | `10.0.1.22` | `10.200.2.0/24` |

2 CNI plugins are configured

### `bridge` plugin

```json
{
    "cniVersion": "0.3.1",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
```

  - `cniVersion` : CNI standard version
  - `name`: the name of the network
  - `type`: the type of the network. Apart from bridge, there're some other options such as `ptp`, `macvlan` etc
  - `bridge`: since we want to create a bridge network, we need to tell the name of this network bridge. This name will show if you run `ifconfig` or `ip addr show`
  - `isGateway`: when set to true, the network bridge will be assigned an IP address, acting as a gateway of containers. This might be a bit hard to understand. Please see below explanation
  - `ipMasq`: set outbound masquerade to Node IP address. This is useful when Pods send traffic to outside the cluster. For example, if the Pod has IP address 10.0.0.2 and it sends traffic to 8.8.8.8, the Node will stripe source IP 10.0.0.2 and replace with its Node IP so the destination know which IP to send traffic back to. Note in CNI, the masquerade is  achieved through iptables rules on host machine. Let's have an example taken from this [article](https://morningspace.github.io/tech/k8s-net-cni/). The Pods CIDR range is 10.15.10.0/24. There's one Pod in this cluster whose IP address is 10.15.10.100/32. When this Pod sends traffic out, the first rule will match, because the source IP is 10.15.10.100/32. Then it enters to rule `CNI-95c521ba458df43df4d8c523`, where the destination also matches so it enters the 3rd rule. In the 3rd rule, the packet destination isn't 224.0.0.0/4 (which is layer 2 broadcaset IP) so it will be masquerade with Node IP address. 
```
$ iptables-save | grep lab-br
-A POSTROUTING -s 10.15.10.100/32 -m comment --comment "name: \"lab-br0\" id: \"lab-ns\"" -j CNI-95c521ba458df43df4d8c523
-A CNI-95c521ba458df43df4d8c523 -d 10.15.10.0/24 -m comment --comment "name: \"lab-br0\" id: \"lab-ns\"" -j ACCEPT
-A CNI-95c521ba458df43df4d8c523 ! -d 224.0.0.0/4 -m comment --comment "name: \"lab-br0\" id: \"lab-ns\"" -j MASQUERADE

```

  - `ipam`: also known as IP Address Management. This configuration provides a set of configurations for IP addressing
      - `type`: the type of the ipam plugin. `host-local` is one option. In `host-local`, we provide the subnet CIDR range and the plugin will assign IP to Pods and make sure the Pods IP doesn't collide. The Pod IP addressing information is saved as a file on Node
      - `ranges`: the CIDR range that the Pods can be assigned to
      - `routes`: set routing table. In this example, we configured a default route: it will route all traffic to Pods gateway, i.e., the bridge. In CNI, the routing table is achieved through iptables rules

### `loopback` plugin

The `loopback` plugin offers a way for multiple containers inside a Pod to talk with each other. There's no many configurations

```bash
cat <<EOF | sudo tee /etc/cni/net.d/99-loopback.conf
{
    "cniVersion": "0.3.1",
    "name": "lo",
    "type": "loopback"
}
EOF
```
