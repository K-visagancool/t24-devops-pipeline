# DevSecOps — Container Security Scanning for T24

Documentation of the DevSecOps practices and multi-layer security scanning approach for containerised T24/Transact deployments on Azure. This covers image scanning with Trivy, code quality with SonarQube, and continuous registry/runtime scanning with Azure Defender.

## The "Shift Left" Principle

Traditional security places checks at the end of the delivery process — after build, test, and deploy. DevSecOps shifts security "left" (earlier) so vulnerabilities are caught during build rather than in production, where they are far more expensive and risky to fix. For banking workloads this early detection is essential for compliance.

## Multi-Layer Scanning Architecture

Security scanning is not a single step — it happens at multiple layers for defense in depth:

```
Developer commits code
        |
        v
[SonarQube / SonarLint]  -> scans source CODE quality + security
        |
        v
Build container image
        |
        v
[Trivy]                  -> scans IMAGE before push (pipeline gate)
        |
        v
Push to Azure Container Registry
        |
        v
[ACR / Defender for Cloud] -> continuous scan of stored images
        |
        v
Deploy to AKS
        |
        v
[Defender for Containers] -> runtime scan of running pods
```

## Layer 1 — Trivy (Image Scanning in Pipeline)

Trivy scans container images for known vulnerabilities in OS packages and application dependencies, checking against a database of CVEs (Common Vulnerabilities and Exposures).

### Manual scanning (investigation)

```bash
# Scan an image
trivy image myapp:1.0

# Filter to serious issues only
trivy image --scanners vuln --severity CRITICAL,HIGH myapp:1.0
```

### Automated scanning (pipeline gate)

In the CI/CD pipeline, Trivy runs after build and before push, with a gate that fails the build on critical findings:

```yaml
- name: Security scan with Trivy
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myregistry.azurecr.io/t24-app:${{ github.sha }}
    exit-code: '1'          # fail the pipeline if issues found
    severity: 'CRITICAL'    # block only on critical
    scanners: 'vuln'
```

The key is `exit-code: '1'` — if critical vulnerabilities are found, Trivy exits non-zero, the pipeline step fails, and the push/deploy steps never run. Insecure images are blocked before reaching the registry.

### Why scan before push

Scanning after build but before push ensures vulnerable images never enter the registry, so no one can accidentally pull and deploy them.

### Understanding severity levels

Vulnerabilities are ranked CRITICAL, HIGH, MEDIUM, and LOW. A common gate blocks on CRITICAL (and sometimes HIGH), while MEDIUM and LOW are tracked but not blocking, to balance security with delivery speed.

### Reducing vulnerabilities

- Use smaller or hardened base images (alpine, distroless) — fewer packages means fewer vulnerabilities
- Rebuild regularly to pick up patched base images
- Only include what the application needs

## Layer 2 — SonarQube (Code Scanning)

While Trivy scans built images, SonarQube analyses source code for bugs, code smells, security vulnerabilities, duplications, and test coverage.

### Developer level (manual/local)

Developers use SonarLint, an IDE plugin (VS Code, IntelliJ), which highlights issues in real time as they write code — catching problems before they even commit.

### Team level (automated/pipeline)

SonarQube runs in the pipeline on every commit or pull request, applying a Quality Gate — pass/fail criteria such as zero new vulnerabilities or minimum code coverage. If the gate fails, the merge is blocked.

```yaml
- name: SonarQube scan
  uses: sonarsource/sonarqube-scan-action@master
  env:
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

Best practice combines both: developers catch issues early with SonarLint, and the pipeline enforces quality with the SonarQube gate.

## Layer 3 — Registry & Runtime Scanning (Azure Defender)

Trivy scans an image once, at build time. But new CVEs are discovered continuously — an image that was clean at build can become vulnerable weeks later. Azure Defender for Cloud addresses this:

- **Registry scanning** — continuously rescans images stored in ACR against newly discovered CVEs, alerting when a previously clean image becomes vulnerable
- **Runtime scanning (Defender for Containers)** — monitors running workloads in AKS for threats and vulnerabilities

This is critical for banking: a deployed T24 image scanned clean months ago could be affected by a new Log4j-style vulnerability, and continuous scanning surfaces it immediately.

## Trivy vs SonarQube vs Defender

| Tool | Scans | When | Blocks |
|------|-------|------|--------|
| SonarQube / SonarLint | Source code | Commit / PR | Quality gate |
| Trivy | Container image | Build (pre-push) | Pipeline gate |
| ACR / Defender | Stored image | Continuous | Alerts |
| Defender for Containers | Running pods | Runtime | Alerts |

## Manual vs Automated

Both tools support manual and automated use:

- **Manual** — an engineer runs Trivy for ad-hoc investigation, or a developer runs SonarLint locally. Useful for investigation and audits but inconsistent and unenforced.
- **Automated** — scanning built into the pipeline. Every image and every commit is scanned consistently, enforced, and logged. This is the goal.

Many environments still rely on manual or periodic security audits separate from deployment. Integrating scanning into the pipeline — so it is automatic, consistent, and enforced — is a key DevSecOps improvement, especially for regulated banking environments.

## Summary

Effective DevSecOps for T24 layers multiple scanning stages: SonarQube for code quality, Trivy as a pipeline gate for image vulnerabilities, and Azure Defender for continuous registry and runtime scanning. Together they provide defense in depth — catching issues before deployment, continuously while stored, and at runtime — which is exactly the security posture banking compliance requires.
