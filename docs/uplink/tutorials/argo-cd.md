# Manage apps on remote Kubernetes clusters with ArgoCD

This tutorial shows how to use [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) with [Inlets Uplink](/uplink/) to manage applications across multiple remote Kubernetes clusters from a centralized GitOps control plane.

With Inlets Uplink, each remote cluster connects back to the control plane over a secure, TLS-encrypted tunnel. This allows Argo CD to manage remote clusters without exposing their APIs to the public internet.

![Architructurel diagram, remote cluster management with ArgoCD and Inlets Uplink](/images/uplink-argocd-remote-cluster-management.png)

In this tutorial, we will:

- Create Uplink tunnels for one or more remote Kubernetes clusters.
- Register those clusters with Argo CD.
- Show how an applicatioin can be deployed to all registerd cluster.

Why use this approach?

- Security - Cluster APIs stay private, with no public exposure. Inlets Uplink creates a secure, encrypted tunnel for Argo CD to connect.
- Centralized management - Operate and deploy to all clusters from a single Argo CD control plane.
- Scalability - Add new clusters easily by creating a tunnel and registering it in Argo CD.

This tutorial has been tested with K3s clusters but works with any Kubernetes distribution.

## Prerequisites

- A central cluster with [Inlets Uplink installed](/uplink/installation/)
- One or more remote K3s clusters to manage
- [arkade](https://arkade.dev) installed
- Basic knowledge of Kubernetes and GitOps

## Step 1: Create Uplink tunnels for remote clusters

For each remote cluster, create a dedicated tunnel. These could be created in separate namespaces for better isolation when e.g. adding clusters for multiple different tenants.

Create a namespace for the first remote cluster:

```bash
export CLUSTER_NAME="production-east"
export NS="tenant1"

kubectl create namespace $NS
kubectl label namespace $NS "inlets.dev/uplink"=1
```

Copy your Uplink license to the new namespace:

```bash
export LICENSE=$(kubectl get secret -n inlets inlets-uplink-license -o jsonpath='{.data.license}' | base64 -d)

kubectl create secret generic \
  -n $NS \
  inlets-uplink-license \
  --from-literal license=$LICENSE
```

Create the tunnel resource for the Kubernetes API:

```bash
kubectl apply -f - <<EOF
apiVersion: uplink.inlets.dev/v1alpha1
kind: Tunnel
metadata:
  name: kubernetes-$CLUSTER_NAME
  namespace: $NS
spec:
  licenseRef:
    name: inlets-uplink-license
    namespace: $NS
  tcpPorts:
    - 6443
EOF
```

> See [create tunnel](/uplink/create-tunnels/) for more details on managing tunnels.

Get the inlets uplink management CLI, and use it to generate a Kubernetes Deployment definition for the tunnel client.

```bash
inlets-pro plugin get tunnel

inlets-pro tunnel connect kubernetes-$CLUSTER_NAME \
  --namespace $NS \
  --domain uplink.example.com \
  --upstream 6443=kubernetes.default.svc:443 \
  --format k8s_yaml > $CLUSTER_NAME-tunnel-client.yaml
```

Replace `uplink.example.com` with your actual Uplink domain.

The client Deployment YAML will have to be applied to the remote cluster.

Switch over to the customer’s Kubernetes cluster and apply the YAML for the tunnel client:

```bash
# Switch to remote cluster context
kubectl config use-context $CLUSTER_NAME

# Deploy the tunnel client
kubectl apply -f $CLUSTER_NAME-tunnel-client.yaml

# Verify connection
kubectl logs deploy/kubernetes-$CLUSTER_NAME-inlets-client
```

> See [connect the tunnel client](/uplink/connect-tunnel-client/) for more details on connecting the client.

Repeat these steps for each remote cluster you want to manage.

## Step 2: Install ArgoCD on the central cluster

Switch back to your central cluster and install ArgoCD:

```bash
arkade install argocd
```

Get the ArgoCD CLI:

```bash
arkade get argocd
```

Port-forward the ArgoCD server:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
```

Login to ArgoCD:

```bash
PASS=$(kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d)

argocd login 127.0.0.1:8080 --insecure \
  --username admin \
  --password $PASS
```

## Step 3: Register remote clusters with ArgoCD

Clusters can be registered declaratively with Argo CD by adding a sectet containing the cluster credentials. Each secret must have label `argocd.argoproj.io/secret-type: cluster`.
Checkout the [Argo CD documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#clusters) for more info.

We provide a small utility to generate these secrets from a kubeconfig file.

Generate and apply the cluster secret for a remote cluster:

```bash
# Ensure the tunnel plugin is installed
inlets-pro plugin get tunnel

inlets-rpo tunnel argo generate $CLUSTER_NAME \
  --kubeconfig ~/.kube/$CLUSTER_NAME.yaml \
  --upstream https://kubernetes-$CLUSTER_NAME.$NS:6443 | \
  kubectl apply -f -
```

Repeat this for each remoter cluster you want to add.

Verify the cluster is registered:

```bash
argocd cluster list
```

You should see your remote cluster listed with the tunneled URL.

## Deploy applications to all clusters

Now that the cluster are reachable and registered with ArgoCD we can start deploying applications to them. From here on we are just going to apply standard ArgoCD practices and patterns.

> Checkout the Argo CD documentation for more info on a [declarative application setup](https://www.openfaas.com/blog/argocd-image-updater-for-functions/)

As an example we are going to install the [OpenFaaS ArgoCD App](https://github.com/welteki/openfaas-argocd-example). This is a sample application that uses the [app of apps pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/) to deploy two applications. The first application is named `openfaas-core` and deploys [OpenFaaS](https://www.openfaas.com/). The second application is named `openfaas-functions` and deploys a set of OpenFaaS functions.

For a detailed description of the application you can read our blog post: [How to update your OpenFaaS functions automatically with the Argo CD Image Updater](https://www.openfaas.com/blog/argocd-image-updater-for-functions/)

We can deploy the application to a single cluster as described in the original article but ideally we want to deploy it to multiple remote clusters. To achieve this we can use an [ApplicationSet](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/) to automatically generate multiple applications.

An ApplicationSet uses generators to create Applications dynamically. The cluster generator discovers all registered clusters and creates an Application for each one. This eliminates the need to manually create separate Application manifests for every cluster.

Example ApplicationSet for deploying to multiple clusters:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: openfaas-multi-cluster
  namespace: argocd
spec:
  generators:
    # Cluster Generator: Creates parameters for every registered cluster
    - clusters:
        # Use a label selector to filter for cluster Secrets.
        # The 'cluster' label is automatically added to Secrets
        # created for *remote* clusters, but not for the local 'in-cluster' setup.
        selector:
          matchLabels:
            argocd.argoproj.io/secret-type: cluster
  template:
    metadata:
      # Template creates unique app names using cluster name: openfaas-production-east
      name: openfaas-{{name}}
      namespace: argocd
    spec:
      project: default
      source:
        repoURL: https://github.com/welteki/openfaas-argocd-example.git
        path: chart/openfaas
        helm:
          parameters:
            # Override the destination server URL and appNameSuffix with the values
            # provided by the generator
            - name: spec.destination.server
              value: "{{server}}"
            - name: appNameSuffix
              value: "{{name}}"
      destination:
        # Deploy to ArgoCD namespace on central cluster (manages remote deployments)
        server: "https://kubernetes.default.svc"
        namespace: argocd
      syncPolicy:
        automated:
          selfHeal: true
```

Apply the ApplicationSet:

```bash
kubectl apply -f openfaas-applicationset.yaml
```

The ApplicationSet will automatically create individual Applications for each registered cluster. You can monitor the deployment progress in the ArgoCD UI or via CLI:

```bash
argocd app list
```

![Screenshot showing the applications for each cluster in the Argo CD UI](/images/uplink-argocd-applicationset-ui.png)
> Argo CD UI showing the openfaas application deployed to 3 different remore clusters.

## Conclusion

You now have a reference architecture for centralized GitOps management of multiple remote Kubernetes clusters using ArgoCD and Inlets Uplink. This approach provides:

- **Security**: No public exposure of cluster APIs
- **Scalability**: Easy addition of new clusters
- **Centralization**: Single point of control for all deployments
- **Reliability**: Secure, encrypted tunnel connections

The combination of Inlets Uplink tunnels and ArgoCD provides a robust foundation for multi-cluster GitOps at scale while maintaining strong security boundaries.

If you run into network reliability issues consider running the Uplink tunnels in [demux mode](/uplink/troubleshooting/#demux-mode) to improve reliability.
