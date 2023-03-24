# Create a tunnel for a customer

## Use separate namespaces for your tunnels

The `inlets` namespace contains the control plane for inlets uplink, so you'll need to create at least one additional namespace for your customer tunnels.

1. Create a namespace per customer (recommended)

    This approach avoids conflicts on names, and gives better isolation between tenants.

    ```bash
    kubectl create namespace acmeco
    ```

    Then, create a copy of the license secret in the new namespace:

    ```bash
    export NS="n1"
    export LICENSE=$(kubectl get secret -n inlets inlets-uplink-license -o jsonpath='{.data.license}' | base64 -d)

    kubectl create secret generic \
      -n $NS \
      inlets-uplink-license \
      --from-literal license=$LICENSE
    ```

2. A single namespace for all customer tunnels (not recommended)

    For development purposes, you could create a single namespace for all your customers.

    ```bash
    kubectl create namespace tunnels
    ```

Finally, if you're using Istio, then you need to label each additional namespace to enable sidecar injection:


```bash
kubectl label namespace inlets \
  istio-injection=enabled --overwrite
```

## Create a Tunnel with an auto-generated token

`Tunnel` describes an inlets-uplink tunnel server. The specification describes a set of ports to use for TCP tunnels.

For example the following Tunnel configuration sets up a http tunnel on port `8000` by default and adds port `8080` for use with TCP tunnels. The `licenceRef` needs to reference a secret containing an inlets-uplink license.

```yaml
apiVersion: uplink.inlets.dev/v1alpha1
kind: Tunnel
metadata:
  name: acmeco
  namespace: tunnels
spec:
  licenseRef:
    name: inlets-uplink-license
    namespace: tunnels
  tcpPorts:
  - 8080 
```

Alternatively the CLI can be used to create a tunnel:

```bash
inlets-pro tunnel create acmeco \
  -n tunnels
  --port 8080
```

## Create a Tunnel with a pre-defined token

If you delete a Tunnel with an auto-generated token, and re-create it later, the token will change. So we recommend that you pre-define your tokens. This style works well for GitOps and automated deployments with Helm.

Make sure the secret is in the same namespace as the Tunnel Custom Resource.

You can use `openssl` to generate a secure token:

```bash
openssl rand -base64 32 > token.txt
```

Create a Kubernetes secret for the token named `custom-token`:

```bash
kubectl create secret generic \
  -n tunnels acmeco-token \
  --from-file token=./token.txt
```

Reference the token when creating a tunnel:

```yaml
apiVersion: uplink.inlets.dev/v1alpha1
kind: Tunnel
metadata:
  name: acmeco
  namespace: tunnels
spec:
  licenseRef:
    name: inlets-uplink-license
    namespace: tunnels
  tokenRef:
    name: acmeco-token
    namespace: tunnels
  tcpPorts:
  - 8080
```

Clients can now connect to the tunnel using the custom token.

### Node selection and annotations for tunnels

The tunnel spec has a `nodeSelector` field that can be used to assign tunnel pods to Nodes. See [Assign Pods to Nodes](https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes/) from the kubernetes docs for more information.

It is also possible to set additional annotations on the tunnel pod using the `podAnnotations` field in the tunnel spec.

The following example adds an annotation with the customer name to the tunnel pod and uses the node selector to specify a target node with a specific region label.

```yaml
apiVersion: uplink.inlets.dev/v1alpha1
kind: Tunnel
metadata:
  name: acmeco
  namespace: tunnels
spec:
  licenseRef:
    name: inlets-uplink-license
    namespace: tunnels
  tcpPorts:
  - 8080
  podAnnotations:
    cutomer: acmeco
  nodeSelector:
    region: east
```

## Connect to tunnels

The `uplink client` command is part of the inlets-pro binary. It is used to connect to tunnels and expose services over the tunnel.

There are several ways to get the binary:

