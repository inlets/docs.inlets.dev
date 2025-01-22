# Inlets

[Inlets](https://inlets.dev) is a cloud-native tunnel built to run over restrictive networks, NAT and firewalls. It was built to connect bare-metal, VMs, containers and Kubernetes to the cloud.

[![Inlets logo](/images/inlets-hero.png){ width="80%"}](https://inlets.dev/)

> You can visit the inlets homepage at [https://inlets.dev/](https://inlets.dev/)

With inlets you are in control of your data, unlike with a SaaS tunnel where shared servers mean your data may be at risk. You can use inlets for local development and in your production environment. It works just as well on bare-metal as in VMs, containers and Kubernetes clusters.

inlets is not just compatible with tricky networks and Cloud Native architecture, it was purpose-built for them.

Common use-cases include:

* Exposing local HTTPS, TCP, or websocket endpoints on the Internet
* Replacing SaaS tunnels that are too restrictive
* Self-hosting from a homelab or on-premises datacenter
* Deploying and monitoring apps across multiple locations
* Receiving webhooks and testing OAuth integrations
* Remote customer support

Do you want to scale to dozens, hundreds or thousands of tunnels? You may be looking for [inlets uplink](https://docs.inlets.dev/uplink/overview/)

## How does it work?

Inlets tunnels connect to each other over a secure websocket with TLS encryption. Over that private connection, you can then tunnel HTTPS or TCP traffic to computers in another network or to the Internet.

One of the most common use-cases is to expose a local HTTP endpoint on the Internet via a HTTPS tunnel. You may be working with webhooks, integrating with OAuth, sharing a draft of a blog post or integrating with a partner's API.

![Access a local service remotely](https://inlets.dev/images/quick.png){ width="60%" }

> After deploying an inlets HTTPS server on a public cloud VM, you can then connect the client and access the local service from the Internet.

There is more that inlets can do for you than exposing local endpoints. inlets also supports local forwarding and can be used to replace legacy solutions like SSH and VPNs.

Learn how inlets compares to VPNs and other solutions in the [inlets FAQ](/reference/faq/).

## Getting started

These guides walk you through a specific use-case with inlets. If you have questions or cannot find what you need, there are options for connecting with the community at the end of this page.

Inlets can tunnel either HTTP or TCP traffic:

* HTTP (L7) tunnels can be used to connect one or more HTTP endpoints from one network to another. A single tunnel can expose multiple websites or hosts, including LoadBalancing and multiple clients to one server.
* TCP (L4) tunnels can be used to connect TCP services such as a database, a reverse proxy, RDP, Kubernetes or SSH to the Internet. A single tunnel can expose multiple ports on an exit-server and load balance between clients

### Downloading inlets

inlets is available for Windows, MacOS (Apple Silicon) and Linux:

* [Download a release](https://github.com/inlets/inlets-pro/releases)

You can also use [the container image from ghcr.io](https://github.com/orgs/inlets/packages/container/package/inlets-pro): `ghcr.io/inlets/inlets-pro:latest`

### Your first HTTPS tunnel with an automated tunnel server (recommended)

Expose one or more HTTPS domains from your local machine.

* [Tutorial: Expose one or more local HTTP services via HTTPS](https://inlets.dev/blog/2021/08/08/private-tunnel.html)

### Install a HTTP tunnel server manually (advanced)

If you don't want to use automation tools to create a server for the inlets-pro server, then you can follow this manual guide to generate and install a systemd service instead.

* [Tutorial: Setting up a HTTP tunnel server manually](/tutorial/manual-http-server/)

### Tunnel TCP services

inlets is not limited to exposing HTTP connections, you can also tunnel TCP protocols like RDP, VNC, SSH, TLS (i.e. reverse proxies, or the Kubernetes API server) and databases.

* [Tutorial: Expose a private SSH server over a TCP tunnel](/tutorial/ssh-tcp-tunnel/)
* [Tutorial: Tunnel a private Postgresql database](/tutorial/postgresql-tcp-tunnel/)
* [Tutorial: Tunnel ports 80 and 443 over TCP for a reverse proxy](https://docs.inlets.dev/tutorial/caddy-http-tunnel/)

### Running multiple tunnel servers on the same host (Advanced)

If you want to mix HTTP and TCP tunnels on the same tunnel server, you could either only use TCP ports, or enable both.

* [Advanced: Setting up dual TCP and HTTPS tunnels](/tutorial/dual-tunnels/)

If you're looking to scale inlets to host many tunnels, then Kubernetes is probably a better option.

### Local port forwarding (Intermediate)

* [Case-study: Reliable local port-forwarding from Kubernetes](https://inlets.dev/blog/2021/04/13/local-port-forwarding-kubernetes.html)

### Connecting with Kubernetes

You may have an on-premises Kubernetes cluster that needs ingress. Perhaps you have a homelab, or Raspberry Pi cluster, that you want to self host services on.

* [Tutorial: Expose a local IngressController with the inlets-operator](/tutorial/kubernetes-ingress/)
* [Tutorial: Expose Kubernetes services in short-lived clusters with helm](https://inlets.dev/blog/2021/07/08/short-lived-clusters.html)

Some teams want to have dev work like production, with tools Istio working locally just like in the cloud.

* [Tutorial: Expose an Istio gateway with the inlets-operator](/tutorial/istio-gateway)

* [Tutorial: Access the Kubernetes API server from anywhere like managed service](/tutorial/kubernetes-api-server/)

See also: [helm charts](https://github.com/inlets/inlets-pro/tree/master/chart)

### Provide tunnels as a managed service or SaaS

Inlets Uplink is a complete solution for Kubernetes that makes it quick and easy to onboard hundreds or thousands of tenants. It can also be used to host tunnel servers on Kubernetes, for smaller amounts of tunnels.

Learn more: [Inlets Uplink](https://docs.inlets.dev/uplink/overview/)

### Monitoring and metrics

Inlets offers you multiple options to monitor your tunnels and get insight in their performance. Find out tunnel statistics, uptime and connected clients with the `inlets-pro status` command or collect the Prometheus metrics from the monitoring endpoint.

* [Monitoring and metrics](/tutorial/monitoring-and-metrics)

## Reference documentation

### inletsctl

Learn how to use inletsctl to provision tunnel servers on various public clouds.

* [inletsctl reference](/reference/inletsctl/)

### inlets-operator

Learn how to set up the inlets-operator for Kubernetes, which provisions public cloud VMs and gives IP addresses to your public LoadBalancers.

* [inlets-operator reference](/reference/inlets-operator/)

## Other resources

For news, use-cases and guides check out the blog:

* [Official Inlets blog](https://inlets.dev/blog/)

Watch a video, or read a blog post from the community:

* [Community tutorials](/tutorial/community/)

Open Source tools for managing inlets tunnels:

* [Inlets Operator for Kubernetes LoadBalancers](https://github.com/inlets/inlets-operator)
* [inletsctl to provision tunnel servers](https://github.com/inlets/inletsctl)
* [inlets helm charts for clients and servers](https://github.com/inlets/inlets-pro/tree/master/chart)

## Connecting with the inlets community

Who built inlets? Inlets &reg; is a commercial solution developed and supported by OpenFaaS Ltd.

You can also contact the team via [the contact page](https://inlets.dev/contact).

The code for this website is open source and available [on GitHub](https://github.com/inlets/docs.inlets.dev/)

> inlets is proud to be featured on the Cloud Native Landscape in the Service Proxy category.

[![CNCF Landscape](/images/cncf-landscape-left-logo.svg){ width="400" }](https://landscape.cncf.io/)
