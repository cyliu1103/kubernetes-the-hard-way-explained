# Configure Kubectl for Remote Access

Nothing special to mention in this chapter. We have configured a `admin` user previously. This chapter configures `kubectl` so we can use `admin` user to interact with Kubernetes cluster

If we have a look at an example `kubectl` configuration `kubeconfig` file, we'll find that effectively it requires some information 

(1) The server URL, in our case, it's the DNS of load balancer

(2) The certificate authority data, i.e., the `ca.pem` file content

(3) The client information (i.e., to prove that I'm `admin` user). This has a number of way to achieve. In this guide, it's using client certificate (`admin.pem` and `amin-key.pem`). If you're using AWS, you can also use AWS IAM

The functionality of `kubectl config set-cluster`, `kubectl config set-credentials`, etc. is similar with kubelet configuration file in previous chapter 5 so I won't go through them.
