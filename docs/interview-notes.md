# Interview notes

Notes written at the end of each stage, in terms of what was actually built in this project. Each section follows the same shape:

- **Concept** — the idea in two or three sentences
- **What we built** — the concrete version in this repo
- **Questions and answers** — likely interview questions with model answers
- **At larger scale** — what an enterprise does differently
- **Mock questions** — asked in the session; my answers and corrections

## Foundations: CI and project setup

_To be written — covers what exists already: the CI workflow, tests in CI, the build artifact, lockfile and Node version pinning._

## Stage 1 — GitHub-to-AWS identity (OIDC)

### Concept

A pipeline that deploys to AWS needs AWS credentials. Instead of storing an access key in GitHub, the pipeline proves who it is with a short-lived token signed by GitHub (OIDC), and AWS swaps that token for temporary credentials on a role. Nothing secret is stored anywhere, and the credentials expire on their own.

Plain version: the role is a visitor badge. The trust policy says who may borrow it; the permissions policy says which doors it opens. GitHub borrows it by showing a signed note saying which repository and branch the run belongs to.

### What we built

- `infra/github-oidc.yml`, a CloudFormation template deployed once from a laptop as stack `my-app-github-oidc`:
  - an OIDC provider, which makes the AWS account accept tokens signed by `token.actions.githubusercontent.com`
  - a role `my-app-github-deploy` whose trust policy requires `aud` = `sts.amazonaws.com` and `sub` = this repository on branch `main`
  - a permissions policy limited to the two site stacks, their buckets, and the CloudFront actions a deploy needs
- A repository variable `AWS_ROLE_ARN` holding the role's ARN (a variable, not a secret: an ARN is an address, not a credential)
- `.github/workflows/aws-identity-check.yml`, a temporary workflow that assumes the role and runs `aws sts get-caller-identity`

### Questions and answers

**How does your pipeline authenticate to AWS?**
Through OIDC federation. The workflow requests a signed token from GitHub, and the `configure-aws-credentials` action sends it to AWS STS with `AssumeRoleWithWebIdentity`. AWS checks the signature against the OIDC provider registered in the account and checks the token's claims against the role's trust policy, then returns credentials that last an hour. There are no access keys in GitHub.

**Why not just store an access key as a GitHub secret?**
A key is long-lived: it works until someone rotates it, from anywhere, for anyone who gets a copy. It can leak through logs, forks or a compromised action. With OIDC there is nothing to leak or rotate, the credentials expire in an hour, and the trust policy ties them to one repository and branch.

**What stops another repository from assuming your role?**
The `sub` condition in the trust policy. The token GitHub signs states the repository and ref of the run, and the role only accepts one exact value. Leaving the `sub` condition out, or using a broad wildcard, is the classic mistake: it would let any GitHub repository assume the role.

**What are `aud` and `sub`?**
Claims inside the token. `aud` (audience) is who the token is meant for, here `sts.amazonaws.com`, so a token issued for another service can't be replayed against AWS. `sub` (subject) is who the token is about: the repository and the branch, tag, pull request or environment the run came from.

**What is the difference between a trust policy and a permissions policy?**
The trust policy answers "who may assume this role". The permissions policy answers "what can the role do once assumed". A role needs both.

**How did you scope the role's permissions?**
Least privilege: CloudFormation actions only on the two named site stacks, S3 only on buckets with those stacks' name prefixes, and the specific CloudFront actions needed. I named the stacks rather than using a `my-app-*` wildcard, because the wildcard would also match the stack that defines the role itself, letting the pipeline change its own permissions. CloudFront is the exception, with `Resource: '*'`, because distribution IDs don't exist until creation.

**Why was this stack deployed by hand when everything else goes through the pipeline?**
It's a bootstrap problem: the pipeline can't create the role it needs in order to run. Identity is set up once by an administrator, and the pipeline is deliberately not allowed to modify it.

**Tell me about a deployment problem you debugged.**
The first run was refused with only "Not authorized to perform sts:AssumeRoleWithWebIdentity". I looked up the failed call in CloudTrail, which records the identity that was presented, and saw the `sub` was `repo:owner@<id>/repo@<id>:ref:refs/heads/main` — GitHub includes numeric owner and repository IDs, and my trust policy had only the names. The comparison is an exact string match, so it failed. I added the IDs to the template and redeployed. Lesson: when access is denied, find out exactly what was presented instead of guessing; CloudTrail has it.

**Why do the numeric IDs matter?**
Names can be released and re-registered by someone else; IDs can't. Matching on the ID closes the case where a deleted or renamed account's name is taken over.

