# Manage customer tunnels

You can use `kubectl` or the tunnel plugin for the `inlets-pro` CLI to manage tunnels.

## List tunnels

List tunnels across all namespaces:

=== "kubectl"

    ```bash
    $ kubectl get tunnels -A

    NAMESPACE     NAME         AUTHTOKENNAME   DEPLOYMENTNAME   TCP PORTS   DOMAINS
    tunnels       acmeco       acmeco          acmeco           [8080]      
    customer1     ssh          ssh             ssh              [50035]
    customer1     prometheus   prometheus      prometheus       []         [prometheus.customer1.example.com]
    ```

=== "cli"

    ```bash
    $ inlets-pro tunnel list -A

    TUNNEL     DOMAINS                              PORTS   CREATED
    acmeco     []                                   [8080]  2022-11-22 11:51:35 +0100 CET
    ssh        []                                   [50035] 2022-11-24 18:19:01 +0100 CET
    prometheus [prometheus.customer1.example.com]   []      2022-11-24 11:43:23 +0100 CET
    ```


To list the tunnels within a namespace:

=== "kubectl"

    ```bash
    $ kubectl get tunnels -n customer1

    NAME         AUTHTOKENNAME   DEPLOYMENTNAME   TCP PORTS   DOMAINS
    ssh          ssh             ssh              [50035]
    ```

=== "cli"

    ```bash
    $ inlets-pro tunnel list -n customer1

    TUNNEL     DOMAINS   PORTS   CREATED
    ssh        []        [50035] 2022-11-22 11:51:35 +0100 CET
    ```

## Delete a tunnel

Deleting a tunnel will remove all resources for the tunnel.

To remove a tunnel run:

=== "kubectl"

    ```bash
    kubectl delete -n tunnels \
      tunnel/acmeco 
    ```

=== "cli"

    ```bash
    inlets-pro tunnel remove acmeco \
      -n tunnels
    ```

Do also remember to stop the customer's inlets uplink client.

## Update the ports or domains for a tunnel

You can update a tunnel and configure its TCP ports or domain names by editing the Tunnel Custom Resource:

```bash
kubectl edit -n tunnels \
  tunnel/acmeco  
```

Imagine you wanted to add port 8081, when you already had port 8080 exposed:

```diff
apiVersion: uplink.inlets.dev/v1alpha1
kind: Tunnel
metadata:
  name: acmeco
  namespace: tunnels
spec:
  licenseRef:
    name: inlets-uplink-license
    namespace: tunnels
  tcpPorts:
  - 8080
+ - 8081
```

Alternatively, if you have the tunnel saved as a YAML file, you can edit it and apply it again with `kubectl apply`.

## Check the logs of a tunnel

The logs for tunnels can be useful for troubleshooting or to see if clients are connecting successfully.

Get the logs for a tunnel deployment: 

```bash
$ kubectl logs -n tunnels deploy/acmeco -f

2022/11/22 12:07:38 Inlets Uplink For SaaS & Service Providers (Inlets Uplink for 5x Customers)
2022/11/22 12:07:38 Licensed to: user@example.com
inlets (tm) uplink server
All rights reserved OpenFaaS Ltd (2022)

Metrics on: 0.0.0.0:8001
Control-plane on: 0.0.0.0:8123
HTTP data-plane on: 0.0.0.0:8000
time="2022/11/22 12:33:34" level=info msg="Added upstream: * => http://127.0.0.1:9090 (9355de15c687471da9766cbe51423e54)"
time="2022/11/22 12:33:34" level=info msg="Handling backend connection request [9355de15c687471da9766cbe51423e54]"
```

## Rotate the secret for a tunnel

You may want to rotate a secret for a customer if you think the secret has been leaked. The token can be rotated manually using `kubectl` or with a single command using the `tunnel` CLI plugin.

=== "kubectl"

    Delete the token secret. The default secret has the same name as the tunnel. The inlets uplink controller will automatically create a new secret.

    ```bash
    kubectl delete -n tunnels \
      secret/acmeco 
    ```

    The tunnel has to be restarted to use the new token. 

    ```bash
    kubectl rollout restart -n tunnels \
      deploy/acmeco
    ```

=== "cli"

    Rotate the tunnel token:

    ```bash
    inlets-pro tunnel rotate acmeco \
      -n tunnels
    ```

Any connected tunnels will disconnect at this point, and won’t be able to reconnect until you configure them with the updated token.

Retrieve the new token for the tunnel and save it to a file:

=== "kubectl"

    ```bash
    kubectl get -n tunnels secret/acmeco \
      -o jsonpath="{.data.token}" | base64 --decode > token.txt 
    ```

=== "cli"

    ```bash
    inlets-pro tunnel token acmeco \
      -n tunnels > token.txt
    ```

The contents will be saved in `token.txt`
