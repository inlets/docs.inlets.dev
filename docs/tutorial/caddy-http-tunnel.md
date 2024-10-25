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

Do not run this command in your home folder, as it will expose your entire home directory.

Instead, create a temporary directory and serve that instead:

```bash
mkdir -p /tmp/shared/
cd /tmp/shared/

echo "Hello world" > WELCOME.txt

inlets-pro fileserver --webroot ./ \
  --allow-browsing
```

The command listens on port `8080` by default, but you can change is as desired with `--port`

The `--allow-browsing` flag allows directory listing and traversal through the browser.

If you're sharing files with a colleague or friend, you can add `--allow-browsing=false` and share the exact URL with them instead.

## Start the inlets-pro client on your local side

Downloads the inlets Pro client:

```sh
sudo inletsctl download
```

Run the inlets-pro client, using the TOKEN and IP given to you from the previous step.

The client will look for your license in `$HOME/.inlets/LICENSE`, but you can also use the `--license/--license-file` flag if you wish.

```sh
export IP=""        # take this from the exit-server
export TOKEN=""     # take this from the exit-server

inlets-pro tcp client \
  --url wss://$IP:8123/connect \
  --ports 80,443 \
  --token $TOKEN \
  --upstream localhost
```

Note that `--upstream localhost` will connect to Caddy running on your computer, if you are running Caddy on another machine, use its IP address here.

## Setup Caddy 2.x

Here's an example Caddyfile that will reverse-proxy to the local file-server using the domain name `service.example.com`:

```Caddyfile
{
  acme_ca https://acme-staging-v02.api.letsencrypt.org/directory
}

service.example.com

reverse_proxy 127.0.0.1:8080 {
}
```

Note the `acme_ca` being used will receive a staging certificate, remove it to obtain a production TLS certificate.

Now [download Caddy 2.x](https://caddyserver.com/download) for your operating system. You can get it from the downloads page, or if you're a Linux user on an amd64 or arm64 machine, you can use arkade to do everything required via `arkade system install caddy`. See `arkade system install --help` for more options.

Once you have the binary, you can run it with the following command:

```bash
sudo ./caddy run \
  -config ./Caddyfile
```

`sudo` - is required to bind to port 80 and 443, although you can potentially update your OS to allow binding to low ports without root access. See this [StackOverflow question for more](https://superuser.com/questions/710253/allow-non-root-process-to-bind-to-port-80-and-443).

You should now be able to access the fileserver via the `https://service.example.com` URL.

If you wanted to expose something else like Grafana, you could simply edit your Caddyfile's `reverse_proxy` line, then restart Caddy.

Caddy also supports multiple domains within the same file, so that you can expose multiple internal or private websites through the same tunnel.

```Caddyfile
{
  email "webmaster@example.com"
}

blog.example.com {
  reverse_proxy 127.0.0.1:4000
}

openfaas.example.com {
      reverse_proxy 127.0.0.1:8080
}
```

If you have services running on other machines you can change `127.0.0.1:8080` to a different IP address such as that of your Raspberry Pi if you had something like [OpenFaaS CE](https://github.com/openfaas/faas) or [faasd CE](https://github.com/openfaas/faasd) running there.

## Check it all worked

You'll see that Caddy can now obtain a TLS certificate.

Go ahead and visit: `https://service.example.com`

Congratulations, you've now served a TLS certificate directly from your laptop. You can close caddy and open it again at a later date. Caddy will re-use the certificate it already obtained and it will be valid for 3 months. To renew, just keep Caddy running or open it again whenever you need it.
