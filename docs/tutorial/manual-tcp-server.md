## Setting up a TCP server manually

In this tutorial we will set up a TCP tunnel server manually.

## Pre-reqs

* A Linux server, Windows and MacOS are also supported
* The inlets-pro binary at /usr/local/bin/

## Log into your existing VM

Generate an authentication token for the tunnel:

```bash
TOKEN="$(openssl rand -base64 32)" > token.txt

# Find the instance's public IPv4 address:
PUBLIC_IP="$(curl -s https://checkip.amazonaws.com)"
```

Let's imagine the public IP resolved to `46.101.128.5` which is part of the DigitalOcean range.

```bash
inlets-pro tcp server \
 --token "$TOKEN" \
 --auto-tls-san $PUBLIC_IP \
 --generate=systemd > inlets-pro.service
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
ExecStart=/usr/local/bin/inlets-pro tcp server --auto-tls --auto-tls-san=46.101.128.5 --control-addr=0.0.0.0 --token="ISgW7E2TQk+ZmbJldN9ophfE96B93eZKk8L1+gBysg4=" --control-port=8124 --auto-tls-path=/tmp/inlets-pro

[Install]
WantedBy=multi-user.target
```

Next install the unit file with:

```bash
sudo cp inlets-pro.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable inlets-pro.service

sudo systemctl restart inlets-pro.service
```

You'll now be able to check the logs for the server:

```bash
sudo journalctl -u inlets-pro
```

Finally you can connect your TCP client from a remote network. In this case, port 5900 is being exposed for VNC, along with port 2222 for SSH. Port 2222 is an extra port added to the `/etc/ssh/sshd_config` file on the Linux machine to avoid conflicting with SSH on the tunnel server itself.

```bash
inlets-pro tcp client \
  --token "ISgW7E2TQk+ZmbJldN9ophfE96B93eZKk8L1+gBysg4=" \
  --upstream 192.168.0.15 \
  --port 2222 \
  --port 5900 \
  --url wss://46.101.128.5:8124
```

You can now connect to the public IP of your server via SSH and VNC:

For example:

```bash
ssh -p 2222 pi@46.101.128.5
```

## Wrapping up

You now have a TCP tunnel server that you can connect as and when you like.

* You can change the ports of the connected client
* You can change the upstream
* You can run multiple `inlets-pro tcp client` commands to load-balance traffic

But bear in mind that you cannot have two clients exposing different ports at the same time unless you're an [inlets uplink user](/uplink/become-a-provider).

We would recommend creating TCP tunnel servers via [inletsctl](/tutorial/ssh-tcp-tunnel) which automates all of the above in a few seconds.
