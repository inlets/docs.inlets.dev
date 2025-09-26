# Install Inlets Uplink

Inlets Uplink requires a Kubernetes cluster, and an inlets uplink subscription.

The installation is performed through a Helm chart (inlets-uplink-provider)) which is published as an OCI artifact in a container registry.

The default installation keeps tunneled services private, with only the control-plane exposed to the public Internet. To expose the data-plane for one or more tunnels, after you've completed the installation, see the page [Expose tunnels](/uplink/expose-tunnels/).

## Before you start

Before you start, you'll need the following:

* A Kubernetes cluster where you can create a LoadBalancer i.e. a managed Kubernetes service like AWS EKS, Azure AKS, Google GKE, etc.
* A domain name clients can use to connect to the tunnel control plane.
* An inlets uplink license (an inlets-pro license cannot be used)
* Optional: [arkade](https://github.com/alexellis/arkade) - a tool for installing popular Kubernetes tools

    To install arkade run:

    ```bash
    curl -sSLf https://get.arkade.dev/ | sudo sh
    ```

You can obtain a subscription for inlets uplink here: [inlets uplink plans](https://inlets.dev/pricing).

## Create a Kubernetes cluster

We recommend creating a Kubernetes cluster with a minimum of three nodes. Each node should have a minimum of 2GB of RAM and 2 CPU cores.

## Install cert-manager

Install [cert-manager](https://cert-manager.io/docs/), which is used to manage TLS certificates for inlets-uplink for the control-plane and the REST API.

You can use Helm, or arkade:

```bash
arkade install cert-manager
```

## Create a namespace for the chart and add the license secret

Make sure to create the target namespace for you installation first.

```bash
kubectl create namespace inlets
```

Create the required secret with your inlets-uplink license.

!!! note "Check that your license key is in lower-case"

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

## Setup up Ingress for the control-plane

Tunnel clients will connect to the client-router component which needs to be exposed via Ingress.

You can use Kubernetes Ingress or Istio. We recommend using Ingress (Option A), unless your team or organisation is already using Istio (Option B).

### A) Install with Kubernetes Ingress

We recommend [ingress-nginx](https://github.com/kubernetes/ingress-nginx) for Ingress, and have finely tuned the configuration to work well for the underlying websocket for inlets. If your organisation uses a different Ingress Controller, you can alter the `class` fields in the chart.

Install ingress-nginx using arkade or Helm:

```bash
arkade install ingress-nginx
```

Create a `values.yaml` file for the inlets-uplink-provider chart:

```yaml
ingress:
  issuer:
    # When set, a production issuer will be generated for you
    # to use a pre-existing issuer, set issuer.enabled=false
    enabled: true
    # Email address used for ACME registration for the production issuer
    email: "user@example.com"
    class: "nginx"

clientRouter:
  # Customer tunnels will connect with a URI of:
  # wss://uplink.example.com/namespace/tunnel
  domain: uplink.example.com

  tls:
    issuerName: letsencrypt-prod
    ingress:
      enabled: true
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
ingress:
  issuer:
    # When set, a production issuer will be generated for you
    # to use a pre-existing issuer, set issuer.enabled=false
    enabled: true
    # Email address used for ACME registration for the production issuer
    email: "user@example.com"
    class: "istio"

clientRouter:
  # Customer tunnels will connect with a URI of:
  # wss://uplink.example.com/namespace/tunnel
  domain: uplink.example.com

  tls:
    issuerName: letsencrypt-prod
    istio:
      enabled: true
```

Make sure to replace the domain and email with your actual domain name and email address.

### Deploy with Helm

!!! note "The chart is served through a container registry (OCI), not GitHub pages"

    Many Helm charts are served over GitHub pages, from a public repository, making it easy to browse and read the source code. We are using an OCI artifact in a container registry, which makes for a more modern alternative. If you want to browse the source, you can simply run `helm template` instead of `helm upgrade`.

    **Unauthorized?**

    The chart artifacts are public and do not require authentication, however if you run into an "Access denied" or authorization error when interacting with `ghcr.io`, try running `helm registry login ghcr.io` to refresh your credentials, or `docker logout ghcr.io`.

The Helm chart is called *inlets-uplink-provider*, you can deploy it using the custom values.yaml file created above:

```bash
helm upgrade --install inlets-uplink \
  oci://ghcr.io/openfaasltd/inlets-uplink-provider \
  --namespace inlets \
  --values ./values.yaml
```

If you want to pin the version of the Helm chart, you can do so with the `--version` flag.

**Where can I see the various options for values.yaml?**

All of the various options for the Helm chart are documented in the [configuration reference](#configuration-reference).

**How can I view the source code?**

See the note on `helm template` under the [configuration reference](#configuration-reference).

**How can I find the latest version of the chart?**

If you omit a version, Helm will use the latest published OCI artifact, however if you do want to pin it, you can browse [all versions of the Helm chart on GitHub](https://ghcr.io/openfaasltd/inlets-uplink-provider)

As an alternative to using ghcr.io's UI, you can get the list of tags, including the latest tag via the crane CLI:

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

kubectl apply -f \
  /tmp/inlets-uplink-provider/crds/uplink.inlets.dev_tunnels.yaml
```

### Upgrading existing customer tunnels

The operator will upgrade the `image:` version of all deployed inlets uplink tunnels automatically based upon the tag set in values.yaml.

If no value is set in your overridden values.yaml file, then whatever the default is in the chart will be used.

```yaml
inletsVersion: 0.9.23
```

When a tunnel is upgraded, you'll see a log line like this:

```bash
2024-01-11T12:25:15.442Z        info    operator/controller.go:860      Upgrading version       {"tunnel": "ce.inlets", "from": "0.9.21", "to": "0.9.23"}
```

## Configuration reference

Looking for the source for the Helm chart? The source is published directly to a container registry as an OCI bundle. View the source with: `helm template oci://ghcr.io/openfaasltd/inlets-uplink-provider`

If you need a configuration option outside of what's already available, feel free to raise an issue on the [inlets-pro repository](https://github.com/inlets/inlets-pro/issues).

Overview of inlets-uplink parameters in `values.yaml`.

| Parameter                | Description                                                                            | Default                        |
| ------------------------ | -------------------------------------------------------------------------------------- | ------------------------------ |
| `pullPolicy` | The a imagePullPolicy applied to inlets-uplink components. | `Always` |
| `operator.image` | Container image used for the uplink operator. | `ghcr.io/openfaasltd/uplink-operator:0.1.5` |
| `ingress.issuer.name` | Name of cert-manager Issuer. | `letsencrypt-prod` |
| `ingress.issuer.enabled` | Create a cert-manager Issuer. Set to false if you wish to specify your own pre-existing object for each component. | `true` |
| `ingress.issuer.email` | Let's Encrypt email. Only used for certificate renewing notifications. | `""` |
| `ingress.class` |  Ingress class for client router ingress. | `nginx` |
| `clientRouter.image` | Container image used for the client router. | `ghcr.io/openfaasltd/uplink-client-router:0.1.5` |
| `clientRouter.domain` | Domain name for inlets uplink. Customer tunnels will connect with a URI of: wss://uplink.example.com/namespace/tunnel. | `""` |
| `clientRouter.tls.ingress.enabled` | Enable ingress for the client router. | `enabled` |
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

## Telemetry and usage data

The inlets-uplink Kubernetes operator will send telemetry data to OpenFaaS Ltd on a periodic basis. This information is used for calculating accurate usage metrics for billing purposes. This data is sent over HTTPS, does not contain any personal information, and is not shared with any third parties.

This data includes the following:

* Number of tunnels deployed
* Number of namespaces with at least one tunnel contained
* Kubernetes version
* Inlets Uplink version
* Number of installations of Inlets Uplink
