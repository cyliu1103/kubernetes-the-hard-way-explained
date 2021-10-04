# Bootstrapping the Kubernetes Worker Nodes - Part 4 Kubelet
Kubelet runs as an agent on each worker node. Kubelet pulls Pod states from API Server, and maintain the desired state by talking to container runtime, containerd. Therefore, we should] be able to see following components in Kubelet configurations

- How to Authenticate and authorise incoming requests and run Kubelet service (`kubelet-config.yaml`)
- Connection parameters to API Server (`kubeconfig` file)
- Connection to the container runtime (`--container-runtime` and `--container-runtime-endpoint` flags)

Let's examine the `kubelet-config.yaml` file
```yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "${POD_CIDR}"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/${WORKER_NAME}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/${WORKER_NAME}-key.pem"
resolvConf: "/run/systemd/resolve/resolv.conf"
```
- `authentication` specifies how requests to the Kubelet's service are authenticated. These requests for example, can be API Server requests to kubelet. There're 3 authentication methods, `x509`, `webhook` and `anonymous` In this configuration, `x509` and `webhook` are enabled, meaning clients can use either of these PKI certificate or bearer token to authenticate. Here, in this configuration, `anonymous` is turned off, meaning any requests from `system:anonymous` username or `system:unauthenticated` group will be rejected with `401 Unauthorized`

Now the question is, what exactly is `system:anonymous` or `system:unauthenticated` Why do we need to disable them? From official [document](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet-authentication-authorization/), 
    
    > By default, requests to the kubelet's HTTPS endpoint that are not rejected by other configured authentication methods are treated as anonymous requests
What this means is that, if the kubelet is configured with `webhook` and `x509` authentication method, and a client passed wrong bearer token, it will be rejected directly with `401 Unauthorized`. However, if the client doesn't provide any token or certificate, it will not be rejected; instead, it will be marked as an anonymous request. It sounds reasonable to enable anonymous, isn't it? But it is not. Go ahead and read `authorization` part. 

- `authorization` specifies authorisation mode. The default is `AlwaysAllow`, meaning allows all requests including anonymous requests. Alternatively, use `Webhook` mode. If `Webhook` mode is enabled, Kubelet will use `SubjectAccessReview` API to determine if client user or group can perform an action.

Now if we combine authentication and authorization, we can see that by default (i.e., without `anonymous.enabled: false`), things can go seriously wrong. Any requests, as long as they're not explicitly rejected, will be treated as anonymous, and anonymous requests are `AlwaysAllow`. This is certainly insecure. There's even a [blog](https://www.cyberark.com/resources/threat-research-blog/using-kubelet-client-to-attack-the-kubernetes-cluster) that shows you how to use this default kubelet configuration to attack the Cluster.  Therefore, it is crucial to ensure anonymous is not authenticated, and even it passed authentication, it cannot do anything. 
- `clusterDomain` is the DNS domain for this Kubernetes cluster. Kubelet will configure all containers to search this domain. This is achieved by inserting records to `/etc/resolv.conf` of each container

- `clusterDNS` is a list of DNS server IP addresses. Kubelet will configure containers DNS server to this address instead of Node's DNS servers
- `podCIDR` specifies the IP address range that will be assigned to Pod.

- `runtimeRequestTimeout` controls timeout for runtime requests. The default value is 2m. Setting this value larger is useful, for example, if a image is too big so it takes over 2m to pull

- `tlsCertFile` and `tlsPrivateKeyFile` are TLS certificate and key file for kubelet service. Kubelet runs a HTTPS service endpoint

- `resolvConf` is the base container DNS resolution configuration. It copies worker Node's DNS resolution file and make adjustments based on `clusterDNS` and `clusterDomain` settings

We have created `kubeconfig` file previously. It contains information about TLS certificate that needs to used to connect API Server and API Server address etc. 

The service unit of kubelet is configured as below. It passed `--config` and `--kubeconfig` as flags. Also, it specifies container runtime and the sock file used by containerd. Note `--container-runtime=remote` This flag has 2 options, `docker` and `remote`

Another note is the `--image-pull-progress-deadline=2m` flag. According to official [document](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/#:~:text=--image-pull-progress-deadline), this should only works when container-runtime is set to `docker` so I think this option is not valid in `containerd` context
```shell
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```
