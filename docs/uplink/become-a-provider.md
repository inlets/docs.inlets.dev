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

Inlets uplink has its own independent subscription from inlets-pro.

Sign-up here: [inlets uplink plans](https://subscribe.openfaas.com/).

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

!!! note

    There is a known issue with LemonSqueezy where the UI will copy the license key in lower-case, it needs to be converted to upper-case before being used with Inlets Uplink.

Convert the license to upper-case, if it's in lower-case:

```bash
(
  mv $HOME/.inlets/LICENSE_UPLINK{,.lower}

  cat $HOME/.inlets/LICENSE_UPLINK.lower | tr '[:lower:]' '[:upper:]' > $HOME/.inlets/LICENSE_UPLINK
  rm $HOME/.inlets/LICENSE_UPLINK.lower
)
```

Create the secret for the license:

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
    issuerName: letsencrypt-prod

    # When set, a production issuer will be generated for you
    # to use a pre-existing issuer, set issuer.enabled=false
    issuer:
      # Create a production issuer as part of the chart installation
      enabled: true

      # Email address used for ACME registration for the production issuer
      email: "user@example.com"

    ingress:
      enabled: true
      class: "nginx"      
```

Make sure to replace the domain and email with your actual domain name and email address.

Want to use the staging issuer for testing?

To use the Let's Encrypt staging issuer, pre-create your own issuer, update `clientRouter.tls.issuerName` with the name you have chosen, and then update `clientRouter.tls.issuer.enabled` and set it to false.

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
    issuerName: letsencrypt-prod

    # When set, a production issuer will be generated for you
    # to use a pre-existing issuer, set issuer.enabled=false
    issuer:
      # Create a production issuer as part of the chart installation
      enabled: true

      # Email address used for ACME registration for the production issuer
      email: "user@example.com"

    istio:
      enabled: true
```

Make sure to replace the domain and email with your actual domain name and email address.

### Deploy with Helm

The Helm chart is called *inlets-uplink-provider*, you can deploy it using the custom values.yaml file created above:

```bash
helm upgrade --install inlets-uplink \
  oci://ghcr.io/openfaasltd/inlets-uplink-provider \
  --namespace inlets \
  --values ./values.yaml
```

If you want to pin the version of the Helm chart, you can do so with the `--version` flag.

You can browse [all versions of the Helm chart on GitHub](https://ghcr.io/openfaasltd/inlets-uplink-provider)

Alternatively, you can get the list of tags, including the latest tag via the crane CLI:

```bash
arkade get crane

# List versions
crane ls ghcr.io/openfaasltd/inlets-uplink-provider

# Get the latest version
LATEST=$(crane ls ghcr.io/openfaasltd/inlets-uplink-provider |tail -n 1)
echo $LATEST
```

## Verify the installation

Once you've installed inlets-uplink, you can verify it is deployed correctly by checking the `inlets` namespace for running pods:

```bash
$ kubectl get pods --namespace inlets

NAME                               READY   STATUS    RESTARTS   AGE
client-router-b5857cf6f-7vrdh      1/1     Running   0          92s
prometheus-74d8d7db9b-2hptm        1/1     Running   0          16s
uplink-operator-7fccc9bdbc-twd2q   1/1     Running   0          92s
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

## Download the tunnel CLI

We provide a CLI to help you create and manage tunnels. It is available as a plugin for the inlets-pro CLI. 

Download the `inlets-pro` binary:

- Download it from the [GitHub releases](https://github.com/inlets/inlets-pro/releases)
- Get it with [arkade](https://github.com/alexellis/arkade): `arkade get inlets-pro`

Get the tunnel plugin:

```bash
inlets-pro plugin get tunnel
```

Run `inlets-pro tunnel --help` to see all available commands.

## Setup the first customer tunnel

Continue the setup here: [Create a customer tunnel](/uplink/create-tunnels)

## Upgrading the chart and components

If you have a copy of values.yaml with pinned image versions, you should update these manually.

Next, run the Helm chart installation command again, and remember to use the sames values.yaml file that you used to install the software originally.

Over time, you may find using a tool like FluxCD or ArgoCD to manage the installation and updates makes more sense than running Helm commands manually.

If the Custom Resource Definition (CRD) has changed, you can extract it from the Chart repo and install it before or after upgrading. As a rule, Helm won't install or upgrade CRDs a second time if there's already an existing version:

```bash
helm template oci://ghcr.io/openfaasltd/inlets-uplink-provider \
  --include-crds=true \
  --output-dir=/tmp

kubectl apply -f /
  tmp/inlets-uplink-provider/crds/uplink.inlets.dev_tunnels.yaml
```

## Configuration reference

Looking for the source for the Helm chart? The source is published directly to a container registry as an OCI bundle. View the source with: `helm template oci://ghcr.io/openfaasltd/inlets-uplink-provider`

If you need a configuration option outside of what's already available, feel free to raise an issue on the [inlets-pro repository](https://github.com/inlets/inlets-pro/issues).

Overview of inlets-uplink parameters in `values.yaml`.

| Parameter                | Description                                                                            | Default                        |
| ------------------------ | -------------------------------------------------------------------------------------- | ------------------------------ |
| `pullPolicy` | The a imagePullPolicy applied to inlets-uplink components. | `Always` |
| `operator.image` | Container image used for the uplink operator. | `ghcr.io/openfaasltd/uplink-operator:0.1.5` |
| `clientRouter.image` | Container image used for the client router. | `ghcr.io/openfaasltd/uplink-client-router:0.1.5` |
| `clientRouter.domain` | Domain name for inlets uplink. Customer tunnels will connect with a URI of: wss://uplink.example.com/namespace/tunnel. | `""` |
| `clientRouter.tls.issuerName` | Name of cert-manager Issuer for the clientRouter domain. | `letsencrypt-prod` |
| `clientRouter.tls.issuer.enabled` | Create a cert-manager Issuer for the clientRouter domain. Set to false if you wish to specify your own pre-existing object in the `clientRouter.tls.issuerName` field. | `true` |
| `clientRouter.tls.issuer.email` | Let's Encrypt email. Only used for certificate renewing notifications. | `""` |
| `clientRouter.tls.ingress.enabled` | Enable ingress for the client router. | `enabled` |
| `clientRouter.tls.ingress.class` | Ingress class for client router ingress. | `nginx` |
| `clientRouter.tls.ingress.annotations` | Annotations to be added to the client router ingress resource. | `{}` |
| `clientRouter.tls.istio.enabled` | Use an Istio Gateway for incoming traffic to the client router. | `false` |
| `clientRouter.service.type` | Client router service type | `ClusterIP` |
| `clientRouter.service.nodePort` | Client router service port for NodePort service type, assigned automatically when left empty. (only if clientRouter.service.type is set to "NodePort")| `nil` |
| `tunnelsNamespace` | Deployments, Services and Secrets will be created in this namespace. Leave blank for a cluster-wide scope, with tunnels in multiple namespaces. | `""` |
| `inletsVersion` | Inlets Pro release version for tunnel server Pods. | `0.9.12` |
| `clientApi.enabled` | Enable tunnel management REST API. | `false` |
| `clientApi.image` | Container image used for the client API. | `ghcr.io/openfaasltd/uplink-api:0.1.5` |
| `prometheus.create` | Create the Prometheus monitoring component. | `true` |
| `prometheus.resources` | Resource limits and requests for prometheus containers. | `{}` |
| `prometheus.image` | Container image used for prometheus. | `prom/prometheus:v2.40.1` |
| `prometheus.service.type` | Prometheus service type | `ClusterIP` |
| `prometheus.service.nodePort` | Prometheus service port for NodePort service type, assigned automatically when left empty. (only if prometheus.service.type is set to "NodePort")| `nil` |
| `nodeSelector` | Node labels for pod assignment. | `{}` |
| `affinity`| Node affinity for pod assignments. | `{}` |
| `tolerations` | Node tolerations for pod assignment. | `[]` |

Specify each parameter using the `--set key=value[,key=value]` argument to `helm install`