## Setting up dual TCP and HTTPS tunnels

In this tutorial we will set both a dual tunnel for exposing HTTP and TCP services from the same server. 

Whilst it's easier to automate two separate servers or cloud instances for your tunnels, you may want to reduce your costs.

The use-case may be that you have a number of OpenFaaS functions running on your Raspberry Pi which serve traffic to users, but you also want to connect via SSH and VNC.

## Pre-reqs

* A Linux server, Windows and MacOS are also supported
* The inlets-pro binary at /usr/local/bin/
* Access to a DNS control plane for a domain you control

## Create the HTTPS tunnel server first

Create a HTTPS tunnel server using the [manual tutorial](/tutorial/manual-http-server/) or [automated tutorial](/tutorial/automated-http-server/).

Once it's running, check you can connect to it, and then log in with SSH.

You'll find a systemd service named `inlets-pro` running the HTTPS tunnel with a specific authentication token and set of parameters.

Now, generate a new systemd unit file for the TCP tunnel.

I would suggest generating a new token for this tunnel.

```bash
TOKEN="$(head -c 32 /dev/urandom | base64 | cut -d "-" -f1)"

# Find the instance's public IPv4 address:
PUBLIC_IP="$(curl -s https://checkip.amazonaws.com)"
```

Let's imagine the public IP resolved to `46.101.128.5` which is part of the DigitalOcean range.

```bash
inlets-pro tcp server \
 --token "$TOKEN" \
 --auto-tls-san $PUBLIC_IP \
 --generate=systemd > inlets-pro-tcp.service
```

Example:

```ini
[Unit]
Description=inlets Pro TCP Server
After=network.target

[Service]
Type=simple
Restart=always
RestartSec=5
StartLimitInterval=0
ExecStart=/usr/local/bin/inlets-pro tcp server --auto-tls --auto-tls-san=46.101.128.5 --control-addr=0.0.0.0 --token="k1wCR+2j41TXqqq/UTLJzcuzhmSJbU5NY32VqnNOnog=" --control-port=8124 --auto-tls-path=/tmp/inlets-pro-tcp

[Install]
WantedBy=multi-user.target
```

We need to update the control-port for this inlets tunnel server via the `--control-port` flag. Use port 8124 since 8123 is already in use by the HTTP tunnel. Add `--control-port 8124` to the `ExecStart` line.

We need to add a new flag so that generated TLS certificates are placed in a unique directory, and don't clash. Add `--auto-tls-path /tmp/inlets-pro-tcp/` to the same line.

Next install the unit file with:

```bash
sudo cp inlets-pro-tcp.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable inlets-pro-tcp.service

sudo systemctl restart inlets-pro-tcp.service
```

You'll now be able to check the logs for the server:

```bash
sudo journalctl -u inlets-pro-tcp
```

Finally you can connect your TCP client:

```bash
inlets-pro tcp client \
  --token "k1wCR+2j41TXqqq/UTLJzcuzhmSJbU5NY32VqnNOnog=" \
  --upstream 192.168.0.15 \
  --ports 2222,5900 \
  --url wss://46.101.128.5:8124
```

Note that `5900` is the default port for VNC. Port `2222` was used for SSH as not to conflict with the version running on the tunnel server.

You can now connect to the public IP of your server via SSH and VNC:

For example:

```bash
ssh -p 2222 pi@46.101.128.5
```

## Wrapping up

You now have a TCP and HTTPS tunnel server running on the same host. This was made possibly by changing the control-plane port and auto-TLS path for the second server, and having it start automatically through a separate systemd service.

This technique may save you a few dollars per month, but it may not be worth your time compared to how quick and easy it is to create two separate servers with `inletsctl create`.
