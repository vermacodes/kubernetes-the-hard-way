# Installing CoreDNS and Metrics Server

In this lab you will install CoreDNS for cluster DNS resolution and metrics-server for resource metrics collection. These are essential addons for a fully functional Kubernetes cluster.

## Prerequisites

The commands in this lab should be run from the `jumpbox` machine.

Verify your cluster is functioning correctly:

```bash
kubectl get nodes
```

```text
NAME     STATUS   ROLES    AGE   VERSION
node-0   Ready    <none>   30m   v1.32.3
node-1   Ready    <none>   30m   v1.32.3
node-2   Ready    <none>   30m   v1.32.3
```

## Installing CoreDNS

CoreDNS is the default DNS server for Kubernetes. It provides service discovery and name resolution within the cluster.

### Deploy CoreDNS

Install CoreDNS using Helm:

```bash
helm repo add coredns https://coredns.github.io/helm
helm repo update
```

```bash
helm install coredns coredns/coredns \
  --namespace kube-system \
  --set service.clusterIP=10.0.0.10
```

> Note: The Helm chart automatically creates the necessary RBAC resources (ClusterRole, ClusterRoleBinding) for CoreDNS to function properly.

### Add Node Hostnames to CoreDNS

To allow metrics-server and other services to resolve node hostnames, add node entries to CoreDNS:

```bash
kubectl edit configmap coredns -n kube-system
```

Add the `hosts` block to the Corefile (adjust IPs to match your `machines.txt`):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        hosts {
           192.168.0.200 node-0 node-0.kubernetes.local
           192.168.0.201 node-1 node-1.kubernetes.local
           192.168.0.202 node-2 node-2.kubernetes.local
           fallthrough
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
```

Restart CoreDNS to apply the configuration:

```bash
kubectl rollout restart deployment coredns -n kube-system
```

### Verification

Wait for CoreDNS pods to be ready:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

```text
NAME                       READY   STATUS    RESTARTS   AGE
coredns-678d788f77-h2dtm   1/1     Running   0          2m
coredns-678d788f77-x9k4p   1/1     Running   0          2m
```

Test DNS resolution:

```bash
kubectl run test-dns --image=busybox:1.28 --rm -it --restart=Never -- nslookup kubernetes
```

```text
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.0.0.1 kubernetes.default.svc.cluster.local
```

## Installing Metrics Server

Metrics Server collects resource metrics from kubelets and exposes them via the Metrics API for use by Horizontal Pod Autoscaler and `kubectl top` commands.

### Deploy Metrics Server

Install metrics-server using Helm:

```bash
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm repo update
```

```bash
helm install metrics-server metrics-server/metrics-server \
  --namespace kube-system \
  --set args[0]="--kubelet-preferred-address-types=Hostname,InternalDNS,InternalIP"
```

### Verify Metrics Server

Wait for the metrics-server pod to be ready:

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=metrics-server
```

```text
NAME                              READY   STATUS    RESTARTS   AGE
metrics-server-5d4dcd5b6f-8xkzm   1/1     Running   0          30s
```

Check the APIService status:

```bash
kubectl get apiservice v1beta1.metrics.k8s.io
```

```text
NAME                     SERVICE                      AVAILABLE   AGE
v1beta1.metrics.k8s.io   kube-system/metrics-server   True        1m
```

Test metrics collection:

```bash
kubectl top nodes
```

```text
NAME     CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
node-0   156m         3%     1204Mi          15%
node-1   98m          2%     987Mi           12%
node-2   112m         2%     1043Mi          13%
```

```bash
kubectl top pods -A
```

```text
NAMESPACE     NAME                              CPU(cores)   MEMORY(bytes)
kube-system   coredns-678d788f77-h2dtm          3m           12Mi
kube-system   coredns-678d788f77-x9k4p          3m           11Mi
kube-system   metrics-server-5d4dcd5b6f-8xkzm   5m           18Mi
```

## Troubleshooting

### CoreDNS Not Starting

If CoreDNS pods fail to start with authentication errors, ensure:

1. The API server certificate includes the service ClusterIP (10.0.0.1 and 10.0.0.10)
2. The Helm chart created RBAC resources correctly
3. Kubelet has been configured with clusterDNS settings

### Metrics Server Connection Issues

If metrics-server cannot reach kubelets:

1. Verify node hostnames are resolvable via CoreDNS
2. Check that kubelet certificates include node hostnames or IPs
3. Ensure the metrics-server can connect using the configured address type

Check metrics-server logs for details:

```bash
kubectl logs -n kube-system deployment/metrics-server
```

Next: [Cleaning Up](14-cleanup.md)

