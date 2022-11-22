# Create tunnels

## Create separate namespace(s) for your tunnels

We recommend deploying customer tunnels into a one or more separate namespaces, that means you can keep the `inlets` namespace for the software coordinates the tunnels.

You could create a single namespace for customers i.e.

```bash
kubectl create namespace tunnels
```

Or you could create one per customer:

```bash
kubectl create namespace acmeco
```

Remember, that if you're an Istio user, you should label each namespace:

```bash
kubectl label namespace inlets \
  istio-injection=enabled --overwrite
```

## Create a tunnel for a customer using the Custom Resource

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

### Define a pre-existing token for your customers

By default a token is generated for tunnels, however if you are using a GitOps workflow, or store your tunnel YAML files in Git, you may want to precreate the tokens for each tunnel.

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

## Connect to tunnels

The `uplink client` command is part of the inlets-pro binary. It is used to connect to tunnels and expose services over the tunnel.

There are several ways to get the binary:

- Download it from the [GitHub releases](https://github.com/inlets/inlets-pro/releases)
- Get it with [arkade](https://github.com/alexellis/arkade): `arakde get inlets-pro`
- Use the [inlets-pro docker image](https://github.com/orgs/inlets/packages/container/package/inlets-pro)

### Exampe: Tunnel a customer HTTP service

We'll use inlets-pro's built in fileserver as an example of how to tunnel a HTTP service.

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

```bash
kubectl get secret -n tunnels acmeco -o jsonpath="{.data.token}" | base64 --decode > token.txt 
```

Start the tunnel client:
```
inlets-pro uplink client \
  --url wss://uplink.welteki.dev/tunnels/acmeco \
  --upstream http://127.0.0.1:8080 \
  --token-file ./token.txt
```

Run a container in the cluster to check the file server is accessible through the http tunnel using curl: `curl -i acmeco.tunnels:8000`

```bash
$ kubectl run -t -i curl --rm --image ghcr.io/openfaas/curl:latest /bin/sh   

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

### Tunnel a customer's TCP service

Perhaps you need to access a customer's Postgres database from their private network?

#### Create a TCP tunnel using a Custom Resouce

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

Now that the tunnel is established, you can connect to the customer's Postgres database from within Kubernetes using its ClusteRIP `prod-database.acmeco.svc.cluster.local`:

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
