# Troubleshooting

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

## Best Practices

### Minimize Proxy Hops

To ensure optimal tunnel performance and reliability, minimize the number of hops through proxies for any tunnels. Each additional proxy hop can introduce latency, potential points of failure, and complicate troubleshooting. Direct connections or configurations with fewer intermediary proxies will generally provide better performance and stability.