### At larger scale

- One AWS account per environment under AWS Organizations, with a separate role in each
- Roles created by a platform team through a central pipeline, not by each application team
- Permission boundaries and service control policies capping what any deploy role can do, whatever its own policy says
- Trust tied to GitHub Environments rather than a branch, so production credentials require an approval (stage 4 here)
- Nobody works as the root user; people sign in through IAM Identity Center with MFA. This project's local CLI still uses root, which is the first thing to change in a real account

### Mock questions

_Asked in the session; answers and corrections to be added._

## Stage 2 — Hosting infrastructure as code

### Concept

A built React app is a folder of static files. The standard AWS way to host it is a private S3 bucket for storage with CloudFront in front as the only public entrance. CloudFront provides HTTPS, copies of the files close to users, and security headers; the bucket itself can't be reached from the internet. All of it is described in a template, so it can be reviewed, repeated for another environment, and removed cleanly.

Plain version: S3 is the stock room, CloudFront is the shop counter. Customers only ever meet the counter, and the stock room door opens only for this shop's counter staff.

### What we built

`infra/site.yml`, one template used for every environment (parameter `Environment`: `staging` or `prod`), with five resources:

- **Bucket** — all public access blocked, versioning on, encrypted, ACLs disabled
- **Origin Access Control (OAC)** — makes CloudFront sign its requests to S3
- **Response headers policy** — HSTS, `X-Content-Type-Options`, `X-Frame-Options: DENY`, `Referrer-Policy`
- **Distribution** — the public address; redirects HTTP to HTTPS, serves `index.html` at `/`, caches with the managed `CachingOptimized` policy, cheapest price class
- **Bucket policy** — allows `s3:GetObject` to the CloudFront service only when the request comes from this distribution

Outputs: bucket name, distribution ID, and the site's domain name — the three values the deploy job needs.

Verified by validating the template and previewing it with a change set, without creating anything.

### Questions and answers

**Why put CloudFront in front of S3 instead of using S3 static website hosting?**
S3 website hosting needs a public bucket and serves plain HTTP only. With CloudFront the bucket stays private, the site gets HTTPS, content is cached at edge locations near users, and there is one place to add security headers, a custom domain, or a firewall later.

**How does CloudFront read a private bucket?**
Through Origin Access Control. CloudFront signs each request to S3 with SigV4, and the bucket policy allows the `cloudfront.amazonaws.com` service to read objects, with a condition that `AWS:SourceArn` equals this distribution's ARN.

**What is the `SourceArn` condition for?**
Without it, the policy would allow the CloudFront service as a whole, meaning any distribution in any AWS account could be pointed at the bucket and read it. This is the confused deputy problem: a trusted service being used by the wrong party. The condition narrows the permission to one specific distribution.

**OAC or OAI?**
OAI (origin access identity) is the older mechanism. OAC replaces it: it uses short-lived signed requests, supports buckets encrypted with KMS keys, and works in all regions. AWS recommends OAC for anything new.

**Why is versioning turned on?**
Each deploy overwrites files. With versioning, the previous copies are kept, so a bad or accidental overwrite or delete can be undone. It is the fallback when rebuilding an old release isn't possible.

**What do the security headers do?**
HSTS tells browsers to use only HTTPS for the site. `X-Content-Type-Options: nosniff` stops browsers guessing file types. `X-Frame-Options: DENY` stops other sites embedding the page, which prevents clickjacking. `Referrer-Policy` limits what the site reveals about itself when users follow links out.

**What does infrastructure as code give you over clicking in the console?**
The setup is in version control, so changes are reviewed in pull requests and there is a history. It is repeatable: staging and production come from the same template, so they can't quietly differ. It is removable as one unit. And changes can be previewed before they happen.

**What is a change set?**
A preview. CloudFormation compares the template with what exists and lists what it would add, change or replace, without doing it. It matters most for updates, where a small-looking edit can mean a resource is replaced.

**Why one template with a parameter instead of one template per environment?**
So the environments stay identical in structure. Anything that differs has to be a visible parameter. Separate copies drift apart over time, and then staging stops being a reliable rehearsal for production.

**Why doesn't the bucket have a fixed name?**
Bucket names are unique across all of AWS, so a fixed name can collide with someone else's, and a named resource can't be replaced by CloudFormation without first being deleted. Letting CloudFormation generate the name avoids both; anything that needs the name reads it from the stack's outputs.

### At larger scale

