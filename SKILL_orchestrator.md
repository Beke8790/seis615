## Purpose
A single-invocation AWS deployment orchestrator that executes both AI agent skills sequentially, validates every requirement, and produces a consolidated deployment and verification report.

Important design requirement: The orchestrator must be idempotent. If a resource already exists, it should inspect and validate it rather than blindly recreate it. It should record whether each resource was EXISTING, CREATED, UPDATED, REUSED, FAILED, or NOT VERIFIED.
Execution should be
ONE COMMAND / ONE AGENT INVOCATION
Deploy Café Application

### Step 1 — skill_imports.md
Execute application and database imports using cafe-elb.yaml where applicable.
Verify EC2, RDS, database connectivity, application files, configuration and IAM permissions.

### Step 2 — skill_elb.md
Execute Elastic Load Balancing configuration using cafe-elb.yaml.
Verify target groups, load balancer, listeners, health checks, security groups and application availability.

### Step 3 — Final verification and audit
Verify all resources and dependencies, capture pre-existing resources, identify missing resources, and produce a consolidated execution report.
report name = validation_overall_execution_result.csv

Required final report
The orchestrator should generate a report similar to this:

| Phase/resource | Status |	Action |

| Application EC2 | EXISTING |	Reused |

It must also report:
- Resources discovered before deployment.
- Resources created or modified.
- Resources reused.
- Failed or skipped requirements.
- Actual verification evidence.
- Final application URL.
- Outstanding issues.

Orchestrator execution rules
The agent should follow these rules:
1. Read both skill files and the YAML before making changes.
2. Build a checklist of every requirement in each skill.
3. Inspect AWS before creating anything.
4. Reuse existing resources when their configuration is compliant.
5. Create missing resources using CloudFormation and AWS APIs.
6. Correct noncompliant configurations where safe.
7. Wait for asynchronous AWS operations to complete.
8. Verify each requirement with actual AWS CLI/Boto3 output.
9. Stop Step 2 if Step 1 has unresolved critical failures.
10. Never report success based solely on a successful CloudFormation deployment.


