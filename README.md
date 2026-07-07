# VaultForge

A local, reusable DevSecOps pipeline that enforces security controls at every stage of the software delivery lifecycle — commit, build, image, deployment, and runtime — instead of relying on a single scan at the end.

The pipeline itself is the deliverable. [OWASP PyGoat](https://github.com/adeyosemanputra/pygoat), an intentionally vulnerable Django application, is used as the test subject to prove every stage actually detects real issues rather than reporting a false "clean" result.

---

## Why this exists

Most CI/CD pipelines answer one question: does the code build? That leaves five real attack surfaces unexamined:

| Stage | Attack surface | Question answered |
|---|---|---|
| Commit | Leaked credentials | Did a secret get committed? |
| Source code | Insecure code patterns | Does the code contain known vulnerability classes (injection, XSS, broken auth)? |
| Build artifact | Vulnerable dependencies | Does the container image ship known CVEs in its OS packages or libraries? |
| Deployment | Exploitable web surface | Does the *running* application expose attackable behavior? |
| Runtime | Post-compromise activity | If an attacker gets a foothold inside the container, does anything notice? |

VaultForge maps a tool to each stage, chosen because no single tool answers all five questions. The pipeline is built to be reusable — swapping the target application means changing three lines in an `env:` block, not rewriting the workflow.

## What it solves

- **Visibility**: every security layer produces a real, quantified result — not a pass/fail checkbox, but actual counts (62 critical CVEs, 52 SAST findings) with full detail reports attached to every run
- **Enforcement**: critical-severity findings block the pipeline by default, not just get logged and ignored
- **Traceability**: critical findings automatically open a GitHub Issue with commit and run references, so detection turns into a tracked action item
- **Reusability**: the same pipeline can scan any containerized application by changing a handful of environment variables

## Architecture

```
                              git push
                                 │
                                 ▼
                     ┌─────────────────────┐
                     │   Gitleaks           │   BLOCKING
                     │   secrets scan       │   (hard stop on match)
                     └──────────┬───────────┘
                                 │ pass
                 ┌───────────────┴────────────────┐
                 ▼                                ▼
       ┌──────────────────┐             ┌───────────────────────┐
       │  Semgrep          │  BLOCKING   │  Docker build          │  BLOCKING
       │  SAST              │  on ERROR   │  + Falco (CI runner    │
       │                    │  severity   │  supply-chain monitor) │
       └──────────────────┘             └──────────┬────────────┘
                                                     │ image built
                                        ┌────────────┴─────────────┐
                                        ▼                          ▼
                              ┌──────────────────┐       ┌──────────────────┐
                              │  Syft             │       │  Trivy            │
                              │  SBOM generation   │      │  image CVE scan   │  BLOCKING
                              └──────────────────┘       │                    │  on CRITICAL
                                                          └──────────┬───────┘
                                                                     │
                                                                     ▼
                                                          ┌──────────────────┐
                                                          │  OWASP ZAP        │  REPORTING
                                                          │  full active scan │
                                                          └──────────┬───────┘
                                                                     │
                                        ┌────────────────────────────┴───┐
                                        ▼                                ▼
                              ┌──────────────────┐          ┌──────────────────┐
                              │  Create GitHub    │          │  Security Summary │
                              │  Issue (if any     │          │  (score + table)   │
                              │  gate failed)      │          └──────────────────┘
                              └──────────────────┘

Deployed separately (not part of CI) to a local kind cluster:

    Helm install ──▶ PyGoat pod running ──▶ Falco (runtime syscall monitoring)
```

**BLOCKING** stages stop the pipeline on failure. **REPORTING** stages record findings and continue. The policy for each is explained below.

## Pipeline flow

The pipeline runs as a job DAG in GitHub Actions, not a flat sequence:

- `secrets-scan` runs first and gates everything downstream via `needs:`
- `sast-scan` (Semgrep) and `build-and-sbom` (Docker + Falco + Syft) run **in parallel** once secrets pass — Semgrep only needs source code, so it doesn't wait on a built image
- `image-scan` (Trivy) and `dast-scan` (ZAP) both depend on `build-and-sbom`, since both need the actual built artifact, but run in parallel with each other
- `create-issue-on-critical` runs only if the Trivy or Semgrep gate failed
- `security-summary` runs last regardless of outcome, aggregating every prior job's output into one report

This shape minimizes wall-clock time — independent checks don't wait on each other — while still enforcing that nothing runs against unscanned code or an unbuilt image.

## Failure policy

Not every finding should stop the pipeline, but some must:

- **Gitleaks (secrets) and the Docker build are hard gates.** A leaked credential or an image that doesn't build are unconditional failures.
- **Trivy blocks on any CRITICAL-severity CVE. Semgrep blocks on any ERROR-severity finding.** These are configurable via the `ALLOW_CRITICAL_CVES` repository variable — set to `true` to bypass while scanning a known-vulnerable demo target like PyGoat, where every run would otherwise fail by design. Against a real application, this stays unset and the gates enforce normally.
- **ZAP reports and continues.** DAST findings against PyGoat are expected on every run; the value here is visibility and quantification, not gatekeeping a known-vulnerable target.
- **A GitHub Issue is auto-created** whenever the Trivy or Semgrep gate actually fails, with the finding counts, commit SHA, and a direct link back to the run — closing the loop from detection to a trackable action item.

## Tools, and why each one is here

| Tool | Stage | Why this one |
|---|---|---|
| **Gitleaks** | Commit | Scans full git history, not just the latest diff — catches secrets committed and later "removed" but still present in history |
| **Semgrep** | Source code | Multi-language pattern matching with targeted rulesets (`p/owasp-top-ten`, `p/django`, `p/python`); chosen over Bandit for cross-language reuse beyond this one Python project |
| **Falco (CI mode)** | Build (pipeline itself) | Monitors the GitHub Actions runner during the build step for supply-chain anomalies — unexpected network calls, unusual process spawns — using Falco's `modern_ebpf` probe designed for CI environments |
| **Syft** | Build artifact | Generates a CycloneDX SBOM — a queryable inventory of every package in the image, so a newly disclosed CVE can be checked against the artifact instantly |
| **Trivy** | Build artifact | Scans OS packages and installed libraries against CVE databases — a different attack surface than Semgrep, which only sees application source |
| **OWASP ZAP** (full active scan) | Deployment | Crawls the running application and actively sends attack payloads (SQLi, XSS, command injection, path traversal) against every discovered input, then inspects responses for confirmed exploitation — not just passive header checks |
| **Falco (runtime)** | Runtime | Deployed via Helm into the kind cluster, monitoring syscalls inside the *running* pod — the only layer that detects post-deployment, in-progress compromise |
| **Helm** | Deployment packaging | Templated Kubernetes manifests with environment-specific values, replacing raw `kubectl apply` |

**Deliberately excluded:**
- **Prometheus / Grafana / Loki** — PyGoat exposes no application metrics endpoint, and the Grafana/Loki datasource auto-provisioning in the `loki-stack` Helm chart did not reliably populate. Falco's output remains fully accessible via `kubectl logs`.
- **ArgoCD** — GitOps sync solves configuration drift against a persistent deployment target. This project deploys to a local, ephemeral kind cluster, so the added component would not solve an actual problem here.

## Why PyGoat

The pipeline is the artifact under test, not the application. PyGoat is used as the scan target because it has known, documented OWASP Top 10 vulnerabilities, which makes it possible to verify that each scanner detects real issues rather than silently passing. A custom-written application risks looking clean simply because the author already avoids their own blind spots; PyGoat removes that bias.

The trade-off: findings are real, but not authored by me, so I can explain *why* a tool flagged something without having written the vulnerable pattern myself. Extending this pipeline to a small custom application with deliberately seeded vulnerabilities is the natural next step.

## Results

Current output from a full pipeline run against PyGoat:

| Stage | Result |
|---|---|
| Secrets (Gitleaks) | PASS |
| SBOM (Syft) | PASS |
| Container CVEs (Trivy) | 62 critical, 903 high |
| SAST (Semgrep) | 52 findings (14 critical) |
| DAST (ZAP, active scan) | Findings reported |
| **Security Score** | **0 / 100** |

The score starts at 100 and deducts per finding (−10 per critical CVE, −3 per high CVE, −2 per Semgrep finding, −10 for a failed DAST job). A score of 0 is the expected, correct outcome — PyGoat runs on an end-of-life base image with a large number of real unpatched CVEs, by design.

Both Trivy and Semgrep produce a structured markdown report on every run (not just raw JSON): a summary table, every CRITICAL/ERROR finding shown in full, and HIGH/WARNING findings grouped by package or rule inside a collapsible section — keeping a 900+ row scan result actually reviewable.

Cross-tool confirmation is visible in the data itself: Semgrep flags insecure deserialization in application code (`views.py`), and Trivy independently flags the vulnerable PyYAML dependency (three CRITICAL RCE-via-deserialization CVEs) that the code relies on — two different tools catching the same risk class from two different angles.

**Runtime detection, captured live:** to confirm Falco detects real in-progress activity, a shell was used to read `/etc/shadow` inside the running pod:

```
Warning: Sensitive file opened for reading
process=cat command=cat /etc/shadow user=root user_uid=0
container_name=vaultforge-app k8s_pod_name=vaultforge-app-586949f867-dtgr5
```

Falco correctly attributed the process, user, and exact pod. This also surfaced a separate finding: the container runs as root, worth remediating independently.

## Known limitations

- **Falco's kernel-module driver does not build against WSL2's custom kernel.** Resolved by switching to the `modern_ebpf` driver, which requires no kernel compilation.
- **DaemonSet pods (Falco) occasionally enter `Unknown` state** after Docker Desktop restarts or system sleep, due to a kubelet state-sync gap specific to kind-on-WSL2. `scripts/fix-stuck-pods.sh` force-deletes affected pods and lets the DaemonSet controller recreate them.
- **Falco's CI-runner monitoring (`falco-actions`) is newer and less battle-tested** than the rest of the stack; it's configured with `continue-on-error` so a failure there doesn't take down the build.
- **SQLite is baked into the image at build time**, so application data does not persist across pod restarts. Acceptable for a scanning demo; not a production data strategy.
- **Grafana/Loki datasource provisioning did not work reliably** via the `loki-stack` Helm chart's sidecar-based ConfigMap mechanism — dropped in favor of direct `kubectl logs` access to Falco output.
- **Signature-based tools (Semgrep, Trivy, ZAP) only catch known vulnerability classes and published CVEs.** They cannot detect zero-days or novel business-logic flaws. Falco's behavior-based runtime detection is the one layer here that doesn't require prior knowledge of the specific exploit technique — it flags anomalous behavior regardless of whether the underlying vulnerability was ever previously documented.

## Future improvements

- Extend the pipeline to a small custom application with deliberately seeded, fully-authored vulnerabilities, and compare detection results against PyGoat
- Resolve the Loki/Grafana datasource provisioning issue and restore log visualization
- Move the `ALLOW_CRITICAL_CVES` bypass to a per-severity threshold once a false-positive baseline is established against a real target application

## Local setup

```bash
git clone https://github.com/sanmathik8/vault-forge-.git
cd vault-forge-

# Create the kind cluster
kind create cluster --config kind/cluster-config.yaml

# Build and load the application image
docker build -t vault-forge-app:latest ./app
kind load docker-image vault-forge-app:latest --name vaultforge

# Deploy with Helm
helm install vaultforge ./helm/vaultforge-chart --kube-context kind-vaultforge

# Deploy Falco (runtime detection)
helm repo add falcosecurity https://falcosecurity.github.io/charts
kubectl create namespace falco --context kind-vaultforge
helm install falco falcosecurity/falco -f falco/values.yaml \
  --namespace falco --kube-context kind-vaultforge

# Access the application
kubectl port-forward svc/vaultforge-service 8000:8000 --context kind-vaultforge
# http://localhost:8000
```

**Requires:** Docker Desktop, WSL2 (if on Windows), `kind`, `helm`, and `kubectl` installed locally.

**To point the pipeline at a different application:** edit the `env:` block at the top of `.github/workflows/pipeline.yml` (`APP_PATH`, `IMAGE_NAME`, `APP_PORT`) — no other changes required.

The GitHub Actions pipeline runs on every push to `main`. See the **Actions** tab for live output, the Security Summary, and downloadable Trivy/Semgrep reports.
