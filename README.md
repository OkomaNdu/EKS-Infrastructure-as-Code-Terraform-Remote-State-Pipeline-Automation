<div align="center">

# AWS EKS Infrastructure as Code

### Production-Ready Kubernetes on AWS — Fully Automated with Terraform

[![Terraform](https://img.shields.io/badge/Terraform-v1.0+-7B42BC?style=for-the-badge&logo=terraform&logoColor=white)](https://www.terraform.io/)
[![AWS](https://img.shields.io/badge/AWS-EKS-FF9900?style=for-the-badge&logo=amazonwebservices&logoColor=white)](https://aws.amazon.com/eks/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.35-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Helm](https://img.shields.io/badge/Helm-Charts-0F1689?style=for-the-badge&logo=helm&logoColor=white)](https://helm.sh/)
[![License](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)](LICENSE)

---

*Deploys a fully managed EKS cluster with VPC networking, managed node groups, serverless Fargate compute, and a MySQL database — all defined as code, with remote state management on S3.*

[Getting Started](#-getting-started) · [Architecture](#-architecture) · [Configuration](#-configuration) · [Outputs](#-outputs) · [Contributing](#-contributing)

</div>

---

## Highlights

- **Networking** — Multi-AZ VPC with public/private subnets, NAT gateway, and automatic EKS subnet tagging
- **Compute** — EKS managed node groups (EC2) + Fargate profiles for serverless workloads
- **Storage** — EBS CSI driver addon for dynamic persistent volume provisioning
- **Database** — MySQL deployed on-cluster via the Bitnami Helm chart
- **State** — Remote Terraform state stored securely in S3
- **Modular** — Built on battle-tested community modules (`terraform-aws-modules`)

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              AWS Cloud (ca-central-1)                               │
│                                                                                     │
│  ┌─ S3 ──────────────────┐                                                         │
│  │ esk-project-bucket    │     Terraform Remote State                               │
│  │ myapp/state.tfstate   │                                                          │
│  └───────────────────────┘                                                          │
│                                                                                     │
│  ┌─ VPC (10.0.0.0/16) ────────────────────────────────────────────────────────────┐ │
│  │                                                                                │ │
│  │   ┌─ Public Subnets ──────────────────────────────────────────────────────┐    │ │
│  │   │                                                                       │    │ │
│  │   │   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐           │    │ │
│  │   │   │ 10.0.4.0/24  │    │ 10.0.5.0/24  │    │ 10.0.6.0/24  │           │    │ │
│  │   │   │    AZ-a       │    │    AZ-b       │    │    AZ-c       │           │    │ │
│  │   │   └──────────────┘    └──────────────┘    └──────────────┘           │    │ │
│  │   │                                                                       │    │ │
│  │   │   Role: kubernetes.io/role/elb = 1  (External Load Balancers)        │    │ │
│  │   └───────────────────────────────────────────────────────────────────────┘    │ │
│  │                              │                                                 │ │
│  │                      ┌──────▼──────┐                                           │ │
│  │                      │ NAT Gateway │                                           │ │
│  │                      └──────┬──────┘                                           │ │
│  │                              │                                                 │ │
│  │   ┌─ Private Subnets ───────▼─────────────────────────────────────────────┐    │ │
│  │   │                                                                       │    │ │
│  │   │   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐           │    │ │
│  │   │   │ 10.0.1.0/24  │    │ 10.0.2.0/24  │    │ 10.0.3.0/24  │           │    │ │
│  │   │   │    AZ-a       │    │    AZ-b       │    │    AZ-c       │           │    │ │
│  │   │   └──────────────┘    └──────────────┘    └──────────────┘           │    │ │
│  │   │                                                                       │    │ │
│  │   │   Role: kubernetes.io/role/internal-elb = 1  (Internal Load Balancers)│    │ │
│  │   └───────────────────────────────────────────────────────────────────────┘    │ │
│  │                              │                                                 │ │
│  └──────────────────────────────┼─────────────────────────────────────────────────┘ │
│                                 │                                                   │
│  ┌─ EKS Cluster (my-cluster) ──▼──────────────────────────────────────────────────┐ │
│  │                                                                                │ │
│  │   ┌───────────────────────────────────────────┐                                │ │
│  │   │         Control Plane (AWS Managed)        │◄──── kubectl / HTTPS :443     │ │
│  │   └────────────────┬──────────────────────────┘                                │ │
│  │                    │                                                            │ │
│  │        ┌───────────┴───────────┐                                               │ │
│  │        ▼                       ▼                                               │ │
│  │   ┌─────────────────┐   ┌─────────────────┐    ┌──────────────────────────┐   │ │
│  │   │  Managed Nodes   │   │ Fargate Profile  │    │     Cluster Addons       │   │ │
│  │   │  t3.small × 3    │   │ ns: my-app       │    │     aws-ebs-csi-driver   │   │ │
│  │   │  (1 min, 3 max)  │   │                  │    │                          │   │ │
│  │   └────────┬─────────┘   └──────────────────┘    └──────────────────────────┘   │ │
│  │            │                                                                    │ │
│  │            ▼                                                                    │ │
│  │   ┌──────────────────────────────┐                                             │ │
│  │   │   MySQL (Bitnami Helm v9.14) │                                             │ │
│  │   │   Release: my-release        │                                             │ │
│  │   └──────────────────────────────┘                                             │ │
│  │                                                                                │ │
│  └────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Tech Stack

| Layer              | Technology                                                                 | Version  |
|--------------------|---------------------------------------------------------------------------|----------|
| **IaC**            | Terraform                                                                 | >= 1.0   |
| **Cloud Provider** | AWS (`hashicorp/aws`)                                                     | ~> 6.0   |
| **Networking**     | [terraform-aws-modules/vpc/aws](https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws) | 5.2.0    |
| **Orchestration**  | [terraform-aws-modules/eks/aws](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws) | 19.20.0  |
| **Database**       | Bitnami MySQL Helm Chart                                                  | 9.14.0   |
| **Kubernetes**     | Amazon EKS                                                                | 1.35     |

---

## Project Structure

```
EKS-Infrastructure-as-Code-Terraform/
│
├── providers.tf          # Required providers & version constraints
├── vpc.tf                # VPC, subnets, NAT gateway, S3 backend config
├── eks-cluster.tf        # EKS cluster, node groups, Fargate profiles
├── mysql.tf              # Kubernetes & Helm providers, MySQL deployment
├── variables.tf          # Input variable declarations
├── output.tf             # Cluster endpoint, kubeconfig, auth outputs
│
├── dev.tfvars            # Development environment overrides
├── values.yaml           # Helm values for MySQL chart customization
└── README.md
```

---

## Prerequisites

| Requirement | Purpose |
|-------------|---------|
| [Terraform](https://developer.hashicorp.com/terraform/install) >= 1.0 | Infrastructure provisioning |
| [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) | AWS authentication & cluster config |
| [kubectl](https://kubernetes.io/docs/tasks/tools/) | Kubernetes cluster interaction |
| [Helm](https://helm.sh/docs/intro/install/) | MySQL chart deployment |
| AWS account with IAM permissions | EKS, VPC, S3, IAM resources |
| S3 bucket: `esk-project-bucket` | Remote Terraform state backend |
| `values.yaml` | Custom MySQL Helm chart values |

---

## Getting Started

### 1. Clone the repository

```bash
git clone https://github.com/<your-username>/EKS-Infrastructure-as-Code-Terraform.git
cd EKS-Infrastructure-as-Code-Terraform
```

### 2. Configure AWS credentials

```bash
aws configure
# — or —
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export AWS_DEFAULT_REGION="ca-central-1"
```

### 3. Initialize Terraform

```bash
terraform init
```

> This downloads provider plugins and configures the S3 remote backend.

### 4. Review the execution plan

```bash
terraform plan -var-file="dev.tfvars"
```

### 5. Deploy the infrastructure

```bash
terraform apply -var-file="dev.tfvars" -auto-approve
```

> Provisioning typically takes **15–20 minutes** (EKS cluster creation is the longest step).

### 6. Connect to the cluster

```bash
aws eks update-kubeconfig \
  --name my-cluster \
  --region ca-central-1

# Verify connectivity
kubectl get nodes
kubectl get pods -A
```

---

## Configuration

### Input Variables

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `env_prefix` | `string` | `"test"` | Environment prefix used for resource naming and tagging |
| `k8s_version` | `string` | `"1.35"` | Kubernetes version for the EKS cluster |
| `cluster_name` | `string` | `"my-test-cluster"` | Name of the EKS cluster |
| `region` | `string` | `"ca-central-1"` | AWS region for all resources |

### Environment Overrides (`dev.tfvars`)

```hcl
env_prefix   = "dev"
k8s_version  = "1.35"
cluster_name = "my-cluster"
region       = "ca-central-1"
```

> Create additional `*.tfvars` files (e.g., `staging.tfvars`, `prod.tfvars`) to manage multiple environments.

---

## Outputs

| Output | Description |
|--------|-------------|
| `cluster_id` | Unique identifier of the EKS cluster |
| `cluster_endpoint` | API server endpoint URL for the EKS control plane |
| `cluster_security_group_id` | Security group IDs attached to the cluster control plane |
| `kubectl_config` | `kubectl` config YAML generated by the EKS module |
| `config_map_aws_auth` | Kubernetes `aws-auth` ConfigMap for IAM ↔ RBAC mapping |
| `cluster_name` | Name of the provisioned Kubernetes cluster |

```bash
# Retrieve a specific output after apply
terraform output cluster_endpoint
```

---

## Infrastructure Details

<details>
<summary><strong>VPC & Networking</strong></summary>

| Property | Value |
|----------|-------|
| CIDR Block | `10.0.0.0/16` |
| Public Subnets | `10.0.4.0/24`, `10.0.5.0/24`, `10.0.6.0/24` |
| Private Subnets | `10.0.1.0/24`, `10.0.2.0/24`, `10.0.3.0/24` |
| NAT Gateway | Single (cost-optimized) |
| DNS Hostnames | Enabled |

Subnets are automatically tagged for EKS load balancer discovery:
- Public subnets → `kubernetes.io/role/elb = 1`
- Private subnets → `kubernetes.io/role/internal-elb = 1`

</details>

<details>
<summary><strong>EKS Cluster</strong></summary>

| Property | Value |
|----------|-------|
| Module | `terraform-aws-modules/eks/aws` v19.20.0 |
| API Endpoint | Public access enabled |
| Addons | `aws-ebs-csi-driver` (persistent volume support) |

**Managed Node Group**
| Property | Value |
|----------|-------|
| Instance Type | `t3.small` |
| Min / Max / Desired | 1 / 3 / 3 |
| IAM Policies | `AmazonEBSCSIDriverPolicy` |

**Fargate Profile**
| Property | Value |
|----------|-------|
| Profile Name | `my-fargate-profile` |
| Namespace Selector | `my-app` |

</details>

<details>
<summary><strong>MySQL Database</strong></summary>

| Property | Value |
|----------|-------|
| Deployment Method | Helm (Bitnami chart) |
| Chart Version | `9.14.0` |
| Release Name | `my-release` |
| Timeout | 1000 seconds |
| Volume Permissions | Enabled |

Configuration is customizable via the `values.yaml` file.

</details>

<details>
<summary><strong>State Management</strong></summary>

| Property | Value |
|----------|-------|
| Backend | S3 |
| Bucket | `esk-project-bucket` |
| State Key | `myapp/state.tfstate` |
| Region | `ca-central-1` |

> For team collaboration, consider enabling [DynamoDB state locking](https://developer.hashicorp.com/terraform/language/settings/backends/s3#dynamodb-table).

</details>

---

## Cleanup

To destroy all provisioned resources:

```bash
terraform destroy -var-file="dev.tfvars"
```

> **Warning:** This will permanently delete the EKS cluster, all running workloads, the VPC, and associated resources. Ensure all data is backed up before proceeding.

---

## Contributing

Contributions are welcome! Please follow these steps:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/my-feature`)
3. Commit your changes (`git commit -m 'Add my feature'`)
4. Push to the branch (`git push origin feature/my-feature`)
5. Open a Pull Request

---

## License

This project is licensed under the [MIT License](LICENSE).

---

<div align="center">

**Built with Terraform on AWS**

[![Terraform](https://img.shields.io/badge/Powered_by-Terraform-7B42BC?logo=terraform&logoColor=white)](https://www.terraform.io/)
[![AWS](https://img.shields.io/badge/Deployed_on-AWS-FF9900?logo=amazonwebservices&logoColor=white)](https://aws.amazon.com/)

</div>
