# Become an inlets uplink provider

inlets uplink makes it easy for Service Providers and SaaS companies to deliver their product and services to customer networks.

To become a provider, you'll need a Kubernetes cluster, an inlets uplink subscription and to install the inlets-uplink-provider Helm chart.

- [Read the Inlets Uplink announcement](https://inlets.dev/blog/2022/11/16/service-provider-uplinks.html)

## Before you start

Before you start, you'll need the following:

* A Kubernetes cluster with LoadBalancer capabilities (i.e. public cloud).
* A domain name clients can use to connect to the tunnel control plane.
* An inlets uplink license (an inlets-pro license cannot be used)
* Optional: [arkade](https://github.com/alexellis/arkade) - a tool for installing popular Kubernetes tools

    To install arkade run:

    ```bash
    curl -sSLf https://get.arkade.dev/ | sudo sh
    ```

Any existing inlets pro subscribers can convert to an [inlets uplink subscription](https://openfaas.gumroad.com/l/inlets-uplink).

## Create a Kubernetes cluster

We recommend creating a Kubernetes cluster with a minimum of three nodes. Each node should have a minimum of 2GB of RAM and 2 CPU cores.

## Install cert-manager

Install [cert-manager](https://cert-manager.io/docs/), which is used to manage TLS certificates for inlets-uplink.

You can use Helm, or arkade:

```bash
arkade install cert-manager
```

## Create a namespace for the inlets-uplink-provider and install your license

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

## Setup up ingress for customer tunnels

Tunnels on your customers' network will connect to your own inlets-uplink-provider.

There are two options for deploying the inlets-uplink-provider.

Use Option A if you're not sure, if your team already uses Istio or prefers Istio, use Option B.

### A) Install with Kubernetes Ingress

We recommend ingress-nginx, and have finely tuned the configuration to work well for the underlying websocket for inlets. That said, you can change the IngressController if you wish.

Install ingress-nginx using arkade or Helm:

```bash
arkade install ingress-nginx
```

Create a `values.yaml` file for the inlets-uplink-provider chart:

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

Label the `inlets` namespace so that Istio can inject its sidecars:

```bash
kubectl label namespace inlets \
  istio-injection=enabled --overwrite
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

The Helm chart is called *inlets-uplink-provider*, you can deploy it using the custom values.yaml file created above:

```bash
helm upgrade --install inlets-uplink \
  oci://ghcr.io/inlets/inlets-uplink-provider \
  --namespace inlets \
  --values ./values.yaml
```

If you want to pin the version of the Helm chart, you can do so with the `--version` flag.

You can browse [all versions of the Helm chart on GitHub](https://ghcr.io/inlets/inlets-uplink-provider)

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

## Setup the first customer tunnel

Continue the setup here: [Create a customer tunnel](/uplink/create-tunnels)
