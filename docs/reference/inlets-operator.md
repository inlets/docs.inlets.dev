# inlets-operator reference documentation

The [inlets/inlets-operator](https://github.com/inlets/inlets-operator) brings LoadBalancers with public IP addresses to your local Kubernetes clusters.

> It works by creating VMs and running an inlets Pro tunnel server for you, the VM's public IP is then attached to the cluster and an inlets client Pod runs for you.

You can install the inlets-operator using a single command with [arkade](https://arkade.dev/) or with helm. arkade is an open-source Kubernetes marketplace and easier to use.

For each provider, the minimum requirements tend to be:

* An access token - for the operator to create VMs for inlets Pro servers
* A region - where to create the VMs

> You can [subscribe to inlets for personal or commercial use via Gumroad](https://inlets.dev/blog/2021/07/27/monthly-subscription.html)

## Install using arkade

```bash
arkade install inlets-operator \
 --provider $PROVIDER \ # Name of the cloud provider to provision the exit-node on.
 --region $REGION \ # Used with cloud providers that require a region.
 --zone $ZONE \ # Used with cloud providers that require zone (e.g. gce).
 --token-file $HOME/Downloads/key.json # Token file/Service Account Key file with the access to the cloud provider.
```

## Install using helm

Checkout the inlets-operator helm chart [README](https://github.com/inlets/inlets-operator/blob/master/chart/inlets-operator/README.md) to know more about the values that can be passed to `--set` and to see provider specific example commands.

```bash
# Create a secret to store the service account key file
kubectl create secret generic inlets-access-key \
  --from-file=inlets-access-key=key.json

# Add and update the inlets-operator helm repo
helm repo add inlets https://inlets.github.io/inlets-operator/

# Create a namespace for inlets-operator
kubectl create namespace inlets

# Create a secret to store the inlets-pro license
kubectl create secret generic -n inlets \
  inlets-license --from-file license=$HOME/.inlets/LICENSE

# Update the local repository
helm repo update

# Install inlets-operator with the required fields
helm upgrade inlets-operator --install inlets/inlets-operator \
  --set provider=$PROJECTID,zone=$ZONE,region=$REGION \
  --set projectID=$PROJECTID \
  --set inletsProLicense=$LICENSE
```

View the code and chart on GitHub: [inlets/inlets-operator](https://github.com/inlets/inlets-operator)

## Instructions per cloud

### Create tunnel servers on DigitalOcean

Install with inlets Pro on [DigitalOcean](https://m.do.co/c/8d4e75e9886f).

Assuming you have created an API key and saved it to `$HOME/Downloads/do-access-token`, run:

```bash
arkade install inlets-operator \
 --provider digitalocean \
 --region lon1 \
 --token-file $HOME/Downloads/do-access-token
```

If you have `doctl` installed, you can list the available regions and see whether they have available capacity to launch a new VM.

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
echo $ACCESS_KEY_JSON | jq -r .AccessKey.AccessKeyId > access-key
echo $ACCESS_KEY_JSON | jq -r .AccessKey.SecretAccessKey > secret-access-key
```

Install with inlets Pro:

```bash
arkade install inlets-operator \
 --provider ec2 \
 --region eu-west-1 \
 --token-file $HOME/Downloads/access-key \
 --secret-key-file $HOME/Downloads/secret-access-key
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

Install the operator:

```bash
arkade install inlets-operator \
    --provider gce \
    --project-id $PROJECTID \
    --zone us-central1-a \
    --token-file key.json
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

Install with inlets Pro:

```bash
arkade install inlets-operator \
 --provider linode \
 --region us-east \
 --access-key $LINODE_ACCESS_KEY
 ```
