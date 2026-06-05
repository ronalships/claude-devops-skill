---
name: devops-best-practices
description: Opinionated production-grade DevOps defaults for Terraform, Kubernetes, CI/CD, Docker, cloud security, observability, cost optimization, and disaster recovery. ALWAYS use this skill when generating, reviewing, or modifying any infrastructure code, Kubernetes manifests (Deployment, Service, StatefulSet, Helm chart, Kustomize), Terraform (.tf files, modules, state config), Dockerfiles, docker-compose, CI/CD pipelines (.github/workflows, .gitlab-ci.yml, Jenkinsfile), cloud resources (AWS/GCP/Azure), IAM policies, security groups, network configs, observability setup (Prometheus, Grafana, OpenTelemetry), shell deployment scripts, systemd units, or DNS/TLS/CDN config — even if the user does not explicitly ask for "best practices." This skill prevents the failure modes that hurt production teams most often: missing PDBs, single replicas in prod, `:latest` tags, public S3 buckets, long-lived credentials, missing observability, untested backups, unbounded costs, and CI/CD supply-chain risks. Apply opinionated defaults by default; surface tradeoffs when the user has reason to deviate.
---

# DevOps Best Practices

This skill encodes opinionated, production-grade DevOps defaults. Apply them whenever generating or reviewing infrastructure code. When the user's request conflicts with a default below, surface the conflict and explain the tradeoff — don't silently override.

These are **opinionated**. Other valid approaches exist. The opinions here are chosen because they prevent the failure modes that hurt teams most often in real production environments.

---

## When to use this skill

Trigger whenever the task involves any of:

- Terraform files (`*.tf`, `*.tfvars`), Terragrunt, Pulumi, CDK
- Kubernetes manifests (Deployment, Service, Ingress, StatefulSet, etc.), Helm charts, Kustomize overlays
- Dockerfiles, `docker-compose.yml`
- CI/CD config (`.github/workflows/*.yml`, `.gitlab-ci.yml`, Jenkinsfile, CircleCI, Buildkite)
- Cloud provider SDKs or CLIs (AWS, GCP, Azure)
- IAM policies, security groups, network ACLs
- Observability config (Prometheus, Grafana, OpenTelemetry, Datadog, CloudWatch)
- Shell scripts deployed to servers (`/etc/init.d`, systemd units, deploy scripts)
- DNS, TLS, CDN configuration

If unsure, default to applying the safety and security sections (they almost never hurt).

---

## Foundational principles (apply to everything)

1. **Default to safety over convenience.** A slightly harder UX that prevents production incidents wins.
2. **Default to least privilege.** Start with zero permissions and add only what the workload demonstrably needs.
3. **Make failure modes loud.** Silent failures destroy trust. Logs > swallowed errors. Alerts > "we'll notice eventually."
4. **Cost is a non-functional requirement.** Generated infra that costs 10x what it should is a bug.
5. **Reproducibility beats cleverness.** If a teammate can't recreate this environment from the repo in 30 minutes, the design is wrong.
6. **Document tradeoffs inline.** If a non-obvious choice was made, leave a comment explaining why so future-you (or future-Claude) doesn't undo it.

---

## Terraform

### Module structure

- One module per logical resource grouping. Modules in `modules/<name>/`. Roots in `environments/<env>/`.
- Always pin module versions and provider versions. Floating versions break reproducibility.
- Use `terraform-aws-modules/*` community modules where available — they're battle-tested and avoid common mistakes.
- Never put state in local files. Use S3 backend with DynamoDB locking (AWS) or GCS backend (GCP). Always.
- One state file per environment. Never share state across dev/staging/prod.

```hcl
# Good
terraform {
  required_version = "~> 1.7"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.30"
    }
  }
  backend "s3" {
    bucket         = "company-tfstate-prod"
    key            = "vpc/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "tfstate-lock"
    encrypt        = true
  }
}
```

### Variable hygiene

