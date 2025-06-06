# Inlets

[Inlets](https://inlets.dev) is a cloud-native tunnel built to run over restrictive networks, NAT and firewalls. Use it to connect bare-metal, VMs, containers and Kubernetes to the cloud.

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

You'll find tutorials in the navigation for HTTP and TCP tunnels, along with a dedicated section for integrating with Kubernetes.

### Downloading inlets

inlets is available for Windows, MacOS (Apple Silicon) and Linux:

* [Download a release](https://github.com/inlets/inlets-pro/releases)

You can also use [the container image from ghcr.io](https://github.com/orgs/inlets/packages/container/package/inlets-pro): `ghcr.io/inlets/inlets-pro:latest`

### Becoming a tunnel provider or operating a hosting service

Inlets Uplink is a complete solution for Kubernetes that makes it quick and easy to onboard hundreds or thousands of tenants. It can also be used to host tunnel servers on Kubernetes, for smaller amounts of tunnels.

Learn more: [Inlets Uplink](https://docs.inlets.dev/uplink/overview/)

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
