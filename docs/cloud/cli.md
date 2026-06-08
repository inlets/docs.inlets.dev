# Cloud CLI Reference

The `inlets-pro cloud` CLI is a plugin that provides a command-line interface to [Inlets Cloud](/cloud/index/). It lets you manage Inlets Cloud tunnels directly from the terminal.

## Install the cloud CLI plugin

The `cloud` plugin is not bundled with `inlets-pro` by default. It is a plugin that needs to be installed separately:

```bash
inlets-pro plugin get cloud
```

After installation, run `inlets-pro cloud` to verify it works:

```bash
inlets-pro cloud version
```

## Authenticate

Before using the CLI, log in with your access token from the Inlets Cloud dashboard:

```bash
inlets-pro cloud auth login --token <ACCESS_TOKEN>
```

Verify the active session:

```bash
inlets-pro cloud auth whoami
```

To log out:

```bash
inlets-pro cloud auth logout
```

## Create tunnels

A tunnel can be created in one of two modes:

* `http` (default) — an HTTPS-terminated tunnel for a single HTTP service. Use a generated `tryinlets.dev` domain or bring your own custom domain.
* `ingress` — a TCP pass-through tunnel for ports 80 and 443. Requires a custom domain.

### Create an HTTP tunnel

This is the quickest way to get started. Inlets Cloud generates a subdomain for you under the `tryinlets.dev` domain.

```bash
inlets-pro cloud tunnel create my-tunnel
```

The `http` mode is the default, so the command above is equivalent to:

```bash
inlets-pro cloud tunnel create --generated my-tunnel
```

Create the tunnel in a specific region:

```bash
inlets-pro cloud tunnel create my-tunnel --region us-east-1
```

### Create an HTTP tunnel with a custom domain

You can serve the tunnel from your own domain instead of a generated one.

