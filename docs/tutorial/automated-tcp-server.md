# Create a tunnel server with inletsctl

When should you create a TCP tunnel server?

It is advisable to only tunnel TCP services which include their own encryption such as TLS, SSH, RDP, or a Reverse Proxy. To expose a plaintext HTTP endpoint such as `http://localhost:3000`, use an [HTTPS tunnel](/docs/tutorial/automated-http-server.md) instead which will provide TLS to your end-users.

TCP tunnel servers can be [set up manually](/docs/tutorial/manual-tcp-server.md) or automatically with inletsctl.

This page shows the steps needed to create a tunnel server via inletsctl.

For a step-by-step guide on exposing a TCP service such as SSH, see [Expose SSH over a TCP tunnel](/docs/tutorial/ssh-tcp-tunnel.md).

## Obtain a cloud API token

inletsctl provisions a new cloud VM with inlets preinstalled using cloud-init.

You'll need to obtain an API token from your provider of choice using [the inletsctl reference documentation](/docs/reference/inletsctl.md).

## Create a tunnel server via inletsctl

Once you've obtained an API token, you can create a tunnel server with the following command:

```bash
export PROVIDER=""
export REGION=""
export ACCESS_TOKEN_FILE_PATH=""

inletsctl create \
    --provider $PROVIDER \
    --access-token-file $ACCESS_TOKEN_FILE_PATH \
    --region $REGION \
    --tcp
```

This will create a new VM in the London region and install inlets-pro on it.

The command will output a sample command for the `inlets-pro client` command:

## Run the tunnel client

The `inletsctl create` command will output a sample command for the `inlets-pro client` command.

To expose SSH from localhost, add:

```
 --upstream 127.0.0.1 \
 --port 2222
```

To expose SSH from a remote machine on your local network i.e. `192.168.1.20`, add:

```
 --upstream 192.168.1.20 \
 --port 2222
```

To expose ports 80 and 443 from a machine where you have a reverse proxy running such as Caddy, add:

```
 --upstream 192.168.1.20 \
 --port 80 \
 --port 443
```

