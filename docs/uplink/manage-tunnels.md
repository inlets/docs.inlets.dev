# Manage tunnels

## Create a tunnel for a customer using the Custom Resource
`Tunnel` describes an inlets-uplink tunnel server. The specification describes a set of ports to use for TCP tunnels.

For example the following Tunnel configuration sets up a http tunnel on port `8000` by default and adds port `8080` for use with TCP tunnels. The `licenceRef` needs to reference a secret containing an inlets-uplink license.

```yaml
apiVersion: uplink.inlets.dev/v1alpha1
kind: Tunnel
metadata:
  name: team
  namespace: inlets
spec:
  licenseRef:
    name: inlets-uplink-license
    namespace: inlets
  tcpPorts:
  - 8080 
```

### How to create a pre-defined token
By default a new token is created when a tunnel is deployed. It is also possible to create the token for a tunnel yourself and reference it from the tunnel CRD.

You can use `openssl` to generate a token:

```bash
openssl rand -base64 32 > pre-defined-token.txt
```

Create a Kubernetes secret for the token named `custom-token`:

```bash
kubectl create secret generic \
  -n inlets pre-defined-token \
  --from-file token=./pre-defined-token.txt
```

Reference the token when creating a tunnel:

```yaml
apiVersion: uplink.inlets.dev/v1alpha1
kind: Tunnel
metadata:
  name: team
  namespace: inlets
spec:
  licenseRef:
    name: inlets-uplink-license
    namespace: inlets
  tokenRef:
    name: pre-defined-token
    namespace: inlets
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


### Tunnel http services
Inlets-pro has a built in file server that we can use to quickly spin up a http server for this example.

```bash
$ inlets-pro fileserver -w ./ -a

Starting inlets Pro fileserver. Version: 0.9.10-rc1-1-g7bc49ae - 7bc49ae494bd9ec789fc5e9eaf500f2b1fe60786
Serving files from: ./
Listening on: 127.0.0.1:8080, allow browsing: true, auth: false
```

Once the server is running connect to your tunnel using the inlets-uplink client. We will connect to the tunnel called `team` (see the example in [Create a tunnel for a customer using the Custom Resource](#Create a tunnel for a customer using the Custom Resource) to create this tunnel).

Retrieve the token for the tunnel:

```bash
kubectl get secret -n inlets team -o jsonpath="{.data.token}" | base64 --decode > token.txt 
```

Start the tunnel client:
```
inlets-pro uplink client \
  --url wss://uplink.welteki.dev/inlets/team \
  --upstream http://127.0.0.1:8080 \
  --token-file ./token.txt
```

Run a container in the cluster to check the file server is accessible through the http tunnel using curl: `curl -i team.inlets:8000`

```bash
$ kubectl run -t -i curl --rm --image ghcr.io/openfaas/curl:latest /bin/sh   

If you don't see a command prompt, try pressing enter.
~ $ curl -i team.inlets:8000
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Date: Thu, 17 Nov 2022 08:39:48 GMT
Last-Modified: Mon, 14 Nov 2022 20:52:53 GMT
Content-Length: 973

<pre>
<a href="README.md">README.md</a>
</pre>
```

### Tunnel TCP services
Imagine that you’d want to forwarded a Postgres database from a customer site `acmeco` to your cloud Kubernetes cluster.

#### Creat a tunnel for the customer

Example Custom Resource to deploy a tunnel for acmeco’s production Postgres database:

```yaml
apiVersion: uplink.inlets.dev/v1alpha1
kind: Tunnel
metadata:
  name: prod
  namespace: acmeco
spec:
  licenseRef:
    name: inlets-uplink-license
    namespace: acmeco
  tcpPorts:
  - 5432
```

#### Run postgresql on your private server
We can run a Postgresql instance using Docker:

```bash
head -c 16 /dev/urandom |shasum 
8cb3efe58df984d3ab89bcf4566b31b49b2b79b9

export PASSWORD="8cb3efe58df984d3ab89bcf4566b31b49b2b79b9"

kubectl run -t -i psql --env PGPORT=5432 --env PGPASSWORD=$PASSWORD --rm --image postgres:latest -- psql -U postgres -h prod.acmeco
```

#### Connect the inlets uplink client

```
export UPLINK_DOMAIN="uplink.example.com"
export TOKEN_FILE="./token.txt"

inlets-pro uplink client \
  --url wss://${UPLINK_DOMAIN}/acmeco/prod \
  --upstream 127.0.0.1:5432 \
  --token-file ${TOKEN_FILE}
```

#### Access the database from your cluster

```bash
export PASSWORD="8cb3efe58df984d3ab89bcf4566b31b49b2b79b9"

kubectl run -it -env PGPORT=5432 -env PGPASSWORD=$PASSWORD --rm postgres:latest psql -U postgres -h potgres-prod.acmeco
```

Try a command such as `CREATE database` or `\l`.
