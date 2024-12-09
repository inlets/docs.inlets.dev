# Inlets Uplink REST API

Inlets uplink tunnels and namespaces can be managed through a REST API.

A quick note on TLS: when you expose the uplink API over the Internet, you should enable HTTPS to prevent snooping and tampering.

## Installation

The Uplink Client API can be enabled through the helm chart. Update the `values.yaml`:

```yaml
clientApi:
    enable: true

    # Domain used for client API ingress
    domain: clientapi.example.com
```

## Authentication

The Inlets Uplink client API supports authentication through API tokens.

The authentication token can be retrieved from the cluster at any time by an administrator.

```sh
export TOKEN=$(kubectl get secret -n inlets client-api-token \
  -o jsonpath="{.data.client-api-token}" \
  | base64 --decode)
```

## Tunnels

### Get a tunnel

```sh
curl -i \
    -H "Authorization: Bearer ${TOKEN}" \
    "$CLIENT_API/v1/tunnels/$NAME?namespace=$NAMESPACE"
```

Adding the query parameter `metrics=1` includes additional tunnel metrics in the response like RX and TX and TCP connection rate.

Path parameters:

* `name` - Name of the tunnel.

Query parameters:

* `namespace` - Namespace where the tunnel should be looked up.
* `metrics` - Include tunnel metrics in the response.

### List tunnels

```sh
curl -i \
    -H "Authorization: Bearer ${TOKEN}" \
    "$CLIENT_API/v1/tunnels?namespace=$NAMESPACE"
```

Query parameters:

* `namespace` - Namespace where the tunnel should be looked up.

### Create a tunnel

```sh
curl -i \
    -X POST \
    -H "Authorization: Bearer ${TOKEN}" \
    "$CLIENT_API/v1/tunnels"
    -d '{ "name": "acmeco", "namespace": "acmeco", "tcpPorts": [ 80, 443 ]  }'
```

### Update a tunnel

```sh
curl -i \
    -X PUT \
    -H "Authorization: Bearer ${TOKEN}" \
    "$CLIENT_API/v1/tunnels"
    -d '{ "name": "acmeco", "namespace": "acmeco", "tcpPorts": [ 80, 443, 4222 ] }'
```

### Delete a tunnel

```sh
curl -i \
    -X DELETE \
    -H "Authorization: Bearer ${TOKEN}" \
    "$CLIENT_API/v1/tunnels/$NAME?namespace=$NAMESPACE"
```

Path parameters:

* `name` - Name of the tunnel.

Query parameters:

* `namespace` - Namespace where the tunnel should be looked up.


## Namespace management

The inlets uplink client API includes REST endpoints for listing, creating and deleting namespaces. Namespaces created through the API are automatically labeled for use with inlets uplink.
The `kube-system` and `inlets` namespace can not be used as tunnel namespaces.

### List uplink namespace

List all inlets uplink namespaces. This endpoint will list all namespaces with a label `inlets.dev/uplink=1`.

```sh
curl -i \
    -H "Authorization: Bearer ${TOKEN}" \
    "$CLIENT_API/v1/namespace
```

### Create a namespace

```sh
curl -i \
    -X POST \
    -H "Authorization: Bearer ${TOKEN}" \
    "$CLIENT_API/v1/tunnels/$NAME?namespace=$NAMESPACE"
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
curl -i \
    -X DELETE \
    -H "Authorization: Bearer ${TOKEN}" \
    "$CLIENT_API/v1/namespace/$NAME"
```