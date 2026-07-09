# Deployment Pipeline Analysis

## Task 1: Analysis of the Unstable Pipeline

### Problems Identified

#### 1. Pipeline triggers on every branch
The original workflow is configured with:

```yaml
branches:
  - '*'
```

This causes deployments from every branch, including feature and experimental branches. Production deployments should only occur from trusted branches such as `main`.

---

#### 2. Deployment happens before validation
The pipeline deploys the application before running linting and tests. This means broken or untested code can reach production.

---

#### 3. No dedicated Source stage
Code checkout is performed inside the deployment job instead of having a separate Source stage that records the commit being built.

---

#### 4. Missing Build stage
There is no dedicated build stage. Dependency installation and deployment are combined into a single job, making failures harder to isolate.

---

#### 5. Incorrect dependency installation
The workflow uses:

```bash
npm install
```

A CI pipeline should use:

```bash
npm ci
```

because it performs a clean, reproducible installation using the lock file.

---

#### 6. Missing Security stage
The original pipeline performs no:

- Dependency vulnerability scanning
- Secret scanning
- Static application security testing (SAST)

This allows vulnerable code to reach production.

---

#### 7. No staging deployment
The application is deployed directly to production without first validating it in a staging environment.

---

#### 8. Missing approval gate
Production deployments happen automatically without any manual approval from reviewers.

---

#### 9. Weak smoke testing
The smoke test ignores failures using:

```bash
curl ... || echo "Smoke test failed, continuing anyway"
```

This allows unhealthy deployments to remain in production.

---

#### 10. No rollback mechanism
The pipeline has no rollback strategy if verification fails after deployment.

---

#### 11. Poor failure isolation
Most operations occur inside one deployment job, making it difficult to identify exactly where failures occur.

---

## Improved Pipeline Design

| Stage | Purpose | Gate Condition |
|--------|---------|----------------|
| Source | Checkout repository and capture commit | Successful checkout |
| Build | Install dependencies, lint and build | Build and lint succeed |
| Test | Execute automated tests | All tests pass |
| Security | Dependency audit and secret scan | No high or critical issues |
| Deploy Staging | Deploy to staging | Previous stages succeed |
| Deploy Production | Production deployment | Staging succeeds and approval is granted |
| Verify | Smoke tests and health checks | Application is healthy |

---

## Stage Dependencies

Source

↓

Build

↓

Test

↓

Security

↓

Deploy Staging

↓

Deploy Production

↓

Verify

Each stage depends on the successful completion of the previous stage using the `needs` keyword. If any stage fails, downstream stages do not execute.

---

## Improvements Implemented

- Added sequential pipeline stages using `needs`
- Replaced `npm install` with `npm ci`
- Added lint and test validation before deployment
- Added security scanning
- Introduced staging before production
- Added production approval through GitHub Environments
- Added smoke tests after deployment
- Added rollback step on verification failure
- Improved traceability by recording commit SHA and timestamps
- Improved failure isolation by separating responsibilities into independent jobs

---

## Conclusion

The redesigned pipeline follows DevOps best practices by enforcing validation gates before deployment. Every stage has a clearly defined purpose, failures are isolated, production deployments are protected through approvals, and post-deployment verification ensures only healthy releases remain in production.