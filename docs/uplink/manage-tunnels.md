# Manage customer tunnels

## Manage tunnels with kubectl

You can use all the `kubectl` commands you would expect to manage tunnels.

### List tunnels

List tunnels across all namespaces:

```bash
$ kubectl get tunnels -A

NAMESPACE   NAME     AUTHTOKENNAME   DEPLOYMENTNAME   TCP PORTS   DOMAINS
inlets      team     team            team             [8080]
tunnels     acmeco   acmeco          acmeco           [8080]      
tunnels     ssh      ssh             ssh              [50035]
```

### Delete a tunnel

To delete a tunnel run `kubectl delete`:

```bash
kubectl delete tunnel/acmeco -n tunnels
```

### Update a tunnel

To update a tunnel and change or add TCP ports you can either edit the Tunnel Custom Resource and run `kubectl apply` or use:

```
kubectl edit tunnel/acmeco \
  -n tunnels
```

## Check logs for a tunnel

The logs for tunnels can be usefull for troubleshooting or to see if clients are connecting successfully.

Get the logs for a tunnel deployment: 

```bash
$ kubectl logs deploy/acmeco -n tunnels -f

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

## Rotate a tunnel's secret

If you inadvertently leaked the token for your tunnel, then you can rotate it. To rotate a token for a tunnel you can delete the secret holding the token. The inlets uplink controller will automatically create a new secret.

We will rotate the token for the acmeco tunnel in the tunnels namespace. The default secret has the same name as the tunnel.

```bash
kubectl delete secret/acmeco -n tunnels
```

The tunnel has to be restarted to use the new token. 

```bash
kubectl rollout restart -n tunnels deploy/acmeco
```

Any connected tunnels will disconnect at this point, and won’t be able to reconnect until you configure them with the updated token.

Retrieve the new token for the tunnel and save it to a file:
