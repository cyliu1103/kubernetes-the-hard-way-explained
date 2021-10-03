# Bootstrapping the etcd Cluster
The etcd provides backend storage for Kubernetes Cluster. In order to run in high availability mode, it is recommended to have 5 members in production (see [official doc](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#:~:text=A%20five%2Dmember%20cluster%20is,clustering%2C%20see%20etcd%20clustering%20documentation.)). Also, since etcd builds on the Raft consensus algorithm to elect leader, it is always a recommended to run etcd as a cluster of odd members. 

etcd will be running on each Controller. I won't go through the downloading and setup steps. Instead, I'll focus on explaining what are required to run an etcd cluster and the flags used in this guide

---
## Certificates and files

The etcd service needs 3 sets of certificates:

- The server certificate and private key to provide service to clients (`kubernetes.pem` and `kubernetes-key.pem`)
- The peert certificate and private key to provide service to other peer members (`kubernetes.pem` and `kubernetes-key.pem`)
- The CA that etcd can use to authenticate client (`ca.pem`)

---
## Configure the etcd Server

Let's have a detailed look at the etcd service unit. The pink highlight items are for server-client certificates whereas the green items are peer certificates.
```bash
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster controller-0=https://10.0.1.10:2380,controller-1=https://10.0.1.11:2380,controller-2=https://10.0.1.12:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
```
- `--name` : a human readable name
- `--cert-file`: the server certificate to provide etcd service
- `--key-file`: the private key of the server certificate
- `--peer-cert-file`: the certificate used between peers
- `--peer-key-file`: the private key of the peer certificate
- `--trusted-ca-file`: trusted CA to authenticate clients
- `--peer-trusted-ca-file`: trusted CA to authenticate peers
- `--peer-client-cert-auth`: etcd will check all incoming peer requests for valid client certificates signed by the supplied CA
- `--client-cert-auth`: etcd will check all incoming client requests for valid client certificates signed by the supplied CA. When this is set, you need `--trusted-ca-file` together; the clients need to pass their certificate in order to connect to etcd
- `--initial-advertise-peer-urls`: it specifies the addresses that etcd clients or other etcd members should use to contact this etcd server. Note this address should be reachable on the remote machines so it has to be the IP address of the EC2 instance, other than `localhost`
- `--listen-peer-urls`: it specifies the address that local etcd server bind to for accepting incoming connections. It listens to `[https://INTERNAL_IP:2380](https://internal_IP:2380)` which means any connection to `INTERNAL_IP:2380` will be picked up by the etcd service.
- `--listen-client-urls`: similar with `listen-peer-urls` except it's the address to listen on for client traffic
- `--advertise-client-urls`: specifies the URLs that etcd member advertise to other members and clients. This address should be reachable from remote machine
- `--initial-cluster-token`: this is the identifier of the etcd cluster. This is useful in multi-cluster environment to prevent etcd members of different cluster from interfering with each other
- `--initial-cluster`: this provides the cluster members addresses. Note this is a static etcd cluster so we know the IP addresses of other etcd members
- `--initial-cluster-state`: setting this to `new` means all members present during initial static bootstrapping. An alternative option is `existing`, where the etcd member will attempt to join an existing cluster
- `--data-dir`: path to the data directory

### A side note of `--client-cert-auth` and `--peer-client-cert-auth`
With `--client-cert-auth` flag set, the etcd member will request clients to provide their certificate. For example
```bash
# etcd member run with flags
$ etcd --name demo --client-cert-auth \
--trusted-ca-file=ca.crt \
--cert-file=server.crt \
--key-file=server.key \
--advertise-client-urls https://127.0.0.1:2379 \
--listen-client-urls https://127.0.0.1:2379

# Client with following request will fail
$ curl --cacert ca.crt https://127.0.0.1:2379/v2/keys/foo 

# Client with following request will succeed
$ curl --cacert ca.crt --cert client.crt --key client.key https://127.0.0.1:2379/v2/keys/foo 
```

### A side note of why peer-cert-file and cert-file are the same in our example
Don't be confused of the purpose of having `peer-cert-file` and `cert-file`. Although in this example they're the same (`kubernetes.pem`), they serve different purposes

The `peer-cert-file` is to authenticate peer members of a etcd cluster whereas the `cert-file` is to authenticate incoming requests from clients. They can be the same certificate, like our example illustrates. But it's recommended to have separate certificates. 

In our example, the peer certificate `kubernetes.pem` has a list of `hosts` that includes each etcd member's actual IP address. This is crucial because otherwise the peer connection with `kubernetes.pem` certificate will fail. This is a TLS authentication mechanism and can be further explained in [official docs](https://etcd.io/docs/v3.4/op-guide/security/#notes-for-host-whitelist). But here's a brief summary.
- When establish peer connections, ectd will deny incoming peer certs with wrong IP address in SAN field. For example, if peer A (IP address is 10.0.0.1) wants to establish connection to peer B and peer A's certificate host field is

```yaml
hosts: ["10.0.0.2", "peer-a.default.svc"]
```
Peer B will reject it, because the IP field of A's peer certificate (`10.0.0.2`) doesn't match its actual IP (`10.0.0.1`)

- However, if A's peer certificate only contains DNS name, no IPs, B will lookup its DNS name. Consider if peer A's IP is `10.0.0.1` and its certificate has host field, Peer B will resolve peer-a.default.svc and a.com see if 10.0.0.1 matches any of the IP resolution. If it matches, B will allow A's connection.
```yaml
hosts: ["peer-a.default.svc", "a.com"]
```
---
## Verification

We use `etcdctl` command line to verify the etcd cluster status. Note this client tool also use the cacert and certificate to authenticate the etcd cluster.