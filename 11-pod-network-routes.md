# Pod Network Routes

With the CNI we installed, we managed to let Pods inside a Node to communicate with each other via bridge network. 

In [Configure CNI networking](https://www.notion.so/Kubernetes-the-Hard-Way-Explained-10963c4400764007be2cab02311add6b), we mentioned that an CNI should enable:

>(1) pods on a node can communicate with all pods on all nodes without NAT
>
>(2) agents on a node (e.g. system daemons, kubelet) can communicate with all pods on that node

Currently, Pods cannot access other Pods in a different Node, because Node has no routing table to Pods on another Node. We need to configure routing table to do so. 

Note this is not the best practice in production. Because in Production, you won't manually setup routing table if a new node or new Pod join in Cluster. In Production, you'd rather use some CNI plugins such as Amazon VPC CNI which will handles routing tables in your VPC to ensure that the newly added Pods or Nodes are in routing table. 

Given that we have 3 worker nodes and each of them have a dedicated Pods CIDR, we need to add to VPC routing table

(1) If destination is 10.200.0.0/24, forward to 10.0.1.20

(2) If destination is 10.200.1.0/24, forward to 10.0.1.21

(3) If destination is 10.200.2.0/24, forward to 10.0.1.22
