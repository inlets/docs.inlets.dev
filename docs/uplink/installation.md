# Install inlets-uplink

Inlets Uplink is a self-hosted solution for managing many tunnels for a team or company.

Learn why we created Inlets Uplink and how it might be a good fit for you in the announcement blog post:

- [Inlets Uplink for SaaS & Service Providers](https://inlets.dev/blog/2022/11/16/service-provider-uplinks.html)

## Pre-reqs
* A Kubernetes cluster with LoadBalancer capabilities (i.e. public cloud).
* A domain name clients can use to connect to the tunnel control plane.
* [arkade](https://github.com/alexellis/arkade), a simple CLI tool that provides a quick way to install various apps and download common binaries much quicker
    
    To install arkade run:

    ```bash
    curl -sSLf https://get.arkade.dev/ | sudo sh
    ```

## Create a Kubernetes cluster

Create a cluster with at least 3 nodes 2-4GB of RAM each

## Install cert-manager

Install [cert-manager](https://cert-manager.io/docs/), which is used to manage TLS certificates for inlets-uplink.
```bash
arkade install cert-manager
```

## Deploy inlets-uplink

Make sure to create the target namespace for you installation first.

```bash
kubectl create namespace inlets
```

Create the required secret with your inlets-uplink license.

```bash
kubectl create secret generic \
  -n inlets inlets-uplink-license \
  --from-file license=$HOME/.inlets/LICENSE_UPLINK
```

There are two options for deploying inlets-uplink. Option A uses Kubernetes Ingress. With option B Ingress is handled by Istio.

### A) Install with Kubernetes Ingress

Install nginx-ingress using arkade:

```bash
arkade install ingress-nginx
```

Create a `values.yaml` file for the inlets-uplink chart:

```yaml
clientRouter:
  # Customer tunnels will connect with a URI of:
  # wss://uplink.example.com/namespace/tunnel
  domain: uplink.example.com

  tls:
    issuer:
      # Email address used for ACME registration
      email: "user@example.com"

    ingress:
      enabled: true
      class: "nginx"      
```

Make sure to replace the domain and email with your actual domain name and email address.

### B) Install with Istio

We have added support in the inlets-uplink chart for Istio to make it as simple as possible to configure with a HTTP01 challenge.

If you don't have Istio setup already you can deploy it with arkade.

```bash
arkade install istio
```
Create a `values.yaml` file for the inlets-uplink chart:

```yaml
clientRouter:
  # Customer tunnels will connect with a URI of:
  # wss://uplink.example.com/namespace/tunnel
  domain: uplink.example.com

  tls:
    issuer:
      # Email address used for ACME registration
      email: "user@example.com"

    istio:
      enabled: true
```

Make sure to replace the domain and email with your actual domain name and email address.

### Deploy with Helm

Deploy inlets-uplink with your configuration values using Helm:

```
helm upgrade --install inlets-uplink \
  oci://ghcr.io/inlets/inlets-uplink-operator \
  --namespace inlets \
  --values ./values.yaml
```

## Verify the installation

Once you've installed inlets-uplink, you can verify it is deployed correctly by checking the `inlets` namespace for running pods:

```bash
$ kubectl get pods --namespace inlets  

NAME                              READY   STATUS    RESTARTS   AGE
client-router-b5857cf6f-p554d     1/1     Running   0          31m
cloud-operator-5495c59bf9-p7ntm   1/1     Running   0          31m
```

You should see the `client-router` and `cloud-operator` in a `Running` state.

If you installed inlets-uplink with Kubernetes ingress, you can verify that ingress for the client-router is setup and that a TLS certificate is issued for your domain using these two commands:

```bash
$ kubectl get -n inlets ingress/client-router

NAME            CLASS    HOSTS                ADDRESS           PORTS     AGE
client-router   <none>   uplink.example.com   188.166.194.102   80, 443   31m
```

```bash
$ kubectl get -n inlets cert/client-router-cert

NAME                 READY   SECRET               AGE
client-router-cert   True    client-router-cert   30m
```
