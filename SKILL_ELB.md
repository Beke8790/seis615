# SKILL: AWS Café Application – Elasticity and High Availability Assignment

## 1. Purpose

You are an AWS Infrastructure Automation Agent responsible for implementing the AWS Café Application Elasticity and High Availability assignment.

Your objective is to create, configure, validate, and test an AWS infrastructure that supports:

- High availability across two Availability Zones.
- An internet-facing Application Load Balancer (ALB).
- EC2 application servers deployed in private subnets.
- An EC2 Launch Template.
- An Auto Scaling Group (ASG).
- Automatic scaling based on application demand.
- Amazon RDS Multi-AZ database availability.
- Application testing through the load balancer.
- Load testing to demonstrate automatic scaling.

## 2. Execution Rules

1. **Reuse existing infrastructure.** Use the existing VPC, subnets, route tables, Internet Gateway, NAT Gateway, security groups, IAM role/instance profile, and RDS resources already created for the Café application (the `cafe-app-vpc` deployment). Do not recreate these resources if they are already correctly configured.
2. **Inspect before changing.** Validate existing resources and their configuration before creating or modifying anything.
3. **No deletions.** Never delete existing AWS resources.
4. **Confirm before changes.** Before changing any existing item, report the planned change and get explicit user confirmation.
5. **No secrets exposure.** Never print, log, or hardcode passwords, secrets, keys, or tokens. Use AWS Secrets Manager / IAM roles where needed.
6. **Verify before reporting success.** Confirm resource state and functional behavior with evidence.

## 3. Existing Baseline to Reuse

Discover and validate the existing Café application resources and reuse them as the baseline:

| Component | Expected existing baseline | Reuse guidance |
|---|---|---|
| VPC | `cafe-app-vpc` (CIDR `10.20.0.0/16`) | Reuse existing VPC; confirm 2 AZs, DNS support/hostnames enabled |
| Public subnets | 2 public subnets across 2 AZs | Use for internet-facing ALB |
| Private subnets | 2 private subnets across 2 AZs | Use for ASG EC2 application servers |
| Internet Gateway | Attached to `cafe-app-vpc` | Reuse |
| NAT Gateway | Existing NAT for private subnet outbound access | Reuse; validate private route to NAT |
| Route tables | Public route to IGW; private route to NAT | Reuse/validate |
| Web security group | `cafe-app-sg` | Reuse for app servers; validate ALB-to-app HTTP access as needed |
| DB security group | `cafe-db-sg` | Reuse; ensure DB access allowed from app tier only |
| IAM role / instance profile | `cafe-app-role` | Reuse for EC2/launch template if permissions are sufficient |
| RDS | Existing private MariaDB `cafe-db` | Reuse existing database; validate Multi-AZ status and report/plan changes only with approval |
| EC2 key pair | `cafe-app-nova` | Reuse if needed by launch template |

If actual names/IDs differ, use discovery results and report the mapping.

## 4. ELB/HA Components (Skill Targets)

Create or validate the following components on top of the existing baseline in `cafe-app-vpc`:

| Component | Name | Status target |
|---|---|---|
| Application Load Balancer | `cafe-app-alb` | Internet-facing ALB in existing public subnets |
| Target Group | `cafe-app-tg` | HTTP:80 targets, health-checked, attached to ALB/ASG |
| Launch Template | `cafe-app-lt` | AL2023 app server template for ASG |
| Auto Scaling Group | `cafe-app-asg` | Private subnets, Min 2 / Desired 2 / Max 6, ELB health checks |

### 4.1 Target Group

| Property | Required value |
|---|---|
| Name | `cafe-app-tg` (or existing equivalent approved by user) |
| Target type | Instances |
| Protocol / Port | HTTP / 80 |
| VPC | Existing `cafe-app-vpc` |
| Health check | HTTP, path `/` or `/cafe/` (confirm application path) |
| Success criteria | Targets become healthy |

### 4.2 Application Load Balancer

| Property | Required value |
|---|---|
| Name | `cafe-app-alb` |
| Scheme | Internet-facing |
| Subnets | Existing public subnets in 2 AZs |
| Security group | Reuse/configure SG allowing inbound HTTP 80 from internet |
| Listener | HTTP 80 forwarding to target group |

### 4.3 Launch Template

| Property | Required value |
|---|---|
| Name | `cafe-app-lt` |
| AMI | Amazon Linux 2023 |
| Instance type | `t3.micro` or approved size |
| Key pair | `cafe-app-nova` (if required) |
| Security group | Existing app security group, validated for ALB access |
| IAM instance profile | Existing `cafe-app-role` |
| User data | Install/start Apache/PHP, deploy Café app, configure DB/Secrets Manager integration as needed |

User data must not contain secrets. If app deployment requires artifacts or secret values, use secure transfer and Secrets Manager.

### 4.4 Auto Scaling Group

| Property | Required value |
|---|---|
| Name | `cafe-app-asg` |
| Launch template | `cafe-app-lt` |
| Subnets | Existing private subnets in 2 AZs |
| Target group | `cafe-app-tg` attached |
| Health check type | ELB |
| Health check grace period | 180 seconds (or validated appropriate value) |
| Minimum size | 2 |
| Desired capacity | 2 |
| Maximum size | 6 |
| Scaling policy | Target tracking based on average CPU utilization (target 25% unless user specifies otherwise) |

**Auto Scaling Group size:** Minimum = **2**, Maximum = **6**.

## 4.5 CloudFormation Automation (`cafe-elb.yaml`)

