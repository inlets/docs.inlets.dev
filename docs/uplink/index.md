# Inlets Uplink

!!! info "What's the difference between Inlets Pro and Inlets Uplink?"
    
    Inlets Pro is a stand-alone binary that can be use to expose local HTTPs and TCP services on a remote machine or network. This is the easiest option for individuals and small teams.

    Inlets Uplink is a complete management solution for tunnels for SaaS companies, infrastructure teams, and service providers. It's designed for scale, multi-tenancy and automation.

Inlets Uplink answers two primary questions:

1. "How do you access customer services from within your own product?"
2. "How to we provide tunnels as a managed service?"

In the first case, of accessing private customer services, you have have already considered building your own agent, using a queue, or a VPN.

The first two options involve considerable work both up front and in the long run. VPNs require firewall changes, specific network conditions, additional paperwork, and can have prohibitive costs associated with them.

The Inlets Uplink runs inside a private network to connect to a public server endpoint using an outbound HTTP connection. Once established, the TLS-encrypted connection is upgraded to a websocket for bi-directional communication. This means it works over NAT, firewalls, and HTTP proxies.

Here are some of the other differences between stand-alone Inlets Pro and Inlets Uplink:

* The management solution is built-in, self-hosted and runs on your own Kubernetes cluster
* You can create a tunnel almost instantly via CLI, REST API or the "Tunnel" Custom Resource
* The license is installed on the server, instead of being attached to the client, to make it easier to give out a connection command to customers and team members
* When exposing TCP ports, they can be remapped to avoid conflicts or needing privileged ports
* A single tunnel can expose HTTP and TCP at the same time
* All tunnels can be monitored centrally for reliability and usage
* By default all tunnels are private and only available for access by your own applications
* Tunnels can also be managed through Helm, ArgoCD or Flux for a GitOps workflow

You can read more about why we created inlets uplink [in the product announcement](https://inlets.dev/blog/2022/11/16/service-provider-uplinks.html).

Reach out to us if you have questions or would like to see a demo: [Contact the inlets team](https://inlets.dev/contact)
