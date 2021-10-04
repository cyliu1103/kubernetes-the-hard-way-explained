# Bootstrapping the Kubernetes Worker Nodes - Part 3 Containerd

`containerd` is the high-level container runtime. We need to install it on each Node so it can run Pods. In this guide, we use `containerd` as the high-level container runtime. You can optionally use `CRI-O`, for example

Note the difference between `containerd` and `runc`. This [article](https://www.tutorialworks.com/difference-docker-containerd-runc-crio-oci/) explains quite well. Containerd is higher level runtime whereas runc is lower level runtime. Runc actually spawn and runs container. When Kubernetes wants to run a container, it talks high-level container runtime (such as containerd or CRI-O) via Container Runtime Interface (CRI). Containerd does not create containers by itself; rather, it talks to containerd-shim, which will then call runc (an OCI compatible, lower-level runtime) and create containers.

Let's have a look the configuration

```jsx
cat << EOF | sudo tee /etc/containerd/config.toml
[plugins]
  [plugins.cri.containerd]
    snapshotter = "overlayfs"
    [plugins.cri.containerd.default_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runc"
      runtime_root = ""
    [plugins.cri.containerd.untrusted_workload_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runsc"
      runtime_root = "/run/containerd/runsc"
EOF
```

- Containerd has a number of [plugins](https://github.com/containerd/containerd/blob/main/docs/PLUGINS.md). If you run `ctr plugin ls` on Worker node you'll find them. To configure these plugins, you need to use `[plugins.<plugin_id>]` under `[plugins]` in containerd's configuration file. In this guide, we mainly configured the CRI plugin. The whole configuration can be found [here](https://github.com/containerd/containerd/blob/main/docs/cri/config.md)
- `snapshotter = "overlayfs"`: Snapshotter is the way that containerd stores and extracts container layers and provide rootfs. The benefits of enabling overlayfs is discussed later in this part.
- `default_runtime` and `untrusted_workload_runtime` fields
    
    When create a Pod, if you add `io.kubernetes.cri.untrusted-workload: "true"` in annotations, the Pod will be executed using untrusted workload runtime. In our case, we use `runsc` (gVisor) as default runtime for untrusted workload
    
    The main difference between `runsc` and `runc` is that `runc` shares host kernel whereas `runsc` provisions a separate application kernel for each container. 
    

Once containerd configuration is ready, we need a service unit to start containerd

```bash
cat <<EOF | sudo tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
EOF
```

One thing to noticed is that `ExecStartPre=/sbin/modprobe overlay` This is the command runs before `ExecStart` . What it does is the enable kernel module overlay that will be used by containerd (see kernel [document](https://www.kernel.org/doc/html/latest/filesystems/overlayfs.html)). Why we need overlay? Because it can save a lot of disk space. For example, if you have a number of containers use Ubuntu base image, copying this base images to all containers will use a lot of disk space. However, with overlay, the base OS (Ubuntu base image) directory and container-specific directory will be merged using kernal's overlay driver. During overlay merge, the base OS directory will become a read-only lower layer and container's directory becomes a write layer. By doing so, containers can share the same base OS layer. Overlay is a kernel driver so you need to enable it first.
