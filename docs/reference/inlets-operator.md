# inlets-operator reference documentation

The [inlets/inlets-operator](https://github.com/inlets/inlets-operator) brings LoadBalancers with public IP addresses to your local Kubernetes clusters.

> It works by creating VMs and running an inlets Pro tunnel server for you, the VM's public IP is then attached to the cluster and an inlets client Pod runs for you.

For each provider, the minimum requirements tend to be:

* An access token - for the operator to create VMs for inlets Pro servers
* A region - where to create the VMs

**Helm or Arkade?**

You can install the inlets-operator's Helm chart using a single command with [arkade](https://arkade.dev/). arkade is an open-source Kubernetes marketplace and easy to use. Helm involves more commands, and is preferred by power users.

> [Get your inlets subscription here](https://inlets.dev/pricing)

## Tunnel Custom Resource Definition (CRD) and lifecycle

The inlets-operator uses a custom resource definition (CRD) to create tunnels. The CRD is called `Tunnel` and its full name is `tunnels.operator.inlets.dev`

```bash
$ kubectl get tunnels -n default
NAMESPACE   NAME             SERVICE   HOSTSTATUS   HOSTIP        CREATED
default     nginx-1-tunnel   nginx-1   active       46.101.1.67   2m45s
```

The CRD can be used to view and monitor tunnels. The `HOSTSTATUS` field shows the status of the tunnel, and the `HOSTIP` field shows the public IP address of the tunnel.

The tunnel's IP address will also be written directly to any `Service` with a type of `LoadBalancer`.

```bash
$ kubectl get svc -n default
NAME         TYPE           CLUSTER-IP    EXTERNAL-IP               PORT(S)        AGE
kubernetes   ClusterIP      10.96.0.1     <none>                    443/TCP        6m26s
nginx-1      LoadBalancer   10.96.94.18   46.101.1.67,46.101.1.67   80:31194/TCP   4m21s
```

The lifecycle of a tunnel is tied to the Service in Kubernetes.

To delete a tunnel permanently, you can delete the Service:

```bash
kubectl delete svc nginx-1
```

To have the tunnel server re-created, you can delete the tunnel CustomResource, this causes the operator to re-create the tunnel:

```bash
kubectl delete tunnel nginx-1-tunnel
```

Bear in mind that if you delete your cluster before you delete the LoadBalancer service, then the inlets-operator will have no way to remove the tunnel servers that have been created for you. Therefore, you should always delete the LoadBalancer service before deleting the cluster. If you should forget, and delete your K3s or KinD cluster, then you can go into your cloud account and delete the VMs manually.

As a rule, the name of the VM will match the name of the service in Kubernetes.

### Working with another LoadBalancer

If you're running metal-lb or kube-vip to provide local IP addresses for LoadBalancer services, then you can annotate the services you wish to expose to the Internet with `operator.inlets.dev/manage=1`, then set `annotatedOnly: true` in the inlets-operator Helm chart.

## Install inlets-operator using arkade

```bash
export REGION=lon1
export PROVIDER=digitalocean

arkade install inlets-operator \
 --provider $PROVIDER \ # Name of the cloud provider to provision the exit-node on.
 --region $REGION \ # Used with cloud providers that require a region.
 --token-file $HOME/Downloads/do-access-token.txt # Token file/Service Account Key file with the access to the cloud provider.
```

## Install inlets-operator using helm

The following instructions are a generic example, you should refer to each specific heading to understand how to create the required API keys for a given cloud provider.

* Some providers require an access key, others also need a secret key.
* Some providers only use a region, others use a zone and projectID too.
* There are additional flags you can set via values.yaml or the `--set` flag.

You can view the [inlets-operator chart on GitHub](https://github.com/inlets/inlets-operator/tree/master/chart/inlets-operator) to learn more.

```bash
# Create a namespace for inlets-operator
kubectl create namespace inlets

# Create a secret to store the service account key file
kubectl create secret generic inlets-access-key \
  --namespace inlets \
  --from-file inlets-access-key=$HOME/Downloads/do-access-token.txt

# Create a secret to store the inlets-pro license
kubectl create secret generic \
  --namespace inlets \
  inlets-license --from-file license=$HOME/.inlets/LICENSE

# Add and update the inlets-operator helm repo
# You only need to do this once.
helm repo add inlets https://inlets.github.io/inlets-operator/

export REGION=lon1
export PROVIDER=digitalocean

# Update the Helm repository and perform an installation
helm repo update && \
  helm upgrade inlets-operator --install inlets/inlets-operator \
  --namespace inlets \
  --set provider=$PROVIDER \
  --set region=$REGION
```

## Instructions per cloud

### Create tunnel servers on DigitalOcean

The [DigitalOcean](https://m.do.co/c/8d4e75e9886f) provider is fast, cost effective and easy to set it. It's recommended for most users.

Create an API access token with full read/write permissions and save it to: `$HOME/Downloads/do-access-token.txt`.

Now, install the chart with arkade using the above options:

```bash
arkade install inlets-operator \
 --provider digitalocean \
 --region lon1 \
 --token-file $HOME/Downloads/do-access-token.txt
```

If you have the DigitalOcean CLI (`doctl`) installed, then you can use it to list available regions and their codes to input into the above command. Bear in mind that some regions are showing no availability for starting new VMs.

```bash
doctl compute region ls
Slug    Name               Available
nyc1    New York 1         true
sfo1    San Francisco 1    false
nyc2    New York 2         false
ams2    Amsterdam 2        false
sgp1    Singapore 1        true
lon1    London 1           true
nyc3    New York 3         true
ams3    Amsterdam 3        true
fra1    Frankfurt 1        true
tor1    Toronto 1          true
sfo2    San Francisco 2    true
blr1    Bangalore 1        true
sfo3    San Francisco 3    true
syd1    Sydney 1           true
```

### Create tunnel servers on AWS EC2

Instructions for [AWS EC2](https://aws.amazon.com/ec2/)

To use the instructions below you must have the AWS CLI configured with sufficient permissions to create users and roles.

- Create a AWS IAM Policy with the following:

Create a file named `policy.json` with the following content

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:AuthorizeSecurityGroupIngress",
        "ec2:DescribeInstances",
        "ec2:DescribeImages",
        "ec2:TerminateInstances",
        "ec2:CreateSecurityGroup",
        "ec2:CreateTags",
        "ec2:DeleteSecurityGroup",
        "ec2:RunInstances",
        "ec2:DescribeInstanceStatus"
      ],
      "Resource": ["*"]
    }
  ]
}
```

Create the policy in AWS

```bash
aws iam create-policy --policy-name inlets-automation --policy-document file://policy.json
```

- Create an IAM user

```bash
aws iam create-user --user-name inlets-automation
```

- Add the Policy to the IAM user

We need to use the policy arn generated above, it should have been printed to the console on success. It also follows the format below.

```bash
export AWS_ACCOUNT_NUMBER="Your AWS Account Number"
aws iam attach-user-policy --user-name inlets-automation --policy-arn arn:aws:iam::${AWS_ACCOUNT_NUMBER}:policy/inlets-automation
```

- Generate an access key for your IAM User

The below commands will create a set of credentials and save them into files for use later on.

> we are using [jq](https://stedolan.github.io/jq/) here. It can be installed using the link provided.
> Alternatively you can print ACCESS_KEY_JSON and create the files manually.

```bash
ACCESS_KEY_JSON=$(aws iam create-access-key --user-name inlets-automation)
echo $ACCESS_KEY_JSON | jq -r .AccessKey.AccessKeyId > ~/Downloads/aws-access-key
echo $ACCESS_KEY_JSON | jq -r .AccessKey.SecretAccessKey > ~/Downloads/aws-secret-access-key
```

Install the chart with arkade using the above options:

```bash
arkade install inlets-operator \
 --provider ec2 \
 --region eu-west-1 \
 --token-file $HOME/Downloads/aws-access-key \
 --secret-key-file $HOME/Downloads/aws-secret-access-key
```

### Create tunnel servers on Google Compute Engine (GCE)

Instructions for [Google Cloud](https://cloud.google.com/compute)

It is assumed that you have gcloud installed and configured on your machine.
If not, then follow the instructions [here](https://cloud.google.com/sdk/docs/quickstarts)

To get your service account key file, follow the steps below:

```bash
# Get current projectID
export PROJECTID=$(gcloud config get-value core/project 2>/dev/null)

# Create a service account
gcloud iam service-accounts create inlets \
  --description "inlets-operator service account" \
  --display-name "inlets"

# Get service account email
export SERVICEACCOUNT=$(gcloud iam service-accounts list | grep inlets | awk '{print $2}')

# Assign appropriate roles to inlets service account
gcloud projects add-iam-policy-binding $PROJECTID \
  --member serviceAccount:$SERVICEACCOUNT \
  --role roles/compute.admin

gcloud projects add-iam-policy-binding $PROJECTID \
  --member serviceAccount:$SERVICEACCOUNT \
  --role roles/iam.serviceAccountUser

# Create inlets service account key file
gcloud iam service-accounts keys create key.json \
  --iam-account $SERVICEACCOUNT
```

Install the chart with arkade using the above options:

```bash
arkade install inlets-operator \
    --provider gce \
    --project-id $PROJECTID \
    --zone us-central1-a \
    --token-file key.json
```

### Create tunnel servers on Hetzner

Create an API key with read/write access, save it to ~/hetzner.txt.

```bash
arkade install inlets-operator \
    --provider hetzner \
    --region eu-central \
    --token-file ~/hetzner.txt
```

### Create tunnel servers on Azure

Instructions for [Azure](https://azure.com)

Prerequisites:

- You will need `az`. See [Install the Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)
- You'll need to have run `az login` also

Generate Azure authentication file:

```sh
SUBSCRIPTION_ID="YOUR_SUBSCRIPTION_ID"
az ad sp create-for-rbac --role Contributor --scopes "/subscriptions/$SUBSCRIPTION_ID" --sdk-auth \
  > $HOME/Downloads/client_credentials.json
```

Find your region code with:

```bash
az account list-locations -o table

DisplayName               Name                 RegionalDisplayName
------------------------  -------------------  -------------------------------------
United Kingdom            ukwest               United Kingdom
```

Install using helm:

```bash
export SUBSCRIPTION_ID="YOUR_SUBSCRIPTION_ID"
export AZURE_REGION="ukwest"
export INLETS_LICENSE="$(cat ~/.inlets/LICENSE)"
export ACCESS_KEY="$HOME/Downloads/client_credentials.json"

kubectl create secret generic inlets-access-key \
  --from-file=inlets-access-key=$ACCESS_KEY

helm repo add inlets https://inlets.github.io/inlets-operator/
helm repo update

helm upgrade inlets-operator --install inlets/inlets-operator \
  --set provider=azure,region=$AZURE_REGION \
  --set subscriptionID=$SUBSCRIPTION_ID
```

### Create tunnel servers on Linode

Instructions for [Linode](https://linode.com)

Install using helm:

```bash
# Create a secret to store the service account key file
kubectl create secret generic inlets-access-key --from-literal inlets-access-key=<Linode API Access Key>

# Add and update the inlets-operator helm repo
helm repo add inlets https://inlets.github.io/inlets-operator/

helm repo update

# Install inlets-operator with the required fields
helm upgrade inlets-operator --install inlets/inlets-operator \
  --set provider=linode \
  --set region=us-east
```

You can also install the inlets-operator using a single command using [arkade](https://arkade.dev/), arkade runs against any Kubernetes cluster.

Install the chart with arkade using the above options:

```bash
arkade install inlets-operator \
 --provider linode \
 --region us-east \
 --access-key $LINODE_ACCESS_KEY
 ```
