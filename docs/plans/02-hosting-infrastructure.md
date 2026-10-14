# 02 — Hosting infrastructure as code

**Status:** Done

**Depends on:** 01

## Context

The built app is a folder of static files (`dist/`). The standard AWS pattern for hosting it is a private S3 bucket as storage with CloudFront in front as the only public entry point: CloudFront gives HTTPS, caching at edge locations, and security headers, while the bucket stays unreachable from the internet. Defining this in a CloudFormation template means the infrastructure is reviewable, repeatable for a second environment, and removable in one command.

## Approach

Write `infra/site.yml`, parameterised by `Environment` (`staging` or `prod`) so stage 4 can reuse it unchanged:

- `AWS::S3::Bucket` — all public access blocked, versioning enabled, default encryption
- `AWS::CloudFront::OriginAccessControl` — lets CloudFront sign its requests to S3
- `AWS::CloudFront::ResponseHeadersPolicy` — security headers (HSTS, `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`)
- `AWS::CloudFront::Distribution`
  - S3 origin using the OAC
  - `DefaultRootObject: index.html`
  - Viewer protocol policy: redirect HTTP to HTTPS
  - Managed `CachingOptimized` cache policy
  - `PriceClass_100`
- `AWS::S3::BucketPolicy` — allows `s3:GetObject` to the `cloudfront.amazonaws.com` service principal only when `AWS:SourceArn` is this distribution
- Outputs: `BucketName`, `DistributionId`, `DistributionDomainName`

No SPA route fallback (403/404 -> `index.html`) yet: the app has no client-side router.

The stack is not deployed in this stage by hand; stage 3's pipeline deploys it. This stage is writing and validating the template.

## Files

- `infra/site.yml` (new)

## Verification

- `aws cloudformation validate-template --template-body file://infra/site.yml` passes
- A change set created against a new stack name lists the expected five resources (create the change set, review it, delete it without executing)

## Interview topics

- Why put CloudFront in front of S3 instead of using S3 static website hosting
- OAC vs the older OAI vs a public bucket
- How the bucket policy's `SourceArn` condition prevents another distribution reading the bucket (confused deputy)
- What infrastructure as code gives you: review, repeatability, drift detection, change sets

## What a larger company does differently

Custom domain with an ACM certificate, WAF in front of CloudFront, access logging to a central logging account, shared modules (CDK constructs or Terraform modules) owned by a platform team rather than a template per app.

## Progress and changes from the plan

- `infra/site.yml` written. `validate-template` passes.
- Change set check done against a throwaway stack name (`my-app-site-template-check`) rather than `my-app-staging`, so the real stack name stays untouched until the pipeline creates it. The change set listed the five expected resources (bucket, bucket policy, origin access control, response headers policy, distribution). It was not executed, and the empty stack shell was deleted afterwards.
- Added beyond the plan: `OwnershipControls: BucketOwnerEnforced` on the bucket (turns ACLs off, so access is decided by the bucket policy alone), HTTP/2 and HTTP/3, IPv6, and compression on the distribution.
- The bucket has no fixed name. CloudFormation generates `<stack name>-sitebucket-<random>`, which is globally unique and falls inside the `my-app-staging-*` / `my-app-prod-*` prefixes the deploy role is allowed to manage.
- Nothing is deployed yet. The first real creation happens in stage 3 through the pipeline, which is also the first real test of the deploy role's permissions.
