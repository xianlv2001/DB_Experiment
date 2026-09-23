# Lab 1: Deploy a TiDB Cluster on AWS EKS

This lab builds a **TiDB** cluster on **Amazon EKS**, managed by **TiDB Operator** and provisioned as code with **Pulumi**.

| Technology | Role in This Lab |
| ---------- | ---------------- |
| Amazon EKS | Managed Kubernetes cluster on AWS |
| TiDB Operator | Deploys and operates TiDB on Kubernetes |
| Pulumi | Defines the whole stack as Infrastructure-as-Code |
| TiDB | Distributed, MySQL-compatible SQL database (HTAP) |
| TiDB Dashboard & Grafana | Cluster diagnostics and monitoring |

<!-- TOC -->
* [Lab 1: Deploy a TiDB Cluster on AWS EKS](#lab-1-deploy-a-tidb-cluster-on-aws-eks)
  * [Prerequisites](#prerequisites)
  * [Lab Roadmap](#lab-roadmap)
  * [AWS Billing](#aws-billing)
<!-- TOC -->

## Prerequisites

- AWS account with API access enabled
- VPN access to AWS and GitHub
- Linux, macOS, or WSL2

## Lab Roadmap

Total: **100 points + 20 bonus points = 120 points**. Complete the steps in order.

| # | Step | Points | Est. Time |
| :- | :--- | -----: | :-------- |
| 0 | [Install dependencies](./0-install-dependencies/README.md) — `kubectl`, Node.js, AWS CLI, Pulumi, Helm | — | — |
| 1 | [Create an EKS cluster](./1-create-an-eks-cluster/README.md) | 25 | >10 min |
| 2 | [Deploy TiDB with TiDB Operator](./2-deploy-tidb-with-tidb-operator/README.md) | 25 | ~10 min |
| 3 | [Explore TiDB basic usage](./3-explore-tidb-basic-usage/README.md) | 20 | ~10 min |
| 4 | [Scale up the TiDB cluster](./4-scale-up-tidb-cluster-with-tidb-operator/README.md) | 20 | ~10 min |
| 5 | [Cleanup: destroy the EKS cluster via Pulumi](./1-create-an-eks-cluster/README.md#do-not-execute-this-step-until-lab-1-finished-destroy-the-eks-cluster-via-pulumi) | 10 | — |
| B | Bonus: [configure the TiDB slow-log threshold](./%5Bbonus%5Dconfig-slow-log-threshold/README.md) | 20 | — |

> - Run steps 1–4 in **the same shell session**; if it is closed, re-run `export KUBECONFIG=$PWD/../1-create-an-eks-cluster/kubeconfig.yaml`.
> - The lab brief is available as [`VLDB ss 2023 lab 1.pdf`](<./VLDB ss 2023 lab 1.pdf>) and [`.pptx`](<./VLDB ss 2023 lab 1.pptx>).

## AWS Billing

| Resource | Quantity | Unit Price |
| -------- | -------- | ---------- |
| EKS control plane | 1 | $0.10 / hour |
| EC2 `t2.medium` worker | 2 | $0.0464 / hour |
| EBS volume (1 GiB) | 4 | negligible |
| **Total** | | **$0.1928 / hour** |

> Destroy the cluster via `pulumi destroy` (step 5) once the lab is finished to stop billing.
