# Ingress for tunnels

!!! info

    Inlets Uplink is designed to connect customer services to a remote Kubernetes cluster for command and control as part of a SaaS product.

    Any tunnelled service can be accessed directly from within the cluster and does not need to be exposed to the public Internet for access.

    Beware: by following these instructions, you are exposing one or more of those tunnels to the public Internet.


Make inlets uplink HTTP tunnels publicly accessible by setting up ingress for the data plane.

The instructions assume that you want to expose two HTTP tunnels. We will configure ingress for the first tunnel, called `grafana`, on the domain `grafana.example.com`. The second tunnel, called `openfaas`, will use the domain `openfaas.example.com`.

Both tunnels can be created with `kubectl` or the `inlets-pro` cli. See [create tunnels](/uplink/create-tunnels/) for more info:

=== "kubectl"

    ```bash
    $ cat <<EOF | kubectl apply -f - 
    apiVersion: uplink.inlets.dev/v1alpha1
    kind: Tunnel
    metadata:
      name: grafana
      namespace: tunnels
    spec:
      licenseRef:
        name: inlets-uplink-license
        namespace: tunnels
    ---
    apiVersion: uplink.inlets.dev/v1alpha1
    kind: Tunnel
    metadata:
      name: openfaas
      namespace: tunnels
    spec:
      licenseRef:
        name: inlets-uplink-license
        namespace: tunnels
    EOF
    ```

=== "cli"

    ```bash
    $ inlets-pro tunnel create grafana
    Created tunnel openfaas. OK.

    $ inlets-pro tunnel create openfaas
    Created tunnel openfaas. OK.
    ```

Follow the instruction for Kubernetes Ingress or Istio depending on how you deployed inlets uplink.

## Setup tunnel ingress

1. Create a new certificate Issuer for tunnels:

    ```bash
    export EMAIL="you@example.com"

    cat > tunnel-issuer-prod.yaml <<EOF
    apiVersion: cert-manager.io/v1
    kind: Issuer
    metadata:
      name: tunnels-letsencrypt-prod
      namespace: inlets
    spec:
      acme:
        server: https://acme-v02.api.letsencrypt.org/directory
        email: $EMAIL
        privateKeySecretRef:
        name: tunnels-letsencrypt-prod
        solvers:
        - http01:
            ingress:
              class: "nginx"
    EOF
    ```

2. Create an ingress resource for the tunnel:

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: grafana-tunnel-ingress
      namespace: inlets
      annotations:
        kubernetes.io/ingress.class: nginx
        cert-manager.io/issuer: tunnels-letsencrypt-prod
    spec:
      rules:
      - host: grafana.example.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: grafana.tunnels
                port:
                  number: 8000
      tls:
      - hosts:
        - grafana.example.com
        secretName: grafana-cert
    ```

    Note that the annotation `cert-manager.io/issuer` is used to reference the certificate issuer created in the first step.

To setup ingress for multiple tunnels simply define multiple ingress resources. For example apply a second ingress resource for the openfaas tunnel:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: openfaas-tunnel-ingress
  namespace: inlets
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/issuer: tunnels-letsencrypt-prod
spec:
  rules:
  - host: openfaas.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: openfaas.tunnels
            port:
              number: 8000
  tls:
  - hosts:
    - openfaas.example.com
    secretName: openfaas-cert
```


## Setup tunnel ingress with an Istio Ingress gateway

1. Create a new certificate Issuer for tunnels:

    ```bash
    export EMAIL="you@example.com"

    cat > tunnel-issuer-prod.yaml <<EOF
    apiVersion: cert-manager.io/v1
    kind: Issuer
    metadata:
      name: tunnels-letsencrypt-prod
      namespace: istio-system
    spec:
      acme:
        server: https://acme-v02.api.letsencrypt.org/directory
        email: $EMAIL
        privateKeySecretRef:
          name: tunnels-letsencrypt-prod
        solvers:
        - http01:
            ingress:
              class: "istio"
    EOF
    ```

    We are using the Let's Encrypt production server which has strict limits on the API. A staging server is also available at https://acme-staging-v02.api.letsencrypt.org/directory. If you are creating a lot of certificates while testing it would be better to use the staging server.

2. Create a new certificate resource. In this case we want to expose two tunnels on their own domain, `grafana.example.com` and `openfaas.example.com`. This will require two certificates, one for each domain:

    ```yaml
    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
      name: grafana-cert
      namespace: istio-system
    spec:
      secretName: grafana-cert
      commonName: grafana.example.com
      dnsNames:
      - grafana.example.com
      issuerRef:
        name: tunnels-letsencrypt-prod
        kind: Issuer

    ---
    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
      name: openfaas-cert
      namespace: istio-system
    spec:
      secretName: openfaas-cert
      commonName: openfaas.example.com
      dnsNames:
      - openfaas.example.com
      issuerRef:
        name: tunnels-letsencrypt-prod
        kind: Issuer
    ```

    Note that both the certificates and issuer are created in the `istio-system` namespace.

3. Configure the ingress gateway for both tunnels. In this case we create a single resource for both hosts but you could also split the configuration into multiple Gateway resources.

    ```yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: Gateway
    metadata:
      name: tunnel-gateway
      namespace: inlets
    spec:
      selector:
          istio: ingressgateway # use Istio default gateway implementation
      servers:
      - port:
          number: 443
          name: https
          protocol: HTTPS  
        tls:
          mode: SIMPLE
          credentialName: grafana-cert
        hosts:
        - grafana.example.com
      - port:
          number: 443
          name: https
          protocol: HTTPS
        tls:
          mode: SIMPLE
          credentialName: openfaas-cert
        hosts:
        - openfaas.example.com
    ```

    Note that the `credentialsName` references the secrets for the certificates created in the previous step.

4. Configure the gateway's traffic routes by defining corresponding virtual services:

    ```yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: grafana
      namespace: inlets
    spec:
      hosts:
      - grafana.example.com
      gateways:
      - tunnel-gateway
      http:
      - match:
        - uri:
            prefix: /
        route:
        - destination:
            host: grafana.tunnels.svc.cluster.local
            port:
              number: 8000
    ---
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: openfaas
      namespace: inlets
    spec:
      hosts:
      - openfaas.example.com
      gateways:
      - tunnel-gateway
      http:
      - match:
        - uri:
            prefix: /
        route:
        - destination:
            host: openfaas.tunnels.svc.cluster.local
            port:
              number: 8000
    ```

After applying these resources you should be able to access the data plane for both tunnels on their custom domain.
