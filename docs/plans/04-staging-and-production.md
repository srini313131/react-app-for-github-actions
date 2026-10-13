# 04 — Staging and production

**Status:** Planned

**Depends on:** 03

## Context

Deploying every merge straight to the only environment is fine for a demo and unacceptable for a real product. Teams put at least one environment before production so a change is seen running before users get it, and they gate production behind an approval. The key rule is promotion: production receives the same artifact that was verified in staging, not a fresh build.

## Approach

1. **Second stack:** deploy `infra/site.yml` again as `my-app-prod` with `Environment=prod`. No template change, which is the payoff of parameterising it in stage 2.
2. **GitHub Environments:** create `staging` and `prod` in the repository settings. `prod` gets a required reviewer (me). Each environment holds its own `AWS_ROLE_ARN` variable.
3. **Per-environment roles:** update `infra/github-oidc.yml` to replace the single role with `my-app-deploy-staging` and `my-app-deploy-prod`. Each trusts only its GitHub Environment (`sub` = `repo:srini313131/react-app-for-github-actions:environment:staging` / `:environment:prod`) and can touch only its own stack and bucket. Redeploy the stack locally.
4. **Reusable workflow:** move the deploy steps into `.github/workflows/deploy.yml` (`on: workflow_call`, input `environment`). `ci.yml` calls it twice:
   - `deploy-staging`: `needs: build`, automatic
   - `deploy-prod`: `needs: deploy-staging`, `environment: prod`, so it pauses for approval
   Both download the same `dist` artifact from the run.

## Hands-on steps for me

- Create the two GitHub Environments and the required-reviewer rule
- `aws login`, then redeploy the OIDC stack
- Approve the first production deployment

## Files

- `infra/github-oidc.yml`
- `.github/workflows/deploy.yml` (new)
- `.github/workflows/ci.yml`

## Verification

- A merge to `main` deploys staging automatically, then waits at `deploy-prod` until approved
- Both CloudFront URLs serve the same build (same hashed asset filenames)
- Rejecting the approval leaves production unchanged
- The staging role cannot update the prod stack (confirmed by a deliberately wrong call, or by reading the policy)

## Interview topics

- Environment promotion and why the artifact must be identical across environments
- Environment parity, and how configuration that differs per environment is handled
- Approval gates: who approves, and what they should check first
- How the OIDC `sub` claim changes when a job targets a GitHub Environment
- Blast radius: why a staging credential must not be able to change production

## What a larger company does differently

A separate AWS account per environment, more environments (dev, QA, pre-prod), automated checks (smoke tests, synthetic monitoring) as the gate instead of a person, change-management approvals for production.