- Every variable has a `description` and a `type`. No exceptions.
- Sensitive variables marked with `sensitive = true`.
- Defaults only for genuinely safe values. Don't default an environment name to `"prod"`.
- Use `validation` blocks for inputs with constraints (e.g., instance type must be in approved list).

### Resource naming

- Consistent naming pattern: `{project}-{env}-{purpose}` (e.g., `acme-prod-api-eks`).
- Tag every resource: `Environment`, `Project`, `Owner`, `CostCenter`, `ManagedBy=terraform`.

### State management

- Run `terraform plan` in CI on every PR. Block merge if plan fails.
- `terraform apply` runs only from CI on merge to main, never from a developer's laptop.
- Use Atlantis, Spacelift, or Terraform Cloud for plan/apply automation.
- Never use `-auto-approve` outside of automated CI/CD with strict guardrails.

### What NOT to do

- ❌ Hardcoded credentials, ARNs, or account IDs in `.tf` files — use variables or data sources
- ❌ `local-exec` provisioners — they break reproducibility; use proper providers
- ❌ One giant `main.tf` — split by resource type or concern
- ❌ Manual changes in the cloud console that aren't reflected in code — drift kills you

---

## Kubernetes

### Workload defaults

Every Deployment / StatefulSet must have:

```yaml
spec:
  replicas: 3  # never 1 in production; minimum 2 for HA, 3 for quorum-based systems
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534  # nobody
        fsGroup: 65534
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: app
          image: registry.example.com/app:sha-abc123  # always a SHA tag, never :latest
          imagePullPolicy: IfNotPresent
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              memory: 256Mi  # set memory limit; do NOT set CPU limit (causes throttling)
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
          startupProbe:
            httpGet:
              path: /healthz
              port: 8080
            failureThreshold: 30
            periodSeconds: 10
```

### Required accompanying resources

For every Deployment, also create:

1. **Service** — even if internal-only
2. **PodDisruptionBudget** — `minAvailable: 1` or `maxUnavailable: 1` based on replicas; required for safe rolling updates and node draining
3. **NetworkPolicy** — default-deny ingress/egress, then allow what's needed
4. **HorizontalPodAutoscaler** — even with conservative limits, prevents single-replica outages under load
5. **ServiceMonitor** (if using Prometheus Operator) — observability is not optional

### Helm vs Kustomize vs raw YAML

**Use Helm.** For everything non-trivial. Reasons:

- Versioned releases with rollback (`helm rollback`)
- Templating handles environment differences cleanly
- De-facto standard — every operator and tool publishes a Helm chart
- Helm 3 has no Tiller (the security objection from Helm 2 is dead)

Use Kustomize only for simple overlays on top of upstream YAML you don't control.

Never use raw YAML for production deployments — no rollback story, no diff, no upgrade safety.

### Namespaces

- One namespace per application or tightly-coupled group
- Apply ResourceQuotas and LimitRanges per namespace
- Never deploy to `default` namespace in production

### Cluster-level

- Use **managed K8s** (EKS, GKE, AKS) — self-managed K8s is not worth the operational burden in 2026
- Enable cluster-level features: audit logging, encryption at rest, private endpoints
- Use **Karpenter** (AWS) or **Cluster Autoscaler** for node management. Karpenter is the default choice in 2026.
- Install: cert-manager (TLS), external-dns (DNS automation), metrics-server (HPA), Prometheus stack, OPA Gatekeeper or Kyverno (policy)

### What NOT to do

- ❌ `:latest` tags in production — always immutable SHAs or semver-pinned versions
- ❌ CPU limits — they cause throttling that's worse than the original problem; set requests, not limits
- ❌ Single replicas in production — minimum 2 for stateless, 3 for stateful
- ❌ Missing readiness probes — without them, traffic flows to unready pods during deploys
- ❌ `hostNetwork: true` or `hostPID: true` — only for system components, never apps
- ❌ Privileged containers — almost never needed; if you think you need it, you probably don't
- ❌ Sharing service accounts across workloads — one SA per workload, least privilege

---

## CI/CD

### GitHub Actions defaults

Every workflow:

