# GitHub Actions Runner Setup for GitHub Enterprise

# Introduction

## Purpose

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

# Set up Kubernetes Cluster on EKS

## Get AWS Credentials

As a first step, be sure to clone this repository to your local computer and then cd into its location.

Depending on your organization or self-accessed AWS channel, there are a number of ways to retrieve AWS "keys" both through the Management Console as well as the AWS CLI. See the [AWS Documentation](https://docs.aws.amazon.com/general/latest/gr/aws-sec-cred-types.html) for more information on how to retrieve these.

You will need the following 3 variables:

- `AWS_ACCESS_KEY_ID` (something like `AKIAIOSFODNN7EXAMPLE`)
- `AWS_SECRET_ACCESS_KEY` (something like `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY`)
- `AWS_SESSION_TOKEN`

> **Note:** The `AWS_SESSION_TOKEN` may not be required and is based on security requirements enforced on the account.


### Troubleshooting

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
