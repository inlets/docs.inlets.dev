# Local port forwarding

Local port forwarding is the opposite use-case of exposing a local service to the Public internet.

Instead, it tunnels a service from a remote machine to your local machine for access over localhost.

For example, you may have a private Prometheus, OpenFaaS, or Grafana dashboard running on a remote server, but you want to access it from your local machine.

Local port-forwarding can also be used with a tunnel server deployed in a Kubernetes cluster. In a related blog post, we describe how a user needed to access NATS for debugging purposes, but the NATS service was only accessible from within the cluster: [Reliable local port-forwarding from Kubernetes](https://inlets.dev/blog/2021/04/13/local-port-forwarding-kubernetes.html)

## Pre-requisites

* A remote server running something you wish to access locally
* A local machine with inlets-pro installed

## Enable local port forwarding on the server

The inlets tcp and http server command disables local port forwarding by default.

To enable it, use the `--client-forwarding` flag.

If you're running the binary directly, use:

```bash
inlets-pro tcp server --client-forwarding
```

Otherwise, run `systemctl cat inlets-pro` to find the service running on your system and add the flag to the `ExecStart` line.

```bash
ExecStart=/usr/local/bin/inlets-pro tcp server --client-forwarding
```

Whether you're running a HTTP server or TCP server, the flag is the same.

## Run the tunnel client

In the example where you want to bring Grafana back to your local machine, you can use:

```bash
inlets-pro [http/tcp] client \
    --local 3000:127.0.0.1:3000
```

Multiple `--local` flags can be used to forward multiple ports i.e. for both Grafana and Prometheus:

```bash
inlets-pro [http/tcp] client \
    --local 3000:127.0.0.1:3000 \
    --local 9090:127.0.0.1:9090
```

If the remote server is running on another machine, but is accessible from the tunnel server's network, you can use the remote server's IP address instead of `127.0.0.1`.

```bash
inlets-pro [http/tcp] client \
    --local 3000:10.0.0.2:3000 \
```

You can then access any of the tunnelled services over localhost.

```bash
curl http://localhost:3000
curl http://localhost:9090
```

TCP services can also be tunneled, and remapped to a different port on your local machine.

Here's an example for if SSH is not publicly accessible, but you want to access it over localhost.

```bash
inlets-pro tcp client \
    --local 2222:127.0.0.1:22
```

You can then access the SSH server over localhost on port 2222.

```bash
ssh -p 2222 localhost
```

