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

_To be written after the stage is complete._

## Stage 3 — Continuous deployment

_To be written after the stage is complete._

## Stage 4 — Staging and production

_To be written after the stage is complete._

## Stage 5 — Operations: rollback, monitoring, guardrails

_To be written after the stage is complete._