- A custom domain with an ACM certificate, and DNS in Route 53
- AWS WAF attached to the distribution for rate limiting and common attack rules
- Access logs sent to a central logging account
- A Content-Security-Policy header, tuned to the app
- Shared building blocks (CDK constructs or Terraform modules) owned by a platform team, so each app doesn't write its own template
- A lifecycle rule expiring old object versions, so versioning doesn't grow storage forever

### Mock questions

_Asked in the session; answers and corrections to be added._

## Stage 3 — Continuous deployment

### Concept

Every change merged to `main` is built, tested and published to a live environment automatically, with no one deploying from a laptop. The build happens once; the deploy publishes exactly those files.

Plain version: the delivery worker from Stage 1 now has a route. When new stock passes inspection, the worker borrows the badge, makes sure the shop exists, puts the stock on the shelves, and tells the counter to stop handing out the old copies.

### What we built

A `deploy` job in `.github/workflows/ci.yml`, run after `build` and only on pushes to `main`:

1. Download the `dist` artifact the build job produced (no rebuild)
2. Borrow the AWS role through OIDC
3. `aws cloudformation deploy` stack `my-app-staging` from `infra/site.yml` — creates it the first time, updates it if the template changed, does nothing otherwise
4. Read the stack's outputs (bucket name, distribution ID, domain)
5. Upload `assets/` with `Cache-Control: public, max-age=31536000, immutable`
6. Upload everything else, including `index.html`, with `Cache-Control: no-cache`, deleting files no longer in the build
7. Invalidate CloudFront's cache
8. Write the site URL to the run summary

Runs on `main` are never cancelled by a newer run; pull request runs still are. Live at https://d1qpoq0ceicfnl.cloudfront.net.

### Questions and answers

**What is the difference between continuous integration, continuous delivery and continuous deployment?**
Continuous integration: every change is merged often and automatically built and tested. Continuous delivery: every change that passes is ready to release, but a person decides when. Continuous deployment: every change that passes goes live with no manual step. This stage is continuous deployment to staging; stage 4 adds an approval before production, which makes production continuous delivery.

**Why doesn't the deploy job build the app again?**
Build once, deploy that artifact. If deploy rebuilt, the files going live wouldn't be the files that were tested: dependencies or the build environment could differ between the two runs. Passing the artifact between jobs guarantees the tested build is the released build, and in stage 4 the same artifact is promoted to production.

**Why are some files cached for a year and `index.html` not at all?**
Vite puts a hash of each file's content in its name, like `index-CVtPK4EU.js`. A changed file gets a new name, so a given name always means the same content and browsers can keep it forever. `index.html` keeps its name on every deploy, so browsers must check for a new copy each visit; it's tiny, and it's what points to the current asset names.

**Why upload the assets before `index.html`?**
If `index.html` went first, a visitor could load a page that references asset files not uploaded yet, and get a broken page. Assets first means the new page only becomes visible once everything it needs is there.

**What does the CloudFront invalidation do, and is it needed?**
It tells the edge locations to drop their cached copies so the next request fetches from S3. With hashed file names it's only really needed for `index.html` and other un-hashed files. The first 1,000 invalidation paths a month are free; `/*` counts as one.

**Why are runs on `main` never cancelled?**
Cancelling mid-deploy could stop CloudFormation halfway through an update or leave half the files uploaded. For pull requests, cancelling an older run when a newer commit arrives just saves time.

**How does the job find the bucket name without hard-coding it?**
The bucket's name is generated by CloudFormation, and the template exposes it as a stack output. The job reads the outputs after deploying. Nothing about the infrastructure is copied into the workflow, so the two can't fall out of step.

**What happens if the stack fails to create the first time?**
CloudFormation rolls back and leaves the stack in `ROLLBACK_COMPLETE`, which can't be updated, only deleted. The pipeline role deliberately can't delete stacks, so an administrator deletes it and the job is re-run.

**How did you verify the deployment?**
Checked the response headers with `curl`: 200 with the security headers, `no-cache` on `index.html`, `immutable` on hashed assets, a 301 from HTTP to HTTPS, and a 403 from the bucket's direct address, proving it's only reachable through CloudFront. Then made a visible code change and watched it reach the site through the pipeline.

### At larger scale

- Artifacts stored in a versioned registry or bucket, so any past release can be redeployed exactly
- A separate deployment system pulling approved artifacts, rather than deploy steps inside the CI workflow
- Automated smoke tests after deploy, and automatic rollback if they fail
- Deployment notifications to chat and change records for audit
- A lifecycle rule expiring old hashed assets after a few weeks

### Mock questions

