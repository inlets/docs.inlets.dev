# Inlets Uplink

!!! info "What's the difference between Inlets Pro and Inlets Uplink?"
    
    Inlets Pro is a stand-alone binary that can be use to expose local HTTPs and TCP services on a remote machine or network.

    Inlets Uplink is a complete management solution for tunnels for SaaS companies and service providers. It's designed for scale, multi-tenancy and easy management.

Inlets Uplink is our answer to the question: "How do you access customer services from within your own product?"

You may consider building your own agent, using a AWS SQS queue, or a VPN.

The first two options involve considerable work both up front and in the long run. VPNs require firewall changes, specific network conditions, and lengthy paperwork.

Inlets Uplink uses a TLS encrypted websocket to make an outbound connection, and can also work over corporate HTTP proxies.

Here are some of the other differences between Inlets Pro and Inlets Uplink:

* The management solution is built-in, self-hosted and runs on your Kubernetes cluster
* You can create a tunnel almost instantly via CLI, REST API or the "Tunnel" Custom Resource
* The license is installed on the server, instead of each client needing it
* TCP ports can be remapped to avoid conflicts
* A single tunnel can expose HTTP and TCP at the same time
* All tunnels can be monitored centrally for reliability and usage
* By default all tunnels are private and only available for access by your own applications

With Uplink, you deploy tunnel servers for a customers to your Kubernetes cluster, and our operator takes care of everything else.

You can read more about why we created inlets uplink [in the product announcement](https://inlets.dev/blog/2022/11/16/service-provider-uplinks.html).

Reach out to us if you have questions: [Contact the inlets team](https://inlets.dev/contact)
