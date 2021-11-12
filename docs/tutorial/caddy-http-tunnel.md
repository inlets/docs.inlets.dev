## Custom reverse proxy with Caddy

In this tutorial we'll set up an inlets TCP tunnel server to forward ports 80 and 443 to a reverse proxy server running on our local machine. Caddy will receive a TCP stream from the public tunnel server for ports 80 and 443. It can terminate TLS and also allow you to host multiple sites with ease.

[Caddy](https://caddyserver.com) is a free and open-source reverse proxy. It's often used on web-servers to add TLS to one or more virtual websites.

## Pre-reqs

* A Linux server, Windows and MacOS are also supported
* The inlets-pro binary at /usr/local/bin/
* Access to a DNS control plane for a domain you control

You can run through the same instructions with other reverse proxies such as [Nginx](http://nginx.org), or [Traefik](https://traefik.io).

Scenario:
* You want to share a file such as a VM image or a ISO over the Internet, with HTTPS, directly from your laptop.
* You have one or more websites or APIs running on-premises or within your home-lab and want to expose them on the Internet.

> You can [subscribe to inlets for personal or commercial use via Gumroad](https://inlets.dev/blog/2021/07/27/monthly-subscription.html)

## Setup your exit node

Provision a cloud VM on DigitalOcean or another IaaS provider using [inletsctl](https://github.com/inlets/inletsctl):

```bash
inletsctl create \
 --provider digitalocean \
 --region lon1 \
 --pro
```

Note the `--url` and `TOKEN` given to you in this step.

## Setup your DNS A record

Setup a DNS A record for the site you want to expose using the public IP of the cloud VM

* `178.128.40.109` = `service.example.com`

## Run a local server to share files

Do not run this command in your home folder.

```bash
mkdir -p /tmp/shared/
cd /tmp/shared/

echo "Hello world" > WELCOME.txt

# If using Python 2.x
python -m SimpleHTTPServer

# Python 3.x
python3 -m http.server
```

This will listen on port `8000` by default.

## Setup Caddy 1.x

* Download the latest Caddy 1.x binary from the [Releases page](https://github.com/caddyserver/caddy/releases)

Pick your operating system, for instance Darwin for MacOS, or Linux.

Download the binary, extract it and install it to `/usr/local/bin`:

```bash
mkdir -p /tmp/caddy
curl -sLSf https://github.com/caddyserver/caddy/releases/download/v1.0.4/caddy_v1.0.4_darwin_amd64.zip > caddy.tar.gz
tar -xvf caddy.tar.gz --strip-components=0 -C /tmp/caddy

sudo cp /tmp/caddy/caddy /usr/local/bin/
```

* Create a Caddyfile

The `Caddyfile` configures which websites Caddy will expose, and which sites need a TLS certificate.

Replace `service.example.com` with your own domain.

Next, edit `proxy / 127.0.0.1:8000` and change the port `8000` to the port of your local webserver, for instance `3000` or `8080`. For our example, keep it as `8000`.

```sh
service.example.com

proxy / 127.0.0.1:8000 {
  transparent
}
```

Start the Caddy binary, it will listen on port 80 and 443.

```
sudo ./caddy
```

If you have more than one website, you can add them to the Caddyfile on new lines.

> You'll need to run caddy as `sudo` so that it can bind to ports 80, and 443 which require additional privileges.

## Start the inlets-pro client on your local side

Downloads the inlets PRO client:

```sh
sudo inletsctl download
```

Run the inlets-pro client, using the TOKEN and IP given to you from the previous step.

The client will look for your license in `$HOME/.inlets/LICENSE`, but you can also use the `--license/--license-file` flag if you wish.

```sh
export IP=""        # take this from the exit-server
export TOKEN=""     # take this from the exit-server

inlets-pro client \
  --url wss://$IP:8123/connect \
  --ports 80,443 \
  --token $TOKEN \
  --upstream localhost
```

Note that `--upstream localhost` will connect to Caddy running on your computer, if you are running Caddy on another machine, use its IP address here.

## Check it all worked

You'll see that Caddy can now obtain a TLS certificate.

Go ahead and visit: `https://service.example.com`

Congratulations, you've now served a TLS certificate directly from your laptop. You can close caddy and open it again at a later date. Caddy will re-use the certificate it already obtained and it will be valid for 3 months. To renew, just keep Caddy running or open it again whenever you need it.

## Setup Caddy 2.x

For Caddy 2.x, the Caddyfile format changes.

Let's say you're running a Node.js service on port 3000, and want to expose it with TLS on the domain "service.example.com":

```
git clone https://github.com/alexellis/expressjs-k8s/
cd expressjs-k8s

npm install
http_port=3000 npm start
```

The local site will be served at http://127.0.0.1:3000

```Caddyfile
{
  acme_ca https://acme-staging-v02.api.letsencrypt.org/directory
}

service.example.com

reverse_proxy 127.0.0.1:3000 {
}
```

Note the `acme_ca` being used will receive a staging certificate, remove it to obtain a production TLS certificate.

Now [download Caddy 2.x](https://caddyserver.com/download) for your operating system.

```bash
sudo ./caddy run \
  -config ./Caddyfile
```

`sudo` - is required to bind to port 80 and 443, although you can potentially update your OS to allow binding to low ports without root access.

You should now be able to access the Node.js website via the `https://service.example.com` URL.

Caddy also supports multiple domains within the same file, so that you can expose multiple internal or private websites through the same tunnel.

```Caddyfile
{
  email "alex@openfaas.com"
}

blog.example.com {
  reverse_proxy 127.0.0.1:4000
}

openfaas.example.com {
      reverse_proxy 127.0.0.1:8080
}
```

If you have services running on other machines you can change `127.0.0.1:8080` to a different IP address such as that of your Raspberry Pi if you had something like [OpenFaaS](https://github.com/openfaas/) running there.

