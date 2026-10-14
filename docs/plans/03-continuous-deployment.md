# 03 — Continuous deployment

**Status:** Done

**Depends on:** 01, 02

## Context

CI already lints, tests, builds, and uploads `dist` as an artifact on every push. This stage adds the deployment half: when a change reaches `main` and CI passes, the pipeline creates or updates the infrastructure and publishes that exact build. No one deploys from a laptop.

Only one environment exists at this point. It is named `staging` so stage 4 can add `prod` beside it without renaming anything.

## Approach

Changes to `.github/workflows/ci.yml`:

1. **Concurrency:** cancel in-progress runs only for pull requests. A deploy on `main` must never be cancelled halfway through a stack update.
2. **New `deploy` job**
   - `needs: build`; runs only on `push` to `main`
   - `permissions: id-token: write, contents: read` (the `id-token` permission is what lets the job request an OIDC token)
   - Steps:
     1. Check out the repo (for `infra/site.yml`)
     2. Download the `dist` artifact produced by the `build` job — the build is not repeated
     3. `aws-actions/configure-aws-credentials` with `role-to-assume: ${{ vars.AWS_ROLE_ARN }}`
     4. `aws cloudformation deploy --stack-name my-app-staging --template-file infra/site.yml --parameter-overrides Environment=staging --no-fail-on-empty-changeset`
     5. Read the stack outputs (bucket, distribution ID, domain)
     6. Upload hashed files: `aws s3 sync dist/assets s3://$BUCKET/assets` with `Cache-Control: public, max-age=31536000, immutable`
     7. Upload the rest, including `index.html`, with `Cache-Control: no-cache`, and `--delete` to remove files no longer in the build
     8. `aws cloudfront create-invalidation --paths "/*"`
     9. Write the site URL to the job summary
3. Remove the temporary identity check added in stage 1.

## Files

- `.github/workflows/ci.yml`
- `.github/workflows/aws-identity-check.yml` (deleted)

## Verification

- A pull request runs `build` only; `deploy` is skipped
- After merging to `main`: `build` then `deploy` are green and stack `my-app-staging` is `CREATE_COMPLETE` (first run takes 5-10 minutes while CloudFront provisions)
- `curl -I https://<DistributionDomainName>` returns 200 with the security headers from stage 2
- `curl -I` on the bucket's direct S3 URL returns 403
- Response headers show `immutable` on a file under `/assets/` and `no-cache` on `/index.html`
- Change visible text in `src/App.tsx`, merge, and see it at the CloudFront URL after the run

## Interview topics

- CI vs continuous delivery vs continuous deployment
- Build once, deploy the artifact: why the deploy job must not rebuild
- Cache strategy: content-hashed assets cached forever, `index.html` never cached, and why that makes deploys safe
- What a CloudFront invalidation does, what it costs, and why hashed filenames mostly make it unnecessary
- Why uploading assets before `index.html` avoids a window of broken pages

## What a larger company does differently

Artifacts stored in a versioned registry or bucket with retention, deployment tooling separated from CI (for example a CD system pulling approved artifacts), change records and deployment notifications to chat.

## Progress and changes from the plan

- Workflow written: `deploy` job added to `ci.yml`, concurrency changed, identity-check workflow deleted.
- Decided: old files under `/assets/` are not deleted on deploy. Their names contain a content hash, so they never clash with new ones, and a visitor who loaded the previous `index.html` can still fetch the files it points to. The cost is that they accumulate; a lifecycle rule would expire them.
- Decided: the deploy role is not given `cloudformation:DeleteStack`. If the first creation fails, the stack is left in `ROLLBACK_COMPLETE` and has to be deleted from a laptop before retrying: `aws cloudformation delete-stack --stack-name my-app-staging --region us-east-1`.
- The deploy role's permission list was enough on the first run: no `AccessDenied`, no change to `infra/github-oidc.yml`.
- First deploy (merge of pull request #7): stack `my-app-staging` reached `CREATE_COMPLETE` with no failed events. Site: https://d1qpoq0ceicfnl.cloudfront.net. Checked with `curl`:
  - `/` returns 200 with `strict-transport-security`, `x-frame-options: DENY`, `x-content-type-options: nosniff`, `referrer-policy`, and `cache-control: no-cache`
  - `/assets/index-CVtPK4EU.js` returns `cache-control: public, max-age=31536000, immutable`
  - `http://` returns 301 to `https://`
  - The bucket's direct S3 address returns 403
- Second deploy test: the heading in `src/App.tsx` changed to "Deployed by GitHub Actions" (test updated to match), to see a code change reach the live site through the pipeline.
