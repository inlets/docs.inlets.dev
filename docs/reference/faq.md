# Inlets FAQ

Inlets concepts and Frequently Asked Questions (FAQ)

## Why did we build inlets?

We built inlets to make it easy to expose a local service on the Internet and to overcome limitations with SaaS tunnels and VPNs. 

* It was built to overcome limitations in SaaS tunnels - such as lack of privacy, control and rate-limits
* It doesn't just integrate with containers and Kubernetes, it was purpose-built to run in them
* It's easy to run on Windows, Linux and MacOS with a self-contained binary
* It doesn't need to run as root, doesn't depend on iptables, doesn't need a tun device or NET_ADMIN capability

There are many different networking tools available such as VPNs and SaaS tunnels - each with its own set of pros and cons, and use-cases. It's very likely that you will use several tools together to get the best out of each of them.

## How does inlets compare to other tools and solutions?

Are you curious about the advantages of using inlets vs. alternatives? We must first ask, advantages vs. what other tool or service.

SaaS tunnels provide a convenient way to expose services for the purposes of development, however they are often:

* blocked by corporate IT
* running on shared infrastructure (servers) with other customers
* subject to stringent rate-limits that affect productivity
* priced per subdomain
* unable to obtain high value TCP ports like 22, 80, 443 and so on

You run inlets on your own servers, so you do not run into those restrictions. Your data remains your own and is kept private.

When compared to VPNs such as Wireguard, Tailscale and OpenVPN, we have to ask what the use-case is.

A traditional VPN is built *to connect hosts and entire IP ranges together*. This can potentially expose a large number of machines and users to each other and requires complex Access Control Lists or authorization rules. If this is your use-case, a traditional VPN is probably the right tool for the job.

Inlets is designed to connect or expose services between networks - either HTTP or TCP.

For example:

* Receiving webhooks to a local application
* Sharing a blog post draft with a colleague or client
* Providing remote access to your homelab when away from home
* Self-hosting websites or services on Kubernetes clusters
* Getting working LoadBalancers with public IPs for local Kubernetes clusters

You can also use inlets to replace Direct Connect or a VPN when you just need to connect a number of services privately and not an entire network range.

Many of the inlets community use a VPN alongside inlets, because they are different tools for different use-cases.