_Asked in the session; answers and corrections to be added._

## Stage 4 — Staging and production

### Concept

Changes reach users in two steps. Every merge deploys to staging automatically, where it can be seen running; production then waits for a person to approve, and receives the exact same build that staging got. Each environment has its own AWS role, so a credential for staging can't change production.

Plain version: one recipe card for "stock the shop", used for two shops. The rehearsal shop is restocked as soon as new stock passes inspection; the real shop is restocked with the same boxes only after a manager signs off, and each shop's key opens only its own door.

### What we built

- **Two stacks from one template:** `my-app-staging` and `my-app-prod`, both from `infra/site.yml`, differing only in the `Environment` parameter
- **Two GitHub Environments:** `staging` (no rules) and `prod` (required reviewer); both accept deployments from `main` only, and each holds its own `AWS_ROLE_ARN` variable
- **Two roles:** `my-app-deploy-staging` and `my-app-deploy-prod`. Each trusts only `sub` = `repo:…:environment:<name>` and can change only its own stack and bucket
- **A reusable workflow:** `.github/workflows/deploy.yml` (`on: workflow_call`, input `environment`). `ci.yml` calls it as `deploy-staging` after `build`, then as `deploy-prod` after staging succeeds. Both download the same `dist` artifact
- Per-environment `concurrency`, so two deploys to the same environment never overlap

Verified: the prod job paused for approval; both sites serve the same hashed asset file; the IAM policy simulator shows the staging role is denied on prod resources.

### Questions and answers

**Walk me through how a change reaches production.**
A pull request runs build and tests only. On merge to `main`, the build job produces one artifact. The staging deploy picks it up automatically. If that succeeds, the production job starts but pauses on the `prod` environment's approval rule. A reviewer checks staging and approves, and the same artifact is deployed to production.

**Why must production get the same artifact as staging?**
Staging is only evidence if production runs exactly what was tested there. Rebuilding for production could produce something different — a new dependency version, a different build machine — and then staging proved nothing. We verified it: both sites load `index-CLkFBzqh.js`.

**How does one workflow deploy to two environments?**
It's a reusable workflow, like a function with an `environment` parameter, called twice. The input picks the stack name and the CloudFormation parameter. It also sets the job's `environment:`, which makes GitHub apply that environment's protection rules, expose that environment's variables (so `vars.AWS_ROLE_ARN` resolves to a different role), and put `environment:<name>` in the OIDC token.

**How does the OIDC `sub` claim change when a job uses an environment?**
Without an environment it names the branch (`ref:refs/heads/main`). With one, it names the environment (`environment:prod`). That's why the prod role can trust "the prod environment" — and the prod environment only lets jobs through from `main` and after approval. The trust policy and GitHub's rules together form the gate.

**Why separate roles per environment instead of one role for both?**
Blast radius. If staging's credentials were misused, or a staging deploy went wrong, it can't touch production. Least privilege applies per environment, not just per application.

**Is the separation complete in your setup?**
Not quite. S3 and CloudFormation are separated by resource name, but CloudFront permissions are `Resource: '*'` in both roles, because distribution IDs don't exist until creation. Full separation would come from a separate AWS account per environment, which is the standard enterprise answer.

**What should an approver check before approving production?**
That staging deployed successfully, the change works in staging, nothing unexpected is in the diff, and it's a sensible time to release. An approval with no check is just a delay; the value is the look at staging.

**What happens if the approval is rejected?**
The production job doesn't run, production stays on the previous version, and staging keeps the new one. The next merge produces a new run with a new approval request.

**How do you handle configuration that differs between environments?**
Keep the structure identical and make the differences explicit inputs: here the `Environment` parameter in the template and the `AWS_ROLE_ARN` variable per GitHub Environment. Anything not expressed as a parameter is guaranteed identical.

**How did you replace the old role without breaking deploys?**
Added the new roles alongside the old one first, switched the workflow over, confirmed in CloudTrail that runs were using the new roles, and only then removed the old one. Expand, migrate, then contract.

### At larger scale

- A separate AWS account per environment under AWS Organizations, so isolation doesn't depend on resource-name patterns
- More environments: dev, QA, pre-production
- Automated gates — smoke tests, synthetic checks, error-rate comparisons — instead of, or before, a human approval
- Change-management records and approval by a release manager or change advisory board
- Progressive delivery in production: canary releases to a small percentage of users first

### Mock questions

_Asked in the session; answers and corrections to be added._

## Stage 5 — Operations: rollback, monitoring, guardrails

_To be written after the stage is complete._