- Pin actions to SHA, not version tag: `uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11`
- Set explicit `permissions:` block at the top — default to read-only, grant write only where needed
- Use OIDC for cloud auth (`aws-actions/configure-aws-credentials` with `role-to-assume`), never long-lived access keys
- Pin runner versions: `runs-on: ubuntu-22.04`, not `ubuntu-latest`
- Cache dependencies: `actions/setup-node@...` with `cache: 'npm'`, etc.

```yaml
name: CI
on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-22.04
    permissions:
      id-token: write   # only this job, only this scope
      contents: read
    steps:
      - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11
      - uses: actions/setup-node@1e60f620b9541d16bece96c5465dc8ee9832be0b
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm test
```

### Secret handling

- Never commit secrets. Use the platform's secret store: GitHub Secrets, AWS Secrets Manager, GCP Secret Manager, HashiCorp Vault
- For CI access to cloud: OIDC > short-lived assumed roles > long-lived access keys
- Rotate any leaked secret immediately, even if "the repo is private"
- Scan PRs for secrets with `gitleaks` or `trufflehog` in CI

### Pipeline structure

- **Fast feedback first**: lint → unit tests → build → integration tests → deploy
- **Parallel where possible**: matrix builds, parallel test shards
- **Cache aggressively**: node_modules, .gradle, Docker layers, Go modules — these are 90% of CI time
- **One pipeline per repo**: monorepo pipelines should detect what changed and run only relevant jobs

### Deploy gates

- Production deploys require: passing tests, security scan, manual approval (for high-stakes changes), and a rollback plan
- Use deployment strategies: blue/green, canary, or progressive (Argo Rollouts, Flagger)
- Always include a rollback step in the pipeline — not just hope-based

### What NOT to do

- ❌ `actions/checkout@v4` (floating tag) — pin to SHA to avoid supply-chain attacks
- ❌ `${{ github.event.pull_request.title }}` interpolation in `run:` blocks — RCE via PR title attack
- ❌ Long-lived AWS access keys in CI — use OIDC
- ❌ Deploying directly from a developer laptop — always through CI
- ❌ Running CI on `pull_request_target` without strict guardrails — exposes secrets to forked PRs

---

## Docker

### Dockerfile defaults

