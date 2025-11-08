# Microblog - Jenkins CI/CD Pipeline

Flask application with a Jenkins CI/CD pipeline for containerized deployment.

## Git Strategy

This repository follows GitHub Flow with the following principles:

- `main` branch reflects the deployed production version
- All changes require pull requests before merging to `main`
- Container images only push to the registry on `main` branch
- Each merge to `main` is automatically deployable

## Jenkins Pipeline

Declarative pipeline with parallel security scanning and container validation.

### Pipeline Stages

| Stage | Purpose | Tool |
|-------|---------|------|
| **Checkout** | Fetch source code | git |
| **Unit Tests** | Run pytest suite | pytest |
| **Security & Linting (Parallel)** | | |
| - SAST Scanning | Scan code for vulnerabilities | Bandit |
| - Secret Scanning | Detect hardcoded secrets & credentials | TruffleHog |
| - Dockerfile Linting | Validate best practices | Hadolint |
| **Build Docker Image** | Create container image | docker build |
| **Container Validation (Parallel)** | | |
| - Container Smoke Test | Verify app runs & health checks | docker run + HTTP test |
| - Container Vulnerability Scan | Scan for CVEs | Trivy |
| **Push to Registry** | Push to GHCR (main branch only) | docker push |

### Generated Artifacts (stored in Jenkins)

- `test-results.xml` - JUnit test results
- `bandit-report.json` - SAST security findings
- `trufflehog-report.json` - Secret scanning results
- `hadolint-report.txt` - Dockerfile lint issues
- Docker images (pushed to GHCR on main branch):
  - `ghcr.io/{user}/microblog:{BUILD_NUMBER}` - Build number tag
  - `ghcr.io/{user}/microblog:{GIT_COMMIT_SHA}` - Immutable Git commit SHA tag
## About Microblog

This is an example application featured in the [Flask Mega-Tutorial](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-i-hello-world). See the tutorial for instructions on how to work with it.

The version of the application featured in this repository corresponds to the 2024 edition of the Flask Mega-Tutorial. You can find the 2018 and 2021 versions of the code [here](https://github.com/miguelgrinberg/microblog-2018). And if for any strange reason you are interested in the original code, dating back to 2012, that is [here](https://github.com/miguelgrinberg/microblog-2012).
## Dependencies

The application requires:
- PostgreSQL database
- Redis cache


