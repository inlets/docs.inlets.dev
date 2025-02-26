# IP filtering

HTTP and TCP tunnels support filtering of incoming requests by IP address through an allow list.

When enabled, only requests from the specified IP addresses or CIDR ranges will be allowed to connect.

You can read more about this feature [in the announcement](https://inlets.dev/blog/2021/10/15/allow-lists.html)

## How to enable IP filtering

To enable IP filtering, you can use the `--allow-cidr` flag to the inlets-pro http or tcp server commands.

You can log into a [host created by inletsctl](/docs/tutorial/automated-http-server.md), or you can [create a host manually](/docs/tutorial/manual-http-server.md).

The below example allows requests from the IP address `192.168.1.1` and the CIDR range `192.168.2.0/24`.

For HTTP tunnels:

```bash
inlets-pro http server \
    --allow-ips 192.168.1.1 \
    --allow-ips 192.168.1.0/24
```

For TCP tunnels:

```bash
inlets-pro tcp server \
    --allow-ips 192.168.1.1 \
    --allow-ips 192.168.1.0/24
```

