## Setting up a HTTP tunnel server manually

In this tutorial we will set up a HTTP tunnel serve manually, without any provisioning tools like inletsctl.

This may be useful for understanding how the server binary works, and how to use it on existing servers that you may have.

## Pre-reqs

* A Linux server, Windows and MacOS are also supported
* The inlets-pro binary at /usr/local/bin/
* Access to a DNS control plane for a domain you control

## Run the server

For this example, your tunnel server should be accessible from the Internet. The tunnel client will connect to it and then expose one or more local websites so that you can access them remotely.

Create a DNS A record for the subdomain or subdomains you want to use, and have each of them point to the public IP address of the server you have provisioned. These short have a short TTL such as 60s to avoid waiting too long for DNS to propagate throughout the Internet. You can increase this value to a higher number later.

First generate an authentication token that the client will use to log in:

```bash
TOKEN="$(head -c 32 /dev/urandom | base64 | cut -d "-" -f1)"
```

We'll use the built-in support for Let's Encrypt to get a valid HTTPS certificate for any services you wish to expose via your tunnel server.

```bash
export DOMAIN="example.com"

  inlets-pro http server \
  --auto-tls \
  --control-port 8123 \
  --auto-tls-san 192.168.0.10 \
  --letsencrypt-domain subdomain1.$DOMAIN \
  --letsencrypt-domain subdomain2.$DOMAIN \
  --letsencrypt-email contact@$DOMAIN \
  --letsencrypt-issuer staging
  --token $TOKEN
```

Notice that `--letsencrypt-domain` can be provided more than one, for each of your subdomains.

We are also defaulting to the "staging" provider for TLS certificates which allows us to obtain a large number of certificates for experimentation purposes only. The default value, if this field is left off is `prod` as you will see by running `inlets-pro http server --help`.

Now the following will happen:

* The tunnel server will start up and listen to TCP traffic on port 80 and 443.
* The server will try to resolve each of your domains passed via `--letsencrypt-domain`.
* Then once each resolves, Let's Encrypt will be contacted for a HTTP01 ACME challenge.
* Once the certificates are obtained, the server will start serving the HTTPS traffic.

Now you can connect your client running on another machine.

Of course you can tunnel whatever HTTP service you like, if you already have one.

Inlets has a built-in HTTP server that we can run on our local / private machine to share files with others. Let's use that as our example:

```bash
mkdir -p /tmp/share

echo "Welcome to my filesharing service." > /tmp/share/welcome.txt

inlets-pro fileserver \
 --allow-browsing \
 --webroot /tmp/share/
 --port 8080
```

Next let's expose that local service running on localhost:8080 via the tunnel server:

```bash
export TOKEN="" # Obtain this from your server
export SERVER_IP="" # Your server's IP
export DOMAIN="example.com"

inlets-pro http client \
  --url wss://$SERVER_IP:8123 \
  --token $TOKEN \
  --upstream http://localhost:8080/
```

If you set up your server for more than one sub-domain then you can specify a domain for each local service such as:

```bash
  --upstream subdomain1.$DOMAIN=http://localhost:8080/,subdomain2.$DOMAIN=http://localhost:3000/
```

Now that your client is connected, you can access the HTTP fileserver we set up earlier via the public DNS name:

```
curl -k -v https://subdomain1.$DOMAIN/welcome.txt
```

Now that you can see everything working, with a staging certificate, you can run the server command again and switch out the `--letsencrypt-issuer staging` flag for `--letsencrypt-issuer prod`.

## Wrapping up

You have now installed an inlets HTTP tunnel server to a machine by hand. The same can be achieved by running the inletsctl tool, which does all of this automatically on a number of cloud providers.

* Can I connect more than one client to the same server?
    Yes, and each can connect difference services. So client 1 exposes subdomain1.$DOMAIN and client 2 exposes subdomain2.$DOMAIN. Alternatively, you can have multiple clients exposing the same domain, for high availability.

* How do I keep the inlets server process running?
    You can run it in the background, by using a systemd unit file. You can generate these via the `inlets-pro http server --generate=systemd` command.

* How do I keep the inlets client process running?
    Do the same as for a server, but use the `inlets-pro http client --generate=systemd` command.

* What else can I do with my server?
    Browse the available options for the tunnel servers with the `inlets-pro http server --help` command.

