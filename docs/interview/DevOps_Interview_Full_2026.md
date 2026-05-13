# DevOps Interview Preparation Guide

**Zen Pharma Project · 5-6 Years Experience · 2026 Edition**

Each question contains **three elements:**
- **Version A** — Candidate voice: conversational, first-person, as spoken in an interview
- **Version B** — Structured reference: key technical bullet points for preparation
- **Diagram** — Architecture or flow diagram where relevant

**Stack:** AWS EKS 1.33 · Terraform 1.10+ · GitHub Actions · ArgoCD · Helm
Prometheus · Grafana · Trivy · Cosign · External Secrets Operator
Spring Boot · Node.js · PostgreSQL 15.7 · Docker

**36 Questions · 9 Sections · Both Answer Styles · Architecture Diagrams**

---

## Table of Contents

1. [Project Overview & Architecture](#01--project-overview--architecture)
2. [CI/CD Pipeline](#02--cicd-pipeline)
3. [GitOps & ArgoCD](#03--gitops--argocd)
4. [Security & DevSecOps](#04--security--devsecops)
5. [Kubernetes](#05--kubernetes)
6. [Terraform & Infrastructure](#06--terraform--infrastructure)
7. [Monitoring & Observability](#07--monitoring--observability)
8. [Behavioural & Situational](#08--behavioural--situational)
9. [Advanced 2026 Topics](#09--advanced-2026-topics)

---

## 01 — Project Overview & Architecture

### Q1 — Can you walk me through your project end-to-end?

**Version A — Candidate Voice**

So the project is a pharma platform — a microservices-based application deployed on AWS. Let me break it down from infrastructure to deployment.

**Infra:** Infrastructure is deployed and managed via Terraform. Terraform state is in S3 with native locking — Terraform 1.10 feature, no DynamoDB needed. In our project, we use a custom VPC, self-managed EKS cluster (1.33) with t3.small node groups (for dev/stg) and t3.medium (for pre-prod and prod), RDS PostgreSQL DB 15.x, ECR repos for each microservice. The terraform plan/apply automation is done using GitHub Actions. When the infra code has changed on a specific branch, the infra is deployed in the specific environment/AWS account. (Consider you have multiple AWS accounts for each environment.)

**Application deployment:** On the CI side, GitHub Actions runs on every push based on the branch. Developers create feature branches, do their changes which runs a lightweight 5-minute check — SAST, tests, no Docker. When code merges to develop branch, pipeline runs build and deploy for the dev environment. Build stages contain Gitleaks, Maven, CodeQL, Semgrep (SAST), OWASP (dependency check), Docker build, Trivy image scan, ECR push, and Cosign signing. Once the image is built, it's pushed to ECR. AWS auth is via OIDC throughout — no static AWS credentials stored anywhere. After the image is pushed, the corresponding image tag is updated in the k8s deployment/rollout spec.

On the CD side is GitOps with ArgoCD. GitHub Actions never runs kubectl. It commits the image tag to zen-gitops. ArgoCD watches that repo and syncs the cluster. If someone manually changes something in the cluster, ArgoCD detects the drift and reverts it within 3 minutes. Git is the single source of truth.

We have dev, QA and prod as separate Kubernetes namespaces in separate EKS clusters. Secrets for the application come from AWS Secrets Manager via External Secrets Operator with IRSA. Monitoring is the full kube-prometheus-stack — Prometheus, Grafana, Alertmanager.

**Version B — Structured Reference**

- **Infrastructure:** Terraform provisions VPC, EKS 1.33 (t3.small, 1-4 nodes), RDS PostgreSQL 15.7 (private subnet), 8 ECR repos. S3 remote state + S3 native locking.
- **CI:** Feature branches = 5 min lightweight (Gitleaks, tests, CodeQL). develop = 15 min full (+ Docker, Trivy, ECR push, Cosign). AWS OIDC auth, no static keys.
- **GitOps:** CI commits image tag to zen-gitops. ArgoCD syncs. `selfHeal: true` auto-reverts drift. Every deployment = git commit.
- **Environments:** dev, qa, prod as Kubernetes namespaces. Different EKS cluster, separate isolation.
- **Secrets:** AWS Secrets Manager + ESO + IRSA. Never in Git. Auto-refresh every 1h.
- **Monitoring:** kube-prometheus-stack in monitoring namespace. Spring Actuator exposes `/actuator/metrics/prometheus`.

**Repos:** Four repositories drive everything:
- `zen-infra` — all Terraform code
- `zen-pharma-frontend` — frontend microservice code
- `zen-pharma-backend` — 8 microservices and GitHub Actions CI
- `zen-gitops` — Helm values and ArgoCD Application manifests (the GitOps repo)

**Branching strategy:**
- Feature branches — developer's code test
- Develop branch — developer's changes deployed in dev environment
- Main branch — production/live application code lives

**Deployment hierarchy:**
- Source code: Develop → PR merges/push → Image tag update in dev values → Deploy to dev
- GitOps code: Image tag update in staging/pre-prod values → Deploy to stg
- GitOps code: Image tag update in prod values → Deploy to prod
- Everything is fine in prod → Code merge from develop to main branch

**Diagram — CI/CD and GitOps Flow**

```
GitHub Actions CI (9 stages on develop)
         │
         ▼
zen-gitops: image.tag: sha-a1b2c3d updated
         │
         ▼
    ArgoCD (polls every 3 min)
         │
    ┌────┼────────────┐
    ▼    ▼            ▼
 dev ns  qa ns     prod ns
auto-sync PR gate  manual sync
    │
    └── AWS VPC 10.0.0.0/16
        Public:  NLB · NAT GW · NGINX Ingress
        Private: EKS nodes · RDS 5432
```

---

### Q2 — Walk me through the architecture

**Version A — Candidate Voice**

The AWS VPC has public subnets for the NLB, NAT Gateway, and NGINX Ingress, and private subnets for EKS worker nodes and RDS. Worker nodes and the database have no public IPs.

The EKS cluster has five namespaces — dev, qa, prod, argocd, and monitoring. ArgoCD manages all deployments. The Prometheus stack runs in monitoring. Secrets come from Secrets Manager via External Secrets Operator.

The flow: developer pushes code, CI builds the image and tags it `sha-a1b2c3d`, updates the tag in zen-gitops, ArgoCD syncs. Prod deployments require two approvals and a manual ArgoCD sync at a maintenance window. Nothing reaches production automatically.

**Version B — Structured Reference**

- **Four repos:** zen-infra (Terraform), zen-pharma-frontend, zen-pharma-backend (CI + 8 services), zen-gitops (Helm values + ArgoCD manifests)
- **VPC 10.0.0.0/16:** Public (NLB, NAT GW, NGINX). Private EKS (worker nodes). Private RDS (PostgreSQL 15.7).
- **EKS namespaces:** dev (8 individual ArgoCD apps, auto-sync), qa (auto-sync), prod (MANUAL sync), argocd, monitoring.
- **8 services:** api-gateway (8080), auth (8081), drug-catalog (8082), inventory (8083), supplier (8084), manufacturing (8085), notification/Node.js (3000), pharma-ui/React (80).
- No manual console changes. Everything is Terraform or GitOps. Every change has an author, timestamp, and is reversible.

**Diagram — AWS Architecture**

```
┌─────────────────────────────────────────────────────────┐
│  AWS Account  us-east-1                                  │
│  VPC  10.0.0.0/16                                        │
│  ├── Public subnets  10.0.1-2.0/24                       │
│  │   ├── NLB (NGINX Ingress entry point)                 │
│  │   └── NAT Gateway (outbound for private nodes)        │
│  ├── Private EKS subnets  10.0.3-4.0/24                  │
│  │   └── EKS nodes (t3.small × 3)                        │
│  │       ├── ns: dev    (auto-sync, 8 apps)              │
│  │       ├── ns: qa     (auto-sync, app-of-apps)         │
│  │       ├── ns: prod   (MANUAL sync, app-of-apps)       │
│  │       ├── ns: argocd                                  │
│  │       └── ns: monitoring (Prometheus stack)           │
│  └── Private RDS subnets  10.0.5-6.0/24                  │
│      └── RDS PostgreSQL 15.7 (port 5432, EKS SG only)   │
│  Also: ECR (8 repos) · Secrets Manager · S3 (tf state)  │
└─────────────────────────────────────────────────────────┘
```

---

## 02 — CI/CD Pipeline

### Q3 — Can you explain each stage of your pipeline?

**Version A — Candidate Voice**

The full pipeline runs on develop and release branches and has 9 sequential stages.

- **Stage 1** is Maven verify — compile, unit tests, integration tests with a Postgres 15 sidecar for DB services, JaCoCo coverage gate at 80%.
- **Stage 2** is CodeQL — it initialises before Maven to instrument the compilation and collect call-graph data, then runs the security-extended query suite. Results to GitHub Security tab as SARIF.
- **Stage 3** is Semgrep — java, owasp-top-ten, spring-boot rules. Advisory with `continue-on-error`. Findings show in the dashboard but do not block the build.
- **Stage 4** is OWASP Dependency Check — Maven deps against NIST NVD. CVSS 7.0 and above. NVD database cached between runs.
- **Stage 5** is Docker build — multi-stage, eclipse-temurin:17-jre, non-root UID 1000.
- **Stage 6** is Trivy — image scan after build, before ECR push. Fails on HIGH and CRITICAL CVEs that have a fix available. `ignore-unfixed: true` so we are not permanently blocked on unfixable OS issues. SARIF to GitHub Security tab.
- **Stage 7** is ECR push — OIDC auth, no static keys. Tag is `sha-<7chars>`. Immutable and git-traceable.
- **Stage 8** is Cosign signing — keyless. GitHub OIDC token to Fulcio CA to Rekor transparency log. Proves this exact image was built by our CI workflow.

After all stages pass, CI commits the image tag to zen-gitops for DEV and opens a QA promotion PR.

**Version B — Structured Reference**

- **Stage 1 — Maven verify:** compile + tests + JaCoCo >= 80%. Postgres sidecar for DB services. Hard block.
- **Stage 2 — CodeQL:** Java call-graph SAST. security-extended queries. SARIF to GitHub Security tab. Hard block.
- **Stage 3 — Semgrep:** java + owasp-top-ten + spring-boot. `continue-on-error: true` (advisory).
- **Stage 4 — OWASP Dependency Check:** pom.xml vs NIST NVD. CVSS >= 7.0. NVD DB cached. Advisory.
- **Stage 5 — Docker:** Multi-stage. eclipse-temurin:17-jre. UID 1000 non-root.
- **Stage 6 — Trivy:** Full image scan. HIGH/CRITICAL with fix only. `ignore-unfixed: true`. SARIF to Security tab. Hard block.
- **Stage 7 — ECR push:** OIDC auth. `sha-<7chars>` tag. Digest captured for signing.
- **Stage 8 — Cosign:** Keyless signing. GitHub OIDC → Fulcio CA → Rekor transparency log.
- **Post-stages:** Commit tag to `zen-gitops/envs/dev`. Open `promote/qa/<svc>/<tag>` PR.
- Feature branches = stages 1-5 only (5 min, no Docker, no ECR). Full 9-stage pipeline only on develop (15 min). 3x faster developer feedback on unreviewed code.

**Diagram — Pipeline Stages**

```
  ├─[1] Gitleaks          secret scan              BLOCK if found
  ├─[2] Maven verify      tests + coverage 80%     BLOCK if fail
  ├─[3] CodeQL            Java SAST                BLOCK if found
  ├─[4] Semgrep           OWASP rules              advisory
  ├─[5] OWASP dep-check   NVD CVSS>=7.0            advisory
  ├─[6] Docker build      multi-stage, UID 1000
  ├─[7] Trivy scan        HIGH/CRITICAL CVEs       BLOCK if fixable
  ├─[8] ECR push          sha-a1b2c3d, OIDC auth
  └─[9] Cosign sign       keyless → Fulcio → Rekor
         │
         ▼
  Commit tag to zen-gitops/envs/dev
  Open promote/qa/<svc>/<tag> PR
```

---

### Q4 — How do you optimize build time?

**Version A — Candidate Voice**

Build time optimisation was something we thought about deliberately. There are several layers to it.

The biggest win is **path filters**. This is a 7-service monorepo. Without path filters, a one-line fix to notification-service triggers all 7 pipelines — 6 completely wasted builds. With path filters scoped per service directory, we eliminated roughly 85% of unnecessary pipeline runs.

Second is the **two-tier pipeline**. A developer pushing 10 to 15 times a day to a feature branch does not need a Docker build and Trivy scan on every push. Feature branches get a 5-minute lightweight check. Full pipeline only runs on develop. Developer feedback is 3 times faster.

Third is **Maven dependency caching**. setup-java caches `~/.m2` keyed on `pom.xml`. After the first run, only new dependencies download. Saves 2 to 3 minutes per build.

Fourth is **Docker layer ordering**. We copy `pom.xml` first, run `mvn dependency:go-offline`, then copy src. The dependency layer is cached unless `pom.xml` changes. Only the JAR copy busts cache on each push.

Fifth is **OWASP NVD caching** — the CVE database is cached keyed on `pom.xml` hash. Drops the OWASP step from 4 minutes to under 30 seconds after the first run.

Finally, we do not run the same stages for staging and prod as we promote the same image built for develop to production.

**Version B — Structured Reference**

- **Path filters:** ~85% reduction in unnecessary builds. One service change triggers only that service pipeline.
- **Two-tier pipeline:** Feature = 5 min (no Docker). develop = 15 min full. 3x faster developer feedback.
- **Maven caching:** `~/.m2` keyed on `pom.xml`. Saves 2-3 min per build.
- `--no-transfer-progress` on all Maven commands: no progress-bar noise in logs.
- **Parallel service pipelines:** all 7 build on separate GitHub Actions runners simultaneously.
- **OWASP NVD caching:** Keyed on `pom.xml` hash. Drops from 4 min to <30 sec after first run.
- **Docker layer ordering:** pom.xml → go-offline → src. Deps layer cached until `pom.xml` changes.
- Saved: ~85% fewer runs · ~6 min per build · 3x faster dev feedback

---

### Q5 — What is your branching strategy?

**Version A — Candidate Voice**

We follow Gitflow. Let me walk through it.

`main` always reflects production. Only release branches and hotfix branches merge here. Every merge is tagged — `v1.2.0`, incrementally afterwards.

`develop` is our integration branch. All completed feature work lands here first.

Feature branches are cut from develop — named `feat/JIRA-101`. The developer gets a CI check on every push, then raises a PR targeting develop. Code review, approval, merge.

At sprint end we cut a release branch — `release/1.2.0`. Develop is now open for next sprint. Release goes through QA. Bugs are fixed on branches off release and merged back. Once QA signs off, we promote to prod.

Hotfixes — this is important — are cut from `main`, not `develop`. Develop may have incomplete sprint work. After the fix, merge to main, tag `v1.1.1`, then back-merge to develop so the fix is not lost.

The back-merge is the step teams most often forget. If you fix bugs on release during QA and do not back-merge, those fixes disappear in the next sprint.

**Version B — Structured Reference**

- **main:** Always reflects prod. Only release + hotfix branches merge here. Tagged on each merge.
- **develop:** Integration branch. Full CI pipeline triggers on push.
- **release/1.2.0:** Cut from develop at sprint end. QA testing. Bug fixes on sub-branches.
- Why Gitflow not trunk-based? Formal QA phase + scheduled sprint releases. Gitflow gives clean separation.
- **Critical:** Always back-merge release into develop after prod. Bug fixes made during QA must flow back or are silently lost next sprint.

**Diagram — Gitflow**

```
main    ●──────────────────────────────●──────────●(v1.2.0)──────────●(v1.2.1)
         \                            /            \                  /
          \                          /              \   hotfix       /
develop    ●──●──●──●──────────────●──────────────────────────────●
              |  |
           feat/A feat/B       release/1.2.0──●──●──● (QA fixes)
```

---

### Q6 — Do you enforce PR reviews? Any automation?

**Version A — Candidate Voice**

Yes — both through process and automation, and they work together.

At the repository level, branch protection rules on `main` and `develop` prevent direct commits from anyone including admins. `zen-infra` requires 2 reviewers because infra changes carry higher risk. Stale review dismissal is on — new commits after approval reset it automatically.

The automation side is Required Status Checks. The merge button is physically blocked until Build, SAST, Scan, Push, and Sign all pass. Even if a reviewer approves, GitHub blocks the merge if checks are red.

We use CODEOWNERS so PRs touching specific service directories auto-request the right team. No one manually tags reviewers.

GitHub Environments gate deployments. dev has no gate. prod requires Required Reviewers — the pipeline pauses and cannot proceed until the Release Manager approves in the GitHub UI.

For Terraform, the plan output is automatically posted as a PR comment. The reviewer sees the exact AWS resources that will change before they approve the merge.

**Version B — Structured Reference**

- **Branch protection:** No direct commits to main or develop. PR required for everyone including admins.
- **Reviewer count:** zen-pharma-backend = 1. zen-infra (Terraform) = 2 (higher risk).
- **Stale review dismissal:** New commits reset approval automatically.
- **Required status checks:** Build · SAST · Scan · Push · Sign must all pass. Hard block on merge.
- **CODEOWNERS:** Service directory changes auto-request the service owner.
- **GitHub Environments:** dev = no gate. prod = Required Reviewers (Release Manager + QA Lead).
- **Terraform PR automation:** plan runs on PR open. Output posted as PR comment. Reviewer sees exact AWS diff.
- **Gitleaks:** Stage 1 of every PR check. Catches committed secrets before code is reviewed.
- The combination of automated gates (Required Status Checks) and human gates (Required Reviewers) means no code reaches prod without passing automated security AND explicit human approval.

**Diagram — PR Merge Gates**

```
PR opened on develop
  │
  ├── Gitleaks + tests + CodeQL must PASS (Required Status Check)
  ├── CODEOWNERS auto-assigns reviewer
  ├── 1 reviewer must APPROVE
  └── GitHub MERGE only allowed when all conditions met

PROD deployment path:
  Release Manager approves  (Required Reviewer)
  QA Lead approves          (Required Reviewer)
  Engineer manually syncs in ArgoCD UI at maintenance window
```

---

## 03 — GitOps & ArgoCD

### Q7 — What is GitOps and how does it differ from traditional CI/CD?

**Version A — Candidate Voice**

In traditional CI/CD, the pipeline pushes changes using `kubectl apply` or `helm upgrade`. Once the pipeline finishes, there is no ongoing guarantee that the cluster stays in the desired state. Someone can `kubectl edit` a deployment, change an image tag manually, and drift silently.

GitOps inverts this. Git is the source of truth. ArgoCD runs inside the cluster, continuously pulls from zen-gitops, and compares desired state against actual cluster state. Any drift is detected and self-healed automatically. Every change is a git commit — auditable, reversible, author-attributed.

In our project specifically — GitHub Actions never runs kubectl. It only commits to zen-gitops. ArgoCD watches that repo and syncs. If someone manually changes a deployment, ArgoCD detects it within 3 minutes and reverts it. That is the key difference. Traditional CI/CD runs once and walks away. GitOps never walks away.

**Version B — Structured Reference**

- **Traditional CI/CD:** Pipeline pushes (`kubectl apply`). Runs once. No ongoing reconciliation. Drift goes undetected.
- **GitOps:** Git = source of truth. ArgoCD continuously reconciles desired vs actual state. `selfHeal: true` auto-reverts drift.
- `prune: true` — resources removed from Git are removed from cluster automatically.
- Every deployment = git commit with author + timestamp + exact diff. Full built-in audit trail.
- Rollback = `git revert`. Not `kubectl rollout undo` (that bypasses Git and causes re-drift on next sync).
- In GitOps: CI only writes to Git. CD is the continuous reconciliation loop inside the cluster. Separated concerns with clear responsibilities.

**Diagram — GitOps vs Traditional**

```
Traditional CI/CD:
  Pipeline → kubectl apply → cluster
  (drift can happen silently, no auto-detection)

GitOps (our project):
  CI → commit to zen-gitops
           │
           ▼
  ArgoCD polls zen-gitops every 3 minutes
           │
  compare: Git state  vs  cluster state
           │
  ├── Match → nothing to do
  └── Drift → auto-revert (selfHeal: true)
```

---

### Q8 — How do you handle multiple environments (dev/qa/prod)?

**Version A — Candidate Voice**

Environment management happens at three layers.

At the **infrastructure layer**, each environment is a completely separate Terraform workspace — `envs/dev`, `envs/qa`, `envs/prod` with its own state file. A `terraform destroy` in dev cannot touch QA or prod state. They call the same modules but with different inputs.

At the **application layer**, one shared Helm chart with per-service per-environment values files in zen-gitops. dev gets DEBUG logs and 1 replica with autoscaling disabled. prod gets WARN logs, minimum 2 replicas, HPA enabled from 2 to 5, and podAntiAffinity so replicas spread across nodes. Secrets are also per-environment — `/pharma/dev/`, `/pharma/qa/`, `/pharma/prod/` — so cross-env credential access is impossible.

For **promotion gates** — dev auto-syncs, no gate. QA requires a PR merge reviewed by the QA team. Prod requires two PR approvals — Release Manager and QA Lead — plus a manual ArgoCD sync at a planned maintenance window. Prod never auto-syncs. That is a hard rule.

**Version B — Structured Reference**

- **Infrastructure:** Separate Terraform state per env. Same modules, different inputs. Destroy in dev cannot touch prod.
- **Config:** One Helm chart. Per-service per-env values files (`zen-gitops/envs/dev|qa|prod`).
- **Config diff:** dev=DEBUG/1 replica/HPA off. prod=WARN/2 replicas min/HPA 2-5/podAntiAffinity.
- **Secrets:** `/pharma/dev/*`, `/pharma/qa/*`, `/pharma/prod/*` — separate paths. Cross-env access impossible.
- dev: auto-sync + selfHeal. No gate.
- qa: PR merge in zen-gitops gates deployment. 1 reviewer. ArgoCD auto-syncs after merge.
- prod: 2 PR approvals + MANUAL ArgoCD sync at maintenance window. NEVER auto-syncs.

**Diagram — Build Once, Deploy Everywhere**

```
Build Once:   sha-a1b2c3d in ECR
                  │
      ┌───────────┼─────────────┐
      ▼           ▼             ▼
 dev namespace  qa namespace  prod namespace
 auto-sync      PR gate       2 approvals + manual sync
 DEBUG logs     INFO logs     WARN logs
 1 replica      1 replica     2 replicas min, HPA 2-5
 /pharma/dev/*  /pharma/qa/*  /pharma/prod/*

Terraform state isolation:
 S3: envs/dev/terraform.tfstate   (own .tflock)
 S3: envs/qa/terraform.tfstate    (own .tflock)
 S3: envs/prod/terraform.tfstate  (own .tflock)
```

---

### Q9 — How do you rollback a failed deployment?

**Version A — Candidate Voice**

This is one of the strongest parts of using GitOps. Rolling back is just changing a value in a YAML file.

**Standard path — about 3 minutes.** I go to zen-gitops, click on closed PRs, open the PR which has the change to prod values, revert the PR. The prod values reverts to the previous tag. Then trigger an ArgoCD manual sync. Kubernetes does a rolling update back to the previous image. Because `maxUnavailable: 0`, old pods keep serving throughout. Zero downtime during the rollback itself.

**If the situation is critical** — `argocd app history pharma-prod` to find the last good history ID, then `argocd app rollback pharma-prod` with that ID. Cluster rolls back in about 60 seconds without a git commit.

But I always follow an ArgoCD direct rollback with a git revert. Otherwise Git says one thing and the cluster runs something different — that is drift. ArgoCD would revert the rollback on the next reconcile cycle.

We never use `kubectl rollout undo`. That works on the cluster without touching Git and ArgoCD would revert it on the next sync.

**Version B — Structured Reference**

- **Standard rollback (~3 min):** Revert the PR which has prod changes → merge revert → ArgoCD manual sync.
- **Fast rollback (~60 sec):** `argocd app rollback pharma-prod <history-id>`. Re-applies cached previous manifests. Always follow with `git revert`.
- `maxUnavailable: 0` — old pods serve throughout rollback. Zero downtime.
- Every image tag in git history with author + timestamp = complete deployment audit trail.
- **NEVER** `kubectl rollout undo` — bypasses Git. ArgoCD re-applies bad image on next reconcile.
- In GitOps, rollback = git revert. Same reconciliation loop as forward deployment. The rollback is itself auditable.

**Diagram — Rollback Decision Tree**

```
Incident detected: Alertmanager fires / ArgoCD Degraded
         │
   ┌─────┴───────────────────────────────┐
   │ CRITICAL (60 sec)     STANDARD (3 min)
   │        │                      │
   │ argocd app rollback    git log envs/prod/values-*.yaml
   │ pharma-prod <id>               │
   │        │               git revert <bad-commit-sha>
   │ cluster reverts                │
   │        │               argocd app sync pharma-prod
   │ FOLLOW UP: git revert          │
   │ (fix drift)            K8s rolling update
   └────────────────────────────────┘
   maxUnavailable: 0 → old pods serve throughout → zero downtime
```

---

### Q10 — What is your deployment strategy?

**Version A — Candidate Voice**

GitOps-driven rolling updates with carefully tuned probe configuration.

Inside Kubernetes: `RollingUpdate` with `maxSurge: 1` and `maxUnavailable: 0`. `maxUnavailable: 0` is the critical setting — no old pods die until new pods pass their readiness probe. Zero downtime is a hard guarantee.

Every service has both probes. Readiness checks `/actuator/health/readiness` with 45 seconds initial delay — Spring Boot needs time to start. If readiness fails, the pod is removed from Service endpoints but keeps running. The rollout stalls. Old pods keep serving. No user impact. Liveness checks `/actuator/health` with 90 seconds initial delay. Failing liveness restarts the pod — catches deadlocks and hung threads.

In prod, podAntiAffinity spreads replicas across different nodes. Single node failure cannot take down all replicas simultaneously.

Database migrations run as ArgoCD PreSync hooks — a Kubernetes Job that completes before the Deployment pods update. Never in application startup. Two pods starting simultaneously would both try to migrate and cause race conditions. Migrations are always backward-compatible so old and new pod versions coexist during rolling updates.

**Version B — Structured Reference**

- **Strategy:** RollingUpdate, `maxSurge: 1`, `maxUnavailable: 0`.
- `maxUnavailable: 0` — no old pod terminates until new pod passes readiness. Zero downtime guaranteed.
- **Prod podAntiAffinity:** replicas on different nodes. Node failure cannot kill all replicas.
- **DB migrations:** Flyway as ArgoCD PreSync hook (K8s Job). Never in app startup. Always backward-compatible.
- Two probes serve different purposes. Readiness = traffic routing. Liveness = pod health. Both required.
- A pod can be alive but not ready (starting), or ready then become not-live (deadlock).

**Diagram — Rolling Update Sequence**

```
Rolling update sequence (prod, 2 replicas):

Initial:  [old-pod-1 READY][old-pod-2 READY]

Step 1:   [old-pod-1 READY][old-pod-2 READY][new-pod starting...]
          maxSurge=1 allows 3 pods temporarily

Step 2:   new-pod passes readiness probe (/actuator/health/readiness, 45s delay)
          old-pod-1 TERMINATES (maxUnavailable=0 → only now safe to remove)

Step 3:   [old-pod-2 READY][new-pod-1 READY][new-pod-2 starting...]
          old-pod-2 terminated ONLY after new-pod-1 ready

Step 4:   [new-pod-1 READY][new-pod-2 READY]
          rollout complete. 0 downtime throughout.

Readiness probe failing → rollout stalls, old pods keep serving, Alertmanager fires.
```

---

## 04 — Security & DevSecOps

### Q11 — Do you scan Docker images?

**Version A — Candidate Voice**

Yes — we scan at two distinct layers and sign on top of that.

Before the Docker build, **OWASP Dependency Check** scans our Maven `pom.xml` against the NIST NVD database. Catches vulnerable Java libraries. But it cannot see the OS layer.

After the Docker build — and this is the critical point — before we push to ECR, **Trivy** scans the complete container image. It sees the OS packages, JRE runtime, libssl, glibc, anything bundled in the image. A vulnerable libssl in the base layer is completely invisible to OWASP. Trivy catches it. `severity: HIGH,CRITICAL`. `ignore-unfixed: true` so we only fail when a fix actually exists.

Results go to the GitHub Security tab as SARIF, tagged per service. ECR itself has scan-on-push enabled — an independent AWS-native second opinion.

After the image is pushed, **Cosign** signs it keylessly. GitHub OIDC token to Fulcio CA to Rekor. The signature proves this exact image was built by our specific workflow. Someone who gains ECR write access and pushes a malicious image — their image has no valid signature and fails verification.

**Version B — Structured Reference**

- **OWASP Dependency Check (pre-build):** `pom.xml` vs NVD. Catches app library CVEs. Cannot see OS packages.
- **Trivy (post-build, pre-push):** Full image — OS, JRE, libssl, glibc. `severity: HIGH,CRITICAL`. `ignore-unfixed: true`. SARIF to Security tab. Hard block.
- **ECR scan-on-push:** All 8 repos. AWS native scanner. Independent second opinion.
- **Cosign keyless:** GitHub OIDC → Fulcio CA → Rekor transparency log. Signs sha256 digest.
- **Roadmap:** Kyverno ClusterPolicy to enforce signature verification at pod admission — reject unsigned images.
- OWASP scans pom.xml (app libs). Trivy scans the built image (OS + runtime + libs). They cover different surfaces. Both are required.

**Diagram — Image Scanning and Signing Pipeline**

```
Maven build (pom.xml)
  │
  ├── [OWASP] scans pom.xml vs NVD (app lib CVEs)
  │
  └── Docker build (multi-stage, eclipse-temurin:17-jre, UID 1000)
          │
          ├── [Trivy] scans built image
          │   ├── OS packages (Alpine/Debian base layer)
          │   ├── JRE runtime libraries
          │   ├── libssl · glibc · zlib
          │   └── BLOCK on HIGH/CRITICAL (with fix)
          │
          ├── [ECR push] sha-a1b2c3d · OIDC auth
          │   └── ECR scan-on-push (AWS native, 2nd opinion)
          │
          └── [Cosign sign] keyless
              GitHub OIDC → Fulcio CA → Rekor
              Signs the sha256 digest (immutable reference)
```

---

### Q12 — How do you manage secrets in CI/CD?

**Version A — Candidate Voice**

Three completely separate layers, and the golden rule throughout is — no secret ever touches Git, a file on disk, or a log.

**In CI**, there are no static AWS credentials. GitHub OIDC exchanges a short-lived token for AWS credentials scoped to `pharma-github-actions-role`. The IAM trust policy `sub` claim is locked to our specific repository. Credentials expire when the workflow ends.

**At the infrastructure level**, DB passwords come in as GitHub Secrets at runtime via Terraform `-var`. Terraform writes them to AWS Secrets Manager under `/pharma/env/` paths. `sensitive=true` redacts them from all plan and apply logs. Never in `.tfvars`, never in committed files.

**At application runtime in Kubernetes**, External Secrets Operator with IRSA. ExternalSecret CRDs declare what to fetch and how to map it. ESO authenticates via IRSA — service account annotated with the IAM role ARN, EKS injects short-lived credentials. Secrets auto-refresh every hour. Nothing sensitive in zen-gitops or any Helm values file.

**Version B — Structured Reference**

- **ESO IAM role:** `secretsmanager:GetSecretValue` on `/pharma/*` paths ONLY. Cannot read other secrets.
- **Gitleaks (Stage 1):** Catches accidentally committed secrets before they spread.
- No secret ever touches Git, a file on disk, or appears in a log. Three-layer principle: OIDC for CI, Secrets Manager for storage, IRSA for runtime.

**Diagram — Three-Layer Secrets Architecture**

```
Layer 1 — CI authentication:
  GitHub Actions OIDC → STS:AssumeRoleWithWebIdentity
  sub claim: "repo:org/zen-pharma-backend:*" (locked to this repo)
  → Short-lived AWS credentials (expire at workflow end)

Layer 2 — Infrastructure secrets:
  GitHub Secret (DEV_DB_PASSWORD)
  → Terraform -var at runtime
  → AWS Secrets Manager /pharma/dev/db-credentials {username, password}
                        /pharma/dev/jwt-secret       {secret}

Layer 3 — Runtime secrets:
  ESO + IRSA → secretsmanager:GetSecretValue (/pharma/* only)
  → Kubernetes Secret (db-credentials, jwt-secret) (refresh 1h)
  → Pod envFrom: secretRef (env vars at startup)
  (NEVER in zen-gitops or any values file)
```

---

### Q13 — How do you control access (RBAC, IAM)?

**Version A — Candidate Voice**

Four distinct layers, least privilege at each one.

**AWS IAM** — no static credentials anywhere. GitHub Actions uses OIDC. ESO pods use IRSA scoped only to `secretsmanager:GetSecretValue` on `/pharma/*` paths. EKS nodes have the three required managed policies. All provisioned by Terraform.

**Kubernetes RBAC** — the `pharma-deployer` role is namespace-scoped, not a ClusterRole. Lists exact resources and verbs. Pods are get/list/watch only — no exec, no delete. A role in dev gives zero access to prod.

**ArgoCD AppProject** — whitelists which repos and namespaces are allowed. Even with a broad ClusterRole, ArgoCD can only source from zen-gitops and deploy to the listed namespaces.

**GitHub Environments** — dev has no gate. prod requires two named reviewers. The pipeline cannot proceed without their approval in the GitHub UI.

**Version B — Structured Reference**

- **AWS IAM:** OIDC for CI. IRSA for ESO (`/pharma/*` only). IRSA for ArgoCD. 3 managed policies for EKS nodes. All in Terraform.
- `sub` claim: locks OIDC/IRSA to specific repo + service account. No lateral movement.
- **K8s RBAC:** `pharma-deployer` Role is namespace-scoped. Pods: get/list/watch only — no exec, no delete. Dev role = zero prod access.
- **ArgoCD AppProject:** `sourceRepos` whitelist (zen-gitops only). `destinations` whitelist (dev/qa/prod/argocd/monitoring).
- **GitHub Environments:** dev = no gate. prod = Required Reviewers (Release Manager + QA Lead).
- Defence in depth: Compromised ESO pod can only read `/pharma/*` secrets. Compromised ArgoCD can only deploy to listed namespaces from listed repos.

**Diagram — Four-Layer Access Control**

```
Layer 1 — AWS IAM:
  GitHub Actions: OIDC → pharma-github-actions-role
  ESO: IRSA → pharma-eso-role (secretsmanager:GetSecretValue /pharma/* only)
  EKS nodes: AmazonEKSWorkerNodePolicy + AmazonEKS_CNI + AmazonEC2ContainerRegistryReadOnly

Layer 2 — Kubernetes RBAC:
  pharma-deployer Role (namespace-scoped, NOT ClusterRole)
  deployments: get list watch create update patch
  pods/log:    get list watch  (NO exec, NO delete)
  Role in dev namespace = ZERO access to prod namespace

Layer 3 — ArgoCD AppProject:
  sourceRepos:  zen-gitops only
  destinations: dev, qa, prod, argocd, monitoring only

Layer 4 — GitHub Environments:
  dev:  no gate (auto)
  prod: Release Manager + QA Lead must approve
```

---

### Q14 — How do you manage ConfigMaps and Secrets?

**Version A — Candidate Voice**

Hard separation between non-sensitive and sensitive config — completely different paths.

**ConfigMaps** hold everything non-secret — Spring profiles, log levels, DB hostnames, port numbers. These live in Helm values files in zen-gitops. Safe to commit. The Helm chart template iterates the `configmap` section and creates a Kubernetes ConfigMap automatically.

There is one really useful trick — the **checksum annotation**. The Deployment pod template has `checksum/config` set to the sha256sum of the ConfigMap content. When config changes, the hash changes, Kubernetes sees the pod template as different, and triggers a rolling restart automatically. No manual `kubectl rollout restart` needed after config changes.

For **secrets**, External Secrets Operator fetches from AWS Secrets Manager. ExternalSecret CRDs declare what to fetch. ESO uses IRSA. Secrets auto-refresh every hour. Nothing sensitive in zen-gitops or any values file.

Both are consumed via `envFrom` in the pod — `configMapRef` and `secretRef`. The application sees no difference between a config value from a ConfigMap and one from a Secret.

**Version B — Structured Reference**

- Both consumed via `envFrom: configMapRef + secretRef`. Application sees no difference.
- Per-env config differences: `SPRING_PROFILES_ACTIVE`, `LOG_LEVEL`, `DB_HOST` are all env-specific.
- **The checksum annotation** eliminates an entire class of operational errors — stale config on running pods after a config-only update. Small addition, big impact.

**Diagram — ConfigMap and Secret Flow**

```
Non-sensitive config:
  zen-gitops/envs/dev/values-auth-service.yaml
    configmap:
      SPRING_PROFILES_ACTIVE: dev
      LOG_LEVEL: DEBUG
      DB_HOST: pharma-dev-postgres...
  → Kubernetes ConfigMap "auth-service"
  → Deployment annotation: checksum/config: sha256sum(ConfigMap)
     (config change → hash changes → rolling restart auto-triggered)

Sensitive config:
  AWS Secrets Manager /pharma/dev/db-credentials
  → ESO ExternalSecret (IRSA auth, refresh 1h)
  → Kubernetes Secret "db-credentials"
  → Pod envFrom: secretRef
  (NEVER in zen-gitops or values files)
```

---

## 05 — Kubernetes

### Q15 — How do you troubleshoot a pod not starting? What issues have you seen?

**Version A — Candidate Voice**

The key discipline is reading the pod status first — the error code tells you exactly which investigation branch to follow.

**Pending** means the scheduler cannot place the pod. `kubectl describe pod` shows Events — usually `Insufficient cpu` or `Insufficient memory`. `kubectl top nodes` then scale via Terraform.

**CreateContainerConfigError** means a Secret or ConfigMap is missing before the container starts. No application logs will help — the container never started. Check `kubectl describe externalsecret` and verify ESO sync status.

**CrashLoopBackOff** means the app is crashing. `kubectl logs --previous` reads the stacktrace. Look for `Caused by` at the bottom — that is the real root cause. Most common in our project: wrong DB_HOST after a cluster rebuild, Flyway migration timeout exceeding probe delay, missing env var.

**ImagePullBackOff** with a 401 from ECR usually means IRSA is broken. If we rebuilt the cluster, the OIDC URL changed and the IAM trust policy no longer matches.

**Running but 0/1 Ready** means the readiness probe is failing. Usually `initialDelaySeconds` is too short for what the service does at startup.

The Events section in `kubectl describe` answers 80 to 90% of questions before you need application logs.

**Version B — Structured Reference**

- **CreateContainerConfigError:** Secret/ConfigMap missing. No app logs. Check ESO sync: `kubectl describe externalsecret`.
- **CrashLoopBackOff:** `kubectl logs --previous`. Look for "Caused by:" at bottom. Common: wrong DB_HOST, Flyway timeout, missing env var.
- **Running 0/1 Ready:** Readiness probe failing. Check describe Events. Increase `initialDelaySeconds` in values file.
- **OOMKilled:** Memory limit exceeded. `kubectl top pods`. Raise memory limit in values file or fix memory leak.
- Events section in `kubectl describe pod` answers 90% of questions. Read it before application logs.
- STATUS column tells you which investigation branch — read it before anything else.

**Diagram — Pod Status Troubleshooting Tree**

```
kubectl get pods → read STATUS column
  │
  ├── Pending
  │     → kubectl describe pod → Events → Insufficient cpu/mem / node affinity / taint
  │
  ├── CreateContainerConfigError
  │     → Secret/ConfigMap missing → kubectl describe externalsecret
  │
  ├── CrashLoopBackOff
  │     → kubectl logs --previous → look for "Caused by:" at bottom
  │
  ├── ImagePullBackOff
  │     → 401 from ECR → IRSA broken / OIDC URL mismatch after cluster rebuild
  │
  ├── Running 0/1 Ready
  │     → readiness probe failing → increase initialDelaySeconds in values file
  │
  └── OOMKilled
        → kubectl top pods → raise memory limit or fix memory leak
```

---

### Q16 — How do you handle scaling?

**Version A — Candidate Voice**

Scaling works at three distinct levels.

**Pod level — HPA.** Disabled in dev and QA to control costs, enabled in prod with minimum 2 and maximum 5 replicas. CPU target is 70%, not 80 or 90%. Spring Boot pods take 45 seconds to pass readiness. At 70% we trigger scale-out early enough that new pods are ready before capacity is exhausted. Memory target at 80%.

**Node level — EKS managed node groups.** Dev uses t3.small, min 1, desired 3, max 4. Prod uses t3.medium, min 2, max 5. Currently manual via Terraform. Cluster Autoscaler is on the roadmap — the module inputs are already wired for `min_size` and `max_size`.

**NGINX Ingress also has HPA** — minimum 2 to maximum 10 pods with podAntiAffinity across nodes. The entry point to the cluster cannot be the bottleneck.

Resource requests are the foundation. HPA calculates utilisation as actual usage divided by request. We tuned requests to P95 actual consumption after a week of monitoring.

**Version B — Structured Reference**

- **Pod HPA (prod only):** min 2 / max 5. CPU target 70%. Memory target 80%. Disabled in dev/QA.
- **Why 70%?** Spring Boot takes 45s to pass readiness. Scale early so new pods ready before saturation.
- **Node group:** dev=t3.small min1/desired3/max4. prod=t3.medium min2/max5. Manual via Terraform currently.
- **Roadmap:** Cluster Autoscaler — Terraform inputs already wired. Watches Pending pods, provisions new EC2.
- **NGINX Ingress HPA:** min 2 / max 10. podAntiAffinity across nodes.
- **Resource requests:** cpu 100m / mem 256Mi request. cpu 500m / mem 512Mi limit. Tuned to P95 actual.
- Without Cluster Autoscaler: HPA scales pods but if nodes are full, pods go Pending. CA is the next step to automate node provisioning.

**Diagram — Three-Level Scaling**

```
Traffic increases:
  CPU > 70% on backend pods (HPA trigger)
  → HPA: create new pod (45s readiness delay for Spring Boot)
  → If nodes full: pod goes Pending
  → [Roadmap] Cluster Autoscaler: provisions new EC2 node
  → Pod scheduled, starts, passes readiness

dev/QA: HPA disabled. Fixed replicas. Manual node sizing.
prod:   HPA 2-5 pods. podAntiAffinity. Node group max 5.
```

---

## 06 — Terraform & Infrastructure

### Q17 — How do you reuse Terraform code?

**Version A — Candidate Voice**

All infrastructure logic lives in modules. Environments are callers with different inputs. Think of a module as a function — inputs, work, outputs.

Six modules: `vpc`, `eks`, `rds`, `ecr`, `iam`, `secrets-manager`. Dev, QA, and prod all call the same modules with different inputs. Dev passes `t3.small` and CIDR `10.0.0.0/16`. Prod passes `t3.medium` and `10.2.0.0/16`. Module code is identical.

Modules share data via output references. E.g.: The EKS module takes `subnet_ids` from `module.vpc.private_eks_subnet_ids`.

**Version B — Structured Reference**

- **Modules:** vpc, eks, rds, ecr, iam, secrets-manager. All envs call same modules with different inputs.
- **Output references:** `module.eks` takes `subnet_ids` from `module.vpc`. Terraform infers ordering. No `depends_on`.
- **`for_each` over `count`:** ECR creates 8 repos from 1 resource block keyed by name. Remove one = destroy only that one.
- **`locals`:** `name_prefix = "pharma-${var.env}"` computed once, used everywhere. DRY naming.
- **Directories not workspaces:** `envs/dev`, `envs/qa`, `envs/prod` — own backend, state, pipeline trigger.
- **Rule:** modules never know which environment they are in. `env` is always an input. No `if env == prod` logic inside modules.

**Diagram — Module Architecture**

```
envs/dev/main.tf (caller):
  module "vpc"
    inputs: env="dev", vpc_cidr="10.0.0.0/16"
    outputs: vpc_id, private_eks_subnet_ids, private_rds_subnet_ids
         │
  module "eks"
    inputs: subnet_ids = module.vpc.private_eks_subnet_ids
    outputs: cluster_name, oidc_provider_arn
         │
  module "rds"
    inputs: subnet_ids = module.vpc.private_rds_subnet_ids
            eks_security_group_id = module.eks.cluster_security_group_id
         │
  module "ecr"
    inputs: for_each = toset(["api-gateway", "auth", ...])
         │
  module "iam"
    inputs: oidc_provider_arn = module.eks.oidc_provider_arn
```

---

### Q18 — How do you manage Terraform state? Have you faced state conflicts?

**Version A — Candidate Voice**

Remote state on S3 — never local. Each environment has a completely separate state file — `envs/dev`, `envs/qa`, `envs/prod`. Separate S3 keys, separate locks.

For locking — we use **S3 native locking**, Terraform 1.10 feature. `use_lockfile: true` in the backend. No DynamoDB table. Terraform creates a `.tflock` file. Second concurrent apply fails immediately with a clear lock error.

**Version B — Structured Reference**

- **Remote backend:** S3. `encrypt: true`. Never local on CI runner (ephemeral).
- **Separate state per env:** `envs/dev/`, `envs/qa/`, `envs/prod/` — own S3 key, own `.tflock`.
- **S3 native locking:** `use_lockfile: true` (Terraform 1.10+). No DynamoDB. `.tflock` file created on apply.
- `cancel-in-progress: false`: Second run waits. Cancelling mid-apply = state corruption.
- **Real conflicts:** Stale lock (force-unlock after verifying). Deleted state + ECR repos (import blocks). Console drift (refresh-only + fix).
- **Failed apply:** Read error. Resources created before failure are in state. Re-run is idempotent. Fix root cause first.
- `terraform force-unlock` is last resort. Only when absolutely certain no apply is running. Using it during a real apply guarantees corruption.

**Diagram — State Management**

```
S3 bucket: zen-pharma-terraform-state
  envs/dev/terraform.tfstate      (own .tflock)
  envs/qa/terraform.tfstate       (own .tflock)
  envs/prod/terraform.tfstate     (own .tflock)
  versioning: ON (every write = recoverable version)

Pipeline flow:
  terraform init → plan → apply
  On failure/cancel: release .tflock automatically

Stale lock scenario:
  Previous apply crashed mid-run → .tflock remains
  Next run: "Error acquiring state lock"
  → Verify no apply running → terraform force-unlock <lock-id>

Deleted state + existing ECR repos:
  apply fails: RepositoryAlreadyExistsException
  Fix: import { to = module.ecr.aws_ecr_repository.main["api-gateway"]
                  id = "api-gateway" }
```

---

## 07 — Monitoring & Observability

### Q19 — How do you monitor your applications? What tools? What alerts? How do you debug production?

**Version A — Candidate Voice**

Full-stack observability across three layers.

**Infrastructure layer** — Node Exporter scrapes CPU, memory, disk, network from every EKS node. kube-state-metrics exposes Kubernetes object state — replica counts, pod phase, container restarts, PVC usage.

**Platform layer** — NGINX Ingress controller exposes Prometheus metrics via pod annotations. Request rates, latency distributions, upstream response times, and 5xx error rates from the ingress layer all scraped automatically.

**Application layer** — every Spring Boot service exposes `/actuator/metrics/prometheus`. We set `serviceMonitorSelectorNilUsesHelmValues` to false so Prometheus discovers all ServiceMonitors across all namespaces automatically. Node.js notification service uses `prom-client` — same pattern.

**Stack:** kube-prometheus-stack — Prometheus, Grafana, Alertmanager, kube-state-metrics, Node Exporter. Prometheus retains 15 days with an 18GB cap on a 20GB gp2 PVC. Grafana at `monitoring.pharma.internal`. Sidecar configured for `searchNamespace: ALL` — any team drops a ConfigMap with dashboard JSON and Grafana picks it up automatically.

**Alerts:** ServiceDegraded when ready pods drop below desired for 5 minutes. HighErrorRate at 5xx rate over 1% for 2 minutes. HighLatency at P95 over 2 seconds for 3 minutes. ResourcePressure at 80% of CPU limit. Plus the full `defaultRules` suite — CrashLoopBackOff, OOMKill, node pressure, etcd, kubelet.

**Debugging sequence** — I always follow the same order. Check ArgoCD status first. Then `kubectl get pods` and read STATUS. Then `kubectl describe pod` and read Events — that answers 80 to 90% of questions. Then `logs --previous` for crashes. Then Grafana for the timeline. Then Prometheus queries for specific investigation.

**Version B — Structured Reference**

- **Stack:** kube-prometheus-stack. Prometheus + Grafana + Alertmanager + kube-state-metrics + Node Exporter. monitoring namespace.
- **Alertmanager:** `group_by: alertname+cluster+service`. `group_wait: 30s`. `repeat_interval: 12h`. `send_resolved: true`. SMTP to devops@pharma.com.
- **Alerts:** ServiceDegraded (ready<desired, 5m) · HighErrorRate (5xx>1%, 2m) · HighLatency (P95>2s, 3m) · ResourcePressure (CPU>80% limit) · defaultRules suite.
- **Debug order:** 1) ArgoCD status 2) pod STATUS 3) describe Events 4) logs --previous 5) Grafana timeline 6) PromQL.
- Slow down 2 min at start of incident. Read carefully. Events in describe answers most questions before needing application logs.

**Diagram — Observability Stack**

```
Metrics sources:
  Node Exporter       → CPU, mem, disk, net per node
  kube-state-metrics  → replica counts, pod phase, restarts, PVC
  NGINX Ingress       → request rate, latency, 5xx rate
  Spring Actuator     → /actuator/metrics/prometheus (all 8 services)
  prom-client         → notification service (Node.js)
         │
         ▼
  Prometheus (15d retention, 20GB PVC)
    ├── Grafana (dashboards at monitoring.pharma.internal)
    └── Alertmanager (group + deduplicate + route)
              └── devops@pharma.com (SMTP)

Debug sequence:
  1) ArgoCD status → 2) pod STATUS → 3) describe Events
  → 4) logs --previous → 5) Grafana timeline → 6) PromQL
```

---

## 08 — Behavioural & Situational

### Q20 — What is the latest thing you worked on?

**Version A — Candidate Voice**

The most recent thing I completed was the full DevSecOps pipeline for the Zen Pharma platform — 8 Spring Boot and Node.js microservices on EKS with GitOps via ArgoCD.

What makes it current for 2026 is the specific stack choices. Terraform 1.10 S3 native locking — no DynamoDB table. Cosign keyless signing using Sigstore Fulcio CA and Rekor — no long-lived signing keys. GitHub OIDC throughout — no static AWS credentials anywhere, pods use IRSA. Every credential is short-lived and identity-bound.

End result: developer pushes code, 5-minute security feedback on feature branches, 15-minute full production-grade build on develop. Production requires two approvals and a manual ArgoCD sync at a maintenance window. Fully auditable, fully reversible.

**Version B — Structured Reference**

- **Project:** Zen Pharma DevSecOps platform. 8 microservices. EKS. GitHub Actions CI. ArgoCD GitOps.
- **Modern stack:** Terraform 1.10 S3 native locking. Cosign keyless signing. GitHub OIDC throughout.
- **Result:** 5 min feature branch feedback. 15 min full build. Prod = 2 approvals + manual sync.
- Lead with specific modern technology choices: S3 native locking (not DynamoDB), Cosign keyless, OIDC everywhere. These signal current 2026 knowledge.

---

### Q21 — What improvement did you recently implement?

**Version A — Candidate Voice**

The most impactful was the **checksum annotation on ConfigMaps**.

Before: changing LOG_LEVEL from DEBUG to INFO updated the ConfigMap but running pods kept the old value in memory. Engineers had to manually run `kubectl rollout restart` after every config change. People forgot. Config changes went silently ignored on running pods for hours.

Fix: one annotation in the Deployment pod template — `checksum/config` set to the sha256sum of the ConfigMap. Config changes, hash changes, Kubernetes sees the pod template as different, triggers rolling restart automatically. Zero manual intervention. Config updates are now atomic.

Also: split pipeline into two tiers reducing feature branch time from 15 minutes to 5 minutes, added Maven and OWASP NVD caching saving about 6 minutes per full build.

**Version B — Structured Reference**

- **Two-tier pipeline:** feature = 5 min. develop = 15 min. 3x faster developer feedback.
- **checksum annotation:** Small addition, big impact. Eliminates entire class of "why is the pod using old config" incidents.

---

### Q22 — Did you reduce cost or improve performance?

**Version A — Candidate Voice**

Two concrete things with measurable impact.

**Cost — path filters** in the monorepo. Before, every push triggered all 7 service pipelines. After path filters, approximately 85% of unnecessary runs eliminated. Direct saving in GitHub Actions minutes and ECR storage.

**Performance — two-tier pipeline.** Developers pushing 15 times a day were waiting 15 minutes each time and started batching changes. After the split: 5-minute feedback on feature branches. Developers push freely and engage with results.

**Infrastructure cost:** dev environment destroyed end-of-day. EKS at $72/month and NAT Gateway at $32/month add up fast if left idle. ECR lifecycle policies keep last 10 images per repo — prevents unbounded storage growth.

**Version B — Structured Reference**

- **Path filters:** ~85% fewer CI runs. Direct GitHub Actions minutes + ECR storage saving.
- **Two-tier pipeline:** feature = 5 min. develop = 15 min. 3x faster feedback.
- **Build caching:** Maven + OWASP NVD = ~6 min saved per full build.
- **Infra:** Dev env destroyed end-of-day (~$104/month saved). ECR lifecycle: last 10 images.
- Quantify: 85% reduction, 3x faster, ~$104/month saved. Numbers make impact concrete.

---

### Q23 — Tell me about a production issue you handled

**Version A — Candidate Voice**

`auth-service` threw 503s for 11 minutes. Alertmanager fired. ArgoCD showed Degraded.

`kubectl get pods` showed two old pods Running/Ready and one new pod Running but 0/1 Ready. `kubectl describe` showed the liveness probe failing. But logs showed Spring Boot still starting — Flyway migrations on prod RDS were taking 85 seconds. The liveness probe had `initialDelaySeconds: 60`.

Because `maxUnavailable: 0`, old pods kept serving throughout. Zero user-facing errors. The 503s were pod-internal health check failures.

**Immediate fix:** patched `initialDelaySeconds` from 60 to 120 in the prod values file. ArgoCD picked it up. Pod stabilised. Zero user impact.

**Root cause fix:** Flyway as ArgoCD PreSync hook. App startup no longer runs migrations. Startup dropped from 85 seconds to under 20 seconds. Added Prometheus alert on migration duration.

**Version B — Structured Reference**

- **Incident:** auth-service 503s. Liveness probe killing pod during Flyway migration (85s > 60s initialDelay).
- **Detection:** Alertmanager error rate. ArgoCD Degraded. `kubectl describe` Events.
- **Post-incident:** Prometheus alert on migration duration.
- `maxUnavailable: 0` prevented user impact. Rolling update safety net worked as designed.

---

### Q24 — What did you automate recently?

**Version A — Candidate Voice**

The **PROD promotion workflow** — `promote-prod.yml` as a GitHub Actions `workflow_dispatch`.

Before: engineer manually reads QA image tag, updates prod values file, raises PR, gets approvals, merges, logs into ArgoCD, clicks sync. Many steps, room for error.

After: Release engineer triggers `workflow_dispatch` from GitHub UI, selects the service. The workflow reads the QA image tag automatically, creates a branch, opens a PR with required reviewers requested. Only the two approvals and ArgoCD sync remain — both intentional gates, not accidental steps.

Also automated the deployment verification script — one command checks pod readiness, ArgoCD health, ExternalSecret sync status, and HTTP endpoints. Complete health picture in seconds.

**Version B — Structured Reference**

- **promote-prod.yml:** `workflow_dispatch`, service dropdown. Reads QA tag automatically. Creates branch. Opens PR. Requests reviewers.
- **Automated verification:** checks pods, ArgoCD health, ExternalSecrets, HTTP endpoints. One command.
- Error rate on promotions: dropped from occasional (wrong tag, wrong service) to zero.
- Automation that eliminates human error in critical processes is more valuable than automation that just speeds things up.

---

### Q25 — Did you ever prevent an issue before it happened?

**Version A — Candidate Voice**

Yes — proactive supply chain security gap closure.

During a security review I noticed CI was using static IAM keys stored as GitHub Secrets. If those leaked, someone could push a malicious image to ECR. Nothing would catch it before it deployed to production.

Two changes: Migrated to **GitHub OIDC** — trust policy `sub` claim locked to our repo. Implemented **Cosign keyless signing** — every image signed with a Fulcio certificate tied to our workflow identity. A manually pushed image has no valid signature and fails verification.

**Version B — Structured Reference**

- **Fix 1:** GitHub OIDC. `sub` claim locked to repo. Short-lived creds.
- **Fix 2:** Cosign keyless signing. Manual push = no valid signature = fails verification.
- Proactive security — especially supply chain — is high-signal in 2026 interviews.

---

### Q26 — How do you collaborate with developers?

**Version A — Candidate Voice**

Not as a gatekeeper — as someone who makes the developer workflow smoother.

Most concrete example: pipeline restructure. The full 15-minute pipeline on every feature branch push meant developers batched changes and ignored CI results. I redesigned to two tiers — 5-minute check on feature branches. Developers push freely, get fast feedback, wait for results.

For security findings — SARIF results go directly to the GitHub Security tab. Developers see CodeQL and Trivy findings in their PR context. Semgrep is advisory with `continue-on-error` — findings show as warnings with remediation hints, not hard failures.

For onboarding a new service — a checklist. Copy a workflow file, create a values file, add an ArgoCD Application manifest. Under an hour without deep platform knowledge.

**Version B — Structured Reference**

- **Security visibility:** SARIF in GitHub Security tab. Findings in PR context. Semgrep advisory not blocking.
- **Alertmanager on Slack:** Developers and ops see prod issues in same channel. No info asymmetry.
- **Goal:** self-service. Developers add a service, deploy, and debug without filing a platform ticket.

---

### Q27 — Production bug came after deployment — how to handle?

**Version A — Candidate Voice**

Two phases — immediate containment then root cause.

First: `git log` on `envs/prod/values-<service>.yaml` in zen-gitops. Every image tag with author and timestamp. If timing aligns with the incident — the deployment is the likely cause.

**Standard rollback** — git revert PR, push, ArgoCD manual sync. Rolling update back to previous image. `maxUnavailable: 0` means old pods serve throughout. Zero downtime during rollback.

**If critical** — `argocd app rollback` with the last good history ID. 60 seconds. Always follow with `git revert` to fix drift.

After containment — root cause via Grafana timeline and pod logs. Fix goes through normal pipeline — feature branch, PR, CI, DEV, QA, prod. No shortcuts even under pressure. The pipeline exists for exactly these moments.

Post-incident: blameless post-mortem. Usually adds a Prometheus alert or test case so monitoring catches this class of issue before users do next time.

**Version B — Structured Reference**

- **Fast rollback (~60 sec):** `argocd app rollback <history-id>`. Always follow with `git revert`.
- **Post-incident:** Blameless post-mortem. Prometheus alert or test case added.
- The pipeline exists for high-pressure moments. Bypassing it to fix faster is how one bug becomes two incidents.

---

## 09 — Advanced 2026 Topics

### Q28 — What is Platform Engineering? How does it differ from DevOps?

**Version A — Candidate Voice**

DevOps is about culture and collaboration between dev and ops. Platform Engineering is about building an Internal Developer Platform — a self-service layer that lets developers deploy and operate without deep infrastructure knowledge.

In our project this already happens. Developers do not write CI pipelines — they call `_java-build.yml` with four parameters. Adding a service is creating a values file and ArgoCD Application manifest — under an hour. The platform team owns the pipeline contract, product teams own their service code. Golden Path — developers use it because it is easier, not because forced.

Our reusable workflow architecture is a platform pattern. Update Trivy version in one place, it applies to all 7 services immediately.

**Version B — Structured Reference**

- **DevOps:** Culture + collaboration. CI/CD, automation, monitoring practices.
- **Platform Engineering:** Internal Developer Platform. Self-service. Golden Path.
- **In our project:** `_java-build.yml` = platform pattern. New service = values file + ArgoCD manifest. < 1 hour.
- **DORA metrics:** Deployment frequency, Lead time, Change failure rate, MTTR.

---

### Q29 — What are SLOs, SLIs, and error budgets?

**Version A — Candidate Voice**

SLI is the measurement — P99 latency, availability. SLO is the target — 99.9% availability over 30 days. Error budget is what is left — at 99.9% you have 43 minutes of downtime per month.

In our project — NGINX Ingress Prometheus metrics give us the raw SLI data. The `kubeApiserverBurnrate` alert in `defaultRules` is a burn rate alert on the API server SLO. We have not formalised error budgets as a deployment gate yet — the observability foundation is in place to do it. That is an honest answer.

**Version B — Structured Reference**

- **SLI:** The measurement. Availability %, P99 latency, error rate.
- **SLO:** The target. 99.9% availability over 30 days.
- **Error budget:** 99.9% = 43 min/month downtime allowed. 99.99% = 4.3 min/month.
- **Our project:** NGINX metrics = SLI data. `kubeApiserverBurnrate` alert = SLO burn rate. Error budgets as deployment gate = roadmap.
- "We have the observability foundation but have not formalised error budgets yet" is honest and credible. Overclaiming full SRE at every project makes interviewers suspicious.

---

### Q30 — What is your DR strategy? RTO and RPO?

**Version A — Candidate Voice**

RTO is how fast you can recover. RPO is how much data you can lose.

Our platform — entire infrastructure is Terraform. Fresh EKS cluster takes 15 to 25 minutes. zen-gitops has all application config — ArgoCD re-syncs automatically after bootstrap. RTO approximately 30 to 45 minutes for full cluster rebuild.

Only stateful component is RDS. Automated backups with 7-day retention in prod. Multi-AZ for availability. Point-in-time restore means RPO under 5 minutes.

Honest gap — no cross-region DR. us-east-1 outage would require re-provisioning in a second region. Route53 failover and replica cluster would address it if the business required RTO under 30 minutes.

**Version B — Structured Reference**

- **RTO:** ~30-45 min (Terraform provision 15-25 min + ArgoCD bootstrap).
- **RPO:** <5 min for RDS (Multi-AZ + point-in-time restore). 7-day backup retention prod.
- EKS is stateless. All app state in RDS. K8s config in zen-gitops.
- **Gap:** Single region (us-east-1). No cross-region DR.
- **Roadmap:** Route53 failover + replica cluster in us-west-2 if RTO < 30 min required.
- "We accept us-east-1 as blast radius at current scale with a documented re-provision runbook" is more credible than overclaiming geographic redundancy.

---

### Q31 — How do you manage and optimise cloud costs (FinOps)?

**Version A — Candidate Voice**

Several deliberate choices.

Dev environment destroyed at end of day and re-provisioned when needed. EKS at $72 and NAT Gateway at $32 add up fast if idle. Destroy pipeline requires typing `destroy` as confirmation.

Right-sizing: t3.small in dev, t3.medium only in prod. HPA disabled in dev and QA.

ECR lifecycle policies keep last 10 images per repo. Without this, years of deployments accumulate unbounded.

Path filters eliminated ~85% of unnecessary CI builds — direct GitHub Actions minutes saving. Roadmap: Spot instances for dev node group could reduce EC2 costs by 60 to 70%.

**Version B — Structured Reference**

- **Dev env destroy end-of-day:** ~$104/month saved (EKS $72 + NAT GW $32). Typed confirmation required.
- **Right-sizing:** t3.small dev / t3.medium prod. HPA off dev+QA.
- **ECR lifecycle:** Last 10 images. Prevents unbounded storage growth.
- **Roadmap:** Spot instances dev (60-70% EC2 saving). Savings Plans prod.
- Knowing approximate cost of each resource and having deliberate controls signals FinOps maturity.

---

### Q32 — What is an SBOM and why does it matter?

**Version A — Candidate Voice**

SBOM — Software Bill of Materials. A machine-readable inventory of every component in your software: OS packages, language libraries, transitive dependencies, versions, licenses.

Why it matters: US Executive Order 14028 mandates SBOMs for software sold to US federal agencies. For a pharma application, regulators want to know exactly what is in your software. When a new CVE is published, an SBOM lets you instantly query which services are affected without re-scanning.

Trivy can generate SBOMs in CycloneDX or SPDX format with one flag. Adding it would be a one-line addition to the existing Trivy step. On our roadmap — trivial to add since the scanner is already there.

**Version B — Structured Reference**

- **SBOM:** Machine-readable inventory. OS packages, libs, transitive deps, versions, licenses.
- **EO 14028:** Mandates SBOMs for US federal software. Pharma regulatory relevance.
- **Generation:** `trivy --format cyclonedx` or `spdx`. One flag. Roadmap item.
- Knowing about SBOMs and having a clear implementation path shows 2026 compliance awareness.

---

### Q33 — If you could redesign this architecture from scratch, what would you do differently?

**Version A — Candidate Voice**

A few things with the benefit of hindsight.

**Separate AWS accounts per environment.** True blast radius isolation. Dev credentials cannot reach prod. Namespace isolation is logical — separate accounts is a hard boundary.

**Cluster Autoscaler from day one.** The Terraform inputs are wired but the controller is not. Pod scaling via HPA is great but if nodes are full, pods go Pending. Retroactive addition is straightforward but caused manual interventions during load tests.

**Network Policies enforced from the beginning.** Much harder to retrofit than to include in the initial Helm chart template. We have a gap — any pod can call auth-service directly.

**Custom HPA metrics from the start.** CPU is a blunt instrument. Queue depth would be a better scaling signal for batch-heavy services like manufacturing-service.

**Version B — Structured Reference**

- **Separate AWS accounts per env:** True blast radius isolation. Dev cannot reach prod. AWS Organizations + Control Tower.
- **Cluster Autoscaler from day one:** Terraform inputs ready, controller not wired. Prevented some manual intervention.
- **Network Policies from start:** Retrofit harder than initial inclusion. Gap: any pod can call auth-service.
- **Custom HPA metrics:** CPU blunt. Queue depth / request rate better for batch-heavy services.
- Self-evaluation scores well. Shows architectural maturity — ability to assess trade-offs and learn. Interviewers trust engineers who critique their own work.

---

### Q34 — How do you handle missing Kubernetes concepts? (PodDisruptionBudget, Network Policies)

**Version A — Candidate Voice**

I think it is important to be honest about gaps rather than overclaiming. There are a few things not implemented yet.

**Network Policies** — we do not have them. In a pharma application with compliance requirements, you would want policies stating only api-gateway can call auth-service. No other pod should reach auth-service directly. A genuine gap we would address before any compliance audit.

**PodDisruptionBudget** — not configured. We would add this for prod services before any EKS node group upgrade. A PDB with `minAvailable: 1` ensures at least one pod stays up during `kubectl drain`.

**Cluster Autoscaler** — Terraform inputs are wired, controller not deployed. Current node scaling is manual.

**Kyverno for image signature enforcement** — Cosign signing is in place but verification at pod admission is not yet implemented. The policy would reject unsigned images at the cluster level.

**Version B — Structured Reference**

- **Network Policies:** Not implemented. Gap: any pod can call auth-service. Roadmap before compliance audit.
- **PodDisruptionBudget:** Not configured. Would add for prod before node group upgrades. `minAvailable: 1`.
- **Cluster Autoscaler:** Terraform inputs wired. Controller not deployed. Node scaling still manual.
- **Kyverno admission:** Cosign signing in place. Enforcement at pod admission = roadmap.
- Saying "not implemented yet, here is the roadmap and the risk" is more credible than overclaiming. Interviewers can tell.

---

### Q35 — Walk me through exactly what happens — developer pushes a commit to user seeing the feature

**Version A — Candidate Voice**

Step by step.

1. Developer pushes to feat branch. GitHub Actions triggers lightweight CI — Maven tests, CodeQL, OWASP. No Docker build. Developer gets feedback in 5 minutes.
2. Developer raises PR targeting develop. Same CI runs on PR. Code review and approval. Merge to develop.
3. Full 9-stage pipeline triggers on develop push. Image `sha-a1b2c3d` built, pushed to ECR, signed with Cosign.
4. CI commits new image tag to `zen-gitops/envs/dev`. ArgoCD detects within 3 minutes and syncs dev namespace. Rolling update — new pods start, readiness probe passes, old pods terminate.
5. CI opens a QA promotion PR in zen-gitops. QA team tests in dev, reviews the PR, merges. ArgoCD auto-syncs QA namespace.
6. At sprint end, engineer triggers `promote-prod.yml` `workflow_dispatch`, selects service. Workflow reads QA image tag, opens prod PR. Two approvals required — Release Manager and QA Lead.
7. After merge, ArgoCD shows `pharma-prod` as OutOfSync. At maintenance window, engineer manually syncs in ArgoCD UI. Rolling update in prod. Readiness probes gate traffic to new pods. Feature is live. User sees it.

Every step is in Git with author, timestamp, and is reversible with `git revert`.

**Version B — Structured Reference**

Practice narrating this fluently in 3-4 minutes without notes. Common final interview question. Shows you understand the whole system.

**Diagram — End-to-End Flow**

```
Developer pushes feat/JIRA-101
         │
[5 min CI] Gitleaks + tests + CodeQL (no Docker)
         │
PR → develop → merge
         │
[15 min CI] all 9 stages → sha-a1b2c3d in ECR → Cosign signed
         │
zen-gitops/envs/dev/values-*.yaml: image.tag updated
         │
ArgoCD detects (3 min) → auto-sync dev namespace
         │
CI opens promote/qa/<svc>/<tag> PR in zen-gitops
         │
QA tests → PR reviewed → merge → ArgoCD auto-sync QA
         │
promote-prod.yml (workflow_dispatch, service dropdown)
Reads QA tag → creates branch → opens prod PR
         │
Release Manager + QA Lead approve → merge
         │
ArgoCD pharma-prod: OutOfSync
         │
Engineer: manual sync in ArgoCD UI at maintenance window
         │
Rolling update (maxUnavailable:0, readiness probes gate traffic)
         │
Feature live. User sees it.

Every step: author + timestamp in Git. Reversible with git revert.
```

---

### Q36 — How would you achieve SOC 2 Type II compliance with this pipeline?

**Version A — Candidate Voice**

The pipeline is already well-positioned. Let me split this into what is already done and what needs to be added.

**Already done:** The zen-gitops PR with approvals is change management evidence — auditable records of what changed, who approved, when. Cosign signatures in Rekor are a cryptographic record of image provenance. SAST, dependency scanning, and image scanning run on every build. GitHub Environments enforce Required Reviewers for production.

**What needs to be added:** Formalise change management — export zen-gitops PR links to Jira per deployment. Audit log retention — GitHub Actions logs and ArgoCD sync history need to be retained for 1 year, export to S3. Access reviews — quarterly review of who has prod Environment approval. Vulnerability SLA tracking — CVSS 9.0 remediated within 24 hours, CVSS 7.0 within 30 days. SBOM generation on every build.

**Version B — Structured Reference**

- **Already done:** zen-gitops PRs = change management evidence. Cosign = image provenance. SAST+scanning on every build. Required Reviewers for prod.
- **Additions needed:** Change records in Jira. Audit log retention (1 year in S3). Quarterly access reviews. Vulnerability SLA tracking. SBOM generation.
- **Honest gap:** Not SOC 2 certified yet. Foundation is in place. Additions are process + configuration, not architectural rebuilds.
- SOC 2 Type II is about demonstrating controls **over time**, not just having them. The GitOps model with git history as audit trail is already a significant advantage.

---

*DevOps Placement Program — Zen Pharma Project · 2026 Edition*