```dockerfile
# Multi-stage builds: build artifacts in one stage, copy to slim runtime
FROM golang:1.22-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /out/app ./cmd/app

# Runtime: minimal base
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

Rules:

- Multi-stage builds for compiled languages (Go, Rust, Java)
- Distroless or Alpine base images, not full OS
- Non-root user always (`USER nonroot` or numeric UID)
- Specific version tags for base images (`alpine:3.19`, not `alpine:latest`)
- Use `.dockerignore` aggressively — secrets, `.git`, `node_modules`, test fixtures
- Cache layers: order Dockerfile from least-frequently-changed to most-frequently-changed
- Sign and scan images: `cosign` for signing, `trivy` or `grype` for scanning

### What NOT to do

- ❌ Running as root in container — use `USER nonroot`
- ❌ `apt-get update && apt-get install -y` without cleanup — bloats image; combine with `&& rm -rf /var/lib/apt/lists/*`
- ❌ Storing secrets in image layers (env vars in `ENV`, files in `COPY`) — they persist in image history forever
- ❌ `COPY . .` early in the Dockerfile — invalidates cache on every code change

---

## Cloud security

### IAM (universally applies across AWS, GCP, Azure)

- **Least privilege.** Start with zero permissions, add only what's demonstrably needed.
- **No wildcards in production policies.** `"Action": "*"` and `"Resource": "*"` are red flags.
- **Use roles, not users.** Humans assume roles via SSO. Workloads use IAM Roles for Service Accounts (AWS IRSA) or Workload Identity (GCP).
- **Rotate any long-lived credentials.** Long-lived access keys are an anti-pattern. Prefer short-lived assumed roles.
- **MFA required** for any human access to production accounts.

### Networking

- **Private subnets by default.** Public subnets only for resources that must accept inbound from the internet (ALB, NAT gateway).
- **Security groups: least-permissive ingress.** No `0.0.0.0/0:22` ever. SSH via SSM Session Manager or Tailscale, not public SSH.
- **VPC endpoints for AWS service access** — avoid sending traffic through the public internet to reach AWS services within the same region.
- **TLS everywhere.** Internal traffic too. Use service mesh (Istio, Linkerd) or sidecar TLS.

### Secrets and data

- **Encryption at rest by default** for storage (S3, EBS, RDS, etc.). It's free in 2026; there's no excuse.
- **Encryption in transit** — TLS 1.3 where supported, TLS 1.2 minimum.
- **Secrets in a secret manager** (AWS Secrets Manager, GCP Secret Manager, Vault). Never in env vars committed to source.
- **KMS-managed keys** for sensitive workloads. Audit key access via CloudTrail equivalents.

### What NOT to do

- ❌ S3 buckets with public read by default — explicit opt-in only, audit regularly
- ❌ IAM user with admin access for daily use — assume roles instead
- ❌ Storing terraform state unencrypted in S3 — `encrypt = true` is one line
- ❌ Allowing 0.0.0.0/0 in security groups for non-internet-facing resources
- ❌ Disabling CloudTrail/CloudWatch to "save money" — you need the audit trail when things go wrong

---

## Observability

### The three pillars

Every production workload must emit:

1. **Logs** — structured JSON, single line per log event, sent to a central log aggregator
2. **Metrics** — Prometheus format, exposed on `/metrics`, scraped by Prometheus or compatible collector
3. **Traces** — OpenTelemetry SDK, traces sent to a backend (Jaeger, Tempo, Honeycomb, Datadog)

### Logging

- Structured logs only (JSON). No printf-style unstructured logs in production.
- Required fields: `timestamp`, `level`, `service`, `trace_id`, `span_id`, `message`
- Log levels: ERROR (real problems), WARN (degraded but working), INFO (significant events), DEBUG (off in production)
- Never log: passwords, tokens, PII, full credit card numbers, full request/response bodies of sensitive endpoints
- Use OpenTelemetry's logging SDK to correlate logs with traces automatically

### Metrics

- Use Prometheus naming conventions: `metric_name_unit{labels}`
- Histograms for latency, counters for events, gauges for instantaneous values
- Cardinality discipline: avoid high-cardinality labels (user IDs, request IDs) — they explode metric storage
- The Four Golden Signals: latency, traffic, errors, saturation — every service must expose these

### Alerting

- Alert on symptoms, not causes (alert on "users seeing errors", not "CPU at 80%")
- Every alert links to a runbook (`docs/runbooks/<alert-name>.md`)
- Alert fatigue is real: if an alert fires more than once a week and isn't actioned, fix it or delete it
- Use SLO-based alerting (burn-rate alerts) for user-facing services

### What NOT to do

- ❌ Unstructured text logs that can't be queried
- ❌ Logging at INFO level for every request — flood your aggregator and your bill
- ❌ Alerts without runbooks — on-call wakes up at 3am with no idea what to do
- ❌ Sampling all traces — sample, but keep error traces and slow traces at 100%

---

## Cost optimization

### Compute

- **Right-size first.** Most workloads are over-provisioned by 2-5x. Use Vertical Pod Autoscaler in recommendation mode for K8s; check CloudWatch metrics for EC2.
- **Spot instances for fault-tolerant workloads.** Batch jobs, dev/staging clusters, stateless web tiers with multiple replicas. Use Karpenter to manage spot pools safely.
- **Reserved Instances or Savings Plans** for steady-state baseline load. Commit to 1-year if usage is predictable; 3-year only if very confident.
- **Auto-scale aggressively in non-prod.** Scale dev clusters to zero overnight and on weekends — `kube-green`, Karpenter consolidation, scheduled scaling.

### Storage

- **Lifecycle policies for object storage.** S3 → Glacier after N days, delete logs after retention period. Cost compounds.
- **Right-size EBS volumes.** Default to gp3 (cheaper and faster than gp2 in 2026). Resize down or delete unused.
- **Compress logs and metrics at rest.** Default compression in modern log aggregators.

### Network

- **VPC endpoints to avoid NAT gateway costs.** NAT charges per GB; endpoints are free for in-region S3/DynamoDB.
- **CloudFront / CDN for egress-heavy workloads.** Edge cache reduces origin egress.
- **Same-AZ where possible** for chatty services. Cross-AZ data transfer charges add up.

### Visibility

- **Cost allocation tags** on every resource. Without tags, you can't attribute spend.
- **Daily cost anomaly detection** (AWS Cost Anomaly Detection, GCP Recommender). Catches surprise spend.
- **Per-team chargeback** with Kubecost or OpenCost in Kubernetes environments.

### What NOT to do

- ❌ Running 24/7 dev clusters that nobody uses on weekends
- ❌ EBS volumes attached to terminated instances ("orphaned volumes") — script regular cleanup
- ❌ Old EBS snapshots accumulating forever — lifecycle policy
- ❌ NAT gateway routing for traffic that could use a VPC endpoint
- ❌ Ignoring Cost Explorer / Billing alerts — set a monthly budget alert at $X, $2X, $5X

---

## Disaster recovery

### Backups

- **Define RPO and RTO** for every system. Without targets, "we have backups" is meaningless.
- **Test restores quarterly.** Untested backups don't exist.
- **3-2-1 rule**: 3 copies, 2 different media, 1 off-site. Modern translation: production data, snapshot in same region, replicated copy in different region.
- **Application-consistent backups** for databases (not just disk snapshots).

### Runbooks

- Every critical system has a runbook in `docs/runbooks/<system>.md`
- Runbook structure: detection → diagnosis → mitigation → resolution → postmortem prompt
- Runbooks are tested, not aspirational — game day exercises catch the gaps

### Chaos engineering

- For high-availability systems, regular chaos drills: kill pods, drain nodes, simulate AZ failure
- Tools: Chaos Mesh, Litmus (K8s), AWS Fault Injection Simulator
- Start with non-production. Move to production only when team is mature.

---

## Common antipatterns (refuse to generate without surfacing the issue)

When the user asks for any of these, explain the risk before generating:

- Public S3 bucket — confirm intent, suggest pre-signed URLs or CloudFront instead
- Security group with 0.0.0.0/0 on SSH/RDP — suggest SSM Session Manager or VPN
- Long-lived AWS access keys in CI — suggest OIDC
- Kubernetes Deployment with `replicas: 1` for production — flag the SPOF risk
- `:latest` image tag — flag the rollback impossibility
- Terraform `local-exec` for cloud resource creation — suggest proper providers
- Disabling audit logs to "reduce noise" — flag the incident-response cost
- Custom encryption implementation — never; use platform KMS
- `chmod 777` in scripts — explain why and suggest specific permissions

---

## When generating infra code

1. Apply the relevant defaults from above
2. Add inline comments where a non-obvious choice was made
3. Include a brief "production checklist" comment at the top of generated files listing what the user still needs to verify
4. If the user's environment is unclear (dev vs prod, scale, compliance requirements), ask before generating

Example top-of-file comment for a generated production manifest:

```yaml
# Production checklist:
# [ ] Verify resource requests/limits match actual workload profile
# [ ] Confirm liveness/readiness probe endpoints exist in the app
# [ ] Apply NetworkPolicy (separate file)
# [ ] Configure HorizontalPodAutoscaler (separate file)
# [ ] Set up ServiceMonitor for Prometheus scraping
# [ ] Document runbook for this service
```

---

## What this skill is NOT

- Not a tutorial — assumes basic familiarity with the tools
- Not exhaustive — covers the failure modes that hurt teams most, not every possible best practice
- Not opinionated about everything — silent on choices where the tradeoff is genuinely subjective (e.g., Go vs Rust for tooling)
- Not a replacement for thinking — defaults are starting points, not unchangeable rules

When a user has a specific reason to deviate, support them. The defaults exist to prevent the common mistakes, not to suppress valid context-specific decisions.
