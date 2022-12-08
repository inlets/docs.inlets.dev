# Monitoring inlets uplink

Inlets Uplink comes with an integrated Prometheus deployment that automatically collects metrics for each tunnel.

!!! note
    Prometheus is deployed with Inlets Uplink by default. If you don't need monitoring you can disable it in the `values.yaml` of the Inlets Uplink Helm chart:

    ```yaml
    prometheus:
      create: false
    ```

## Metrics for the control-plane

The control-plane metrics can give you insights into the number of clients that are connected and the number of http requests made to the control-plane endpoint for each tunnel.

| Metric | Type | Description | Labels |
|--------|------|-------------|--------|
| controlplane_connected_gauge | gauge | gauge of inlets clients connected to the control plane | `tunnel_name` |
| controlplane_requests_total | counter | total HTTP requests processed by connecting clients on the control plane | `code`, `tunnel_name`|

## Metrics for the data-plane

The data-plane metrics can give you insights in the services that are exposed through your tunnel.

| Metric | Type | Description | Labels |
|--------|------|-------------|--------|
| dataplane_connections_gauge | gauge | gauge of connections established over data plane | `port`, `type`, `tunnel_name` |
| dataplane_connections_total | counter | total count of connections established over data plane | `port`, `type`, `tunnel_name` |
| dataplane_requests_total | counter | total HTTP requests processed | `code`, `host`, `method`, `tunnel_name` |
| dataplane_request_duration_seconds | histogram | seconds spent serving HTTP requests | `code`, `host`, `method`, `tunnel_name`, |

The connections metrics show the number of connections that are open at this point in time, and on which ports. The `type` label indicates whether the connection is for a `http` or `tcp` upstream.

The request metrics only include HTTP upstreams. These metrics can be used to get Rate, Error, Duration (RED) information for any API or website that is connected through the tunnel.