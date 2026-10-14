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

_To be written after the stage is complete._

## Stage 4 — Staging and production

_To be written after the stage is complete._

## Stage 5 — Operations: rollback, monitoring, guardrails

_To be written after the stage is complete._
