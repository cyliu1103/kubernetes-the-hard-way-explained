# Bootstrapping the Kubernetes Control Plane - Part 3 Scheduler and RBAC for Kubelet Authorization

In previous 2 parts, we've covered API server and Controller Manager. We'll discuss Scheduler and RBAC for Kubelet authorization in this part

## Configure the Kubernetes Scheduler
Scheduler assigns Pods to Nodes by determining which Nodes are valid according to Node's constraints and available resources. If multiple Nodes found, Scheduler will rank each Node and binds the Pod to a suitable Node. 

Kubernetes scheduler doesn't need to pass a bunch of parameters. You have 

Let's examine the configuration and service unit of Scheduler

```yaml
apiVersion: kubescheduler.config.k8s.io/v1alpha1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
```
Above kubeconfig is relatively simple. It set kubeconfig file path and enable leader election of Scheduler. The leader election is similar with Controller Manager

Note above is almost the minimal configuration required for Scheduler. There're a number of available options such as stage extension points (which fine grained functions of each scheduling stage), profiles (so different Pods can use different scheduling profiles or policies in `.spec.schedulerName`). More information can be found [here](https://kubernetes.io/docs/reference/scheduling/config/)

Scheduler service unit only contains the configuration file and a log level flag

```bash
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
```
---
## Start the Controller Services

In the end, we need to startup all components of the Control Plane, including API Server, Controller Manager and Scheduler

If there's anything wrong, logs can be checked from `/var/log/syslog` (as we started this tutorial with Ubuntu). The error is likely because of invalid certificate configuration (e.g., mismatch of the SAN field with actual host IPs) or listening/connection address setup.

---
## RBAC for Kubelet Authorisation

At this stage the Kubelet can talk to API Server, because we enabled Node Authorizer in `--authorization-mode` of the API Server. However, API Server still cannot talk to Kubelet. We need to create a `ClusterRole` that enables several node operations, then bind this role to API Server with `ClusterRoleBinding`

The User that binds the ClusterRole should be the one that API Server is used to make request to Kubelet. It uses `--kubelet-client-certificate=kubernetes.pem` certificate, whose CN is `kubernetes`. Therefore, the username should be `kubernetes`
