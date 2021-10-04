# Deploying the DNS Cluster Add-on

The purpose of having a DNS add-on is that in Kubernetes you can resolve `service` and `pod` to their IP addresses. The DNS cluster add-on will provision a pod and a service on the cluster, in `kube-system` namespace. Once we configure Kube DNS service, any service will have an A record with format `<service_name>.<namespace>.svc.<clusterDomain>` and any Pod will have an A record with format `<pod_IP>.<namespace>.pod.<clusterDomain>`. 

In our example, we've configured `clusterDomain` to be `cluster.local` in Kubelet, so for example, if we lookup the service `kubernetes` which runs on `default` namespace, we will get A record `kubernetes.default.svc.cluster.local` . If we resolve `kube-dns` service runs on `kube-system` service, we will get an A records `kube-dns.kube-system.svc.cluster.local`

But how does individual container knows the existence of this DNS service? Well, we have told it. In `kubelet-config.yaml` we configured the `clusterDNS` and `clusterDomain` so any new containers will have these DNS server injected to their `/etc/resolv.conf` file.
