# Inlets Uplink

Inlets Uplink is a complete management solution for tunnels for SaaS companies, infrastructure teams, and service providers. It's designed for scale, multi-tenancy and automation.

![Conceptual overview](/images/uplink/conceptual.png)
> Conceptual architecture: Inlets Uplink is deployed to one or more central Kubernetes cluster(s).

Inlets Uplink answers two primary questions:

1. "How do you access customer services from within your own product?"
2. "How to we provide tunnels as a managed service?"

In the first case, of accessing private customer services, you have have already considered building your own agent, using a queue, or a VPN.

The first two options involve considerable work both up front and in the long run. VPNs require firewall changes, specific network conditions, additional paperwork, and can have prohibitive costs associated with them.

For the second use-case, managed options can be convenient, however they can be expensive as they scale up, may present a security risk, and can't be customised to your specific needs.

!!! info "What's the difference between Inlets Pro and Inlets Uplink?"
    
    Inlets aka "Inlets Pro" is a stand-alone binary that can be use to expose local HTTPs and TCP services on a remote machine or network. This is the easiest option for individuals and small teams.
    
    Uplink is an enterprise tunnel solution for Kubernetes.

## Inlets works from behind restrictive networks

The Inlets Uplink runs inside a private network to connect to a public server endpoint using an outbound HTTP connection. Once established, the TLS-encrypted connection is upgraded to a websocket for bi-directional communication. This means it works over NAT, firewalls, and HTTP proxies.

## Private or Public tunnels

Depending on your use-case, you can keep the [data-plane private](/uplink/private-tunnels/) or [expose it to the Internet](/uplink/expose-tunnels/) on a per-tunnel basis.

By default, the data-plane for each inlets-uplink tunnel is kept private and can only be accessed from within the Kubernetes cluster where inlets-uplink is installed.

You can then expose the data-plane for any tunnel to the Public Internet if required.

## Features & benefits

* The management solution is built-in, self-hosted and runs on your own Kubernetes cluster
* You can create a tunnel almost instantly via CLI, REST API or the "Tunnel" Custom Resource
* The license is installed on the server, instead of being attached to the client, to make it easier to give out a connection command to customers and team members
* When exposing TCP ports, they can be remapped to avoid conflicts or needing privileged ports
* A single tunnel can expose HTTP and TCP at the same time
* All tunnels can be monitored centrally for reliability and usage
* By default all tunnels are private and only available for access by your own applications
* Tunnels can also be managed through Helm, ArgoCD or Flux for a GitOps workflow

Support via email is included for the installation and operation of your inlets-uplink installation.

You can read more about why we created inlets uplink [in the original product announcement](https://inlets.dev/blog/2022/11/16/service-provider-uplinks.html).

## Installation

Learn more about how to [install Inlets Uplink](/uplink/installation/).

Reach out to us if you have questions or would like to see a demo: [Contact the inlets team](https://inlets.dev/contact)