- Download it from the [GitHub releases](https://github.com/inlets/inlets-pro/releases)
- Get it with [arkade](https://github.com/alexellis/arkade): `arkade get inlets-pro`
- Use the [inlets-pro docker image](https://github.com/orgs/inlets/packages/container/package/inlets-pro)

### Example: Tunnel a customer HTTP service

We'll use inlets-pro's built in file server as an example of how to tunnel a HTTP service.

Run this command on a private network or on your workstation:

```bash
mkdir -p /tmp/share
cd /tmp/share
echo "Hello World" > README.md

inlets-pro fileserver -w /tmp/share -a

Starting inlets Pro fileserver. Version: 0.9.10-rc1-1-g7bc49ae - 7bc49ae494bd9ec789fc5e9eaf500f2b1fe60786
Serving files from: /tmp/share
Listening on: 127.0.0.1:8080, allow browsing: true, auth: false
```

Once the server is running connect to your tunnel using the inlets-uplink client. We will connect to the tunnel called `acmeco` (see the example in [Create a tunnel for a customer using the Custom Resource](#Create a tunnel for a customer using the Custom Resource) to create this tunnel).

Retrieve the token for the tunnel:

=== "kubectl"

    ```bash
    kubectl get -n tunnels \
      secret/acmeco -o jsonpath="{.data.token}" | base64 --decode > token.txt 
    ```

=== "cli"

    ```bash
    inlets-pro tunnel token acmeco \
      -n tunnels > token.txt
    ```

The contents will be saved in token.txt.

Start the tunnel client:

```bash
inlets-pro uplink client \
  --url wss://uplink.example.com/tunnels/acmeco \
  --upstream http://127.0.0.1:8080 \
  --token-file ./token.txt
```

!!! tip "Tip: get connection instructions"
    The tunnel plugin for the inlets-pro CLI can be used to get connection instructions for a tunnel.

    ```bash
    inlets-pro tunnel connect acmeco \
      --domain uplink.example.com \
      --upstream http://127.0.0.1:8080
    ```

    Running the command above will print out the instructions to connect to the tunnel:
    
    ```bash
    # Access your tunnel via ClusterIP: acmeco.tunnels
    inlets-pro uplink client \
      --url=wss://uplink.example.com/tunnels/acmeco \
      --upstream=http://127.0.0.1:8080 \
      --token=z4oubxcamiv89V0dy8ytmjUEPwAmY0yFyQ6uaBmXsIQHKtAzlT3PcGZRgK
    ```


Run a container in the cluster to check the file server is accessible through the http tunnel using curl: `curl -i acmeco.tunnels:8000`

```bash
$ kubectl run -t -i curl --rm \
  --image ghcr.io/openfaas/curl:latest /bin/sh   

$ curl -i acmeco.tunnels:8000
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Date: Thu, 17 Nov 2022 08:39:48 GMT
Last-Modified: Mon, 14 Nov 2022 20:52:53 GMT
Content-Length: 973

<pre>
<a href="README.md">README.md</a>
</pre>
```

#### How to tunnel multiple HTTP services from a customer

The following example shows how to access more than one HTTP service over the same tunnel. It is possible to expose multiple upstream services over a single tunnel.

Start a tunnel client and add multiple upstreams:

```bash
inlets-pro uplink client \
  --url wss://uplink.example.com/tunnels/acmeco \
  --upstream prometheus=http://127.0.0.1:9090 \
  --upstream gateway=http://127.0.0.1:8080 \
  --token-file ./token.txt

```

Access both services using `curl`:

```bash
$ kubectl run -t -i curl --rm \
  --image ghcr.io/openfaas/curl:latest /bin/sh   

$ curl -i -H "Host: prometheus" acmeco.tunnels:8000
HTTP/1.1 302 Found
Content-Length: 29
Content-Type: text/html; charset=utf-8
Date: Thu, 16 Feb 2023 16:29:09 GMT
Location: /graph

<a href="/graph">Found</a>.


$ curl -i -H "Host: gateway" acmeco.tunnels:8000
HTTP/1.1 301 Moved Permanently
Content-Length: 39
Content-Type: text/html; charset=utf-8
Date: Thu, 16 Feb 2023 16:29:11 GMT
Location: /ui/

<a href="/ui/">Moved Permanently</a>.
```

Note that the `Host` header has to be set in the request so the tunnel knows which upstream to send the request to.

### Tunnel a customer's TCP service

Perhaps you need to access a customer's Postgres database from their private network?

#### Create a TCP tunnel using a Custom Resource

Example Custom Resource to deploy a tunnel for acmeco’s production Postgres database:

```yaml
apiVersion: uplink.inlets.dev/v1alpha1
kind: Tunnel
metadata:
  name: prod-database
  namespace: acmeco
spec:
  licenseRef:
    name: inlets-uplink-license
    namespace: acmeco
  tcpPorts:
  - 5432
```

Alternatively the cli can be used to create a new tunnel:

```bash
inlets-pro tunnel create prod-database \
  -n acmeco
  --port 5432
```

#### Run postgresql on your private server

The quickest way to spin up a Postgres instance on your own machine would be to use Docker:

```bash
head -c 16 /dev/urandom |shasum 
8cb3efe58df984d3ab89bcf4566b31b49b2b79b9

export PASSWORD="8cb3efe58df984d3ab89bcf4566b31b49b2b79b9"

docker run --rm --name postgres \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=8cb3efe58df984d3ab89bcf4566b31b49b2b79b9 \
  -ti postgres:latest
```

#### Connect with an inlets uplink client

```bash
export UPLINK_DOMAIN="uplink.example.com"

inlets-pro uplink client \
  --url wss://${UPLINK_DOMAIN}/acmeco/prod-database \
  --upstream 127.0.0.1:5432 \
  --token-file ./token.txt
```

#### Access the customer database from within Kubernetes

Now that the tunnel is established, you can connect to the customer's Postgres database from within Kubernetes using its ClusterIP `prod-database.acmeco.svc.cluster.local`:

Try it out:

```bash
export PASSWORD="8cb3efe58df984d3ab89bcf4566b31b49b2b79b9"

kubectl run -i -t psql \
  --env PGPORT=5432 \
  --env PGPASSWORD=$PASSWORD --rm \
  --image postgres:latest -- psql -U postgres -h prod-database.acmeco
```

Try a command such as `CREATE database websites (url TEXT)`, `\dt` or `\l`.

## Getting help

Feel free to [reach out to our team via email](mailto:support@openfaas.com) for technical support.
