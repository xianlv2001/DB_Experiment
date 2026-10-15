# Lab 1: Deploy TiDB Cluster on AWS EKS

This lab deploys a **TiDB cluster** on **AWS EKS**: the cluster is managed by the **TiDB Operator**, and the deployment is automated with **Pulumi** (infrastructure as code). In short: one Pulumi program creates a Kubernetes cluster, a second one installs TiDB on it, and the remaining steps teach you how to use, scale, and tune the database.

<!-- TOC -->
* [Lab 1: Deploy TiDB Cluster on AWS EKS](#lab-1-deploy-tidb-cluster-on-aws-eks)
  * [What You Will Learn](#what-you-will-learn)
  * [Before You Start](#before-you-start)
  * [Syllabus](#syllabus)
  * [Versions Used](#versions-used)
  * [Repository Layout](#repository-layout)
  * [What Gets Deployed](#what-gets-deployed)
  * [Services and Ports](#services-and-ports)
  * [How to Run](#how-to-run)
  * [Troubleshooting](#troubleshooting)
  * [Cost](#cost)
<!-- TOC -->

## What You Will Learn

- The basics of Kubernetes and its managed AWS service, EKS
- The operator pattern: how the TiDB Operator manages a database on Kubernetes
- How to deploy, use, and scale a TiDB cluster
- How to automate all of it with Pulumi

## Before You Start

- An AWS account (this lab bills ~**0.19 USD/hour**, see [Cost](#cost))
- VPN access to the AWS API and GitHub
- Linux, macOS, or WSL2
- Tools from [Step 0](./0-install-dependencies/README.md): Node.js 16, AWS CLI, Pulumi, `kubectl`, Helm
- A `mysql` command-line client (Step 3 and Bonus)
- Time: roughly 10-20 minutes per `pulumi up` (Steps 1, 2 and 4), a few minutes for the rest

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

> **Order matters:** Steps 0 -> 1 -> 2 -> 3 -> 4 -> Bonus, in order, in the same
> shell session. Finish the Bonus **before** Step 5 destroys the cluster.

## Versions Used

| Component | Version |
| --- | --- |
| TiDB / TiKV / PD | v7.1.0 |
| TiDB Operator | v1.4.4 |
| Kubernetes (EKS default) | 1.27 |
| Node.js | 16 |
| Prometheus / Grafana | v2.27.1 / 7.5.11 |

All versions are pinned in the manifests and Pulumi programs, so re-running the
lab reproduces the same stack. Change them only if you are prepared to re-verify
the affected steps.

## Repository Layout

Every numbered directory (and the bonus directory) is **self-contained**, with
its own step-by-step `README.md`. Steps 1 and 2 also contain a standalone
**Pulumi program** (TypeScript): `Pulumi.yaml` (project), `Pulumi.default.yaml`
(state encryption salt for the `default` stack), and `index.ts` (entrypoint).

```
DB_Experiment/
├── 0-install-dependencies/   # Tool installation guide
├── 1-create-an-eks-cluster/  # Step 1: Pulumi program (EKS cluster + EBS CSI driver); also holds Step 5 destroy instructions
├── 2-deploy-tidb-with-tidb-operator/
│   ├── index.ts              # Step 2: Pulumi program (TiDB Operator via Helm + CRDs + manifests)
│   ├── crds/                 #   TiDB Operator CRDs (v1.4.4)
│   └── tidb-cluster-manifests/
│       ├── tidb-cluster.yaml #   TidbCluster CR (PD / TiKV / TiDB)
│       ├── tidb-monitor.yaml #   TidbMonitor CR (Prometheus + Grafana)
│       └── tidb-dashboard.yaml # TidbDashboard CR
├── 3-explore-tidb-basic-usage/     # Step 3 guide + set-up-port-forward.sh
├── 4-scale-up-tidb-cluster-with-tidb-operator/  # Step 4 guide (edit Step 2 manifest, re-run Pulumi)
├── [bonus]config-slow-log-threshold/  # Bonus guide + cheat_scripts.sh (test data)
├── .imgs/                    # Images referenced by the READMEs
├── Makefile                  # `make install` (npm install), `make lint` (default target)
└── package.json              # Root npm dependencies (Pulumi AWS/EKS/Kubernetes SDKs)
```

(`VLDB ss 2023 lab 1.pdf` / `.pptx` at the root are the lecture slides.)

## What Gets Deployed

Step 1 creates the infrastructure; Step 2 fills it with the database:

- **Step 1** creates an EKS cluster (`my-eks`, two `t2.medium` nodes) plus the
  **EBS CSI driver** add-on, which lets pods mount EBS volumes. It exports a
  `kubeconfig` that every later step uses.
- **Step 2** installs the TiDB Operator v1.4.4 (via Helm) and applies the CRDs,
  then creates three custom resources: one `TidbCluster` (`basic`), one
  `TidbMonitor`, and one `TidbDashboard`.
- The `TidbCluster` runs **TiDB** (MySQL-compatible SQL layer), **TiKV** (storage)
  and **PD** (placement driver), each starting with 1 replica (TiKV grows to 2 in
  Step 4). Each component gets a **1 GiB** EBS volume -- 4 volumes total, 5 after Step 4.

For reference, the lab creates these tables along the way:

| Table | Created in | Purpose |
| --- | --- | --- |
| `test.hello_world` | Step 3, via `mysql` client | Inspect region status, `tidb_version()`, cluster info |
| `test.table1` / `test.table2` | Bonus, via `cheat_scripts.sh` | 5,000 random rows each, for designing slow queries |

## Services and Ports

Everything Step 2 deploys is `ClusterIP` inside the cluster; on your laptop you
reach it through the port-forwards started by `set-up-port-forward.sh` in Step 3.

| Component | Service | Port | Access after port-forward |
| --- | --- | --- | --- |
| TiDB (SQL) | `basic-tidb` | 4000 | `mysql --comments -h 127.0.0.1 -P 4000 -u root` |
| TiDB Dashboard | `basic-tidb-dashboard-exposed` | 12333 | http://localhost:12333, empty password |
| Grafana | `basic-grafana` | 3000 | http://localhost:3000, `admin`/`admin` |

PD (2379), TiKV (20160), Prometheus (9090) and the TiDB status API (10080) stay
internal -- the lab does not port-forward them.

## How to Run

Run the steps in order, ideally in **one shell session**. Three things live in
that session and must be re-exported if you open a new shell:

| Variable | Where it comes from |
| --- | --- |
| `KUBECONFIG` | Step 1: `export KUBECONFIG=$PWD/kubeconfig.yaml` after `pulumi stack output kubeconfig > kubeconfig.yaml` |
| `PULUMI_CONFIG_PASSPHRASE=""` | export before every `pulumi up` / `pulumi destroy` |
| AWS credentials | `aws configure` in Step 0 |

1. `make install` at the repository root, plus the manual tools in
   [Step 0](./0-install-dependencies/README.md).
2. [Step 1](./1-create-an-eks-cluster/README.md): from `1-create-an-eks-cluster/`,
   run `pulumi login --local`, `export PULUMI_CONFIG_PASSPHRASE=""`,
   `pulumi stack select default -c`, then `pulumi up`. Afterwards:
   `pulumi stack output kubeconfig > kubeconfig.yaml` and
   `export KUBECONFIG=$PWD/kubeconfig.yaml`.
   **Verify:** `kubectl get nodes` shows 2 `Ready` nodes.
3. [Step 2](./2-deploy-tidb-with-tidb-operator/README.md): from
   `2-deploy-tidb-with-tidb-operator/`, select the stack and `pulumi up` again.
   If you opened a new shell, first run
   `export KUBECONFIG=$PWD/../1-create-an-eks-cluster/kubeconfig.yaml`.
   **Verify:** `kubectl get po` eventually shows `basic-tidb-0`, `basic-tikv-0`,
   `basic-pd-0` all `Running` (several minutes).
4. [Step 3](./3-explore-tidb-basic-usage/README.md): `./set-up-port-forward.sh`,
   then `mysql --comments -h 127.0.0.1 -P 4000 -u root`.
   **Verify:** `select tidb_version();` returns v7.1.0.
5. [Step 4](./4-scale-up-tidb-cluster-with-tidb-operator/README.md): edit
   `tidb-cluster.yaml` in Step 2 (`tikv.replicas: 1 -> 2`), re-run `pulumi up`
   from the Step 2 directory.
   **Verify:** `kubectl get po` shows a second TiKV pod, `basic-tikv-1`.
6. [Bonus](./%5Bbonus%5Dconfig-slow-log-threshold/README.md): raise
   `tidb_slow_log_threshold` in the `TidbCluster` manifest (via `pulumi up`),
   run the slow SQL, check TiDB Dashboard.
7. [Step 5 cleanup](./1-create-an-eks-cluster/README.md#do-not-execute-this-step-until-lab-1-finished-destroy-the-eks-cluster-via-pulumi):
   `pulumi destroy -y -s default` in the Step 1 directory, **only after** the
   whole lab (including the Bonus) is finished.

## Troubleshooting

- **`pulumi up` fails to decrypt configuration**: run from the step directory
  with `export PULUMI_CONFIG_PASSPHRASE=""` exactly as documented -- the stack
  state is tied to that passphrase and the salt in `Pulumi.default.yaml`.
- **`no stack named 'default' found` / `no stack selected`**: run
  `pulumi stack select default -c` (first time) or `pulumi stack select default`
  (afterwards) in the step directory that owns the program.
- **`kubectl` cannot connect**: `KUBECONFIG` is not set. Re-export it:
  `export KUBECONFIG=$PWD/../1-create-an-eks-cluster/kubeconfig.yaml`.
- **`Permission denied` on `./set-up-port-forward.sh`**: run
  `chmod +x set-up-port-forward.sh` or `bash set-up-port-forward.sh`. Keep the
  script running in its own terminal; `Ctrl+C` stops all port-forwards.
- **TiDB pods stuck in `Pending`**: usually the EBS CSI driver is missing (run
  `pulumi up` in Step 1's directory, not Step 2's) or there is no default
  storage class. Check with `kubectl describe po <pod>` and `kubectl get sc`.
- **Bonus: slow queries not in TiDB Dashboard**: the new
  `instance.tidb_slow_log_threshold` only takes effect after `pulumi up` and a
  TiDB pod restart -- then re-run the slow SQL from a fresh `mysql` session.

## Cost

| Item | Rate |
| --- | --- |
| EKS control plane | 1 x 0.10 USD/hour |
| EC2 `t2.medium` workers | 2 x 0.0464 USD/hour |
| EBS volumes (1 GiB each) | negligible |
| **Total** | **~0.1928 USD/hour** (≈ 1.54 USD for an 8-hour session) |

> **Do not forget Step 5**: until you `pulumi destroy` the cluster, it keeps
> billing ~0.19 USD/hour.
