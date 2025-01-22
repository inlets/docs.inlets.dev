# Connect the tunnel client

The tunnel plugin for the inlets-pro CLI can be used to get connection instructions for a tunnel.

Whether the client needs to be deployed as a systemd service on the customers server or as a Kubernetes service, with the CLI it is easy to generate connection instructions for these different formats by setting the `--format` flag.

Supported formats:

- [CLI command](#get-connection-instructions)
- [Systemd](#deploy-the-client-as-a-systemd-service) 
- [Kubernetes YAML Deployment](#deploy-the-client-in-a-kubernetes-cluster)

Make sure you have the latest version of the tunnel command available:

```bash
inlets-pro plugin get tunnel
```

## Get connection instructions

Generate the client command for the selected tunnel:

```bash
$ inlets-pro tunnel connect openfaas \
    --domain uplink.example.com \
    --upstream http://127.0.0.1:8080

# Access your HTTP tunnel via: http://openfaas.tunnels:8000

# Access your TCP tunnel via ClusterIP: 
#  openfaas.tunnels:5432

inlets-pro uplink client \
  --url=wss://uplink.example.com/tunnels/openfaas \
  --token=tbAd4HooCKLRicfcaB5tZvG3Qj36pjFSL3Qob6b9DBlgtslmildACjWZUD \
  --upstream=http://127.0.0.1:8080
```

Optionally the `--quiet` flag can be set to print the CLI command without the additional info.

## Deploy the client as a systemd service

To generate a systemd service file for the tunnel client command set the `--format` flag to `systemd`. 

```bash
$ inlets-pro tunnel connect openfaas \
    --domain uplink.example.com \ 
    --upstream http://127.0.0.1:8080 \
    --format systemd

[Unit]
Description=openfaas inlets client
After=network.target

[Service]
Type=simple
Restart=always
RestartSec=5
StartLimitInterval=0
ExecStart=/usr/local/bin/inlets-pro uplink client --url=wss://uplink.example.com/tunnels/openfaas --token=tbAd4HooCKLRicfcaB5tZvG3Qj36pjFSL3Qob6b9DBlgtslmildACjWZUD --upstream=http://127.0.0.1:8080

[Install]
WantedBy=multi-user.target
```

Copy the service file over to the customer's host. Save the unit file as: `/etc/systemd/system/openfaas-tunnel.service`.

Once the file is in place start the service for the first time:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now openfaas-tunnel
```

Verify the tunnel client is running:

```bash
systemctl status openfaas-tunnel
```

You can also check the logs to see if the client connected successfully:

```bash
journalctl -u openfaas-tunnel
```

## Deploy the client in a Kubernetes cluster

To generate a YAML deployment for a selected tunnel, set the `--format` flag to `k8s_yaml`. The generated resource can be deployed in the customers cluster.

```bash
inlets-pro tunnel connect openfaas \
    --domain uplink.example.com \
    --upstream http://gateway.openfaas:8080 \
    --format k8s_yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: openfaas-inlets-client
spec:
  replicas: 1
  selector:
    matchLabels:
      app: openfaas-inlets-client
  template:
    metadata:
      labels:
        app: openfaas-inlets-client
    spec:
      containers:
      - name: openfaas-inlets-client
        image: ghcr.io/inlets/inlets-pro:0.9.14
        imagePullPolicy: IfNotPresent
        command: ["inlets-pro"]
        args:
        - "uplink"
        - "client"
        - "--url=wss://uplink.example.com/tunnels/openfaas"
        - "--token=tbAd4HooCKLRicfcaB5tZvG3Qj36pjFSL3Qob6b9DBlgtslmildACjWZUD"
        - "--upstream=http://gateway.openfaas:8080"
```

In this example we create a tunnel to uplink an [OpenFaaS](https://www.openfaas.com/) deployment.

Get the logs for the client and check it connected successfully:

```bash
kubectl logs deploy/openfaas-inlets-client
```