Create or validate CloudFormation automation for the ELB/ASG layer in `cafe-app/cafe-elb.yaml`. This template provisions the load balancing and auto scaling components on top of the existing `cafe-infra.yaml` baseline.

### Template

| Property | Value |
|---|---|
| File | `cafe-app/cafe-elb.yaml` |
| Stack name (recommended) | `cafe-elb-stack` |
| Scope | ALB, ALB security group, target group, listener, launch template, ASG, scaling policy |
| Baseline dependency | Existing `cafe-app-vpc` resources from `cafe-app/cafe-infra.yaml` |

### Required parameters

| Parameter | Purpose |
|---|---|
| `VpcId` | Existing Cafe VPC ID |
| `PublicSubnet1Id`, `PublicSubnet2Id` | Existing public subnets for internet-facing ALB |
| `PrivateSubnet1Id`, `PrivateSubnet2Id` | Existing private subnets for ASG instances |
| `AppSecurityGroupId` | Existing app/web security group ID; template adds ALB->app ingress on 80 |
| `InstanceProfileName` | Existing IAM instance profile (default `cafe-app-role`) |
| `KeyName` | Existing EC2 key pair (default `cafe-app-nova`) |
| `LatestAmiId` | Amazon Linux 2023 AMI SSM parameter |
| `InstanceType` | Launch template instance type (default `t3.micro`) |

### Resources created/managed

- `AWS::EC2::SecurityGroup` - `cafe-alb-sg` allowing inbound HTTP 80 from `0.0.0.0/0`.
- `AWS::EC2::SecurityGroupIngress` - allows HTTP 80 from `cafe-alb-sg` to existing app security group.
- `AWS::ElasticLoadBalancingV2::TargetGroup` - `cafe-app-tg`, HTTP:80, instance targets, health check HTTP `/cafe/` (fallback `/` acceptable per app validation).
- `AWS::ElasticLoadBalancingV2::LoadBalancer` - `cafe-app-alb`, internet-facing, two public subnets.
- `AWS::ElasticLoadBalancingV2::Listener` - HTTP:80 forwarding to target group.
- `AWS::EC2::LaunchTemplate` - `cafe-app-lt`, AL2023, no secrets in user data.
- `AWS::AutoScaling::AutoScalingGroup` - `cafe-app-asg`, private subnets, Min 2 / Desired 2 / Max 6, ELB health checks.
- `AWS::AutoScaling::ScalingPolicy` - target tracking average CPU utilization, target 25%.

### Validation/deploy

1. Validate template:
   ```powershell
   aws cloudformation validate-template --template-body file://cafe-app/cafe-elb.yaml --region us-east-1
   ```
2. Deploy (example using existing stack outputs/values; use `UsePreviousValue` or explicit values as appropriate, never expose secrets):
   ```powershell
   aws cloudformation deploy `
     --stack-name cafe-elb-stack `
     --template-file cafe-app/cafe-elb.yaml `
     --parameter-overrides `
       VpcId=<vpc-id> `
       PublicSubnet1Id=<public-subnet-1> `
       PublicSubnet2Id=<public-subnet-2> `
       PrivateSubnet1Id=<private-subnet-1> `
       PrivateSubnet2Id=<private-subnet-2> `
       AppSecurityGroupId=<cafe-app-sg-id> `
       InstanceProfileName=cafe-app-role `
       KeyName=cafe-app-nova `
     --capabilities CAPABILITY_NAMED_IAM `
     --region us-east-1
   ```
3. Verify stack outputs and resource health: ALB active, target group healthy, ASG instances InService across 2 AZs, scaling policy attached.
4. Before updating an existing stack, inspect current state, review change set/impact, and get explicit user approval for potentially disruptive changes.

## 5. Workflow

### Phase 1 — Discovery and Validation
- Confirm AWS account/region/identity.
- Inventory existing `cafe-app-vpc` baseline resources and ELB/ASG resources.
- Report what exists, what matches, and what requires creation or change.
- Get user confirmation before changing any existing item.

### Phase 2 — ALB and Target Group
- Validate/create target group in existing VPC.
- Validate/create internet-facing ALB in existing public subnets.
- Configure HTTP listener to forward to target group.
- Validate ALB active state.

### Phase 3 — Launch Template and ASG
- Validate/create launch template using existing IAM/SG/key pair.
- Validate/create ASG in existing private subnets across 2 AZs.
- Attach ASG to target group and enable ELB health checks.
- Set capacity: min 2, desired 2, max 6.
- Configure target tracking scaling policy.

### Phase 4 — RDS Multi-AZ Validation
- Inspect existing `cafe-db`.
- Verify Multi-AZ status, subnet group, SG, and connectivity from app tier.
- If Multi-AZ is not enabled, report impact and obtain explicit approval before modifying.

### Phase 5 — Testing
- Verify two healthy instances across two AZs.
- Test application through ALB DNS name.
- Run approved load test to generate CPU load and observe scaling.
- Confirm new instances register healthy and ALB remains available.
- Include test sample output in validation_elb.csv

## 6. Final Report

Report PASS/FAIL/NOT IMPLEMENTED/BLOCKED/NOT TESTED for:

- Existing VPC/baseline reuse
- ALB configuration and DNS test
- Target group health
- Launch template configuration
- ASG min/desired/max and AZ distribution
- Scaling policy and load-test evidence
- RDS Multi-AZ validation
- Overall end-to-end application availability

Include resource IDs, evidence, and any required approvals. Do not include secrets.

Save report in cafe-app including evidence the test balance loader/auto scale works in validation_elb.csv