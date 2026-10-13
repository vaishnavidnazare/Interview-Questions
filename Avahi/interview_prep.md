## 1. AWS fundamentals

- **Region**: a geographic area (e.g. `us-east-1`, `ap-south-1` Mumbai). Resources are mostly regional. Data does not leave a region unless you move it.
- **Availability Zone (AZ)**: one or more isolated data centers inside a region. Spread workloads across at least 2 AZs for high availability.
- **Edge locations**: used by CloudFront (CDN) and Route 53.
- **Shared responsibility model**: AWS secures the cloud (hardware, facilities, hypervisor). You secure what you put in it (IAM, OS patching on EC2, data encryption, security groups, app code). For managed services like RDS, AWS also handles OS patching, but you still own access and data.
- **AWS Organizations**: manage many accounts under one org. Use separate accounts for prod, staging, dev, security and logging. **Service Control Policies (SCPs)** set the maximum permissions an account can ever have.
- **Control Tower / Landing Zone**: a pre-built multi-account setup with guardrails. Customers migrating to AWS usually want this first.
- **Ways to interact with AWS**: Console (web), CLI, SDKs, and IaC (CloudFormation, Terraform, CDK). In a real job, almost everything should be IaC.

---

## 2. IAM and access: users, roles, login, CLI

This is where interviewers often start. Know it cold.

### 2.1 The identities

| Identity | What it is | Use it for |
|---|---|---|
| **Root user** | The email you signed up with. Has unlimited power. | Almost nothing. Enable MFA, lock it away, no access keys. |
| **IAM user** | A long-lived identity with a password and/or access keys. | Legacy and small setups. Avoid for humans in a mature org. |
| **IAM group** | A collection of IAM users to attach policies to. | Grouping permissions (e.g. `developers`). Groups can't be assumed or nested. |
| **IAM role** | An identity with **no permanent credentials**. Someone or something *assumes* it and gets temporary credentials. | EC2, Lambda, EKS pods, cross-account access, CI/CD, federated humans. |
| **IAM Identity Center (SSO)** | Central login for humans across accounts, backed by its own directory or an IdP (Okta, Entra ID, Google). | The recommended way for people to access AWS today. |

**Golden rule**: humans use Identity Center (SSO), workloads use roles, and nobody uses long-lived access keys unless unavoidable.

### 2.2 Policies

