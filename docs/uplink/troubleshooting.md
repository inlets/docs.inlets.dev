# Troubleshooting

## Client connections

Depending on how you are running the client, with Kubernetes or as a systemd service, use the following commands to check the logs.

- kubectl logs command (if deployed in Kubernetes):
    ```bash
    kubectl logs deploy/<tunnel-name>-inlets-client -f
    ```

- systemd logs (if deployed as service):
    ```bash
    journalctl -u <tunnel-name>-tunnel -f
    ```

Logs for the client will show you the connection status, reconnect attempts and errors and warnings that occurred during the connection.

- Verify the client is using a valid auth token for the tunnel.
- Verify the client is using the same inlets version as the tunnel server
- Verify the tunnel exist and is running, see: [troubleshoot tunnel servers](#tunnel-servers)
- Verify the tunnels is reachable through the client-router. See: [troubleshoot the client router](#client-router)

## Tunnel servers

Each tunnel gets its own tunnel server deployment that handles the actual data forwarding.

Check the tunnel exists:

```sh
kubectl get tunnels.uplink.inlets.dev -A
```

Check the tunnel deployment is running and healthy

```sh
kubectl get deploy -n <tunnel-namespace>
```

Check the service for the tunnel got created and has the correct ports:

```sh
kubectl get svc -n <tunnel-namespace>
```

A tunnels service should have 8123/TCP,8000/TCP,8001/TCP + any additional TCP ports from the tunnel spec.

Check the logs for the tunnel deployment:

```sh
kubectl logs -n <tunnel-namespace> deploy/<tunnel-name>
```

Check event in the tunnel namespace:

```sh
kubectl get events -n <tunnel-namespace> \
  --sort-by=.metadata.creationTimestamp
```

Common issues:

- The Uplink license has not been copied to the tunnel namespace.
- The tunnel secret referenced by `tokenRef` in the tunnel spec does not exist.
- You have reached the maximum number of tunnels allowed by your license.

See [troubleshoot the uplink operator](#uplink-operator) if the deployment or service for the tunnel does not exist.

## Uplink operator

The uplink operator manages the lifecycle of tunnels, creating deployments, services, and secrets for each tunnel.

Check the uplink operator is running:

```sh
kubectl get deploy/uplink-operator -n inlets
```

Issues creating the tunnel resources will be shown in the logs.

```sh
kubectl logs -n inlets deploy/uplink-operator -f
```

The logs will show you if a tunnel did not get reconciled because you reached the maximum number of tunnels allowed by your license.

## Client router

The client router handles incoming WebSocket connections from tunnel clients and routes them to the appropriate tunnel server.

Check the client router is running:

```sh
kubectl get deploy/client-router -n inlets
```

All client connection attempts are logged by the router. If there are any issues reaching the upstream tunnel server will be shown in the logs.

```sh
kubectl logs -n inlets deploy/client-router -f
```

If client connections are not showing up in the client-router logs make sure to check the client router is reachable. Check your ingress configuration and the logs of your ingress server.

## Network reliability issues

Some protocols may experience network reliability issues under the standard inlets uplink mode due to Head-of-Line (HOL) blocking or slow reader/writer problems that can affect connection stability.

### Demux Mode

The demux (demultiplexed) mode addresses these issues by opening a separate WebSocket connection for each remote connection to a forwarded port. This prevents slow or problematic connections from affecting other connections on the same tunnel.

Enable demux mode by setting the `uplink_demux` environment variable in the Tunnel spec:

```diff
apiVersion: uplink.inlets.dev/v1alpha1
kind: Tunnel
metadata:
  name: kubernetes-api
  namespace: tenant1
spec:
+ env:
+   uplink_demux: "1"
  licenseRef:
    name: inlets-uplink-license
    namespace: tenant1
  tcpPorts:
    - 6443
```

## Proxy Protocol Issues

When using TCP tunnels, Proxy Protocol can be a common source of connectivity issues. Proxy Protocol is used to preserve the original client IP address when traffic passes through multiple proxy layers.

### Common Proxy Protocol Problems

1. **Misconfigured Proxy Protocol at any hop**: If Proxy Protocol is enabled at one point in the chain but not properly handled at every subsequent hop, connections will fail.

2. **Version mismatch**: Ensure all components use the same Proxy Protocol version (v1 or v2).

3. **Service configuration**: The tunneled service must be configured to expect and parse Proxy Protocol headers when enabled.

### Troubleshooting Proxy Protocol

Check if your tunneled service supports Proxy Protocol and verify the configuration at each network hop:

- Load balancers (AWS ALB, Traefik, HAProxy)
- Ingress controllers
- Service meshes (Istio, Linkerd)
- The tunneled application itself

Disable Proxy Protocol temporarily to isolate whether it's the source of connection issues.

## Best Practices

### Minimize Proxy Hops

To ensure optimal tunnel performance and reliability, minimize the number of hops through proxies for any tunnels. Each additional proxy hop can introduce latency, potential points of failure, and complicate troubleshooting. Direct connections or configurations with fewer intermediary proxies will generally provide better performance and stability.
