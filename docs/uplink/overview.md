# Inlets Uplink overview

!!! Inlets Pro vs Inlets Uplink
    
    Inlets Pro is a stand alone tunnel server that can be use to expose local HTTPs and TCP services on a remote machine or network.

    Inlets Uplink is a complete management solution for tunnels for SaaS companies and service providers.

Inlets Uplinks is our answer to the question: "How do we access customer services from our product?"

VPNs require firewall changes, specific network conditions, and lengthy paperwork. Inlets Uplink uses a TLS encrypted websocket to make an outbound connection, and can also work over corporate HTTP proxies.

* The license is installed on the server, instead of the client
* TCP port ranges can be remapped to avoid conflicts
* A single tunnel can expose HTTP and TCP at the same time
* All tunnels can be monitored centrally for reliability and usage
* By default all tunnels are private and only available for access by your own applications

With Uplink, you deploy tunnel servers for a customers to your Kubernetes cluster, and our operator takes care of everything else.

You have three options to set up a new tunnel, using the `inlets-pro tunnel` CLI, the REST API or a Kubernetes `Tunnel` Custom Resource. Then you you can get a connection command for that tunnel, and have the customer connect - all within seconds.

You can read more about why we created inlets uplink [in the product announcement](https://inlets.dev/blog/2022/11/16/service-provider-uplinks.html).

## Table of Contents

* [Become a provider](/uplink/become-a-provider)
* [Create a tunnel](/uplink/create-tunnels)
* [Connect to a tunnel](/uplink/connect-to-tunnels)
* [Manage tunnels](/uplink/manage-tunnels)
* [Expose tunnels publicly (optional)](/uplink/ingress-for-tunnels)
* [Monitor tunnels](/uplink/monitoring-tunnels)

You can reach out to us if you have questions: [Contact the inlets team](https://inlets.dev/contact)
