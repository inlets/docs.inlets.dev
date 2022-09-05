# Monitoring and metrics

Learn how you can monitor your tunnel servers using the `status` command and Prometheus metrics.

This can help you understand how tunnels are being used and answer questions like:

  - What are the Rate, Error, Duration (RED) metrics for any HTTP APIs or websites that are being hosted?
  - How many connections are open at this point in time, and on which ports?
  - Have any clients attempted to connect which failed authentication?

## Introduction
All the information for monitoring tunnels is exposed via the inlets control-plane. It provides a connection endpoint for clients, a status endpoint and a monitoring endpoint.

> Checkout the FAQ to [learn about the difference between the data-plane and control-plane](https://docs.inlets.dev/reference/faq/#whats-the-difference-between-the-data-plane-and-control-plane)

Inlets provides two distinct ways to monitor tunnels. You can use the `status` command that is part of the CLI or collect Prometheus metrics for background monitoring and alerting. We will explore both methods.

## The status command
With the `inlets-pro status` command you can find out some basic tunnel statistics without logging in with a console SSH session. It shows you a list of the connected clients along with the version and uptime information of the server and can be used with both HTTP and TCP tunnels.

Here’s an example of a TCP tunnel server:

```bash
$ inlets-pro status \
  --url wss://178.62.70.130:8123 \
  --token "$TOKEN" \
  --auto-tls

Querying server status. Version DEV - unknown
Hostname: unruffled-banzai4
Started: 49 minutes
Mode: tcp
Version:        0.8.9-rc1

Client ID                        Remote Address     Connected Upstreams
730aa1bb96474cbc9f7e76c135e81da8 81.99.136.188:58102 15 minutes localhost:8001, localhost:8000, localhost:2222
22fbfe123c884e8284ee0da3680c1311 81.99.136.188:64018 6 minutes  localhost:8001, localhost:8000, localhost:2222
```

We can see the clients that are connected and the ports they make available on the server. In this case there are two clients. All traffic to the data plane for ports 8001, 8000 and 2222 will be load-balanced between the two clients for HA.

The response from a HTTP tunnel:

```bash
$ inlets-pro status \
  --url wss://147.62.70.101:8123 \
  --token "$TOKEN"  \
  --auto-tls

Server info:
Hostname: creative-pine6
Started: 1 day
Mode:           http
Version:        0.8.9-rc1
Connected clients:
Client ID                        Remote Address     Connected Upstreams
4e35edf5c6a646b79cc580984eac4ea9 192.168.0.19:34988 5 minutes example.com=http://localhost:8000, prometheus.example.com=http://localhost:9090
```

In this example we can see that there is only one client connected to the server at the moment. This client provides two separate domains.


The command uses the status endpoint that is exposed on the control-plane. It is possible to invoke the HTTP endpoint yourself. The token that is set up for the server has to be set in the Authorization header.

```bash
$ curl -ksLS https://127.0.0.1:8123/status \
  -H "Authorization: Bearer $TOKEN"
```

Example response from a HTTP tunnel:

```json
{
  "info": {
    "version": "0.8.9-18-gf4fc15b",
    "sha": "f4fc15b9604efd0b0ca3cc604c19c200ae6a1d7b",
    "mode": "http",
    "startTime": "2021-08-13T12:23:17.321388+01:00",
    "hostname": "am1.local"
  },
  "clients": [
    {
      "clientID": "0c5f2a1ca0174ee3a177c3be7cd6d950",
      "remoteAddr": "[::1]:63671",
      "since": "2021-08-13T12:23:19.72286+01:00",
      "upstreams": [
        "*=http://127.0.0.1:8080"
      ]
    }
  ]
}
```

## Monitor inlets with Prometheus
The server collects metrics for both the data-plane and the control-plane. These metrics are exposed through the monitoring endpoint on the control-plane. Prometheus can be set up for metrics collection and alerting.

The name of the metrics and the kind of metrics that are exported will depend on the mode that the server is running in. For TCP tunnels the metric name starts with `tcp_` for HTTP tunnels this will be `http_`.

You don’t need to be a Kubernetes user to take advantage of Prometheus. You can run it locally on your machine by [downloading the binary here](https://prometheus.io/download/).

> As an alternative, [Grafana Cloud](https://grafana.com/products/cloud/) can give you a complete monitoring stack for your tunnels without having to worry about finding somewhere to run and maintain Prometheus and Grafana. We have a write up on our blog that shows you how to set this up: [Monitor inlets tunnels with Grafana Cloud](https://inlets.dev/blog/2022/09/02/monitor-inlets-with-grafana.html).

Create a `prometheus.yaml` file to configure Prometheus. Replace TOKEN with the token from your server.

```yaml
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
    static_configs:
    - targets: ['localhost:9090']
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'http-tunnel'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
    static_configs:
    - targets: ['localhost:8123']
    scheme: https

    authorization:
      type: Bearer
      credentials: TOKEN
    tls_config:
      insecure_skip_verify: true

```

Start Prometheus with this command. It will listen on port 9090.

```bash
$ prometheus --config.file=./prometheus.yaml

level=info ts=2021-08-13T11:25:31.791Z caller=main.go:428 msg="Starting Prometheus" version="(version=2.29.1, branch=HEAD, revision=dcb07e8eac34b5ea37cd229545000b857f1c1637)"
level=info ts=2021-08-13T11:25:31.931Z caller=main.go:784 msg="Server is ready to receive web requests."
```

### Metrics for the control-plane
The control-plane metrics can give you insights into the number of clients that are connected and the number of http requests made to the different control-plane endpoints.

HTTP tunnels

| Metric | Type | Description | Labels |
|--------|------|-------------|--------|
| http_controlplane_connected_gauge | gauge | gauge of inlets clients connected to the control plane | |
| http_controlplane_requests_total | counter | total HTTP requests processed by connecting clients on the control plane | `code`, `path`|

TCP tunnels

| Metric | Type | Description | Labels |
|--------|------|-------------|--------|
| tcp_controlplane_connected_gauge | gauge | gauge of inlets clients connected to the control plane | |
| tcp_controlplane_requests_total | counter | total HTTP requests processed by connecting clients on the control plane | `code`, `path`|

These metrics can for instance be used to tell you whether there are a lot of clients that attempted to connect but failed authentication.

If running on Kubernetes, the connected gauge could be used to scale tunnels down to zero replicas, and back up again in a similar way to OpenFaaS. This could be important for very large-scale installations of devices or tenants that have partial connectivity.

### Metrics for the data-plane
The data-plane metrics can give you insights in the services that are exposed through your tunnel.

HTTP tunnels

| Metric | Type | Description | Labels |
|--------|------|-------------|--------|
| http_dataplane_requests_total | counter | total HTTP requests processed | `code`, `host`, `method` |
| http_dataplane_request_duration_seconds | histogram | Seconds spent serving HTTP requests. | `code`, `host`, `method` |


TCP tunnels

| Metric | Type | Description | Labels |
|--------|------|-------------|--------|
| tcp_dataplane_connections_gauge | gauge | gauge of TCP connections established over data plane | `port` |
| tcp_dataplane_connections_total | counter | total count of TCP connections established over data plane | `port` |


For HTTP tunnels these metrics can be used to get Rate, Error, Duration (RED) information for any API or website that is connected through the tunnel. This essentially allows you to collect basic metrics for your services even if they do not export any metrics themselves.

For TCP tunnels these metrics can help answer questions like:

- How many connections are open at this point in time, and on which ports? i.e. if exposing SSH on port 2222, how many connections are open?

## Wrapping up

We showed two different options that can be used to monitor your inlets tunnels.

The CLI provides a quick and easy way to get some status information for a tunnel. The endpoint that exposes this information can also be invoked directly using HTTP.

Prometheus metrics can be collected from the monitoring endpoint. These metrics are useful for background monitoring and alerting. They can provide you with Rate, Error, Duration (RED) metrics for HTTP services that are exposed through Inlets.

### You may also like
* [Blog post: Measure and monitor your inlets tunnels](https://inlets.dev/blog/2021/08/18/measure-and-monitor.html)