# Plans

One plan doc per feature or roadmap stage. A plan is written and approved before any code for it is written, and its status is kept current.

## Deployment roadmap

Goal: host this app on a private S3 bucket behind CloudFront, deployed by GitHub Actions, built the way an enterprise team would.

| # | Plan | Status |
|---|------|--------|
| 01 | [GitHub-to-AWS identity (OIDC)](01-github-oidc.md) | Done |
| 02 | [Hosting infrastructure as code](02-hosting-infrastructure.md) | Planned |
| 03 | [Continuous deployment](03-continuous-deployment.md) | Planned |
| 04 | [Staging and production](04-staging-and-production.md) | Planned |
| 05 | [Operations: rollback, monitoring, guardrails](05-operations.md) | Planned |

## Target architecture

```
PR      -> CI: lint, test, build
main    -> CI -> deploy staging (automatic) -> approval gate -> deploy prod
              same build artifact promoted to both environments

browser -> CloudFront (HTTPS, security headers) -> private S3 bucket (OAC)
GitHub Actions -> AWS via OIDC role (no stored keys)
```

## Fixed choices

- CloudFormation for infrastructure
- Region `us-east-1`
- CloudFront default URL (`*.cloudfront.net`); a custom domain is an optional extra after stage 5

## Optional extras after stage 5

Custom domain with ACM and Route 53, WAF, per-PR preview environments, SPA route fallback once the app has a router.

## Cost and teardown

Free tier or cents at learning-project traffic. To remove everything: empty the buckets, delete the site stacks, then delete the OIDC stack.
