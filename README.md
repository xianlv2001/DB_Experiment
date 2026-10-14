# Lab 1: Deploy TiDB Cluster on AWS EKS

This lab deploys a **TiDB cluster** on **AWS EKS**. The cluster is managed by the
**TiDB Operator**, and the whole deployment process is automated with **Pulumi**
(infrastructure as code).

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
  * [Troubleshooting](#troubleshooting)
  * [AWS Billing Price](#aws-billing-price)
<!-- TOC -->

## Introduction

**Cloud computing** is the delivery of computing **services** -- including
servers, storage, databases, networking, software, analytics, and intelligence
-- over the internet ("the cloud") to offer faster innovation, flexible
resources, and economies of scale, as your business needs change.

**Amazon EKS** is a managed Kubernetes service that makes it easy to run
**Kubernetes** on AWS. Kubernetes automates the deployment, scaling, and
management of containerized applications.

**TiDB** is an open-source distributed SQL database that supports HTAP
workloads. It provides users with a one-stop database solution, and helps
improve the scalability, availability and reliability of users' data storage
systems.

Deploying TiDB clusters on AWS EKS gives you all the features provided by TiDB
while leveraging the benefits of operating TiDB clusters on managed Kubernetes
services.

## Learning Objectives

- Understand basic usage of Kubernetes
- Understand basic usage of the AWS Kubernetes service (EKS)
- Understand the fundamentals of the operator pattern
- Learn to deploy TiDB clusters on Kubernetes with the TiDB Operator
- Understand basic usage of TiDB clusters
- Automate the deployment process with the Pulumi IaC framework

## Pre-Requisites

- An AWS account
- VPN for connecting to the AWS API and GitHub
- Linux, macOS, or WSL2 environments
- Tools installed in [Step 0](./0-install-dependencies/README.md): Node.js 16,
  AWS CLI, Pulumi, `kubectl`, Helm
- A `mysql` command-line client (used in Step 3 and the Bonus step)

## Syllabus

> 100 basic points + 20 bonus points = 120 total points.

| Step | Content | Points |
| --- | --- | --- |
| 0 | [Install Dependencies](./0-install-dependencies/README.md) | -- |
| 1 | [Create an EKS cluster](./1-create-an-eks-cluster/README.md) | 25 |
| 2 | [Deploy TiDB with the TiDB Operator](./2-deploy-tidb-with-tidb-operator/README.md) | 25 |
| 3 | [Explore TiDB basic usage](./3-explore-tidb-basic-usage/README.md) | 20 |
| 4 | [Scale up the TiDB cluster with the TiDB Operator](./4-scale-up-tidb-cluster-with-tidb-operator/README.md) | 20 |
| 5 | [Cleanup: destroy the EKS cluster via Pulumi](./1-create-an-eks-cluster/README.md#do-not-execute-this-step-until-lab-1-finished-destroy-the-eks-cluster-via-pulumi) | 10 |
| Bonus | [Configure the TiDB slow-log threshold and update the cluster with Pulumi](./%5Bbonus%5Dconfig-slow-log-threshold/README.md) | 20 |

> **Order matters:** Steps 0 -> 1 -> 2 -> 3 -> 4 -> Bonus must be done in order,
> each reusing the same shell session (or re-exporting `KUBECONFIG`, see
> [How to Run](#how-to-run)). The Bonus step must be finished **before** Step 5
> destroys the cluster.

## Repository Layout

Every numbered directory (and the bonus directory) is **self-contained**, with
its own step-by-step `README.md`. Steps 1 and 2 additionally contain a
standalone **Pulumi program** (TypeScript): each has a `Pulumi.yaml` (project +
local `file://~/.pulumi` backend), a `Pulumi.default.yaml` (state encryption
salt for the `default` stack), and an `index.ts` entrypoint.

```
DB_Experiment/
├── 0-install-dependencies/
│   └── README.md                     # Tool installation (Node.js 16, AWS CLI, Pulumi, kubectl, Helm)
├── 1-create-an-eks-cluster/
│   ├── README.md                     # Step 1 guide + EKS destroy instructions (Step 5 cleanup)
│   ├── index.ts                      # Pulumi program: EKS cluster + EBS CSI driver addon
│   ├── Pulumi.yaml                   # Pulumi project definition
│   └── Pulumi.default.yaml           # Encryption salt for the `default` stack
├── 2-deploy-tidb-with-tidb-operator/
│   ├── README.md                     # Step 2 guide: operator pattern + deployment
│   ├── index.ts                      # Pulumi program: TiDB Operator (Helm) + CRDs + TiDB cluster manifests
│   ├── Pulumi.yaml                   # Pulumi project definition
│   ├── Pulumi.default.yaml           # Encryption salt for the `default` stack
│   ├── crds/
│   │   └── tidb-operator-v1.4.4.yaml # All TiDB Operator CRDs (v1.4.4), applied via ConfigGroup
│   └── tidb-cluster-manifests/       # Applied by the same Pulumi program:
│       ├── tidb-cluster.yaml         #   TidbCluster CR (PD / TiKV / TiDB)
│       ├── tidb-monitor.yaml         #   TidbMonitor CR (Prometheus + Grafana)
│       └── tidb-dashboard.yaml       #   TidbDashboard CR
├── 3-explore-tidb-basic-usage/
│   ├── README.md                     # Step 3 guide
│   └── set-up-port-forward.sh        # Port-forwards TiDB (4000), Dashboard (12333), Grafana (3000)
├── 4-scale-up-tidb-cluster-with-tidb-operator/
│   └── README.md                     # Step 4 guide: edit Step 2 manifest, re-run Pulumi
├── [bonus]config-slow-log-threshold/
│   ├── README.md                     # Bonus guide: slow-log threshold
│   └── cheat_scripts.sh              # Creates table1/table2, inserts 5,000 rows each
├── .imgs/                            # Images referenced by the READMEs
├── Makefile                          # `make install` (npm install), `make lint` (eslint --fix, the default target)
├── package.json                      # Root npm dependencies (Pulumi AWS/EKS/Kubernetes SDKs)
└── VLDB ss 2023 lab 1.pdf / .pptx    # Lecture slides for this lab
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
- Applies `tidb-cluster-manifests/*.yaml`, creating one `TidbCluster`
  (`basic`), one `TidbMonitor`, and one `TidbDashboard` custom resource.

### Steps 0, 3, 4, and Bonus (documentation / scripts only)

- `0-install-dependencies`: tool installation; also covers `make install`.
- `3-explore-tidb-basic-usage`: services, port-forwarding, and first SQL
  statements; uses `set-up-port-forward.sh`.
- `4-scale-up-tidb-cluster-with-tidb-operator`: edits
  `2-deploy-tidb-with-tidb-operator/tidb-cluster-manifests/tidb-cluster.yaml`
  (TiKV `replicas: 1 -> 2`), then re-runs `pulumi up` from the Step 2
  directory.
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
  PD / TiKV / TiDB, and the `TidbDashboard` manifest requests one more 1 GiB
  volume for its own PVC -- **4 volumes** in total (**5** after Step 4 scales
  TiKV to 2 replicas).

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
   slow-log threshold in the `TidbCluster` manifest, run the slow SQL, verify
   in TiDB Dashboard.
7. [Step 5 cleanup](./1-create-an-eks-cluster/README.md#do-not-execute-this-step-until-lab-1-finished-destroy-the-eks-cluster-via-pulumi):
   `pulumi destroy -y -s default` in the Step 1 directory **only after** the
   lab (including the Bonus) is finished.

## Troubleshooting

- **`pulumi up` fails with a decryption/salt error** (e.g. `failed to decrypt
  encrypted configuration value`): the stack state is bound to the salt in
  `Pulumi.default.yaml` and your `PULUMI_CONFIG_PASSPHRASE`. Always run from
  the step directory with the passphrase exported exactly as documented:
  `export PULUMI_CONFIG_PASSPHRASE=""`.
- **`error: no stack named 'default' found` / `no stack selected`**: run
  `pulumi stack select default -c` (first run) or `pulumi stack select default`
  (subsequent runs) from the step directory that owns the program.
- **`kubectl` cannot connect / `connection refused`**: `KUBECONFIG` is not
  pointing at the Step 1 output. Re-export it:
  `export KUBECONFIG=$PWD/../1-create-an-eks-cluster/kubeconfig.yaml` (or the
  absolute path from wherever you are).
- **`Permission denied` while running `./set-up-port-forward.sh`**: the script
  is not executable yet. Run `chmod +x set-up-port-forward.sh`, or use
  `bash set-up-port-forward.sh`. Keep the script running in its own terminal:
  `Ctrl+C` tears down all the port-forwards it started.
- **TiDB pods stuck in `Pending`**: usually the EBS CSI driver was not
  installed (Step 1 must run `pulumi up` before Step 2) or the cluster has no
  default storage class. Inspect with `kubectl describe po <pod>` and
  `kubectl get sc`.
- **Bonus: slow queries do not appear in TiDB Dashboard**:
  `instance.tidb_slow_log_threshold` only takes effect after `pulumi up`
  re-applies the `TidbCluster` manifest and the TiDB pods have restarted --
  then re-run the slow SQL from a fresh `mysql` session.

## AWS Billing Price

This lab will incur charges under the AWS account, described in detail at:

- New EKS cluster control plane: **1** cluster x **0.10** USD per hour
- Two EKS worker EC2 `t2.medium` instances: **2** instances x **0.0464** USD
  per hour
- EBS volumes of 1 GiB each, with negligible cost (**4** volumes before Step 4
  scales TiKV to 2 replicas, **5** after that step)

Total: **0.1928** USD per hour.

> **Cleanup reminder:** always finish with
> [Step 5](./1-create-an-eks-cluster/README.md#do-not-execute-this-step-until-lab-1-finished-destroy-the-eks-cluster-via-pulumi)
> (`pulumi destroy -y -s default`), otherwise the cluster keeps billing at
> ~0.19 USD/hour until it is destroyed.
