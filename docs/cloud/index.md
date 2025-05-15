# Inlets Cloud

[Inlets Cloud](https://inlets.dev/cloud) is the quickest and easiest way to expose your local services to the Internet. 

A tunnel server is set up for you in your preferred region, and then you can generate a CLI command or Kubernetes manifest to connect the tunnel to it.

Blog posts/tutorials:

* [Managed HTTPS tunnels in one-click with inlets cloud](https://inlets.dev/blog/tutorial/2025/04/01/one-click-hosted-http-tunnels.html)
* [SSH Into Any Private Host With Inlets Cloud](https://inlets.dev/blog/tutorial/2024/10/17/ssh-with-inlets-cloud.html)

## Supported regions

Each Inlets Cloud region is an independent installation of [inlets-uplink](/uplink/index/) which is managed centrally from the Inlets Cloud dashboard.

The following regions are available:

* cambs1 - London, United Kingdom
* us-east-1 - Virginia, USA
* ap-southeast-1 - Singapore

If you'd like to request an additional region, send us a message on the Inlets Discord server.

## Types of tunnel

* HTTPS terminated tunnels (tryinlets.dev)

    Try out a HTTPS tunnel with a random name generated for you under the `tryinlets.dev` domain.

    Copy the CLI command or Kubernetes YAML and connect your tunnel.

    You can then access your local service i.e. mkdocs via the generated URL i.e. `https://heavy-snow.cambs1.tryinlets.dev`.

* HTTPS terminated tunnels (Bring Your Own Domain)

    First off, verify one of your domains (i.e. `example.com` by creating a TXT record as directed in the dashboard.
    
    Next, deploy a HTTPS tunnel and specify your custom domain. Inlets Cloud will create a tunnel server and a TLS certificate for you. Then connect the client to the tunnel server using the connection string provided in the dashboard.

    This is perfect for dashboards, web applications, OpenFaaS, blogs, and any other HTTP service you want to expose to the Internet.

* Ingress tunnels

    Use an Ingress tunnel expose ports 80 and 443 with TCP pass-through to your local reverse proxy or Kubernetes Ingress Controller/Istio.

* SSH tunnels

    You can expose one or more SSH services to the Internet using the `inlets-pro snimux` command and an Ingress Tunnel. This is useful for SSH access to your home network or a remote server.

    Unlike traditional SSH tunnels, you can also implement an IP allow list in the config file for the `inlets-pro snimux server`.

## Proxy Protocol

When using the Ingress tunnel type, you can enable the [Proxy Protocol](https://www.haproxy.org/download/1.8/doc/proxy-protocol.txt). This is useful for services that need to know the original client IP address.

Whenever you enable the Proxy Protocol for a tunnel, you'll need to configure your tunneled service to accept the Proxy Protocol header.

Both v1 and v2 are supported.

## Get started with Inlets Cloud

An inlets subscription is a pre-requisite. If you don't have one yet, you can [sign-up here](https://inlets.dev/pricing/) on a monthly basis.

During beta, there is no additional charge or fee for using Inlets Cloud. You can use it to test the service and provide feedback.

Just [register](https://cloud.inlets.dev/register) with the same email you use for your inlets subscription and we will send you an invite.

You can log into the inlets cloud dashboard using a magic link sent to your email address.

## Comments, questions and suggestions

For any kind of feedback, please use the Inlets Discord server. You'll find a link in your Inlets Cloud dashboard.

