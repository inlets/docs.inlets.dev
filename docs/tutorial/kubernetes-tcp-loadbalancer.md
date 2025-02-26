# Create a public TCP LoadBalancer in Kubernetes

There are a number of ways to run inlets clients from within Kubernetes:

1. Run the inlets-pro client as a Deployment, after running `inlets-pro tcp client --generate=k8s_yaml`
2. Install the inlets-pro client via Helm
3. Use the inlets-operator to watch for LoadBalancer services and create pairs of tunnel servers and clients

This tutorial shows how to create a public LoadBalancer in Kubernetes using the inlets-operator.

The [inlets-operator](https://github.com/inlets/inlets-operator) is an open-source Kubernetes operator that watches for LoadBalancer services, then creates the tunnel VM using the same approach as [inletsctl](/docs/reference/inletsctl). After the VM is booted up and ready for connections, the operator will create a tunnel client, and update the service's public IP on the Kubernetes service.

## Pre-requisites

* A Kubernetes cluster
* Helm

## Install the inlets-operator

There are several cloud providers supported, so [use the reference guide to install the chart](/docs/reference/inlets-operator.md).

## Create a service and expose it as a LoadBalancer

Of course, you will already have your own applications that you want to expose. You won't tend to want to expose a HTTP endpoint from a container directly, but through an Ingress Controller or Istio Gateway.

As a sample, we can run Nginx as a Pod and then create a Service to expose it as a LoadBalancer only using kubectl.

```bash
# Run Nginx in the background in the default namespace
kubectl run nginx-1 --image=nginx:latest --restart=Always --port=80 --labels app=nginx

# Expose the Nginx service as a LoadBalancer
kubectl expose deployment nginx-1 --port=80 --type=LoadBalancer
```

## Find the LoadBalancer IP

There are two ways to find the LoadBalancer IP.

1. Use the CRD

```bash
$ kubectl get tunnels -w
NAMESPACE   NAME             SERVICE   HOSTSTATUS   HOSTIP        CREATED
default     nginx-1-tunnel   nginx-1   active       46.101.1.67   2m45s
```

2. Use the LoadBalancer service

$ kubectl get svc -n default
NAME         TYPE           CLUSTER-IP    EXTERNAL-IP               PORT(S)        AGE
kubernetes   ClusterIP      10.96.0.1     <none>                    443/TCP        6m26s
nginx-1      LoadBalancer   10.96.94.18   46.101.1.67               80:31194/TCP   4m21s
```

You can then access the Nginx service using the LoadBalancer IP.

```bash
curl http://46.101.1.67
```

## Delete the tunnel server

In order to delete the tunnel server, you need to delete the LoadBalancer service.

```bash
kubectl delete svc nginx-1
```

## Co-existing with other LoadBalancers

If you're running metal-lb or kube-vip to provide local IP addresses for LoadBalancer services, then you can annotate the services you wish to expose to the Internet with `operator.inlets.dev/manage=1`, then set `annotatedOnly: true` in the inlets-operator Helm chart.

i.e.

```bash
helm install inlets-operator inlets/inlets-operator --set annotatedOnly=true
```


```bash
kubectl annotate svc nginx-1 operator.inlets.dev/manage=1
```

