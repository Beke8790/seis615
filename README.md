## Purpose
This repo (seis615) is an AWS Cafe application deployment and high-availability workspace. It contains infrastructure-as-code (CloudFormation), operational “skill” playbooks, Cafe web/database artifacts, 
and validation reports for importing the app to EC2/RDS and extending it with an ALB + Auto Scaling architecture.

This document summarizes the high-level workflow for deploying, integrating, and validating the Cafe application infrastructure in this repository.

## Process Flow

```mermaid
flowchart TD
    A[Start: Review skills & artifacts] --> B[Preflight / Discovery]
    B --> B1[Check AWS identity, account, region]
    B --> B2[Inventory local artifacts in cafe-app/]
    B --> B3[Inspect existing AWS resources]

    B1 --> C[Deploy/Validate Baseline Infrastructure<br/>cafe-infra.yaml]
    B2 --> C
    B3 --> C

    C --> C1[VPC, public/private subnets, IGW/NAT, route tables]
    C --> C2[Security groups: app SG, DB SG]
    C --> C3[IAM role/instance profile for EC2]
    C --> C4[EC2 cafe-app in public subnet]
    C --> C5[Private RDS MariaDB, Multi-AZ enabled]

    C4 --> D[Deploy Web App to EC2]
    D --> D1[SSH/SCP artifacts: setup.zip, cafe.zip, aws.zip, aws.phar]
    D --> D2[Install Apache/PHP/MariaDB client]
    D --> D3[Extract/configure app files in /var/www/html]

    C5 --> E[Import Database to RDS]
    D1 --> E
    E --> E1[Stage db.zip on EC2]
    E --> E2[Retrieve DB password from Secrets Manager /cafe/dbPassword]
    E --> E3[Import SQL into private RDS from EC2]

    D3 --> F[Integrate App with RDS]
    E3 --> F
    F --> F1[Configure app to use RDS endpoint]
    F --> F2[Use Secrets Manager for credentials]
    F --> F3[Restart/verify web service]

    F --> G[Deploy/Validate ELB & HA Layer<br/>cafe-elb.yaml]
    G --> G1[ALB security group + listener]
    G --> G2[Target group + internet-facing ALB]
    G --> G3[Launch template for app instances]
    G --> G4[ASG in private subnets<br/>Min 2 / Desired 2 / Max 6]
    G --> G5[Target tracking CPU scaling policy]

    G --> H[Testing & Validation]
    H --> H1[Infra checks: EC2, RDS, SGs, IAM]
    H --> H2[App test via ALB DNS / HTTP]
    H --> H3[DB connectivity & app data checks]
    H --> H4[Load test and ASG scaling evidence]

    H --> I[Generate Validation Reports]
    I --> I1[validation_imports.csv]
    I --> I2[validation_elb.csv]
    I --> I3[validation_overall_execution_result.csv]
```

## Key Artifacts

- **Skills/playbooks**: `SKILL_IMPORTS.md`, `SKILL_ELB.md`, `SKILL_orchestrator.md`, `cafe-app/SKILL_IMPORTS.md`, `cafe-app/SKILL_ELB.md`
- **Infrastructure templates**: `cafe-infra.yaml`, `cafe-elb.yaml`, `cafe-app/cafe-infra.yaml`, `cafe-app/cafe-elb.yaml`
- **Application artifacts**: `cafe-app/web_app/` (web app, SDK, setup scripts), `cafe-app/db_app/db.zip`
- **Validation outputs**: `validation_imports.csv`, `validation_elb.csv`, `validation_overall_execution_result.csv`

**Top-Level Objects**

<img width="1031" height="337" alt="image" src="https://github.com/user-attachments/assets/81f95a2a-2ad5-498d-bb77-9858f07baae5" />

**cafe-app/ Contents**

<img width="1031" height="337" alt="image" src="https://github.com/user-attachments/assets/d2d2339c-a28c-4375-aff7-befff1c1c88e" />
# SEIS615 Cafe App - High-Level Process Flow