> We often write about use-cases for public and private inlets tunnels [on the blog](https://inlets.dev/blog/).

## What's the difference between inlets, inletsctl and inlets-operator?

[inlets-pro](https://github.com/inlets/inlets-pro) aka "inlets" is the command-line tool that contains both the client and server required to set up HTTP and TCP tunnels.

The inlets-pro server is usually set up on a computer with a public IP address, then the inlets-pro client is run on your own machine, or a separate computer that can reach the service or server you want to expose.

You can download inlets-pro and inletsctl with the "curl | sh" commands provided at the start of each tutorial, this works best on a Linux host, or with Git Bash if using Windows.

> Did you know? You can also download binaries for inlets-pro and inletsctl on GitHub, for Windows users you'll want "inlets-pro.exe" and for MacOS, you'll want "inlets-pro-darwin".

For instance, on Windows machines you'll need "inlets-pro.exe"

See also: [inlets-pro releases](https://github.com/inlets/inletsctl/releases)

[inletsctl](https://github.com/inlets/inletsctl) is a tool that can set up a tunnel server for you on around a dozen popular clouds. It exists to make it quicker and more convenience to set up a HTTPS or TCP tunnel to expose a local service.

It has three jobs:

1. Create the VM for you
2. Install the inlets-pro server in TCP or HTTPS mode (as specified) with systemd
3. Inform you of the token and connection string

You can download the inletsctl tool with "curl | sh" or from the [inletsctl releases](https://github.com/inlets/inletsctl/releases) page.

Find out more: [inletsctl reference page](/reference/inletsctl)

[inlets-operator](https://github.com/inlets/inlets-operator) is a Kubernetes Operator that will create tunnel servers for you, on your chosen cloud for any LoadBalancers that you expose within a private cluster.

Find out more: [inlets-operator reference page](/reference/inlets-operator)

## What is the networking model for inlets?

Whilst some networking tools such as Bittorrent use a peer-to-peer network, inlets uses a more traditional client/server model.

One or more client tunnels connect to a tunnel server and advertise which services they are able to provide. Then, whenever the server receives traffic for one of those advertised services, it will forward it through the tunnel to the client. The tunnel client will then forward that on to the service it advertised.

> The tunnel server may also be referred to as an "exit" server because it is the connection point for the client to another network or the Internet.

If you install and run the inlets server on a computer, it can be referred to as a *tunnel server* or *exit server*. These servers can also be automated through cloud-init, terraform, or tools maintained by the inlets community such as [inletsctl](/reference/inletsctl/).

![Conceptual architecture](/images/conceptual.png)

> Pictured: the website `http://127.0.0.1:3000` is exposed through an encrypted tunnel to users at: `https://example.com`

For remote forwarding, the client tends to be run within a private network, with an `--upstream` flag used to specify where incoming traffic needs to be routed. The tunnel server can then be run on an Internet-facing network, or any other network reachable by the client.

## What kind of layers and protocols are supported?

Inlets works at a higher level than traditional VPNs because it is designed to connect services together, rather than hosts directly.

* HTTP - Layer 7 of the OSI model, used for web traffic such as websites and RESTful APIs
* TCP - Layer 4 of the OSI model, used for TCP traffic like SSH, TLS, databases, RDP, etc

Because VPNs are designed to connect hosts together over a shared IP space, they also involve tedious IP address management and allocation.

Inlets connects services, so for TCP traffic, you need only think about TCP ports.

For HTTP traffic, you need only to think about domain names.

## Do I want a TCP or HTTPS tunnel?

If you're exposing websites, blogs, docs, APIs and webhooks, you should use a HTTPS tunnel.

For HTTP tunnels, Rate Error and Duration (RED) metrics are collected for any service you expose, even if it doesn't have its own instrumentation support.

For anything that doesn't fit into that model, a TCP tunnel may be a better option.

Common examples are: RDP, VNC, SSH, TLS, database protocols, legacy medical protocols such as DiCom.

TCP tunnels can also be used to forward traffic to a reverse proxy like Nginx, Caddy, or Traefik, sitting behind a firewall or NAT by forwarding port 80 and 443.

TCP traffic is forwarded directly between the two hosts without any decryption of bytes. The active connection count and frequency can be monitored along with the amount of throughput.

## Does inlets use TCP or UDP?

Inlets uses a websocket over TCP, so that it can penetrate HTTP proxies, captive portals, firewalls, and other kinds of NAT. As long as the client can make an outbound connection, a tunnel can be established. The use of HTTPS means that inlets will have similar latency and throughput to a HTTPS server or SSH tunnel.

Once you have an inlets tunnel established, you can use it to tunnel traffic to TCP and HTTPS sockets within the private network of the client.

Most VPNs tend to use UDP for communication due to its low overhead which results in lower latency and higher throughput. Certain tools and products such as OpenVPN, SSH and Tailscale can be configured to emulate a TCP stack over a TCP connection, this can lead to [unexpected issues](http://sites.inka.de/~bigred/devel/tcp-tcp.html).

Inlets connections send data, rather than emulating a TCP over TCP stack, so doesn't suffer from this problem.

## Are both remote and local forwarding supported?

Remote forwarding is where a local service is forwarded from the client's network to the inlets tunnel server.

![Remote forwarding pushes a local endpoint to a remote host for access on another network](https://inlets.dev/images/2021-04-23-local-forwarding/remote-forwarding.jpg)

> Remote forwarding pushes a local endpoint to a remote host for access on another network

This is the most common use-case and would be used to expose a local HTTP server to the public Internet via a tunnel server.

Local forwarding is used to forward a service on the tunnel server or tunnel server's network back to the client, so that it can be accessed using a port on localhost.

![Local forwarding brings a remote service back to localhost for accessing](https://inlets.dev/images/2021-04-23-local-forwarding/local-forwarding.jpg)
> Local forwarding brings a remote service back to localhost for accessing

An example would be that you have a webserver and MySQL database. The HTTP server is public and can access the database via its own loopback adapter, but the Internet cannot. So how do you access that MySQL database from CI, or from your local machine? Connect a client with local forwarding, and bring the MySQL port back to your local machine or CI runner, and then use the MySQL CLI to access it.

A developer at the UK Government uses inlets to forward a NATS message queue from a staging environment to his local machine for testing. [Learn more](https://inlets.dev/blog/2021/04/13/local-port-forwarding-kubernetes.html)

## What's the difference between the data plane and control plane?

The data plane is any service or port that carries traffic from the tunnel server to the tunnel client, and your private TCP or HTTP services. It can be exposed on all interfaces, or only bound to loopback for private access, in a similar way to a VPN.

If you were exposing SSH on an internal machine from port `2222`, your data-plane may be exposed on port `2222`

The control-plane is a TLS-encrypted, authenticated websocket that is used to connect clients to servers. All traffic ultimately passes over the control-plane's link, so remains encrypted and private.

Your control-plane's port is usually `8123` when used directly, or `443` when used behind a reverse proxy or Kubernetes Ingress Controller.

An example from the article: [The Simple Way To Connect Existing Apps to Public Cloud](https://inlets.dev/blog/2021/04/07/simple-hybrid-cloud.html)

A legacy MSSQL server runs on Windows Server behind the firewall in a private datacenter. Your organisation cannot risk migrating it to an AWS EC2 instance at this time, but can move the microservice that needs to access it.

The inlets tunnel allows for the MSSQL service to be tunneled privately to the EC2 instance's local network for accessing, but is not exposed on the Internet. All traffic is encrypted over the wire due to the TLS connection of inlets.

![Hybrid Cloud in action using an inlets tunnel to access the on-premises database](https://inlets.dev/images/2021-simple-hybrid-cloud/hybrid-in-action.jpg)

> Hybrid Cloud in action using an inlets tunnel to access the on-premises database

This concept is referred to as a a "split plane" because the control plane is available to public clients on all adapters, and the data plane is only available on local or private adapters on the server.

## Is there a reference guide to the CLI?

The inlets-pro binary has built-in help commands and examples, just run `inlets-pro tcp/http client/server --help`.

A separate CLI reference guide is also available here: [inlets-pro CLI reference](https://github.com/inlets/inlets-pro/blob/master/docs/cli-reference.md)

## Is inlets secure?

All traffic sent over an inlets tunnel is encapsulated in a TLS-encrypted websocket, which prevents eavesdropping. This is technically similar to HTTPS, but you'll see a URL of `wss://` instead.

The tunnel client is authenticated using an API token which is generated by the tunnel administrator, or by automated tooling.

Additional authentication mechanisms can be set up using a reverse proxy such as Nginx.

## Do I have to expose services on the Internet to use inlets?

No, inlets can be used to tunnel one or more services to another network without exposing them on the Internet.

The `--data-addr 127.0.0.1:` flag for inlets servers binds the data plane to the server's loopback address, meaning that only other processing running on it can access the tunneled services. You could also use a private network adapter or VPC IP address in the `--data-addr` flag.

## How do I monitor inlets?

See the following blog post for details on the `inlets status` command and the various Prometheus metrics that are made available.

[Measure and monitor your inlets tunnels](https://inlets.dev/blog/2021/08/18/measure-and-monitor.html)

## How do you scale inlets?

Inlets HTTP servers can support a high number of clients, either for load-balancing the same internal service to a number of clients, or for a number of distinct endpoints.

Tunnel servers are easy to scale through the use of containers, and can benefit from the resilience that a Kubernetes cluster can bring:

See also: [How we scaled inlets to thousands of tunnels with Kubernetes](https://inlets.dev/blog/2021/03/15/scaling-inlets.html)

## Does inlets support High Availability (HA)?

For the inlets client, it is possible to connect multiple inlets tunnel clients for the same service, such as a company blog. Traffic will be distributed across the clients and if one of those clients goes down or crashes, the other will continue to serve requests.

For the inlets tunnel server, the easiest option is to run the server in a supervisor that can restart the tunnel service quickly or allow it to run more than one replica. Systemd can be used to restart tunnel servers should they run into issues, likewise you can run the server in a container, or as a Kubernetes Pod.

![HA VIP](https://github.com/inlets/inlets-pro/blob/master/docs/images/inlets-pro-vip-ha.png?raw=true)

> HA example with an AWS ELB

For example, you may place a cloud load-balancer in front of the data-plane port of two inlets server processes. Requests to the stable load-balancer IP address will be distributed between the two virtual machines and their respective inlets server tunnel processes.  

## Is IPv6 supported?

Yes, see also: [How to serve traffic to IPv6 users with inlets](https://inlets.dev/blog/2020/11/16/inlets-the-ipv6-proxy.html)

## What if the websocket disconnects?

The client will reconnect automatically and can be configured with systemd or a Windows service to stay running in the background. See also `inlets pro tcp/http server/client --generate=systemd` for generating systemd unit files.

When used in combination with a Kubernetes ingress controller or reverse proxy of your own, then the websocket may timeout. These timeout settings can usually be configured to remove any potential issue.

Monitoring in inlets allows for you to monitor the reliability of your clients and servers, which are often running in distinct networks.

## How much does inlets cost?

Monthly and annual subscriptions are available via Gumroad.

You can also purchase a static license for offline or air-gapped environments.

For more, see the [Pricing page](https://inlets.dev/pricing/)

## What happens when the license expires?

If you're using a Gumroad license, and keep your billing relationship active, then the software will work for as long as you keep paying. The Gumroad license server needs to be reachable by the inlets client.

If you're using a static license, then the software will continue to run, even after your license has expired, unless you restart the software. You can either rotate the token on your inlets clients in an automated or manual fashion, or purchase a token for a longer period of time up front.

## Can I get professional help?

Inlets is designed to be self-service and is well documented, but perhaps you could use some direction?

Business licenses come with support via email, however you are welcome to [contact OpenFaaS Ltd](https://inlets.dev/contact) to ask about a consulting project.
