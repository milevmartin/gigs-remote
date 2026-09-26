# AWS Target Architecture — European SaaS Platform

This repository contains the target AWS architecture and operating model
designed for a fast-growing European SaaS company migrating from a
fragmented multi-account environment to a governed, scalable platform.

---

## Context

The company operates a mixture of EC2, ECS, Lambda, and Kubernetes with
independently managed databases and networking across multiple AWS accounts
and regions. The environment has grown organically and has accumulated
challenges around security, networking, deployment consistency, observability,
disaster recovery, and cloud costs.

**Key constraints driving the design:**

| Constraint | Requirement |
|---|---|
| Data residency | Customer data must remain within the European Economic Area |
| Availability | Multi-AZ for all critical workloads |
| Recovery Time Objective | ≤ 60 minutes |
| Recovery Point Objective | ≤ 15 minutes |
| Scale | Millions of events per hour today; 10× growth expected within 12 months |
| Governance | Application teams autonomous without losing central control |

---

## Document index

### 1. AWS Estate — Accounts, Environments and Governance
[1-account-setup.md](1-account-setup.md)

AWS Control Tower landing zone, account structure (Management, Security OU,
Infrastructure, Non-Production, Production, DR), AFT account vending pipeline,
SCPs, and governance model.

---

### 2. Networking — VPCs, Connectivity and Traffic Flow
[2-networking.md](2-networking.md)

Transit Gateway hub-and-spoke topology, per-account VPCs, subnet tiers
(public / private / database / intra), route table design, and hub-and-spoke
isolation between environments.

---

### 3. Application Platform — EC2, Containers, Kubernetes and Serverless

| File | Content |
|---|---|
| [3-1-kubernetes.md](3-1-kubernetes.md) | EKS clusters, Karpenter vs Cluster Autoscaler, HPA vs KEDA, add-ons, Envoy Gateway, Linkerd mTLS, Pod Identity |
| [3-2-ec2.md](3-2-ec2.md) | EC2 for stateful workloads, ASGs, ALB placement, IAM instance profiles, Ansible configuration |
| [3-3-ecs.md](3-3-ecs.md) | ECS with Fargate, when to use it, and recommendation relative to EKS |
| [3-4-lambda.md](3-4-lambda.md) | Lambda for event-driven workloads, VPC placement, triggers, IAM execution roles, cold starts |
| [3-5-compute-summary.md](3-5-compute-summary.md) | Decision guide — which compute tier to use for a given workload |

---

### 4. Data & Event Processing

| File | Content |
|---|---|
| [4-1-databases.md](4-1-databases.md) | Aurora PostgreSQL, ElastiCache Redis, DynamoDB, S3 — placement, Multi-AZ, PITR, RPO/RTO summary |
| [4-2-event-streaming.md](4-2-event-streaming.md) | SQS, SNS fanout, Kinesis Data Streams vs MSK, Data Firehose |

---

### 5. Security & Identity
[5-security.md](5-security.md)

IAM Identity Center (human access), IAM roles (machine access), Secrets
Manager, KMS encryption at rest, services enabled by Control Tower by
default, GuardDuty (threat detection), Security Hub (compliance dashboard).

---

### 6. CI/CD and Developer Experience
[6-cicd.md](6-cicd.md)

GitHub as source of truth, self-hosted runners in Shared Services, OIDC
authentication, ECR in Shared Services, ArgoCD per cluster (GitOps),
Ansible for EC2, database access from Shared Services via TGW.

---

### 7. Observability & Operations
[7-observability.md](7-observability.md)

Metrics (CloudWatch + Container Insights + Linkerd golden metrics), logs
(Fluent Bit → CloudWatch Logs), traces (X-Ray + OpenTelemetry), alerting
(CloudWatch Alarms → SNS), Grafana dashboards, optional Dynatrace/New Relic path.

---

### 8. DR Strategy
[8-dr.md](8-dr.md)

Pilot Light pattern in `eu-west-1`, Aurora Global Database, DynamoDB Global
Tables, S3 CRR, Route 53 Failover routing with CloudWatch health check alarm,
GitHub Actions failover workflow (`workflow_dispatch`), failover runbook.

---

### 9. Roadmap and Key Risks
[9-roadmap.md](9-roadmap.md)

Five-phase migration roadmap (Foundation → CI/CD → Data → DR →
Optimisation), phase milestones, and a risk register covering migration
complexity, skill gaps, cost overrun, EEA data residency, and DR readiness.

---

## Architecture principles

- **Account-per-environment** — blast-radius containment; a failure in
  non-production cannot reach production.
- **Infrastructure as code everywhere** — all resources provisioned by
  Terraform via AFT; no manual console changes.
- **No long-lived credentials** — human access via IAM Identity Center
  (federated, short-lived); machine access via IAM roles (Pod Identity,
  instance profiles, execution roles); OIDC for CI/CD.
- **Encrypt everything at rest** — KMS CMK on every resource that supports
  it; enforced by SCP.
- **EEA data residency enforced by SCP** — non-EEA regions blocked at the
  organisation level; no workload can be created outside `eu-central-1`
  and `eu-west-1`.
- **Build once, promote by tag** — container images built once in Shared
  Services ECR and promoted from Dev → Staging → Prod without rebuilding.
- **GitOps for Kubernetes** — ArgoCD is the only path to cluster state
  changes; CI runners never call `kubectl` directly.
