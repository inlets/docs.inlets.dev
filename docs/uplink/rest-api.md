# Inlets Uplink REST API

Inlets uplink tunnels and namespaces can be managed through a REST API.

A quick note on TLS: when you expose the uplink API over the Internet, you should enable HTTPS to prevent snooping and tampering.

## Installation

The REST API is part of the client-api component, and is disabled by default.
You'll need to configure authentication, then update the values.yaml file and install the chart again.

An access token needs to be created for the Client API before it can be deployed.

```sh
# Generate a new access token
export token=$(openssl rand -base64 32|tr -d '\n')
echo -n $token > $HOME/.inlets/client-api

# Store the access token in a secret in the inlets namespace.
kubectl create secret generic \
  client-api-token \
  -n inlets \
  --from-file client-api-token=$HOME/.inlets/client-api
```

This token can be used to authenticate with the API.

> See the [OAuth configuration](#configure-oauth) section for instructions on how to enable OAuth.

Add the following parameters to your uplink `values.yaml` file and update the deployment:

```yaml
clientApi:
  enable: true

  tls:
    ingress:
      enabled: true
```

By default, the client-api will be exposed on the same domain as the client-router under the `/v1` path prefix. For example, if your client-router domain is `uplink.example.com`, the API will be available at `https://uplink.example.com/v1`.

If you prefer to use a separate domain for the client-api, set the `clientApi.domain` field:

```yaml
clientApi:
  enable: true

  # Use a dedicated domain for the client API
  domain: clientapi.example.com

  tls:
    ingress:
      enabled: true
```

When a dedicated domain is set, a separate Ingress resource is created for the client-api.

## Authentication

The Inlets Uplink client API supports authentication through a static API token or using OAuth.

### Static API token

The authentication token can be retrieved from the cluster at any time by an administrator.

```sh
export TOKEN=$(kubectl get secret -n inlets client-api-token \
  -o jsonpath="{.data.client-api-token}" \
  | base64 --decode)
```

Use the token as bearer token in the `Authorization` header when making requests to the API.

### OAuth

If you have OAuth enabled you can obtain a token from your provider that can be used to invoke the Uplink Client API.

The example uses the client credentials grant. Replace the token url, client id and client secret with the values obtained from your identity provider.

```sh
export IDP_TOKEN_URL="https://myprovider.example.com/token"
export CLIENT_ID="inlets-uplink"
export CLIENT_SECRET="$(cat ./client-secret.txt)"

curl -S -L -X POST "${IDP_TOKEN_URL}" \
  --header 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode "client_id=${CLIENT_ID}" \
  --data-urlencode "client_secret=${CLIENT_SECRET}" \
  --data-urlencode 'scope=openid' \
  --data-urlencode 'grant_type=client_credentials'
```

Use the token as bearer token in the `Authorization` header when making requests to the API.

```sh
export CLIENT_API="https://uplink.example.com"
export NAME="acmeco"
export NAMESPACE="acmeco"

curl -i \
  -H "Authorization: Bearer ${TOKEN}" \
  "$CLIENT_API/v1/tunnels/$NAME?namespace=$NAMESPACE"
```

## Tunnel management

We will be create an tunnel named `acmeco` in the `acmeco` namespace in the API examples. 

### Get a tunnel

```sh
export CLIENT_API="https://uplink.example.com"
export NAME="acmeco"
export NAMESPACE="acmeco"

curl -i \
  -H "Authorization: Bearer ${TOKEN}" \
  "$CLIENT_API/v1/tunnels/$NAME?namespace=$NAMESPACE"
```

Adding the query parameter `metrics=1` includes additional tunnel metrics in the response like RX and TX and TCP connection rate.

Path parameters:

- `name` - Name of the tunnel.

Query parameters:

- `namespace` - Namespace where the tunnel should be looked up.
- `metrics` - Include tunnel metrics in the response.

Example response with metrics:

```json
{
  "name": "acmeco",
  "namespace": "acmeco",
  "tcpPorts": [80, 443],
  "authToken": "TAjFZExVq6qUfnqojwR2HOej347fRXqV3vLexlyoP6GcRZ2SjIUALY8Jdx8",
  "connectedClients": 1,
  "created": "2024-09-10T14:48:21Z",
  "metrics": {
    "rx": 195482,
    "tx": 32348,
    "tcpConnectionRate": 62.99
  }
}
```

The metrics section includes rx/tx bytes per second and tcp connection rate over the last 5 minutes.

### List tunnels

```sh
export CLIENT_API="https://uplink.example.com"
export NAMESPACE="acmeco"

curl -i \
  -H "Authorization: Bearer ${TOKEN}" \
  "$CLIENT_API/v1/tunnels?namespace=$NAMESPACE"
```

Query parameters:

- `namespace` - Namespace where the tunnel should be looked up.

### Create a tunnel

```sh
export CLIENT_API="https://uplink.example.com"

curl -i \
  -X POST \
  -H "Authorization: Bearer ${TOKEN}" \
  "$CLIENT_API/v1/tunnels"
  -d '{ "name": "acmeco", "namespace": "acmeco", "tcpPorts": [ 80, 443 ]  }'
```

### Update a tunnel

```sh
export CLIENT_API="https://uplink.example.com"

curl -i \
  -X PUT \
  -H "Authorization: Bearer ${TOKEN}" \
  "$CLIENT_API/v1/tunnels"
  -d '{ "name": "acmeco", "namespace": "acmeco", "tcpPorts": [ 80, 443, 4222 ] }'
```

### Delete a tunnel

```sh
export CLIENT_API="https://uplink.example.com"
export NAME="acmeco"
export NAMESPACE="acmeco"

curl -i \
  -X DELETE \
  -H "Authorization: Bearer ${TOKEN}" \
  "$CLIENT_API/v1/tunnels/$NAME?namespace=$NAMESPACE"
```

Path parameters:

- `name` - Name of the tunnel.

Query parameters:

- `namespace` - Namespace where the tunnel should be looked up.

## Namespace management

The inlets uplink client API includes REST endpoints for listing, creating and deleting namespaces. Namespaces created through the API are automatically labeled for use with inlets uplink.
The `kube-system` and `inlets` namespace can not be used as tunnel namespaces.

### List uplink namespace

List all inlets uplink namespaces. This endpoint will list all namespaces with a label `inlets.dev/uplink=1`.

```sh
export CLIENT_API="https://uplink.example.com"

curl -i \
  -H "Authorization: Bearer ${TOKEN}" \
  "$CLIENT_API/v1/namespace
```

### Create a namespace

```sh
export CLIENT_API="https://uplink.example.com"

curl -i \
  -X POST \
  -H "Authorization: Bearer ${TOKEN}" \
  "$CLIENT_API/v1/namespace
  -d '{ "name": "acmeco" }'
```

Every namespace created through the API will have the `inlets.dev/uplink=1` label set.

The API supports adding additional namespace labels and annotations:

```json
{
  "name": "acmeco",
  "annotations": {
    "customer": "acmeco"
  },
  "labels": {
    "customer": "acmeco"
  }
}
```

### Delete a namespace

```sh
export CLIENT_API="https://uplink.example.com"
export NAME="acmeco"

curl -i \
  -X DELETE \
  -H "Authorization: Bearer ${TOKEN}" \
  "$CLIENT_API/v1/namespace/$NAME"
```

## Configure OAuth

You can configure any OpenID Connect (OIDC) compatible identity provider for use with Inlets Uplink.

1. Register a new client (application) for Inlets Uplink with your identity provider.
2. Enable the required authentication flows.
  The Client Credentials flow is ideal for serve-to-server interactions where there is no direct user involvement. This is the flow we recommend and use in our examples any other authentication flow can be picked depending on your use case.
3. Configure Client API

    Update your `values.yaml` file and add to following parameters to the `clientApi` section:

    ```yaml
    clientApi:
      # OIDC provider url.
      issuerURL: "https://myprovider.example.com"

      # The audience is generally the same as the value of the domain field, however
      # some issuers like keycloak make the audience the client_id of the application/client.
      audience: "uplink.example.com"
    ```

  The `issuerURL` needs to be set to the url of your provider, eg. `https://accounts.google.com` for google or `https://example.eu.auth0.com/` for Auth0.
  
  The `audience` is usually the client apis public URL although for some providers it can also be the client id.