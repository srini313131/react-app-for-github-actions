# 05 — Operations: rollback, monitoring, guardrails

**Status:** Planned

**Depends on:** 04

## Context

Shipping is half of deployment; the other half is what happens when a release is bad. This stage adds the three things a team needs before calling a pipeline production-ready: a fast way back to the last good version, a signal that something is wrong, and rules that stop unreviewed changes reaching `main`.

## Approach

1. **Rollback**
   - New workflow `.github/workflows/rollback.yml`, triggered manually (`workflow_dispatch`) with inputs `environment` and `ref` (a commit SHA or tag).
   - It checks out that ref, builds it, and calls the reusable deploy workflow from stage 4. Production rollback still passes through the `prod` approval.
   - S3 versioning (enabled in stage 2) is the safety net if a build can't be reproduced.
2. **Monitoring**
   - Add to `infra/site.yml`: a log bucket and CloudFront standard access logging, plus a CloudWatch alarm on the distribution's `5xxErrorRate`.
   - Alarm notifies an SNS topic with an email subscription.
3. **Guardrails**
   - Branch protection on `main`: pull request required, the `build` check must pass, no direct pushes.

## Hands-on steps for me

- Confirm the SNS email subscription
- Configure branch protection in the repository settings
- Run a rollback drill

## Files

- `.github/workflows/rollback.yml` (new)
- `infra/site.yml`
- `infra/github-oidc.yml` (role permissions for the new resource types)

## Verification

- Rollback drill: deploy a visible change, run the rollback workflow against the previous commit, and see the old version served
- Log files appear in the log bucket after some traffic
- The alarm exists and is in `OK`; setting its state manually with `aws cloudwatch set-alarm-state` sends the email
- A direct push to `main` is rejected; a PR with a failing test cannot be merged

## Interview topics

- "A bad deploy just went out. What do you do?" — detect, mitigate (roll back), then diagnose
- Rollback vs roll-forward, and when each is right
- What you would monitor for a static front end, and what a useful alarm looks like
- Blue/green and canary deployments at a conceptual level, and how CloudFront continuous deployment or weighted routing would implement them
- Why branch protection is part of deployment safety

## What a larger company does differently

Automatic rollback triggered by health checks, canary releases with gradual traffic shifting, real user monitoring and front-end error tracking, on-call rotation with paging, post-incident reviews.
