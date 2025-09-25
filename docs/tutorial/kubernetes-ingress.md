# Tutorial: Expose a local IngressController with the inlets-operator

In this quick-start we will configure the [inlets-operator](https://github.com/inlets/inlets-operator) to use inlets-pro in TCP mode to expose ports 80 and 443 of an Ingress Controller (ingress-nginx) so that it can receive HTTPS certificates via LetsEncrypt and [cert-manager](https://github.com/jetstack/cert-manager).

The inlets-operator creates a VM for each tunnel server in the cloud of your choice, then plumbs in an inlets client to connect to it using a Deployment. There is an alternative approach that we also recommend which involves creating the tunnel server with [inletsctl](https://github.com/inlets/inletsctl), followed by installing the inlets client with Helm: [Fixing Ingress for short-lived local Kubernetes clusters](https://inlets.dev/blog/2021/07/08/short-lived-clusters.html).

> [Get your inlets subscription here](https://inlets.dev/pricing)

## Pre-reqs

* A computer or laptop running MacOS or Linux, or Git Bash or WSL on Windows
* Docker for Mac / Docker Daemon - installed in the normal way, you probably have this already
* [KinD](https://github.com/kubernetes-sigs/kind) - the "darling" of the Kubernetes community is Kubernetes IN Docker, a small one-shot cluster that can run inside a Docker container
* [arkade](https://github.com/alexellis/arkade) - arkade is an app installer that takes a helm chart and bundles it behind a simple CLI

## Install arkade

You can use [arkade](https://arkade.dev) or helm to install the various applications we are going to add to the cluster below. arkade provides an apps ecosystem that makes things much quicker.

MacOS and Linux users:

```bash
curl -sSLf https://get.arkade.dev/ | sudo sh
```

Windows users should install [Git Bash](https://git-scm.com/downloads) and run the above without `sudo`.

## Create a Kubernetes cluster with KinD

We're going to use [KinD](https://github.com/kubernetes-sigs/kind), which runs inside a container with Docker for Mac or the Docker daemon. MacOS cannot actually run containers or Kubernetes itself, so projects like Docker for Mac create a small Linux VM and hide it away.

You can use an alternative to KinD if you have a preferred tool.

Get a KinD binary release and `kubectl` (the Kubernetes CLI):

```bash
arkade get kind --version v0.9.0
arkade get kubectl --version v1.19.3
```

Now create a cluster:

```bash
$ kind create cluster
```

The initial creation could take a few minutes, but subsequent clusters creations are much faster.

```
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.19.0) 🖼
 ✓ Preparing nodes 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Have a nice day! 👋
```

We can check that our single node is ready now:

```bash
kubectl get node -o wide

NAME                 STATUS     ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE       KERNEL-VERSION     CONTAINER-RUNTIME
kind-control-plane      Ready   master   35s   v1.18.0   172.17.0.2    <none>        Ubuntu 19.10   5.3.0-26-generic   containerd://1.3.2
```

The above shows one node Ready, so we are ready to move on.

## Install the inlets-operator

Save an access token for your cloud provider as `$HOME/access-token`, in this example we're using DigitalOcean. Other providers may also need a secret token in addition to the API key.

Your inlets license should be already saved at: `$HOME/.inlets/LICENSE`, if it's not, you can move it there or use the `--license-file` flag.

```bash
export ACCESS_TOKEN=$HOME/access-token

arkade install inlets-operator \
 --provider digitalocean \
 --region lon1 \
 --token-file $ACCESS_TOKEN \
 --license-file "$HOME/.inlets/LICENSE"
```

> You can run `arkade install inlets-operator --help` to see a list of other cloud providers.

* Set the `--region` flag as required, it's best to have low latency between your current location and where the exit-servers will be provisioned.

## Install nginx-ingress

This installs nginx-ingress using its Helm chart:

```bash
arkade install nginx-ingress
```

## Install cert-manager

Install [cert-manager](https://cert-manager.io/docs/), which can obtain TLS certificates through NginxIngress.

```bash
arkade install cert-manager
```

## A quick review

Here's what we have so far:

* nginx-ingress

    An IngressController, Traefik or Caddy are also valid options. It comes with a Service of type LoadBalancer that will get a public address via the tunnel

* inlets-operator configured to use inlets-pro in TCP mode

    Provides us with a public VirtualIP for the IngressController service.

* cert-manager

    Provides TLS certificates through the HTTP01 or DNS01 challenges from LetsEncrypt

## Deploy an application and get a TLS certificate

This is the final step that shows everything working end to end.

TLS certificates require a domain name and DNS A or CNAME entry, so let's set that up

Find the External-IP:

```bash
kubectl get svc
```

Now create a DNS A record in your admin panel, so for example: `expressjs.example.com`.

Now when you install a Kubernetes application with an Ingress definition, NginxIngress and cert-manager will work together to provide a TLS certificate.

Create a staging issuer for cert-manager `staging-issuer.yaml` and make sure you edit the `email` value.

```bash
export EMAIL="you@example.com"

cat > issuer-staging.yaml <<EOF
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-staging
  namespace: default
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: $EMAIL
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - selector: {}
      http01:
        ingress:
          class: nginx
EOF
```

Apply the file with `kubectl apply -f staging-issuer.yaml`

While the Let's Encrypt production server has strict limits on the API, the staging server is more forgiving, and
should be used while you are testing a deployment.

Edit `email`, then run: `kubectl apply -f issuer.yaml`.

Let's use helm3 to install Alex's example Node.js API [available on GitHub](https://github.com/alexellis/expressjs-k8s)

Create a `custom.yaml` file with the following:

```yaml
ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/issuer: "letsencrypt-staging"
  hosts:
    - host: expressjs.inlets.dev
      paths: ["/"]
  tls:
   - secretName: expressjs-tls
     hosts:
       - expressjs.inlets.dev
```

Replace the string `expressjs.inlets.dev` with your own sub-domain created earlier i.e. `expressjs.example.com`.

You can download around a dozen other CLI tools using arkade including helm. Use arkade to download helm and put it in your `PATH`:

```bash
arkade get helm

# Put arkade in your path:
export PATH=$PATH:$HOME/.arkade/bin/helm3/

# Or alternatively install to /usr/local/bin
sudo cp $HOME/.arkade/bin/helm3/helm /usr/local/bin/
```

Now install the chart using helm:

```bash
helm repo add expressjs-k8s https://alexellis.github.io/expressjs-k8s/

# Then they run an update
helm repo update

# And finally they install
helm upgrade --install express expressjs-k8s/expressjs-k8s \
  --values custom.yaml
```

## Test it out

Now check the certificate has been created and visit the webpage in a browser:

```bash
kubectl get certificate

NAME            READY   SECRET          AGE
expressjs-tls   True    expressjs-tls   49s
```

Open the webpage i.e. https://api.example.com. Since this is a staging certificate, you will get a warning
from your browser. You can accept the certificate in order to test your site.

## Getting a Production Certificate

Create a production certificate issuer `issuer-prod.yaml`, similar to the staging issuer you produced
earlier. Be sure to change the email address to your email.

```bash
export EMAIL="you@example.com"

cat > issuer-prod.yaml <<EOF
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-prod
  namespace: default
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: $EMAIL
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - selector: {}
      http01:
        ingress:
          class: nginx
EOF
```

Then run `kubectl apply -f issuer-prod.yaml`

Now you must update your `expressjs` deployment to use the new certificate issuer. Create a new
helm3 overrides file `custom-prod.yaml`:

```bash
cat > custom-prod.yaml <<EOF
ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/issuer: "letsencrypt-prod"
  hosts:
    - host: expressjs.inlets.dev
      paths: ["/"]
  tls:
   - secretName: expressjs-tls
     hosts:
       - expressjs.inlets.dev
EOF
```

Be sure to change the above domain name to your domain name for the sample server.

You can update your deployment using the helm command below:

```bash
helm upgrade express expressjs-k8s/expressjs-k8s \
  --values custom-prod.yaml
```

Here's my example on my own domain:

![The page with TLS](../images/operator-pro-webpage.png)

You can view the certificate the certificate that's being served directly from your local cluster and see that it's valid:

![Green lock](../images/operator-pro-webpage-letsencrypt.png)

## Install a real-world application

Using arkade you can now install OpenFaaS or a Docker Registry with a couple of commands, and since you have Nginx and cert-manager in place, this will only take a few moments.

### OpenFaaS with TLS

OpenFaaS is a platform for Kubernetes that provides FaaS functionality and microservices. The motto of the project is [Serverless Functions Made Simple](https://www.openfaas.com/) and you can deploy it along with TLS in just a couple of commands:

```bash
export DOMAIN=gateway.example.com
arkade install openfaas
arkade install openfaas-ingress \
  --email webmaster@$DOMAIN \
  --domain $DOMAIN
```

That's it, you'll now be able to access your gateway at https://$DOMAIN/

For more, see the [OpenFaaS workshop](https://github.com/openfaas/workshop/)

### Docker Registry with TLS

A self-hosted Docker Registry with TLS and private authentication can be hard to set up, but we can now do that with two commands.

```bash
export DOMAIN=registry.example.com
arkade install docker-registry
arkade install docker-registry-ingress \
  --email webmaster@$DOMAIN \
  --domain $DOMAIN
```

Now try your registry:

```bash
docker login $DOMAIN
docker pull alpine:3.22.1
docker tag alpine:3.22.1 $DOMAIN/alpine:3.22.1

docker push $DOMAIN/alpine:3.22.1
```

You can even combine the new private registry with OpenFaaS if you like, [checkout the docs for more](https://docs.openfaas.com/).

## Wrapping up

Through the use of inlets-pro we have an encrypted control-plane for the websocket tunnel, and encryption for the traffic going to our Express.js app using a TLS certificate from LetsEncrypt.

You can now get a green lock and a valid TLS certificate for your local cluster, which also means that this will work with bare-metal Kubernetes, on-premises and with your Raspberry Pi cluster.

