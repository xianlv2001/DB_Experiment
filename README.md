# Lab 1: Deploy TiDB Cluster on AWS EKS

This lab deploys a **TiDB cluster** on **AWS EKS**. The TiDB cluster is managed by **TiDB Operator**, and the deployment process is automated with **Pulumi**.

<!-- TOC -->
* [Lab 1: Deploy TiDB Cluster on AWS EKS](#lab-1-deploy-tidb-cluster-on-aws-eks)
  * [Introduction](#introduction)
  * [Learning Objectives](#learning-objectives)
  * [Pre-Requisites](#pre-requisites)
  * [Repository Layout](#repository-layout)
  * [Syllabus](#syllabus)
  * [AWS billing price](#aws-billing-price)
<!-- TOC -->

## Introduction

**Cloud computing** is the delivery of computing **services** -- including servers, storage, databases, networking, software, analytics, and intelligence -- over the internet ("the cloud") to offer faster innovation, flexible resources, and economies of scale, as your business needs change.

**Amazon EKS** is a managed Kubernetes service that makes it easy for you to run **Kubernetes** on AWS. Kubernetes offers automating deployment, scaling, and management of containerized applications.

**TiDB** is an open-source distributed SQL database that supports HTAP workloads. It provides users with a one-stop database solution, and helps improve scalability, availability and reliability for users' data storage systems.

When deploying TiDB clusters on AWS EKS, users can gain all features provided by TiDB, while leverage the benefits of operating TiDB clusters on managed Kubernetes services.

## Learning Objectives

- Understand basic usage of Kubernetes
- Understand basic usage of AWS Kubernetes service (EKS)
- Understand the fundamentals of operator pattern
- Learn to deploy TiDB clusters on Kubernetes with TiDB Operator
- Understand basic usage of TiDB cluster
- Automate deployment process with Pulumi IaC framework

## Pre-Requisites

- An AWS account with permissions to create EKS clusters, EC2 instances, IAM roles, and VPC resources
- VPN for connecting to AWS API and GitHub
- Linux or MacOS or WSL2 environments
- A MySQL client (used by the scoring point in [Step 3](./3-explore-tidb-basic-usage/README.md)); install it in [Step 0](./0-install-dependencies/README.md)

## Repository Layout

This repository is a TypeScript workspace: each lab step is an independent **Pulumi project** (its own `Pulumi.yaml`), and all infrastructure is defined in the `index.ts` of each step directory.

```text
.
|-- 0-install-dependencies/            # Setup guide (kubectl, Node.js 16, AWS CLI, Pulumi, Helm). No code.
|-- 1-create-an-eks-cluster/           # Pulumi project: EKS cluster + EBS CSI driver addon.
|   |-- index.ts                       #   Pulumi program. Exports the `kubeconfig` stack output.
|   |-- Pulumi.yaml                    #   Project metadata; pins the backend to `file://~/.pulumi`.
|   `-- Pulumi.default.yaml            #   `default` stack config (encryption salt).
|-- 2-deploy-tidb-with-tidb-operator/  # Pulumi project: TiDB Operator + TiDB cluster.
|   |-- index.ts                       #   Reads Step 1's `kubeconfig` via StackReference, then installs CRDs,
|   |                                  #   the TiDB Operator Helm chart, and the cluster manifests below.
|   |-- crds/tidb-operator-v1.4.4.yaml #   TiDB Operator CRDs (TidbCluster, TidbMonitor, TidbDashboard, ...).
|   `-- tidb-cluster-manifests/        #   TiDB cluster CR manifests, applied through the TiDB Operator:
|       |-- tidb-cluster.yaml          #     `TidbCluster` CR: PD + TiKV + TiDB (1 replica each, v7.1.0).
|       |-- tidb-dashboard.yaml        #     `TidbDashboard` CR: the web UI for the cluster.
|       `-- tidb-monitor.yaml          #     `TidbMonitor` CR: Prometheus + Grafana monitoring stack.
|-- 3-explore-tidb-basic-usage/        # Read-only exploration of the deployed cluster.
|   `-- set-up-port-forward.sh         #   Port-forwards TiDB (4000), TiDB Dashboard (12333), Grafana (3000).
|-- 4-scale-up-tidb-cluster-with-tidb-operator/  # Guide: scale TiKV 1 -> 2 by editing Step 2's manifest.
`-- [bonus]config-slow-log-threshold/  # Guide + helper script for the slow-log threshold bonus task.
```

Notes:

- **Run every Pulumi command from its own step directory.** Each `Pulumi.yaml` declares the project name; running `pulumi up` from the wrong directory creates or updates the wrong stack.
- Step 2 depends on Step 1: it resolves `kubeconfig` through a `StackReference` to `organization/1-create-an-eks-cluster/default` (see the [Step 2 README](./2-deploy-tidb-with-tidb-operator/README.md) for the required one-line change).
- `Makefile`: `make install` runs `npm install`; the default target `make lint` runs `npx eslint --fix .` (it rewrites files in place).
- `.imgs/` holds the images referenced by the READMEs; `VLDB ss 2023 lab 1.pdf/pptx` are the original course slides.

## Syllabus

> 100 basic points + 20 bonus points = 120 total points.

- Step 0: Install Dependencies [`0-install-dependencies`](./0-install-dependencies/README.md)

1. (25 points) Create an EKS cluster [`1-create-an-eks-cluster`](./1-create-an-eks-cluster/README.md)
2. (25 points) Deploy TiDB with TiDB
   Operator [`2-deploy-tidb-with-tidb-operator`](./2-deploy-tidb-with-tidb-operator/README.md)
3. (20 points) Explore TiDB basic usage [`3-explore-tidb-basic-usage`](./3-explore-tidb-basic-usage/README.md)
4. (20 points) Scale up TiDB cluster with TiDB
   Operator [`4-scale-up-tidb-cluster-with-tidb-operator`](./4-scale-up-tidb-cluster-with-tidb-operator/README.md)
5. (10 points) Cleanup: Destroy the EKS cluster (documented at the end of the Step 1 README: [destroy the EKS cluster via
   Pulumi](./1-create-an-eks-cluster/README.md#do-not-execute-this-step-until-the-whole-lab-is-finished-destroy-the-eks-cluster-via-pulumi))

- (20 bonus points) Bonus: Config the TiDB slow-log threshold and update the cluster with Pulumi [[bonus]config-slow-log-threshold](./[bonus]config-slow-log-threshold/README.md)

---

## AWS billing price

This lab will incur charges under the AWS account, described in detail at:

- New EKS cluster control plane, **_1_** cluster x **_0.10_** USD per hour
- Two EKS worker EC2 `t2.medium` instances, **_2_** instances * **_0.0464_** USD per hour
- 4 EBS of size 1 GiB, with negligible cost

Total **_0.1928_** USD per hour.

> - Prices vary by AWS region and change over time; check your own AWS bill for the real numbers.
> - The TiDB cluster manifest sets `pvReclaimPolicy: Retain`, so the EBS volumes behind the PD/TiKV/TiDB Dashboard pods are **not** deleted by `pulumi destroy`. Remove them manually in the AWS console if you no longer need them.
> - The manifest declares 1 GiB volumes for PD, TiKV and the dashboard; the monitor stack creates its own PVCs through operator defaults, and actual prices vary by region -- the numbers above are the original 2023 lab estimates.
