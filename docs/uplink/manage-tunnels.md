# Manage customer tunnels

## Manage tunnels with kubectl

You can use all the `kubectl` commands you would expect to manage tunnels.

### List tunnels

List tunnels across all namespaces:

```bash
$ kubectl get tunnels -A

NAMESPACE     NAME     AUTHTOKENNAME   DEPLOYMENTNAME   TCP PORTS   DOMAINS
tunnels       acmeco   acmeco          acmeco           [8080]      
customer1     ssh      ssh             ssh              [50035]
```

To list the tunnels within a namespace:

```bash
$ kubectl get tunnels -n customer1
```

### Delete a tunnel

To delete a tunnel run `kubectl delete`:

```bash
kubectl delete -n tunnels \
  tunnel/acmeco 
```

This will remove all resources for the tunnel.

Do also remember to stop the customer's inlets uplink client.

### Update the ports or domains for a tunnel

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

You may want to rotate a secret for a customer if you think the secret has been leaked. To rotate a token for a tunnel you can delete the secret holding the token. The inlets uplink controller will automatically create a new secret.

We will rotate the token for the acmeco tunnel in the tunnels namespace. The default secret has the same name as the tunnel.

```bash
kubectl delete -n tunnels \
  secret/acmeco 
```

The tunnel has to be restarted to use the new token. 

```bash
kubectl rollout restart -n tunnels \
  deploy/acmeco
```

Any connected tunnels will disconnect at this point, and won’t be able to reconnect until you configure them with the updated token.

Retrieve the new token for the tunnel and save it to a file:

```bash
kubectl get -n tunnels secret/acmeco \
  -o jsonpath="{.data.token}" | base64 --decode > token.txt 
```

The contents will be saved in `token.txt`
