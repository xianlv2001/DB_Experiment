# Lab 1: Deploy TiDB Cluster on AWS EKS

This lab deploys a **TiDB cluster** on an **AWS EKS**. The TiDB cluster is managed by **TiDB-Operator**, and the deployment process is automated with **Pulumi**.

<!-- TOC -->
* [Lab 1: Deploy TiDB Cluster on AWS EKS](#lab-1-deploy-tidb-cluster-on-aws-eks)
  * [Introduction](#introduction)
  * [Learning Objectives](#learning-objectives)
  * [Pre-Requisites](#pre-requisites)
  * [Syllabus](#syllabus)
  * [Repository Layout](#repository-layout)
  * [Core Modules](#core-modules)
  * [Database Components](#database-components)
  * [How to Run](#how-to-run)
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

- An AWS account
- VPN for connecting to AWS API and GitHub
- Linux or MacOS or WSL2 environments
- Tools installed in [Step 0](./0-install-dependencies/README.md): Node.js 16, AWS CLI, Pulumi, `kubectl`, Helm
- A `mysql` command-line client (used in Step 3 and the Bonus step)

## Syllabus

> 100 basic points + 20 bonus points = 120 total points.

- Step 0: Install Dependencies [`0-install-dependencies`](./0-install-dependencies/README.md)

1. (25 points) Create an EKS cluster [`1-create-an-eks-cluster`](./1-create-an-eks-cluster/README.md)
2. (25 points) Deploy TiDB with TiDB Operator [`2-deploy-tidb-with-tidb-operator`](./2-deploy-tidb-with-tidb-operator/README.md)
3. (20 points) Explore TiDB basic usage [`3-explore-tidb-basic-usage`](./3-explore-tidb-basic-usage/README.md)
4. (20 points) Scale up TiDB cluster with TiDB Operator [`4-scale-up-tidb-cluster-with-tidb-operator`](./4-scale-up-tidb-cluster-with-tidb-operator/README.md)
5. (10 points) Cleanup: Destroy the EKS cluster [`finished-destroy-the-eks-cluster-via-pulumi`](./1-create-an-eks-cluster/README.md#do-not-execute-this-step-until-lab-1-finished-destroy-the-eks-cluster-via-pulumi)

- (20 bonus points) Bonus: Config the TiDB slow-log threshold and update the cluster with Pulumi [`[bonus]config-slow-log-threshold`](./%5Bbonus%5Dconfig-slow-log-threshold/README.md)

> **Order matters:** Steps 0 -> 1 -> 2 -> 3 -> 4 -> Bonus must be done in order, each
> reusing the same shell session (or re-exporting `KUBECONFIG`, see
> [How to Run](#how-to-run)). The Bonus step must be finished **before** Step 5
> destroys the cluster.

## Repository Layout

Every numbered directory (and the bonus directory) is **self-contained**, with its
own step-by-step `README.md`. Steps 1 and 2 additionally contain a standalone
**Pulumi program** (TypeScript): each has a `Pulumi.yaml` (project + local
`file://~/.pulumi` backend), a `Pulumi.default.yaml` (state encryption salt for the
`default` stack), and an `index.ts` entrypoint.

```
DB_Experiment/
+-- 0-install-dependencies/
|   +-- README.md                     # Tool installation (Node.js 16, AWS CLI, Pulumi, kubectl, Helm)
+-- 1-create-an-eks-cluster/
|   +-- README.md                     # Step 1 guide + EKS destroy instructions (Step 5 cleanup)
|   +-- index.ts                      # Pulumi program: EKS cluster + EBS CSI driver addon
|   +-- Pulumi.yaml                   # Pulumi project definition
|   +-- Pulumi.default.yaml           # Encryption salt for the `default` stack
+-- 2-deploy-tidb-with-tidb-operator/
|   +-- README.md                     # Step 2 guide: operator pattern + deployment
|   +-- index.ts                      # Pulumi program: TiDB Operator (Helm) + CRDs + TiDB cluster manifests
|   +-- Pulumi.yaml                   # Pulumi project definition
|   +-- Pulumi.default.yaml           # Encryption salt for the `default` stack
|   +-- crds/
|   |   +-- tidb-operator-v1.4.4.yaml # All TiDB Operator CRDs (v1.4.4), applied via ConfigGroup
|   +-- tidb-cluster-manifests/       # Applied by the same Pulumi program:
|       +-- tidb-cluster.yaml         #   TidbCluster CR (PD / TiKV / TiDB)
|       +-- tidb-monitor.yaml         #   TidbMonitor CR (Prometheus + Grafana)
|       +-- tidb-dashboard.yaml       #   TidbDashboard CR
+-- 3-explore-tidb-basic-usage/
|   +-- README.md                     # Step 3 guide
|   +-- set-up-port-forward.sh        # Port-forwards TiDB (4000), Dashboard (12333), Grafana (3000)
+-- 4-scale-up-tidb-cluster-with-tidb-operator/
|   +-- README.md                     # Step 4 guide: edit Step 2 manifest, re-run Pulumi
+-- [bonus]config-slow-log-threshold/
|   +-- README.md                     # Bonus guide: slow-log threshold
|   +-- cheat_scripts.sh              # Creates table1/table2, inserts 5,000 rows each
+-- .imgs/                            # Images referenced by the READMEs
+-- Makefile                          # `make install` (npm install), `make lint` (eslint --fix, the default target)
+-- package.json                      # Root npm dependencies (Pulumi AWS/EKS/Kubernetes SDKs)
+-- VLDB ss 2023 lab 1.pdf / .pptx    # Lecture slides for this lab
```

## Core Modules

### `1-create-an-eks-cluster` (Pulumi program, run from its directory)

- Creates the EKS cluster `my-eks` with two `t2.medium` worker nodes.
- Installs the **EBS CSI driver** add-on: it creates the cluster OIDC provider,
  an IAM role bound to the `ebs-csi-controller-sa` service account, and the
  `aws-ebs-csi-driver` add-on -- required so TiDB pods can mount EBS volumes.
- Exports `kubeconfig` as a stack output (`pulumi stack output kubeconfig`).

### `2-deploy-tidb-with-tidb-operator` (Pulumi program, run from its directory)

- Reads the Step 1 stack output via `StackReference`
  (`organization/1-create-an-eks-cluster/default`) and builds a Kubernetes
  provider from its `kubeconfig`.
- Applies the CRDs from `crds/*.yaml` via a `ConfigGroup`.
- Installs **TiDB Operator v1.4.4** from the `https://charts.pingcap.org/` Helm
  repository.
- Applies `tidb-cluster-manifests/*.yaml`, creating one `TidbCluster` (`basic`),
  one `TidbMonitor`, and one `TidbDashboard` custom resource.

### Steps 0, 3, 4, and Bonus (documentation / scripts only)

- `0-install-dependencies`: tool installation; also covers `make install`.
- `3-explore-tidb-basic-usage`: services, port-forwarding, and first SQL
  statements; uses `set-up-port-forward.sh`.
- `4-scale-up-tidb-cluster-with-tidb-operator`: edits
  `2-deploy-tidb-with-tidb-operator/tidb-cluster-manifests/tidb-cluster.yaml`
  (TiKV `replicas: 1 -> 2`), then re-runs `pulumi up` from the Step 2 directory.
- `[bonus]config-slow-log-threshold`: adjusts `tidb_slow_log_threshold` in the
  `TidbCluster` manifest, designs and runs slow SQL, and verifies it in TiDB
  Dashboard; `cheat_scripts.sh` prepares the test tables.

## Database Components

Deployed by the Step 2 Pulumi program (versions are pinned in the manifests as
of this lab; check them when re-running with newer tooling):

- **TiDB v7.1.0** (`TidbCluster` CR `basic`): MySQL-compatible SQL layer,
  1 replica, ClusterIP service `basic-tidb` on port **4000**.
- **TiKV v7.1.0** / **PD v7.1.0**: row storage engine and placement driver
  (cluster-wide version from `spec.version`), each starts with **1 replica**
  (TiKV is scaled to 2 in Step 4), ports 20160 (TiKV) and 2379 (PD).
- **TiDB Dashboard** (`TidbDashboard` CR): web UI, port-forwarded on
  `localhost:12333`.
- **TiDB Monitor** (`TidbMonitor` CR): Prometheus v2.27.1 + Grafana 7.5.11,
  port-forwarded on `localhost:3000` (Grafana `admin`/`admin`).
- The `TidbCluster` manifest requests a **1 GiB** EBS volume for each of
  PD / TiKV / TiDB (3 volumes; 4 after Step 4 scales TiKV to 2 replicas).

The TiDB Operator CRDs (`crds/tidb-operator-v1.4.4.yaml`) define the full
operator family (`TidbCluster`, `TidbMonitor`, `TidbDashboard`, `Backup`,
`BackupSchedule`, `DMCluster`, ...). This lab only uses `TidbCluster`,
`TidbMonitor`, and `TidbDashboard`.

### Database tables used in the lab

| Table | Created in | Purpose |
| --- | --- | --- |
| `test.hello_world (id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY, v VARCHAR(32))` | Step 3, manually via `mysql` client | Inspect `information_schema.tikv_region_status`, `tidb_version()`, `information_schema.cluster_info` |
| `test.table1 (id INT AUTO_INCREMENT PRIMARY KEY, random_num FLOAT)` | Bonus, by `cheat_scripts.sh` | Load data for designing slow queries (5,000 random rows) |
| `test.table2 (id INT AUTO_INCREMENT PRIMARY KEY, random_num FLOAT)` | Bonus, by `cheat_scripts.sh` | Load data for designing slow queries (5,000 random rows) |

## How to Run

Run the steps in order, ideally inside **one shell session**:

1. `make install` at the repository root, plus the manual tools in
   [Step 0](./0-install-dependencies/README.md) (`kubectl`, AWS CLI, Pulumi,
   Helm, Node.js 16).
2. [Step 1](./1-create-an-eks-cluster/README.md): from
   `1-create-an-eks-cluster/`, run `pulumi login --local`,
   `export PULUMI_CONFIG_PASSPHRASE=""`, `pulumi stack select default -c`, then
   `pulumi up`. Afterwards: `pulumi stack output kubeconfig > kubeconfig.yaml`
   and `export KUBECONFIG=$PWD/kubeconfig.yaml`.
3. [Step 2](./2-deploy-tidb-with-tidb-operator/README.md): from
   `2-deploy-tidb-with-tidb-operator/`, select the stack and `pulumi up` again.
   If you opened a new shell, first run
   `export KUBECONFIG=$PWD/../1-create-an-eks-cluster/kubeconfig.yaml`.
4. [Step 3](./3-explore-tidb-basic-usage/README.md): `./set-up-port-forward.sh`,
   then connect with `mysql --comments -h 127.0.0.1 -P 4000 -u root`.
5. [Step 4](./4-scale-up-tidb-cluster-with-tidb-operator/README.md): edit
   `tidb-cluster.yaml` in Step 2, re-run `pulumi up` from the Step 2 directory.
6. [Bonus](./%5Bbonus%5Dconfig-slow-log-threshold/README.md): adjust the
   slow-log threshold in the `TidbCluster` manifest, run the slow SQL, verify in
   TiDB Dashboard.
7. [Step 5 cleanup](./1-create-an-eks-cluster/README.md#do-not-execute-this-step-until-lab-1-finished-destroy-the-eks-cluster-via-pulumi):
   `pulumi destroy -y -s default` in the Step 1 directory **only after** the lab
   (including the Bonus) is finished.

## AWS billing price

This lab will incur charges under the AWS account, described in detail at:

- New EKS cluster control plane, **_1_** cluster x **_0.10_** USD per hour
- Two EKS worker EC2 `t2.medium` instances, **_2_** instances * **_0.0464_** USD per hour
- 4 EBS of size 1 GiB, with negligible cost
  (3 before Step 4 scales TiKV to 2 replicas, 5 after that step)

Total **_0.1928_** USD per hour.
