# CI/CD & DevSecOps Architecture Decisions

This document outlines the design decisions and steps taken to build the CI/CD pipeline for the Sample Node.js application. 

## a. Version Bumping & Git Workflow
**Decision:** GitHub Flow with Git SHA versioning.
- **Workflow:** We assume a standard GitHub Flow where developers branch off `main`, open Pull Requests, and merge back into `main`. The pipeline runs on both PRs (to test and scan) and on merges to `main` (to build, push, and deploy).
- **Version Bumping:** Instead of manually bumping semantic versions in a `package.json` for every deployment, the pipeline tags Docker images with the exact **Git SHA** (e.g., `sha-a1b2c3d`). This is a DevOps best practice for continuous deployment because it guarantees that every image tag corresponds 1:1 with an exact, immutable commit in Git, eliminating any ambiguity about what code is running in production.

## b. DevSecOps (Security Checks)
We integrated security directly into the pipeline (`.github/workflows/ci-cd.yaml`) to fail fast if vulnerabilities are found.
- **SAST (Static Application Security Testing):** 
  - **Tool:** `njsscan` (An OWASP-recommended static analysis tool specifically tailored for Node.js).
  - **Implementation:** The pipeline runs `njsscan` on the source code. If it detects any critical or high-severity vulnerabilities (like hardcoded credentials, insecure regex, or bad crypto implementations), it exits with a non-zero code, blocking the build.
- **Bonus Security Tool:** 
  - **Tool:** `TruffleHog`. 
  - **Implementation:** Integrated to scan every commit for accidentally exposed API keys, tokens, or passwords.
- **Vulnerability Scanning (Docker Image):**
  - **Tool:** Aqua Security's `Trivy`.
  - **Implementation:** Before pushing the image to a registry, the pipeline builds it locally and scans the filesystem/dependencies inside the container. It is configured to explicitly block deployment (`exit-code: '1'`) if any `HIGH` or `CRITICAL` vulnerabilities are found in the OS packages or Node modules.

## c. Build and Dockerization
**Decision:** GitHub Actions + Plain Docker + GitHub Container Registry (GHCR)
- The pipeline uses standard Docker (`docker/build-push-action`) instead of Kaniko, matching your preference.
- It leverages Docker Buildx for efficient caching and cross-platform builds if needed.
- The built image is pushed to **GHCR (GitHub Container Registry)**, acting as our private registry. We authenticate seamlessly using the built-in `${{ secrets.GITHUB_TOKEN }}`.

## d. Deployment Stage (ArgoCD & GitOps)
**Decision:** A Separate GitOps Repository monitored by ArgoCD.
- **Why this method?** While ArgoCD can pull manifests directly from the app's repository, separating the application code from the infrastructure/deployment code is the industry standard GitOps best practice. 
  1. **Security:** CI pipelines only need permission to push to the GitOps repo, rather than having direct access to the Kubernetes cluster.
  2. **Clean Auditing:** Your GitOps repo becomes an exact ledger of what is deployed in production, free from application code noise.
  3. **No CI Loops:** Committing a new image tag to an app repo triggers the CI pipeline again (an infinite loop) unless complex ignore rules are added. A separate repo solves this natively.
- **Implementation:** 
  - The final step of the GitHub Actions pipeline checks out the hypothetical GitOps repo (`your-org/sample-nodejs-gitops`), uses `sed` to bump the image tag in the Helm `values.yaml` to the new Git SHA, and commits the change.
  - ArgoCD monitors this GitOps repo (as configured in `deploy/argocd/application.yaml`). When it sees the new commit, it automatically pulls the changes and applies them to the Kubernetes cluster, completing the deployment.