A policy is a JSON document: **Effect** (Allow/Deny), **Action** (e.g. `s3:GetObject`), **Resource** (ARN), optional **Condition**.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": ["arn:aws:s3:::my-bucket", "arn:aws:s3:::my-bucket/*"]
  }]
}
```

Policy types to know:
- **Identity-based**: attached to a user, group or role.
- **Resource-based**: attached to the resource (S3 bucket policy, KMS key policy, SQS queue policy, role **trust policy**).
- **Permissions boundary**: caps the maximum permissions a user or role can have.
- **SCP**: caps permissions for an entire account or OU.
- **Session policy**: further restricts a role session.

**Evaluation logic (very common question)**: everything is denied by default. An explicit **Deny** anywhere always wins. Otherwise you need an explicit **Allow**, and that allow must not be blocked by an SCP or boundary.

**Role = trust policy + permissions policy.** The trust policy says *who can assume* the role. The permissions policy says *what the role can do*.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ec2.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
```

### 2.3 How to create a user (step by step)

**Console**: IAM, then Users, then Create user. Choose console access if needed, add the user to a group (don't attach policies directly), then enable MFA.

**CLI**:
```bash
aws iam create-user --user-name alice
aws iam create-group --group-name developers
aws iam add-user-to-group --user-name alice --group-name developers
aws iam attach-group-policy --group-name developers \
  --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess

# Console password (user must reset at first login)
aws iam create-login-profile --user-name alice \
  --password 'TempP@ssw0rd!' --password-reset-required

# Programmatic keys (avoid if you can)
aws iam create-access-key --user-name alice
```

### 2.4 How to log in and access AWS

1. **Console**: sign-in URL is `https://<account-id-or-alias>.signin.aws.amazon.com/console` for IAM users, or your Identity Center portal URL (`https://<name>.awsapps.com/start`) for SSO users.
2. **CLI with SSO (preferred)**:
   ```bash
   aws configure sso            # one-time setup, creates a named profile
   aws sso login --profile dev  # opens the browser to authenticate
   aws sts get-caller-identity --profile dev   # "who am I?" check
   ```
3. **CLI with access keys (legacy)**:
   ```bash
   aws configure   # prompts for access key, secret key, default region, output
   ```
   Credentials land in `~/.aws/credentials`, config in `~/.aws/config`.
4. **Assume a role** (cross-account or elevated access):
   ```bash
   aws sts assume-role --role-arn arn:aws:iam::123456789012:role/Admin \
     --role-session-name my-session
   ```
   Or in `~/.aws/config`:
   ```ini
   [profile prod-admin]
   role_arn = arn:aws:iam::123456789012:role/Admin
   source_profile = default
   ```
5. **Credential lookup order** for the CLI and SDKs: environment variables, then the profile files, then container credentials (ECS/EKS), then the EC2 instance profile. This is why code on EC2 needs no keys.

**Handy debug command**: `aws sts get-caller-identity`. It tells you exactly which identity AWS sees. Use it first whenever you get `AccessDenied`.

### 2.5 Roles for workloads

- **EC2**: attach an *instance profile* (wraps a role).
- **Lambda**: *execution role*.
- **ECS**: *task role* (what your app can do) vs *task execution role* (what ECS needs to pull images and write logs).
- **EKS pods**: **IRSA** (IAM Roles for Service Accounts, uses the cluster's OIDC provider) or the newer **EKS Pod Identity**. This gives each pod its own least-privilege role instead of sharing the node's role.
- **GitHub Actions**: **OIDC federation**, so there are no stored AWS keys in GitHub (see Section 7).

### 2.6 Security best practices (say these in the interview)

- MFA on root and all humans; no root access keys.
- Least privilege; start narrow and widen with IAM Access Analyzer's help.
- Use roles and temporary credentials, rotate or eliminate access keys.
- Use CloudTrail (audit log of every API call) in all regions, sent to a locked-down S3 bucket in a separate account.
- Use Secrets Manager or SSM Parameter Store for secrets, never put them in code or env files.
- Encrypt with KMS; use customer-managed keys where compliance needs it.

---

## 3. Networking (VPC)

Expect at least one networking design question.

- **VPC**: your private network in a region, with a CIDR range (e.g. `10.0.0.0/16`).
- **Subnets**: slices of the VPC, each in one AZ. **Public** subnet = route table has a route to an **Internet Gateway (IGW)**. **Private** subnet = no direct route to the internet.
- **NAT Gateway**: lets private subnets reach the internet outbound only. It lives in a public subnet. It is **expensive** (hourly plus per-GB), so mention cost awareness. One per AZ for HA, or one shared for dev to save money.
- **Route tables**: decide where traffic goes.
- **Security Groups (SG)**: stateful firewall at the instance/ENI level, allow rules only. **NACLs**: stateless firewall at the subnet level, allow and deny rules.
- **VPC endpoints**: private access to AWS services without the internet. *Gateway endpoints* (S3, DynamoDB, free) and *Interface endpoints* (PrivateLink, most other services, hourly cost). Cuts NAT cost and improves security.
- **Connecting networks**: VPC peering (simple, non-transitive), **Transit Gateway** (hub-and-spoke for many VPCs), Site-to-Site VPN, Direct Connect (dedicated line).
- **Load balancers**: **ALB** (layer 7, HTTP/HTTPS, path/host routing, best for web and containers), **NLB** (layer 4, TCP/UDP, very high performance, static IPs), **GWLB** (security appliances).
- **Route 53**: DNS, with routing policies (simple, weighted, latency, failover, geolocation) and health checks.
- **CloudFront**: CDN in front of S3 or ALB.

**Standard 3-tier layout**: public subnets (ALB, NAT) across 2-3 AZs, private app subnets (EC2/ECS/EKS nodes), isolated private DB subnets (RDS). Only the ALB is internet-facing.

---

## 4. Core compute, storage and database services

**Compute**
- **EC2**: virtual machines. Know instance families (general `t/m`, compute `c`, memory `r`, GPU `g/p`), AMIs, user data, **Auto Scaling Groups** with **Launch Templates**, pricing (On-Demand, Reserved/Savings Plans, Spot).
- **Lambda**: serverless functions, event driven, up to 15 minutes per invocation. Watch cold starts, concurrency limits, and VPC attachment considerations.
- **Elastic Beanstalk / App Runner**: easier PaaS-style options. Common targets when migrating off Heroku.

**Storage**
- **S3**: object storage. Know storage classes (Standard, Intelligent-Tiering, Standard-IA, Glacier tiers), versioning, lifecycle rules, encryption (SSE-S3, SSE-KMS), Block Public Access, bucket policies, replication, pre-signed URLs.
- **EBS**: block volumes for EC2 (gp3 is the usual default). **EFS**: shared NFS file system. **FSx**: managed Windows/Lustre/NetApp.

**Databases**
- **RDS / Aurora**: managed relational DB. **Multi-AZ** = high availability (synchronous standby, automatic failover). **Read replicas** = read scaling (asynchronous). They are not the same thing, and interviewers love this question. Aurora has a shared storage layer and faster failover.
- **DynamoDB**: serverless NoSQL, single-digit ms latency, on-demand or provisioned capacity.
- **ElastiCache**: Redis/Valkey/Memcached caching.
- **Redshift**: data warehouse. **Athena**: SQL directly on S3.

**Messaging**: SQS (queue), SNS (pub/sub), EventBridge (event bus), Kinesis (streaming), Step Functions (workflows).

---

## 5. Containers: Docker, ECS, EKS, and Kubernetes without EKS

### 5.1 Docker basics

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
EXPOSE 3000
USER node
CMD ["node", "server.js"]
```

Know: image vs container, layers and caching, multi-stage builds (smaller images), running as non-root, `.dockerignore`, image scanning (ECR scan or Trivy), tagging (don't rely on `latest` in prod, use commit SHA).

**Push to ECR (Elastic Container Registry)**:
```bash
aws ecr create-repository --repository-name myapp
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com
docker build -t myapp:1.0 .
docker tag myapp:1.0 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0
```

### 5.2 ECS vs EKS: how to choose

| | **ECS** | **EKS** |
|---|---|---|
| What it is | AWS-native container orchestrator | Managed Kubernetes control plane |
| Learning curve | Lower | Higher |
| Portability | AWS only | Standard Kubernetes, portable |
| Ecosystem | AWS integrations | Huge (Helm, Argo CD, operators, service meshes) |
| Control plane cost | Free | About $0.10/hour per cluster (check current pricing) |
| Best for | Simpler apps, small teams, quick wins | Complex platforms, multi-cloud, teams with K8s skills |

**Launch types**: **Fargate** (serverless, no nodes to manage) or **EC2** (you manage the instances, cheaper at scale, more control, GPU support).

**ECS concepts**: cluster, task definition (the "recipe": image, CPU/memory, ports, env, roles), task (a running instance), service (keeps N tasks running, integrates with ALB, rolling or blue/green deploys).

**Create and connect to an ECS cluster (no Kubernetes at all)**:
```bash
aws ecs create-cluster --cluster-name demo
aws ecs register-task-definition --cli-input-json file://taskdef.json
aws ecs create-service --cluster demo --service-name web \
  --task-definition web:1 --desired-count 2 --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-aaa,subnet-bbb],securityGroups=[sg-123],assignPublicIp=DISABLED}" \
  --enable-execute-command

# "Connect" into a running container (like kubectl exec)
aws ecs execute-command --cluster demo --task <task-id> \
  --container web --interactive --command "/bin/sh"
```
`execute-command` needs `enableExecuteCommand` on the service and SSM permissions (`ssmmessages:*`) on the **task role**.

### 5.3 EKS: creating a cluster

**Quickest way (eksctl)**:
```bash
eksctl create cluster --name demo --region us-east-1 \
  --nodegroup-name ng-1 --node-type t3.medium --nodes 2 --managed
```
This builds a VPC, the control plane, a managed node group, and writes your kubeconfig. It uses CloudFormation under the hood, a nice fact to mention.

**Proper way (IaC)**: Terraform with the `terraform-aws-modules/eks/aws` module, or CloudFormation / CDK. In production, define the VPC, subnets, cluster, node groups, add-ons and IAM in code.

**Node options**:
- **Managed node groups**: AWS manages the EC2 lifecycle and upgrades. A good default.
- **Fargate profiles**: serverless pods, no nodes (some limitations, e.g. no DaemonSets).
- **Self-managed nodes**: your own ASG, maximum control.
- **Karpenter**: modern autoscaler that provisions right-sized nodes just in time (often better and cheaper than Cluster Autoscaler).
- **EKS Auto Mode**: AWS also offers a mode that manages more of the compute, storage and networking for you. Mention it as a newer option and check the latest docs.

**Key add-ons**: Amazon **VPC CNI** (gives pods real VPC IPs), **CoreDNS**, **kube-proxy**, **EBS CSI driver** (persistent volumes), **AWS Load Balancer Controller** (creates ALBs/NLBs from Ingress/Service objects).

### 5.4 EKS: connecting to a cluster (this is the question you asked about)

```bash
# 1. Make sure the AWS CLI is authenticated as the right identity
aws sts get-caller-identity

# 2. Generate/update your kubeconfig entry
aws eks update-kubeconfig --region us-east-1 --name demo

# 3. Test
kubectl get nodes
kubectl config current-context
```

**How authentication actually works** (great depth to show): `kubectl` calls `aws eks get-token`, which creates a signed STS token from your AWS identity. The EKS API server validates that token, then maps your IAM identity to a Kubernetes user/group. Authorization after that is normal Kubernetes **RBAC**.

**How an IAM identity gets mapped to Kubernetes permissions**:
- **Access entries (current recommended method)**: managed through the EKS API.
  ```bash
  aws eks create-access-entry --cluster-name demo \
    --principal-arn arn:aws:iam::123456789012:role/DevOps
  aws eks associate-access-policy --cluster-name demo \
    --principal-arn arn:aws:iam::123456789012:role/DevOps \
    --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
    --access-scope type=cluster
  ```
- **aws-auth ConfigMap (older method)**: a ConfigMap in `kube-system` mapping IAM roles/users to K8s groups. Still seen in many existing clusters. Cluster authentication mode can be `CONFIG_MAP`, `API_AND_CONFIG_MAP` or `API`.
- By default, the IAM identity that **created** the cluster gets admin access. A classic gotcha: someone else can't connect because only the creator was mapped.

**Network access to the API endpoint**: the endpoint can be public, public and private, or private only. If private only, you need to be inside the VPC (bastion, SSM session, VPN, or Direct Connect). If you can't connect, check: wrong identity, no access entry, endpoint is private, security group or CIDR allowlist blocks you, or wrong region/cluster name in kubeconfig.

### 5.4b Pods calling AWS services (IRSA / Pod Identity)

1. Cluster has an OIDC provider (`eksctl utils associate-iam-oidc-provider --cluster demo --approve`).
2. Create an IAM role whose trust policy trusts that OIDC provider and a specific service account (`system:serviceaccount:<namespace>:<sa-name>`).
3. Annotate the Kubernetes ServiceAccount with `eks.amazonaws.com/role-arn`.
4. Pods using that ServiceAccount get temporary credentials automatically. **EKS Pod Identity** simplifies this further without needing OIDC trust setup per cluster.

### 5.5 Kubernetes WITHOUT EKS (self-managed on EC2)

Interviewers may ask "how would you run Kubernetes on AWS without EKS, and why would you?" Reasons: full control of the control plane, custom versions or flags, cost at very small scale, learning, or air-gapped needs. Downsides: you own upgrades, etcd backups, HA of the control plane, security patching, certificates.

**Option A: kubeadm on EC2** (the classic):
1. Launch EC2 instances (1 or 3 control plane, N workers) in private subnets. Open the needed ports in security groups (API server 6443, etcd 2379-2380 between control plane nodes, kubelet 10250, node communication).
2. On every node: disable swap, install a container runtime (containerd), install `kubeadm`, `kubelet`, `kubectl`.
3. On the control plane:
   ```bash
   sudo kubeadm init --pod-network-cidr=10.244.0.0/16
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```
4. Install a CNI plugin (Calico, Cilium or Flannel).
5. On each worker, run the `kubeadm join <control-plane-ip>:6443 --token ... --discovery-token-ca-cert-hash sha256:...` command printed by `init`.
6. **Connect**: your kubeconfig lives at `/etc/kubernetes/admin.conf`. Copy it to your laptop (or `~/.kube/config`), and make sure the server address points to a reachable address (a load balancer or the node's IP, and that IP/name is in the API server certificate's SANs).
7. For HA: 3 control plane nodes behind an NLB, external or stacked etcd, regular etcd snapshots.

**Option B: k3s** (lightweight, one-command install, good for labs and edge): `curl -sfL https://get.k3s.io | sh -`, kubeconfig at `/etc/rancher/k3s/k3s.yaml`.

**Option C: kops** (tool to create production-grade K8s on AWS; manages the instances, ASGs and DNS for you).

**Connecting to the EC2 nodes themselves**: prefer **SSM Session Manager** over SSH. No open port 22, no key pairs, full audit logging.
```bash
aws ssm start-session --target i-0123456789abcdef0
```
(Instance needs the SSM agent plus an instance profile with `AmazonSSMManagedInstanceCore`.)

### 5.6 Kubernetes essentials to know

Pod, Deployment, ReplicaSet, Service (ClusterIP/NodePort/LoadBalancer), Ingress, ConfigMap, Secret, Namespace, StatefulSet, DaemonSet, Job/CronJob, PV/PVC, HPA (pod autoscaling), probes (liveness/readiness/startup), resource requests and limits, RBAC, NetworkPolicy, taints/tolerations, node affinity, PodDisruptionBudget, Helm, kustomize.

**Everyday commands**:
```bash
kubectl get pods -A
kubectl describe pod <pod>
kubectl logs <pod> -f
kubectl exec -it <pod> -- sh
kubectl rollout status deployment/web
kubectl rollout undo deployment/web
kubectl top nodes
```

**Troubleshooting a pod (common scenario question)**:
- `Pending`: not enough resources, node selector/taint mismatch, PVC unbound. Check `kubectl describe pod` events.
- `CrashLoopBackOff`: app crashes. Check `kubectl logs --previous`, bad config/env, failing probes.
- `ImagePullBackOff`: wrong image name/tag, or no permission to ECR (node role or pull secret).
- Service not reachable: check selectors match labels, endpoints exist (`kubectl get endpoints`), security groups, ingress or target group health.

---

## 6. Infrastructure as Code: CloudFormation and Terraform

### 6.1 CloudFormation

**What it is**: AWS-native IaC. You write a YAML/JSON **template**, deploy it as a **stack**, and CloudFormation creates, updates and deletes the resources in the right order.

**Template anatomy**:
```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Demo bucket
Parameters:
  Env:
    Type: String
    AllowedValues: [dev, prod]
Conditions:
  IsProd: !Equals [!Ref Env, prod]
Resources:
  DataBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain
    Properties:
      BucketName: !Sub "demo-data-${AWS::AccountId}-${Env}"
      VersioningConfiguration:
        Status: !If [IsProd, Enabled, Suspended]
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
Outputs:
  BucketArn:
    Value: !GetAtt DataBucket.Arn
    Export:
      Name: !Sub "${Env}-data-bucket-arn"
```
Sections: `Parameters`, `Mappings`, `Conditions`, `Resources` (the only required one), `Outputs`.

**Intrinsic functions to know**: `!Ref`, `!GetAtt`, `!Sub`, `!Join`, `!If`, `!Select`, `!ImportValue`, `!FindInMap`.

**Deploying**:
```bash
aws cloudformation deploy --template-file template.yaml \
  --stack-name demo --parameter-overrides Env=dev \
  --capabilities CAPABILITY_NAMED_IAM          # needed when creating IAM resources

aws cloudformation validate-template --template-body file://template.yaml
aws cloudformation describe-stack-events --stack-name demo   # first stop when debugging
aws cloudformation delete-stack --stack-name demo
```

**Features interviewers ask about**:
- **Change sets**: preview what an update will create, modify or replace *before* applying. Essential for production safety. Watch for "Replacement: True", which means the resource is destroyed and recreated.
- **Rollback**: on failure, a stack rolls back automatically. `UPDATE_ROLLBACK_FAILED` can be recovered with `continue-update-rollback`.
- **Drift detection**: finds resources changed manually outside CloudFormation.
- **Nested stacks**: break big templates into reusable pieces. **StackSets**: deploy a stack across many accounts and regions (key for multi-account orgs).
- **Stack policies, DeletionPolicy / UpdateReplacePolicy**: protect critical resources (databases, buckets).
- **Cross-stack references**: `Export`/`ImportValue`, or use SSM parameters to reduce coupling.
- **Custom resources** (Lambda-backed) for things CloudFormation can't do natively. **Resource import** to bring existing resources under management.
- **Linting and policy checks**: `cfn-lint`, `cfn-guard`, `checkov`.
- **AWS CDK**: write infrastructure in TypeScript/Python etc., which synthesizes to CloudFormation. **SAM** is a CloudFormation extension for serverless.

### 6.2 Terraform

**What it is**: cloud-agnostic IaC by HashiCorp (now IBM). HCL language. OpenTofu is the open-source fork, worth knowing the name.

```hcl
terraform {
  required_version = ">= 1.6"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
  backend "s3" {
    bucket         = "my-tf-state-bucket"
    key            = "prod/network/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "tf-locks"       # state locking
    encrypt        = true
  }
}

provider "aws" { region = "us-east-1" }

resource "aws_s3_bucket" "data" {
  bucket = "demo-data-bucket-12345"
  tags   = { Env = "dev" }
}
```

**Core workflow**: `terraform init`, `terraform fmt`, `terraform validate`, `terraform plan`, `terraform apply`, `terraform destroy`.

**Concepts**:
- **State**: Terraform's record of what it manages. Never commit it to Git. Store it remotely (S3 with encryption and versioning) with **locking** (DynamoDB table; newer Terraform versions also support S3 native locking, check the version you use).
- **Modules**: reusable building blocks (VPC module, EKS module). Use the public registry for well-maintained ones.
- **Variables, outputs, locals, data sources**, `count` / `for_each`, `lifecycle` blocks (`prevent_destroy`, `create_before_destroy`, `ignore_changes`).
- **Workspaces vs separate state per environment**: many teams prefer separate directories/state files per environment for safer blast radius.
- **`terraform import`** and `import` blocks: adopt existing resources. Essential in migration and modernization work.
- **Drift**: `terraform plan` shows differences between code, state and real infrastructure.
- **Secrets**: don't hard-code. State can contain secrets, so encrypt it and restrict access.
- **Pipeline pattern**: run `fmt`/`validate`/`tflint`/`checkov` on pull requests, post the `plan` output to the PR, require approval, then `apply` on merge.

### 6.3 CloudFormation vs Terraform (likely question)

| | CloudFormation | Terraform |
|---|---|---|
| Scope | AWS only | Multi-cloud and SaaS providers |
| State | Managed by AWS, no state file to handle | You manage the state backend |
| Language | YAML/JSON (or CDK) | HCL |
| New AWS features | Often first, sometimes same day | Provider lag of days to weeks |
| Rollback | Automatic on failure | None; you fix forward |
| Strengths | Native, StackSets, no extra tooling | Modules, ecosystem, plan output, multi-cloud |

**Good answer**: "I'd choose based on the customer. If they're all-in on AWS and want no extra tooling, CloudFormation or CDK. If they're multi-cloud, already use Terraform, or want the module ecosystem, Terraform. Whatever we pick, everything goes through code review and a pipeline, and nothing is changed by hand."

---

## 7. CI/CD: GitHub Actions, AWS CodeSuite, CircleCI

### 7.1 CI/CD concepts

- **CI (Continuous Integration)**: every push builds and tests the code automatically.
- **Continuous Delivery**: every change is deployable, with a manual approval to release. **Continuous Deployment**: releases automatically.
- **Typical pipeline**: checkout, lint and unit tests, security scans (SAST, dependency scan, secret scan), build Docker image, scan image, push to ECR, deploy to dev, integration tests, approval, deploy to prod, smoke test, rollback on failure.
- **Deployment strategies**: **rolling** (replace gradually), **blue/green** (new environment alongside, switch traffic, instant rollback), **canary** (small % of traffic first), **recreate** (downtime). Also feature flags.
- **GitOps**: Git is the source of truth for cluster state; **Argo CD** or **Flux** pull and reconcile changes into Kubernetes. Pull-based and auditable.

### 7.2 GitHub Actions with AWS (OIDC, no stored keys)

**One-time AWS setup**:
1. IAM, Identity providers, add an OpenID Connect provider: URL `https://token.actions.githubusercontent.com`, audience `sts.amazonaws.com`.
2. Create a role whose trust policy only allows your repo and branch:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
      "StringLike":   { "token.actions.githubusercontent.com:sub": "repo:my-org/my-repo:ref:refs/heads/main" }
    }
  }]
}
```

**Workflow** (`.github/workflows/deploy.yml`):
```yaml
name: build-and-deploy
on:
  push:
    branches: [main]

permissions:
  id-token: write     # required for OIDC
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4   # check for the latest major version
        with:
          role-to-assume: arn:aws:iam::123456789012:role/gha-deploy
          aws-region: us-east-1

      - id: ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push
        env:
          REGISTRY: ${{ steps.ecr.outputs.registry }}
          TAG: ${{ github.sha }}
        run: |
          docker build -t $REGISTRY/myapp:$TAG .
          docker push $REGISTRY/myapp:$TAG

      - name: Deploy to EKS
        run: |
          aws eks update-kubeconfig --name demo --region us-east-1
          kubectl set image deployment/web web=$REGISTRY/myapp:${{ github.sha }}
          kubectl rollout status deployment/web --timeout=180s
```
(For ECS you would instead render a new task definition and use `aws-actions/amazon-ecs-deploy-task-definition`.)

**Talking points**: reusable workflows, environments with required approvers, caching, matrix builds, self-hosted runners (for private network access), pinning actions to a version or commit SHA, least-privilege deploy roles, separate roles per environment.

### 7.3 AWS CodeSuite

| Service | Role |
|---|---|
| **CodePipeline** | Orchestrates stages (source, build, test, deploy, approval) |
| **CodeBuild** | Managed build runners; steps defined in `buildspec.yml` |
| **CodeDeploy** | Deploys to EC2/on-prem, ECS (blue/green), Lambda (canary/linear), with rollback |
| **CodeCommit** | AWS-hosted Git. Its availability for new customers has changed over time, so check its current status; most teams use GitHub or GitLab |
| **CodeArtifact** | Package repository (npm, pip, Maven) |

```yaml
# buildspec.yml
version: 0.2
phases:
  pre_build:
    commands:
      - aws ecr get-login-password | docker login --username AWS --password-stdin $REGISTRY
  build:
    commands:
      - docker build -t $REGISTRY/myapp:$CODEBUILD_RESOLVED_SOURCE_VERSION .
      - docker push $REGISTRY/myapp:$CODEBUILD_RESOLVED_SOURCE_VERSION
artifacts:
  files: ["imagedefinitions.json"]
```

**When to pick which**: CodeSuite shines when the customer wants everything inside AWS (compliance, IAM-native, no external SaaS). GitHub Actions wins on developer experience and ecosystem. A common hybrid: GitHub for source and CI, CodeDeploy or CodePipeline for deployment.

### 7.4 CircleCI with AWS

```yaml
version: 2.1
orbs:
  aws-cli: circleci/aws-cli@4
  aws-ecr: circleci/aws-ecr@9
workflows:
  build-deploy:
    jobs:
      - aws-ecr/build_and_push_image:
          repo: myapp
          tag: "${CIRCLE_SHA1}"
          auth:
            - aws-cli/setup:
                role_arn: arn:aws:iam::123456789012:role/circleci-deploy   # OIDC role
```
CircleCI concepts: **config.yml**, **jobs**, **workflows**, **executors** (docker/machine), **orbs** (reusable packages), **contexts** (shared secrets), **caching**, **workspaces**, parallelism. It also supports OIDC tokens to assume AWS roles, so avoid stored keys. Orb versions change, so verify them in the registry before using.

---

## 8. AWS migration and modernization

This is Avahi's bread and butter. Spend real time here.

### 8.1 The 7 Rs (migration strategies)

| R | Meaning | Example |
|---|---|---|
| **Retire** | Turn it off | Unused legacy app |
| **Retain** | Keep where it is for now | Compliance or too risky |
| **Rehost** | "Lift and shift" | VM to EC2 using MGN |
| **Relocate** | Move without changes | VMware to VMware Cloud on AWS |
| **Repurchase** | Move to SaaS | Self-hosted CRM to Salesforce |
| **Replatform** | "Lift, tinker and shift" | Self-managed MySQL to RDS/Aurora |
| **Refactor** | Re-architect cloud-native | Monolith to microservices/serverless |

### 8.2 Phases (AWS Migration Acceleration Program style)

1. **Assess**: inventory servers, dependencies, licensing, TCO and business case (Application Discovery Service, Migration Evaluator, Migration Hub).
2. **Mobilize**: build the landing zone (accounts, network, IAM, logging, guardrails via Control Tower), train people, pick a pilot.
3. **Migrate and modernize**: move in waves, test, cut over, optimize.

### 8.3 Tools

- **AWS Application Migration Service (MGN)**: continuous block-level replication of servers to EC2, then a cutover.
- **AWS Database Migration Service (DMS)** + **Schema Conversion Tool (SCT)**: migrate databases, including heterogeneous (Oracle or SQL Server to PostgreSQL/Aurora). Supports ongoing replication (CDC) for near-zero downtime.
- **DataSync, Snowball family, Transfer Family**: bulk data and file movement.
- **Migration Hub**: tracking across tools.

### 8.4 Cutover approach (a strong answer)

1. Set up networking (VPN/Direct Connect) and replicate data continuously.
2. Build and test the target environment in parallel.
3. Run performance and functional tests, plus a rollback plan agreed in advance.
4. Lower DNS TTL before the cutover window.
5. Freeze writes, do the final sync, switch DNS or traffic gradually (weighted Route 53), monitor.
6. Keep the old environment for a defined period, then decommission.

### 8.5 Avahi-specific migration paths (shows you did homework)

- **Heroku to AWS**: dynos map to ECS Fargate, App Runner or Elastic Beanstalk (or EKS if the team wants Kubernetes); Heroku Postgres to RDS/Aurora PostgreSQL (DMS, logical replication, or dump/restore); Redis add-on to ElastiCache; config vars to Secrets Manager/Parameter Store; build packs to Dockerfiles; Heroku pipelines to CI/CD.
- **Azure to AWS**: map VMs to EC2, AKS to EKS, Azure SQL to RDS, Blob to S3, Entra ID to IAM Identity Center, ARM/Bicep to Terraform/CloudFormation.
- **OpenAI or Gemini to Amazon Bedrock**: swap model API calls to Bedrock (e.g. Claude, Nova, Llama models), keep data in the customer's AWS account, handle prompt differences, evaluate output quality and cost, add Guardrails.

### 8.6 Modernization

Monolith to microservices (strangler fig pattern), containers, serverless (Lambda, Step Functions), managed databases, event-driven design with SQS/SNS/EventBridge, CI/CD, observability. Don't modernize for its own sake: tie it to business outcomes such as speed of delivery, cost, or resilience.

---

## 9. Reliability, security, cost: Well-Architected and HA design

### 9.1 The six pillars (memorize)

1. **Operational Excellence**: IaC, CI/CD, runbooks, observability, learn from incidents.
2. **Security**: identity, detection, infrastructure protection, data protection, incident response.
3. **Reliability**: recover from failure, scale horizontally, test recovery.
4. **Performance Efficiency**: right resource types, caching, experiments.
5. **Cost Optimization**: pay only for what you need, rightsize, commitments, visibility.
6. **Sustainability**: efficient resource use, reduce waste.

Tool: the **AWS Well-Architected Tool** runs structured reviews, and "Well-Architected Reviews" are a common customer engagement for partners.

### 9.2 High availability and fault tolerance

- Spread across **multiple AZs**: ALB + Auto Scaling Group across AZs, RDS Multi-AZ, EKS nodes in several AZs with pod topology spread constraints.
- **Health checks** and automatic replacement; **stateless** app tiers with state in RDS/DynamoDB/S3/ElastiCache.
- **Decouple** with queues so spikes don't take you down.
- **Disaster recovery** strategies (cheapest to most expensive): **backup and restore**, **pilot light**, **warm standby**, **multi-site active/active**. Define **RPO** (how much data loss is acceptable) and **RTO** (how long to recover) first, then choose.
- **Test it**: game days, failover drills, chaos testing.
- Backups: AWS Backup, cross-region and cross-account copies, test restores.

### 9.3 Performance

Caching (CloudFront, ElastiCache), read replicas, right-sizing, autoscaling policies (target tracking), Graviton instances for price-performance, database indexing and connection pooling (RDS Proxy), async processing.

### 9.4 Security services map

| Need | Service |
|---|---|
| Audit trail | CloudTrail |
| Config history and compliance rules | AWS Config |
| Threat detection | GuardDuty |
| Central findings | Security Hub |
| Vulnerability scanning | Inspector, ECR scanning |
| Web protection | WAF, Shield |
| Secrets | Secrets Manager, SSM Parameter Store |
| Encryption keys | KMS |
| Permissions analysis | IAM Access Analyzer |
| Multi-account guardrails | SCPs, Control Tower |

### 9.5 Cost optimization checklist

- Tag everything; use Cost Explorer, Budgets and Cost Anomaly Detection.
- Rightsize with Compute Optimizer; shut down non-prod off-hours.
- **Savings Plans / Reserved Instances** for steady load; **Spot** for fault-tolerant work (EKS with Karpenter, batch, CI runners).
- Graviton instances; S3 lifecycle and Intelligent-Tiering; delete unattached EBS volumes and old snapshots.
- Watch hidden costs: **NAT Gateway data processing**, cross-AZ and cross-region data transfer, idle load balancers, CloudWatch Logs retention.
- Use VPC endpoints for S3/DynamoDB to cut NAT traffic.

### 9.6 Observability

CloudWatch (metrics, logs, alarms, dashboards), **Container Insights**, X-Ray or OpenTelemetry for tracing, Prometheus and Grafana (Amazon Managed Service for Prometheus / Managed Grafana), centralized logging (CloudWatch Logs, OpenSearch, Fluent Bit). Know the difference between metrics, logs and traces, and what an SLO and an error budget are.

---

## 10. Data, ML and AI exposure (the "plus" in the JD, and Avahi's focus)

Avahi positions itself as an AI company on AWS, so even basic fluency here helps a lot.

- **Amazon Bedrock**: managed access to foundation models (including Anthropic Claude, Amazon Nova and others) through one API, with no infrastructure to run. Key features: **Knowledge Bases** (managed RAG), **Agents**, **Guardrails** (content and PII filters), model evaluation, fine-tuning for some models, provisioned throughput.
- **RAG (Retrieval Augmented Generation)**: embed documents into a vector store (OpenSearch Serverless, Aurora pgvector, etc.), retrieve relevant chunks at query time, and pass them to the model with the question. Lets the model answer from private data without training.
- **SageMaker**: build, train, deploy custom ML models; endpoints, pipelines, notebooks.
- **Task-specific AI services**: Textract (document extraction), Transcribe (speech to text), Comprehend (NLP), Rekognition (images/video), Translate, Polly, Amazon Connect + Lex (voice and contact center). These map directly to Avahi's solutions (data extraction, voice agents, translation, image analysis).
- **Data platform pieces**: S3 data lake, Glue (ETL and catalog), Athena, Redshift, Lake Formation, Kinesis/MSK for streaming, QuickSight for BI.
- **Infra angle for AI** (your edge as a DevOps lead): IAM and VPC endpoints for Bedrock so traffic stays private, cost control (token usage, caching, model choice), logging and guardrails for compliance, GPU capacity on EKS, and IaC for the whole stack.

If asked "what AI have you worked with?", be truthful about your exposure and describe what you've built or tried in the labs below.

---

# Part 3: Interview questions with model answers

Practice saying these out loud in your own words. Don't memorize word for word.

## A. IAM and security

**1. How would you give a developer in another AWS account access to an S3 bucket?**
Create a role in the bucket's account with a permissions policy for that bucket and a trust policy naming the other account (or specific principal). The developer assumes the role with `sts:AssumeRole` and gets temporary credentials. If the bucket policy also needs to allow it, add that. No shared keys.

**2. What is the difference between an IAM user and a role?**
A user has long-lived credentials and represents a person or app. A role has no credentials of its own; it is assumed and returns temporary credentials. Roles are preferred for EC2, Lambda, pods, CI/CD and cross-account access.

**3. How does IAM evaluate a request?**
Default deny. An explicit deny always wins. Otherwise there must be an allow, and it must also pass any SCP, permissions boundary or session policy. For resource-based policies in the same account, either the identity or the resource policy allowing is enough, while cross-account needs both sides.

**4. How do you secure a CI/CD pipeline's access to AWS?**
OIDC federation so there are no stored keys, a dedicated role per environment with least privilege, trust policy scoped to a specific repo and branch, required approvals for production, short session durations, and CloudTrail alerts on unusual use.

**5. A developer says "I get AccessDenied". How do you debug it?**
Run `aws sts get-caller-identity` to confirm who they really are. Check the identity policy, resource policy, SCPs, permissions boundary, and any condition keys (region, MFA, source IP, VPC endpoint). Use the IAM Policy Simulator and CloudTrail to see the exact denied action and the reason. Check KMS key policies if encryption is involved.

**6. How do you protect data in S3?**
Block Public Access, bucket policies with least privilege, encryption (SSE-KMS where required), versioning plus MFA delete or Object Lock for critical data, access logging, VPC endpoint policies, Macie for sensitive data discovery, and replication for DR.

## B. Networking

**7. Design a VPC for a three-tier web app.**
`/16` VPC across 3 AZs. Public subnets for ALB and NAT gateways, private app subnets, isolated DB subnets. ALB takes traffic, app tier in an Auto Scaling Group or containers, RDS Multi-AZ. Security groups chain (ALB SG to app SG to DB SG). VPC endpoints for S3 and other services. Flow logs on. WAF in front of the ALB.

**8. Security group vs NACL?**
SGs are stateful, attached to ENIs, allow-only. NACLs are stateless, at the subnet level, with allow and deny rules and numbered order. Most control lives in SGs; NACLs are a coarse extra layer.

**9. A private instance can't reach the internet. What do you check?**
Route table has `0.0.0.0/0` pointing to a NAT gateway; the NAT gateway is in a public subnet with a route to the IGW and has an Elastic IP; security group and NACL outbound rules; DNS resolution settings; or use a VPC endpoint if only AWS services are needed.

## C. Containers and Kubernetes

**10. ECS or EKS: which would you recommend?**
It depends on the customer. For a small team that wants simplicity and AWS-only, ECS on Fargate gets them running quickly with less operational overhead. For teams that already know Kubernetes, need portability, or want the ecosystem (Helm, Argo CD, operators), EKS. I'd also factor in cost, hiring, and how complex the workloads are.

**11. How do you connect to an EKS cluster?**
Authenticate with the AWS CLI, run `aws eks update-kubeconfig --name <cluster> --region <region>`, then use `kubectl`. Under the hood `kubectl` uses `aws eks get-token`. The IAM identity must be mapped to Kubernetes permissions via EKS access entries (or the older aws-auth ConfigMap), and the network must be able to reach the API endpoint.

**12. A colleague can't run kubectl against the cluster you created. Why?**
By default only the creator identity has admin. They need an access entry (or an aws-auth mapping) plus an associated access policy or RBAC binding. Also check the endpoint is reachable (private endpoint, IP allowlist) and the right profile and region are in use.

**13. How would you give a pod access to S3 safely?**
IRSA or EKS Pod Identity: a dedicated IAM role for the pod's ServiceAccount with only the needed S3 actions. Never use the node's instance role for app permissions or embed keys in the image.

**14. How would you run Kubernetes without EKS, and why would you?**
kubeadm (or kops/k3s) on EC2: install containerd and kubeadm, `kubeadm init`, install a CNI, join workers, then use the admin kubeconfig. I'd only do this for special requirements, because we then own control plane HA, etcd backups, upgrades and certificates. In most customer cases EKS removes that burden.

**15. How do you upgrade an EKS cluster with minimal risk?**
Read the release notes and check deprecated APIs (e.g. with a tool like pluto or kubent). Upgrade in a lower environment first. Upgrade the control plane one minor version at a time, then add-ons, then node groups (managed groups roll nodes using surge). Use PodDisruptionBudgets, multiple replicas and readiness probes so there is no downtime. Have rollback or blue/green cluster as the fallback plan.

**16. A pod is in CrashLoopBackOff. Walk me through it.**
`kubectl describe pod` for events and exit codes, `kubectl logs --previous`, check config and secrets, probes, resource limits (OOMKilled), image version, and dependencies like DB reachability.

**17. How do you autoscale on EKS?**
HPA for pods (CPU/memory/custom metrics), Karpenter or Cluster Autoscaler for nodes, KEDA for event-driven scaling. Set sensible requests and limits, since autoscaling depends on them.

## D. IaC and CI/CD

**18. How do you manage Terraform state in a team?**
Remote backend in S3 with versioning and encryption, state locking, separate state per environment/component, restricted access, no secrets in plain text, and changes only through the pipeline.

**19. What happens if someone changes a resource manually in the console?**
That's drift. Detect it with `terraform plan` or CloudFormation drift detection, decide whether to revert it or codify it, and prevent it in future with restricted console permissions and SCPs/policies for production.

**20. Describe a CI/CD pipeline you'd build for a containerized app on AWS.**
PR triggers lint, tests, SAST and dependency scans. Merge to main builds an image tagged with the commit SHA, scans it, pushes to ECR. Deploy to dev automatically via GitHub Actions with OIDC; integration tests; manual approval; deploy to prod using a rolling or blue/green strategy with health checks and automatic rollback. IaC changes go through plan and review. Notifications and DORA metrics tracked.

**21. How do you do zero-downtime deployments?**
Multiple replicas across AZs, readiness probes, rolling update with `maxUnavailable: 0`, connection draining, backward-compatible database migrations (expand and contract), and blue/green or canary for riskier changes, plus automated rollback on alarms.

**22. How do you handle secrets in the pipeline and in apps?**
Secrets Manager or Parameter Store, injected at runtime (External Secrets Operator or CSI driver on EKS, task definition `secrets` on ECS), encrypted with KMS, rotated, never in Git or images, with secret scanning in CI.

## E. Migration and architecture

**23. A customer wants to move from Heroku to AWS. What's your approach?**
Discovery (apps, add-ons, data size, traffic, downtime tolerance, team skills), choose target (Fargate or App Runner for simplicity), containerize, build IaC and a landing zone, migrate Postgres with DMS or logical replication for minimal downtime, replicate config/secrets, set up CI/CD and monitoring, test, then a planned cutover via DNS with rollback. Then optimize cost and security.

**24. How would you migrate a large on-prem database with minimal downtime?**
DMS with full load plus CDC to keep the target in sync, SCT if the engine changes, performance test the target, then a short write-freeze, final sync, and cut over the application connection, with rollback ready.

**25. How do you approach a customer's high AWS bill?**
Start with Cost Explorer and tags to find the top spend, look for idle/oversized resources, NAT and data transfer, unused storage, then commitments (Savings Plans), Spot, Graviton, lifecycle policies, and scheduling non-prod. Set budgets and alerts so it stays fixed.

**26. How would you design for high availability and DR?**
Multi-AZ everything for HA. Ask for RPO and RTO, then pick backup/restore, pilot light, warm standby or active-active accordingly. Automate failover where possible and test it regularly.

**27. A production site is down. What do you do?**
Stabilize first: check alarms and recent changes (deploys, config), roll back if a change caused it, communicate status. Then diagnose using metrics, logs and traces (load balancer 5xx, target health, DB connections, resource saturation). After recovery, run a blameless postmortem with action items.

## F. AI-related

**28. How would you help a customer move from OpenAI to Bedrock?**
Assess current usage (models, prompts, volume, latency, cost), pick equivalent Bedrock models, create an abstraction layer for the model API, test quality with an evaluation set, configure IAM, VPC endpoints, Guardrails and logging, roll out gradually behind a flag, and compare cost and performance.

**29. What is RAG and when would you use it?**
Retrieval Augmented Generation: fetch relevant private documents at query time and give them to the LLM as context. Use it when answers must reflect company data that changes, without retraining a model. Bedrock Knowledge Bases manages much of this.

## G. Leadership and behavioral (the JD stresses mentoring and process design)

Use STAR (Situation, Task, Action, Result). Prepare 4 stories:

1. **A hard technical problem you solved** (an outage, a failed deployment, a performance issue).
2. **Mentoring someone** (how you helped a junior grow, what changed).
3. **Improving a process** (a manual task you automated, a pipeline you made faster, time saved).
4. **Disagreement or a mistake** (what you learned, what you changed afterward).

**Questions about process design they may ask**: "How would you standardize project delivery across teams?" Answer ideas: reusable IaC modules and templates, a landing zone baseline, a standard pipeline, definition of done including security and cost checks, runbooks, documentation as code, architecture reviews, regular retrospectives.

## H. "I don't have deep experience in X" (use this honestly)

When you hit a gap, don't bluff. Try: "I haven't run that in production. Here's how I understand it works, here's what I built in a lab, and here's how I'd validate it before recommending it to a customer." Interviewers respect clear reasoning and honest boundaries far more than confident wrong answers.

## I. Questions to ask THEM

- What does a typical customer engagement look like from discovery to handover?
- What's the split between migrations, modernization and AI projects right now?
- How are standards enforced across projects (IaC modules, pipelines, reviews)?
- How does the team work with AWS (funding programs, partner team)?
- What would success look like in the first 90 days?
- How do you support certifications and learning?

---

# Part 4: Hands-on lab plan (2 weeks)

**Cost safety first**: set a **budget alert** (AWS Budgets) at a low amount before anything else. Use a personal account, tear down at the end of each session. The costly items are the **EKS control plane (about $0.10/hour)**, **NAT gateways**, load balancers and large instances. Check the Free Tier terms for your account type, as they have changed over time.

| Day | Lab | Outcome |
|---|---|---|
| 1 | Create account setup: root MFA, budget alert, IAM Identity Center user, `aws configure sso`, `get-caller-identity` | You can explain login and access end to end |
| 2 | IAM: create user, group, policy, role, assume it via CLI; break a policy and debug with the simulator | Real `AccessDenied` stories |
| 3 | VPC from scratch (public/private subnets, IGW, NAT, SGs); launch EC2 in private subnet, access with SSM | Networking confidence |
| 4 | Launch template + ASG + ALB across 2 AZs; kill an instance, watch recovery | HA demo |
| 5 | Docker: containerize a small app, push to ECR | Docker + ECR |
| 6 | ECS Fargate service behind an ALB; try `execute-command` | Non-K8s container path |
| 7 | CloudFormation: template for VPC + S3; change sets; trigger a failure and rollback; drift detection | CFN depth |
| 8 | Terraform: same VPC, S3 remote state with locking; modules; `import` an existing resource | Terraform depth |
| 9 | EKS via `eksctl` or Terraform module; `update-kubeconfig`; deploy app + Service + Ingress with AWS Load Balancer Controller | Core EKS skill |
| 10 | EKS access: add a second IAM role via access entry; IRSA or Pod Identity for S3 access | The auth story you asked about |
| 11 | Kubernetes without EKS: 2-3 EC2 instances, `kubeadm` cluster, copy kubeconfig, deploy an app (use small instances, tear down fast) | Self-managed K8s |
| 12 | GitHub Actions with OIDC: build, push to ECR, deploy to EKS or ECS; add an approval step | CI/CD |
| 13 | CodePipeline + CodeBuild version of the same; optional CircleCI | CodeSuite and CircleCI familiarity |
| 14 | Bedrock: call a model via CLI/SDK, build a tiny RAG with a Knowledge Base; migration mock-plan write-up (Heroku to AWS) | AI + migration story |

**Destroy everything** at the end: `eksctl delete cluster`, `terraform destroy`, delete stacks, release Elastic IPs, delete NAT gateways and load balancers.

**Certifications**: the JD says Professional and specialty certs are a plus. Realistic path: Solutions Architect Associate, then DevOps Engineer Professional or Solutions Architect Professional; useful specialties include Security and Advanced Networking. If you can't finish one before the interview, say what you're studying and your timeline.

---

# Part 5: Cheat sheet

```bash
# Identity
aws sts get-caller-identity
aws configure sso && aws sso login --profile <p>

# IAM
aws iam create-user --user-name <u>
aws iam attach-role-policy --role-name <r> --policy-arn <arn>

# EC2 / SSM
aws ec2 describe-instances --query "Reservations[].Instances[].[InstanceId,State.Name]"
aws ssm start-session --target <instance-id>

# ECR / ECS
aws ecr get-login-password | docker login --username AWS --password-stdin <registry>
aws ecs list-services --cluster <c>
aws ecs execute-command --cluster <c> --task <t> --container <n> --interactive --command "/bin/sh"

# EKS / kubectl
eksctl create cluster --name <n> --region <r>
aws eks update-kubeconfig --name <n> --region <r>
kubectl get nodes; kubectl get pods -A
kubectl describe pod <p>; kubectl logs <p> --previous
kubectl rollout undo deployment/<d>

# CloudFormation
aws cloudformation deploy --template-file t.yaml --stack-name <s> --capabilities CAPABILITY_NAMED_IAM
aws cloudformation describe-stack-events --stack-name <s>

# Terraform
terraform init && terraform plan -out tfplan && terraform apply tfplan
```

**One-line memory hooks**
- Humans: SSO. Workloads: roles. Pipelines: OIDC. Pods: IRSA/Pod Identity.
- Multi-AZ = HA. Read replica = scale reads. Backups + cross-region = DR.
- State in S3 with locking. Everything through a pipeline. Nothing by hand.
- Well-Architected: Ops, Security, Reliability, Performance, Cost, Sustainability.
- Migration: 7 Rs, MGN for servers, DMS for databases, cut over with a rollback plan.