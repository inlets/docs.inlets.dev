## Access tunnels privately

By default, only the control-plane is made public, and the data-plane is kept private.

This suits two use-cases:

* A SaaS or Service Provider who needs to give customers a way to tunnel a private endpoint or service to their control plane
* An infrastructure team managing their own applications or services across multiple environments

Once the [tunnel client is connected](/uplink/connect-tunnel-client/), the data-plane of the tunnel can be access from within the Inlets Uplink control-plane cluster.

You can deploy your applications to the management Kubernetes Cluster, then access them via the tunnel's ClusterIP just like they were running directly within your cluster.

The traffic will go to the ClusterIP, which points at the tunnel server's data-plane ports, which then causes a connection to be dialed back to the client.

[![Uplink with no external ingress or exposed tunnels](/images/uplink/no-external-ingress.png)](/images/uplink/no-external-ingress.png)
> Uplink with no external ingress or exposed tunnels

If you need external ingress for one or more tunnels, then you should see the [expose tunnels](/uplink/expose-tunnels/) page.

### Access a TCP service via ClusterIP

If you have forwarded a TCP service, then you can access it via the ClusterIP of the service.

Let's say that you port-forwarded SSH and remapped port 22 to 2222 to avoid needing to use any additional privileged for the tunnel server Pod:

```bash
kubectl run -t -i --rm tcp-access --image=ubuntu --restart=Never -- sh
```

Then from within the pod, you can access the service via the ClusterIP:

```bash
# apt update
# apt install -qy openssh-client

# ssh -p 2222 user@tunnel1.customer1
```

The address is made up of the tunnel name plus the namespace you used for the customer.

### Access a HTTP service via ClusterIP

If you have forwarded an HTTP service, then you can access it via the ClusterIP of the service, using port 8000.

If you have forwarded multiple HTTP endpoints, use the HTTP "Host" header to access one or the other.

```bash
kubectl run -t -i --rm http-access --image=ubuntu --restart=Never -- sh
```

Then from within the pod, you can access the service via the ClusterIP:

```bash
# apt update
# apt install -qy curl

# curl -i http://tunnel1.customer1:8000
```

To access a specific domain, use the HTTP "Host" header:

```bash
# apt update
# apt install -qy curl

# curl -H "Host:  www.example.com" -i http://tunnel1.customer1:8000
```

