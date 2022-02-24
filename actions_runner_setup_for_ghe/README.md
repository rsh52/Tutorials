# GitHub Actions Runner Setup for GitHub Enterprise

---

# Purpose

This tutorial describes how to set up a **Kubernetes** cluster on **Amazon Web Services (AWS)** to launch **runners** accessible to applications on a **GitHub Enterprise Server (GHES)**. This tutorial is not targetted towards Kubernetes experts and is meant to be accessible and approachable by all audiences. 

While it is possible for GHES actions runners to be set up and streamlined for all members of an organization, not all places have the infrastructure or resources to enable it. Under these circumstances, this tutorial will be helpful.

### Prerequisites

The following are required for setup:

- A Mac computer outfitted with `homebrew`
- An AWS account
- A GitHub Enterprise server
- An understanding of [Continuous Integration (CI)](https://docs.github.com/en/actions/automating-builds-and-tests/about-continuous-integration)
- An understanding of [GitHub Actions anatomy](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions)

## Background

**GitHub Actions (GHA)** is a powerful workflow automation engine. However, to use GHA on GHES, you need to provide your own runtime environments, also known as runners.

Runners need to be configured to talk to GHES and advertise that they are able to accept **jobs**. In addition, if there is more than one **workflow** that gets triggered by a GHA, more than one runner needs to be available. For more information about GHA components see [GithHub's documentation](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions).

A widely used open-source community-maintained solution is available [**here**](https://github.com/actions-runner-controller/actions-runner-controller). The actions-runner-controller repo provides a Kubernetes controller for self-hosted runner deployments. It supports the following features:

- Can be used with GitHub Enterprise Server
- Can be registered on the repo, org and enterprise level and may register with custom labels inside runner groups
- Autoscaling based on pending and running jobs or percentage of runners already busy
- Automatic cleanup after each run
- Allows customization of software installed in the self-hosted runner image

In order to set up a Kubernetes cluster, AWS Elastic Kubernetes Service (EKS) works as the managed service for automation of scaling, deployment, and overall management of the cluster(s).

---

# Set up Kubernetes Cluster on EKS

## Get AWS Credentials

As a first step, be sure to clone this repository to your local computer and then cd into its location.

Depending on your organization or self-accessed AWS channel, there are a number of ways to retrieve AWS "keys" both through the Management Console as well as the AWS CLI. See the [AWS Documentation](https://docs.aws.amazon.com/general/latest/gr/aws-sec-cred-types.html) for more information on how to retrieve these.

You will need the following 3 variables:

- `AWS_ACCESS_KEY_ID` (something like `AKIAIOSFODNN7EXAMPLE`)
- `AWS_SECRET_ACCESS_KEY` (something like `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY`)
- `AWS_SESSION_TOKEN`

> **Note:** The `AWS_SESSION_TOKEN` may not be required and is based on security requirements enforced on the account.

## Install the Kubernetes CLI (if necessary)

The `kubectl` command line utility (CLI) must be within +/-1 minor versions of the AWS control plane. Install the newest version of `kubectl`, which at the time of this writing is **v1.20**.

```bash
# on Mac
brew install kubernetes-cli
```

For installation options check the [Kubernetes Install Tools page](https://kubernetes.io/docs/tasks/tools/).

## Install the `eksctl` utility (if necessary)
This tool automates common workflows on EKS.

```bash
# on Mac
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
```

## Create the Kubernetes Cluster

Now it's showtime!

Let's create our `cluster.yaml` manifest which will provide the instructions for the following:

- Kubernetes control plane
- Metadata: name, region, version
- VPC subnets
- Node information: name, type, amount, size, etc.

```yaml
apiVersion: eksctl.io/v1alpha5
Kind: ClusterConfig

metadata:
  name: mycluster
  region: us-east-1
  version: "1.20"

vpc:
  subnets:
    private:
      us-east-1a: { id: "subnet-<yoursubnetinfo1>" }
      us-east-1b: { id: "subnet-<yoursubnetinfo2>" }

managedNodeGroups:
  - name: ekstest-ng
    instanceType: t2.micro
    desiredCapacity: 2
    privateNetworking: true
    volumeSize: 30
    ssh:
      allow: true
      publicKeyName: mykey

```

Things to note: 

- Replace the necessary elements with your cluster information
- The current YAML uses t2.micro (free tier) EC2 instances which may not meet requirements for most projects

Once set up, run the following to create the cluster. This will take **20-30minutes to complete** ☕:

```bash
eksctl create cluster -f cluster.yaml
```

A lot of dialogue will fly across your terminal, the important thing is to see "ready" statuses after completion and a final:

```
2021-07-07 12:08:01 [✔]  EKS cluster "cgt" in "us-east-1" region is ready
```

You can then confirm the available nodes with:

```bash
kubectl get nodes -o wide
```

## Deploy the GitHub Actions Runner Application

If you've made it this far, congrats! 🎉 Just a little ways left to go. In this section we will walk through the following:

1. Install cert-manager
2. Install actions-runner-controller
3. Generate a GitHub Enterprise personal access token (PAT) with appropriate permissions
4. Apply a RunnerDeployment object to the cluster

### Install cert-manager

The actions-runner-controller app requires [cert-manager](https://cert-manager.io/docs/). This is a Kubernetes controller that implements a general purpose certification management system and is needed for the GHA controller to talk to GitHub/GHES. Installation instructions can be found [here](https://cert-manager.io/docs/installation/kubernetes/).

Run the following:

```bash
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.3.1/cert-manager.yaml
```

Once again, a lot of stuff will fly across your terminal, but luckily this won't take very long.

Next, tell the controller manager about your organization's GHES instance:

```bash
kubectl set env deploy controller-manager \
  -c manager GITHUB_ENTERPRISE_URL=<YOUR GHE URL HERE> \
  --namespace actions-runner-system
```

And confrim the following return:

```
deployment.apps/controller-manager env updated
```

### Generate Personal Access Token on GHE Servers (if necesary)

This section may not be necessary for all users, but if a PAT already exists you may want to check the following settings are enabled.

Log into GitHub Enterprise > in the upper-right corner of any page, click your profile photo > click **Settings**.

In the left sidebar, click **Developer settings** > **Personal access tokens** > **Generate new token**.

Allow the following:

- [x] repo (Full control)
- [x] workflow (Full control)
- [x] admin:org (Full control)
- [x] admin:public_key (read:public_key)
- [x] admin:repo_hook (read:repo_hook)
- [x] admin:org_hook (Full control)
- [x] notifications (Full control)
- [x] admin:enterprise (Full control)

Click **Generate token**.

On the next page, copy the PAT and store it in a safe place. In order to use it, you can create an environment variable:

```bash
kubectl create secret generic controller-manager \
    -n actions-runner-system \
    --from-literal=github_token=$GHA_GHES_TOKEN
```

And confirm:

```
secret/controller-manager created
```

### Create a Runner Deployment

The `actions-runner-controller` supports a Kubernetes object of the kind `RunnerDeployment` which is analogous to the `Deployment` object in default Kubernetes and allows spinning up multiple pods and providing self-healing and fault tolerance.

The goal here is to accomplish the following:

- Create a `RunnerDeployment` named `my-org-runners`
- Have 3 runners available at all times
- The scope of the runner should be your GitHub organization
- Runners should have the extra label `my-runner`. This will allow targeting of GitHub Actions workflows to these specific runners.

The `runners.yaml` file which implements these settings is reproduced below:

```yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: my-org-runners
spec:
  replicas: 3
  template:
    spec:
      organization: CGTDataOps
      labels:
      - my-runner
```

To deploy, run:

```bash
kubectl apply -f runners.yaml
```

```
runnerdeployment.actions.summerwind.dev/cgt-dataops-runners created
```

After a short while, the pods should be online. You can see your pods with:

```bash
kubectl get pods -o wide
```

### Set up the Github Organization (only needed once)

Go to GHES > Settings > Organizations > YourOrganization Settings > Actions > scroll down. In the **Runner Groups** table, expand the **Default** ribbon. You should see at least 3 runners! You may also see "Offline" runners from previous deployments. The 3 runners we just deployed should be "Idle" at this time.

By default, GitHub Enterprise makes runners available to private repos only, however in your org you may like to keep repos public so others can see your work. To make the runners accessible to public repos, click on ... at the far right of the **Runner Groups** ribbon > **Edit name and repository access** > ✓ **Allow public repositories** > **Save group**.

### Using the Runners
There is no additional setup needed on a repository level to use the runners, as long as the repository is within your organization.

To verify that you have runners available for your repo, go to Settings > Actions, and scroll all the way to the bottom. You should see that the 3 runners are shared with the repository.

## Clean Up

Should you choose to tear down the cluster, run the following command and allot about 5 minutes for it to complete:

```bash
eksctl delete cluster -f cluster.yaml
```

And confirm the terminal ends with something like the following:

```
2021-07-08 11:14:49 [✔]  all cluster resources were deleted
```

---

## Troubleshooting

#### _Problem: AWS EKS clusters terminate during creation_

AWS permissions may need to be elevated for the user that are currently disabled. If security settings on the AWS account are set to rotate keys, it is possible the AWS Access Key/Secret Access Key changes during EKS cluster creation and causes creation failure.

#### _Problem: `kubectl` version incorrect_

This can arise from having previously installed Docker. Check if the `kubectl` on the machine is derived symbolically from inside the `Docker.app` folder using:

```bash
ls -al /usr/local/bin/kubectl
```

If it is a symbolic link, remove it and link the versio from `homebrew` (if on Mac):

```bash
sudo rm /usr/local/bin/kubectl
brew link kubernetes-cli
```

---

## Sources

- [**AWS Getting Started `eksctl`**](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html)
- [**Github Actions Runner Controller**](https://github.com/actions-runner-controller/actions-runner-controller)
