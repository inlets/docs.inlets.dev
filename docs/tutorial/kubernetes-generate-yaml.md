# Generate YAML for a tunnel client

There are a number of ways to run the inlets-pro client in Kubernetes.

One way is to generate YAML for a Kubernetes Deployment, then apply it to your cluster.

## Pre-requisites

* A Kubernetes cluster
* inlets-pro binary

## Generate the YAML for a HTTP tunnel

If you have already created a HTTPS tunnel server, and want to connect a service from within your cluster to the tunnel server, then you can generate the YAML for a tunnel client.

For instance, if you wanted to expose the OpenFaaS gateway accessible at `http://gateway.openfaas.svc.cluster.local:8080` to the Internet:

```bash
export URL=wss://43.245.192.10:8123
export TOKEN="TOKEN"

inlets-pro http client \
    --url $URL \
    --generate=k8s_yaml \
    --generate-name openfaas-gateway-tunnel \
    --upstream http://gateway.openfaas.svc.cluster.local:8080 \
    --token $TOKEN > openfaas-gateway-tunnel.yaml
```

Three objects will be outputted in the YAML:

1. A `Deployment` for the inlets-pro binary
2. A `Secret` for the license key
3. A `Service` for the tunnel client

You can then apply the YAML to your cluster:

```bash
kubectl apply -f openfaas-gateway-tunnel.yaml
```

Based upon the name passed in `--generate-name`, the tunnel client will be named `openfaas-gateway-tunnel-client`, so you can check its logs with:

```bash
kubectl logs deploy/openfaas-gateway-tunnel-client -f
```

## Generate the YAML for a TCP tunnel

The same commands will work for a TCP tunnel, i.e. for Ingress Nginx:

```bash
export URL=wss://43.245.192.10:8123
export TOKEN="TOKEN"

inlets-pro tcp client \
    --url $URL \
    --generate=k8s_yaml \
    --generate-name ingress-nginx-tunnel \
    --upstream ingress-nginx-controller.ingress-nginx.svc.cluster.local \
    --ports 80 \
    --ports 443 \
    --token $TOKEN > ingress-nginx-tunnel.yaml
```

You can then apply the YAML to your cluster.

