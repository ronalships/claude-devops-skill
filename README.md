# claude-devops-skill

> Opinionated, production-grade DevOps defaults for Claude Code — so generated infrastructure is right by default, not just "technically correct."

<p align="center">
  <a href="#install">Install</a> ·
  <a href="#what-it-does">What it does</a> ·
  <a href="#coverage">Coverage</a> ·
  <a href="#philosophy">Philosophy</a> ·
  <a href="#contributing">Contributing</a>
</p>

---

Claude Code is excellent at writing code. But infrastructure code has a particular failure mode: it can be **technically correct yet missing every senior-engineer instinct** that prevents 3am pages. A Deployment without a PDB. A `:latest` tag in production. An S3 bucket that's public by accident. A CI pipeline with a long-lived AWS access key.

This skill fixes that. It encodes the opinionated defaults that experienced DevOps engineers apply without thinking, and injects them into every Claude Code session that touches infrastructure.

## What it does

When Claude Code detects you're working on infrastructure — Terraform, Kubernetes manifests, Dockerfiles, CI/CD pipelines, cloud resources, IAM policies, observability config — this skill triggers automatically and applies production-grade defaults:

- ✅ Pinned versions, not floating tags
- ✅ Least-privilege IAM by default
- ✅ PodDisruptionBudgets on every Deployment
- ✅ Encryption at rest and in transit
- ✅ Structured logging with trace correlation
- ✅ Resource limits sized for real workloads, not copy-pasted defaults
- ✅ Observability built in, not bolted on later
- ✅ Cost-aware patterns (right-sizing, lifecycle policies, spot where safe)
- ✅ Reversible, auditable, reproducible

And it actively flags antipatterns before generating risky code:

- 🚫 Public S3 buckets, security groups open to `0.0.0.0/0`
- 🚫 Single-replica production Deployments
- 🚫 `:latest` image tags
- 🚫 Long-lived AWS access keys in CI
- 🚫 Missing readiness probes
- 🚫 Floating action versions in GitHub workflows (supply-chain risk)
- 🚫 Privileged containers
- 🚫 CPU limits (causes throttling worse than the original problem)

## Install

### Option A — Claude Code (recommended)

```bash
# Clone into your Claude Code skills directory
git clone https://github.com/ronalships/claude-devops-skill ~/.claude/skills/devops-best-practices
```

Claude Code automatically discovers skills in this directory. No restart needed.

### Option B — Project-local

If you want the skill scoped to a single project:

```bash
mkdir -p .claude/skills
git clone https://github.com/ronalships/claude-devops-skill .claude/skills/devops-best-practices
```

Then check `.claude/skills/` into your repo so the whole team benefits.

### Option C — Curl install

```bash
mkdir -p ~/.claude/skills/devops-best-practices
curl -fsSL https://raw.githubusercontent.com/ronalships/claude-devops-skill/main/SKILL.md \
  -o ~/.claude/skills/devops-best-practices/SKILL.md
```

## Coverage

| Domain | What's covered |
|---|---|
| **Terraform** | Module structure, state management, variable hygiene, naming conventions, provider versioning, drift prevention |
| **Kubernetes** | Workload defaults (probes, security contexts, resource limits), PDBs, NetworkPolicies, HPAs, namespace hygiene, cluster components |
| **CI/CD** | GitHub Actions security, OIDC over long-lived keys, action pinning, secret handling, deploy gates, rollback strategies |
| **Docker** | Multi-stage builds, distroless bases, non-root containers, image scanning, layer caching, signing |
| **Cloud security** | Least-privilege IAM, network segmentation, TLS everywhere, secret managers, encryption defaults, MFA, audit logging |
| **Observability** | Structured logging, Prometheus metrics, OpenTelemetry traces, SLO-based alerting, runbook discipline |
| **Cost** | Right-sizing, Reserved Instances vs Spot, storage lifecycle, network cost avoidance, cost allocation tags |
| **Disaster recovery** | Backup strategies, RPO/RTO definition, restore testing, runbook structure, chaos engineering |

## Philosophy

This skill is **opinionated**. Other valid approaches exist. The opinions here are chosen because they prevent the failure modes that hurt teams most often in real production environments.

Six guiding principles:

1. **Default to safety over convenience.** A slightly harder UX that prevents production incidents wins.
2. **Default to least privilege.** Start with zero permissions and add only what the workload demonstrably needs.
3. **Make failure modes loud.** Silent failures destroy trust.
4. **Cost is a non-functional requirement.** Generated infra that costs 10x what it should is a bug.
5. **Reproducibility beats cleverness.** If a teammate can't recreate this environment from the repo in 30 minutes, the design is wrong.
6. **Document tradeoffs inline.** If a non-obvious choice was made, leave a comment explaining why.

When your context has a legitimate reason to deviate from a default, tell Claude — the skill supports overrides explicitly. The defaults exist to prevent the common mistakes, not to suppress valid context-specific decisions.

## Examples

**Without the skill:**

```yaml
# Claude generates this when you say "give me a Deployment for my web app"
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: app
          image: myapp:latest
```

**With the skill:**

```yaml
# Production checklist:
# [ ] Verify resource requests/limits match actual workload profile
# [ ] Confirm liveness/readiness probe endpoints exist in the app
# [ ] Apply NetworkPolicy (separate file)
# [ ] Configure HorizontalPodAutoscaler (separate file)
# [ ] Set up ServiceMonitor for Prometheus scraping
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 3
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
        fsGroup: 65534
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: app
          image: registry.example.com/myapp:sha-abc123
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
              memory: 256Mi  # no CPU limit — throttling is worse than over-use
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 30
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8080
---
# PDB is required for safe rolling updates and node draining
apiVersion: policy/v1
kind: PodDisruptionBudget
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: myapp
```

Same task. Different outcome. That's the entire pitch.

## Roadmap

- [x] v0.1 — broad coverage, single SKILL.md
- [ ] v0.2 — split into reference files (one per domain) for progressive disclosure
- [ ] v0.3 — Pulumi, CDK, and Crossplane-specific patterns
- [ ] v0.4 — language-specific bundled scripts (Helm chart linting, Terraform module validation)
- [ ] v0.5 — observability backends (Datadog, New Relic, Honeycomb) specific patterns

## Contributing

This skill encodes opinions. Some of those opinions you'll disagree with — that's healthy. Open an issue with:

- The current opinion you're challenging
- The context where it's wrong
- The alternative you'd recommend

PRs welcome for:

- New antipatterns to flag
- Additional cloud providers or tools
- Better examples
- Coverage gaps

## License

[MIT](LICENSE)

---

Part of a series of tools that make developer pain visible and fixable. See also: [envsnap](https://github.com/ronalships/envsnap), [ghactrace](https://github.com/ronalships/ghactrace).

Built by [@RonalRaj4](https://x.com/RonalRaj4). If this skill saved your team from a 3am page, a ⭐ on the repo is the best way to say thanks.
