# IaC -- AWS Infrastructure Automation (Terraform)

Infrastructure as Code (IaC) project using **Terraform** to provision AWS networking, IAM roles, and EC2 instances for an application server and a self-hosted GitLab Runner, automated via **GitLab CI/CD**.

## What Gets Provisioned

- **VPC & Networking**: Custom VPC (`10.0.0.0/16`) with 3 public and 3 private subnets across availability zones, NAT Gateway, and VPN Gateway.
- **Security Groups**:
  - `main`: Allows internal VPC traffic (`10.0.0.0/16`) and all outbound traffic.
  - `app-server`: Allows internal VPC traffic, inbound TCP on port `3000`, and all outbound traffic.
- **IAM Roles**:
  - `app-server-role`: Grants EC2 access to AWS Systems Manager (SSM) and Amazon ECR.
  - `gitlab-runner-role`: Grants EC2 full access to SSM and Amazon ECR for deployments.
- **EC2 Instances** (Ubuntu 22.04 LTS, `t3.small`):
  - `app-server`: Application host instance.
  - `gitlab-runner`: Self-hosted GitLab CI/CD runner instance.
- **Remote State**: Stored in an AWS S3 bucket (`infra-bucket-03` in `eu-west-2`).

## EC2 Bootstrap Scripts

Both EC2 instances are configured on launch using `user_data` template scripts in the `scripts/` directory:

- **`scripts/script.tpl`** (`app-server`): Updates apt repositories, installs Docker and AWS CLI, and configures Docker group permissions.
- **`scripts/script-gitlab.tpl`** (`gitlab-runner`): Installs Docker, AWS CLI, and `gitlab-runner`, configures permissions, and automatically registers the runner with GitLab (`shell` executor) using `runner_registration_token`.

## Project Structure

```text
.
├── .gitlab-ci.yml            # GitLab CI/CD pipeline with Trivy IaC scanning
├── providers.tf              # AWS provider & S3 backend configuration
├── main.tf                   # IAM, VPC, Security Groups, and EC2 modules
├── data.tf                   # AWS data sources (AMI, VPC, IAM policies, AZs)
├── variables.tf              # Input variables
├── locals.tf                 # Template rendering for EC2 user_data scripts
├── outputs.tf                # Rendered bootstrap script outputs
└── scripts/
    ├── script.tpl            # Bootstrap script for app-server
    └── script-gitlab.tpl     # Bootstrap script for gitlab-runner
```

## Variables

| Name | Description | Default | Sensitive |
| :--- | :--- | :--- | :---: |
| `aws_region` | AWS region to deploy resources into | `eu-west-2` | No |
| `env_prefix` | Environment tag prefix | `dev` | No |
| `aws_access_key_id` | AWS Access Key ID | `""` | No |
| `aws_secret_access_key` | AWS Secret Access Key | `""` | No |
| `runner_registration_token` | Token used to register the GitLab Runner | `""` | Yes |

### GitLab CI/CD Pipeline

The `.gitlab-ci.yml` pipeline automates validation, security scanning, deployment, and teardown:

1. **`init`**: Initializes Terraform and providers.
2. **`test`**: Runs `terraform validate` and scans Terraform files for misconfigurations with **Trivy**.
3. **`build`**: Generates a `terraform plan` artifact.
4. **`deploy`** *(manual)*: Applies the planned changes to AWS.
5. **`plan-destroy` & `destroy`** *(manual)*: Plans and tears down the AWS infrastructure.

Make sure `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and `RUNNER_TOKEN` are configured in your GitLab CI/CD variables.