!!! note "Register and verify the domain first"
    Before you can create a tunnel with a custom domain, the domain must be
    registered and verified. First [add the domain](#add-a-domain), then
    [verify it](#verify-a-domain). A tunnel that references an unverified domain will be rejected.

Once the domain is verified, pass it with `--domain`:

```bash
inlets-pro cloud tunnel create my-tunnel --domain myapp.example.com
```

### Create an Ingress tunnel

An Ingress tunnel passes TCP through on ports 80 and 443 to your own reverse proxy or Kubernetes Ingress Controller. It always uses a custom domain, which must be [registered and verified first](#domains):

```bash
inlets-pro cloud tunnel create ingress my-tunnel --domain myapp.example.com
```

### List tunnels

```bash
inlets-pro cloud tunnel list
```

```
NAME                  TYPE     REGION   STATUS    DOMAIN                                  CONNECTED
my-tunnel             http     cambs1   Created   leveto.cambs1.tryinlets.dev             1
openfaas              ingress  cambs1   Created   *.openfaas.example.dev                  1
```

You can include RX/TX traffic metrics by adding the `--verbose` when listing tunnels:

```bash
inlets-pro cloud tunnel list --verbose
```

### Delete a tunnel

Delete by name:

```bash
inlets-pro cloud tunnel delete my-tunnel
```

Delete by domain:

```bash
inlets-pro cloud tunnel delete example.com
```

### Connect to a tunnel

The `connect` command prints the configuration needed to run the inlets client for a tunnel, including the tunnel URL, token and upstream targets. The output is written to stdout so you can copy it or pipe it into a file. Use `--format` to choose how it is generated.

By default it prints a ready-to-run `inlets-pro uplink client` CLI one-liner:

```bash
inlets-pro cloud tunnel connect my-tunnel
```

Override the upstream targets that traffic is forwarded to:

```bash
inlets-pro cloud tunnel connect my-tunnel --upstream "8080=127.0.0.1:8080"
```

Use `--format k8s` to print a Kubernetes Deployment manifest that runs the inlets client as a Pod. The manifest is written to stdout, so redirect it to a file and apply it with `kubectl`:

```bash
inlets-pro cloud tunnel connect --format k8s my-tunnel > inlets-client.yaml
kubectl apply -f inlets-client.yaml
```

Set the namespace and inlets-pro image version used in the manifest:

```bash
inlets-pro cloud tunnel connect --format k8s \
  --namespace ingress \
  --inlets-version 0.11.1 \
  my-tunnel
```

Use `--format systemd` to print a systemd unit file that runs the inlets client as a service. The unit is written to stdout, so save it under `/etc/systemd/system/`, then reload and enable it to keep the tunnel connected across reboots:

```bash
inlets-pro cloud tunnel connect --format systemd my-tunnel \
  | sudo tee /etc/systemd/system/my-tunnel-inlets-client.service
sudo systemctl daemon-reload
sudo systemctl enable --now my-tunnel-inlets-client
```

### Manage Ingress tunnel domains

Attach an additional domain to an Ingress tunnel:

```bash
inlets-pro cloud tunnel domain add my-tunnel myapp.example.com
```

Remove a domain from an Ingress tunnel:

```bash
inlets-pro cloud tunnel domain remove my-tunnel myapp.example.com
```

### Enable Proxy Protocol

Set Proxy Protocol (v1 or v2) on an Ingress tunnel to preserve the original client IP:

```bash
inlets-pro cloud tunnel proxy-protocol set my-tunnel v2
```

Disable Proxy Protocol:

```bash
inlets-pro cloud tunnel proxy-protocol set my-tunnel none
```

## Domains

Manage custom apex domains for HTTPS tunnels. Before creating a [tunnel with a custom domain](#create-an-http-tunnel-with-a-custom-domain) or an [Ingress tunnel](#create-an-ingress-tunnel), you must first register and verify the domain using the steps below.

### Add a domain

Register an apex domain with Inlets Cloud:

```bash
inlets-pro cloud domain add example.com
```

### Verify a domain

Verifying a domain proves you own it before Inlets Cloud will issue TLS certificates and route traffic for it. This is a one-time DNS TXT record challenge.

**1. Get the verification challenge**

```bash
inlets-pro cloud domain get example.com
```

```
Domain: example.com
Verified: no
Required TXT record: inlets-domain-verify=ba4a4ab1-3324-4b1c-b684-e0ff2cc53e08
```

**2. Create the TXT record at your DNS provider**

Add a `TXT` record on the apex of the domain using the value from the previous step:

| Type | Name      | Value                                                  |
|------|-----------|--------------------------------------------------------|
| TXT  | `@`       | `inlets-domain-verify=ba4a4ab1-3324-4b1c-b684-e0ff2cc53e08` |

The `@` name (also shown as the bare domain by some providers) places the record on the apex, e.g. `example.com`. DNS changes can take a few minutes to propagate.

**3. Verify the domain**

Once the record has propagated, ask Inlets Cloud to check it:

```bash
inlets-pro cloud domain verify example.com
```

If the TXT record cannot be found yet, the command will report the domain as unverified. Wait for DNS to propagate and run it again. You can re-run `domain get` at any time to recheck the status and re-fetch the challenge value.

### List domains

List all registered domains:

```bash
inlets-pro cloud domain list
```

```
DOMAIN       VERIFIED  LAST_VERIFIED
example.com  yes       2026-03-11T16:12:48Z
```

### Delete a domain

```bash
inlets-pro cloud domain delete example.com
```

## Regions

List available Inlets Cloud regions:

```bash
inlets-pro cloud region list
```

```
NAME       DESCRIPTION  HTTPS_GENERATED  HTTPS_CUSTOM  WILDCARD_DOMAIN
cambs1     eu-west-1    yes              yes           cambs1.tryinlets.dev
us-east-1  us-east-1    yes              yes           us-east-1.tryinlets.dev
```

Each row is a region you can target. The columns are:

- `NAME` — the region's identifier. Pass this value to `--region` when creating a tunnel.
- `DESCRIPTION` — the underlying cloud provider region the tunnel servers run in.
- `HTTPS_GENERATED` — `yes` if the region can host HTTP tunnels with an auto-generated `tryinlets.dev` domain.
- `HTTPS_CUSTOM` — `yes` if the region can host HTTP tunnels with your own domain.
- `WILDCARD_DOMAIN` — the wildcard domain that generated tunnel names are created under in this region (for example `cambs1.tryinlets.dev`).

## ACLs

Manage the IP allow list for a tunnel's [Security Group](/cloud/security-groups/). Each tunnel has a Security Group containing the CIDRs/IPs that are allowed to reach it. These commands let you view and update that allow list from the CLI.

List ACL entries for a tunnel:

```bash
inlets-pro cloud acl list my-tunnel
```

```
Tunnel: my-tunnel
Security Group: default
ALLOW
0.0.0.0/0
```

Set ACL entries (replaces all existing entries):

```bash
inlets-pro cloud acl set my-tunnel --allow 203.0.113.0/24 --allow 198.51.100.5
```

## JSON output

Most commands support `--json` for machine-readable output, which is useful for scripting:

```bash
inlets-pro cloud tunnel list --json
inlets-pro cloud tunnel connect my-tunnel --json 
inlets-pro cloud auth whoami --json
inlets-pro cloud region list --json
```
