# Test Pipeline App

A simple Node.js application used to demonstrate a production-like Jenkins CI/CD pipeline.

## Pipeline Flow

```text
Developer Push
      |
      v
GitHub App Repo
      |
      v
Jenkins Pipeline
      |
      ├── Checkout
      |
      ├── Parallel Code Validation
      |     ├── Unit Tests
      |     ├── Lint
      |     └── Dependency Audit
      |
      ├── Docker Build
      |
      ├── Image Smoke Test
      |
      ├── Trivy Image Scan
      |
      └── Push Docker Image