# 01 — GitHub-to-AWS identity (OIDC)

**Status:** Done

## Context

GitHub Actions has to create and update AWS resources, so it needs AWS credentials. The old way is an IAM user's access key stored as a GitHub secret: long-lived, easy to leak, and rarely rotated. The current practice is OIDC federation: GitHub issues a short-lived signed token for each workflow run, and AWS exchanges it for temporary credentials on a role that trusts only this repository. Nothing secret is stored in GitHub.

This is the one stage deployed from a local machine, because the pipeline can't grant itself access.

## Approach

1. Check whether the account already has a GitHub OIDC provider (`aws iam list-open-id-connect-providers`); an account can only have one per issuer URL.
2. Write `infra/github-oidc.yml` (CloudFormation):
   - `AWS::IAM::OIDCProvider` for `https://token.actions.githubusercontent.com`, audience `sts.amazonaws.com` (created only if step 1 found none)
   - `AWS::IAM::Role` `my-app-github-deploy`
     - Trust policy: `sts:AssumeRoleWithWebIdentity` from the provider, with `sub` restricted to `repo:srini313131/react-app-for-github-actions:ref:refs/heads/main`
     - Permissions (least privilege): CloudFormation on stacks named `my-app-*`, S3 on buckets named `my-app-*`, CloudFront distribution, origin access control, response headers policy and invalidation actions
   - Output: the role ARN
3. Deploy it locally: `aws cloudformation deploy --stack-name my-app-github-oidc --template-file infra/github-oidc.yml --capabilities CAPABILITY_NAMED_IAM`
4. Store the role ARN as a GitHub Actions repository variable `AWS_ROLE_ARN`.
5. Add a temporary workflow step that assumes the role and runs `aws sts get-caller-identity`, to prove the trust works before building on it.

## Hands-on steps for me

- `aws login` (session is expired)
- Install and authenticate `gh`, or set the repository variable in the browser (Settings -> Secrets and variables -> Actions -> Variables)

## Files

- `infra/github-oidc.yml` (new)
- `.github/workflows/aws-identity-check.yml` (new, temporary identity check)

## Verification

- `aws cloudformation validate-template --template-body file://infra/github-oidc.yml` passes
- Stack `my-app-github-oidc` reaches `CREATE_COMPLETE`
- A workflow run on `main` prints the assumed-role ARN from `aws sts get-caller-identity`
- The same step on a non-`main` branch is refused by AWS (proves the `sub` condition)

## Interview topics

- OIDC federation vs long-lived access keys
- What the `sub` and `aud` claims are and what the trust policy condition restricts
- Least privilege, and how you would scope a deploy role
- Why the root user should not be used for day-to-day work (the local CLI currently logs in as root)

## What a larger company does differently

Separate AWS accounts per environment under AWS Organizations, roles created by a platform team through a central pipeline, permission boundaries and SCPs limiting what any deploy role can do.

## Progress and changes from the plan

- Step 1 done: the account had no OIDC providers, so the template always creates one (no conditional needed).
- Step 2 done: `infra/github-oidc.yml` written and validated.
- Changed: CloudFormation permissions are limited to the two named stacks (`my-app-staging`, `my-app-prod`) instead of `my-app-*`. The wildcard would also have matched this stack (`my-app-github-oidc`), letting the pipeline attempt changes to its own identity setup.
- Changed: S3 permissions are limited to buckets named `my-app-staging-*` and `my-app-prod-*`, matching the names CloudFormation generates from those stack names.
- The S3 and CloudFront action lists are a best estimate of what stages 2-3 need. If a deploy fails with `AccessDenied`, the missing action is added here and this stack is redeployed.
- Changed: the identity check is its own temporary workflow (`.github/workflows/aws-identity-check.yml`) instead of a step in `ci.yml`. A step in CI would fail on every pull request, because the role only trusts `main`. The separate workflow runs on pushes to `main` and can be started by hand from any branch to see the refusal.
- Step 3 done: stack `my-app-github-oidc` is `CREATE_COMPLETE`. Role ARN: `arn:aws:iam::263935782586:role/my-app-github-deploy`.
- Found in verification: the first run on `main` was refused with `Not authorized to perform sts:AssumeRoleWithWebIdentity`. CloudTrail showed the `sub` GitHub actually sends is `repo:srini313131@52782561/react-app-for-github-actions@1416544802:ref:refs/heads/main` — it carries the numeric owner and repository IDs, which the trust policy did not include. The template now takes `GitHubOwnerId` and `GitHubRepoId` and matches the full value.
- Steps 4-5 done: `AWS_ROLE_ARN` is set as a repository variable and the stack was redeployed with the corrected trust policy. The "AWS identity check" workflow passes on `main` (reported from the Actions tab).
- Not yet observed: the refusal from a branch other than `main`. To see it, run the workflow by hand from another branch (Actions -> AWS identity check -> Run workflow); it should fail with `Not authorized to perform sts:AssumeRoleWithWebIdentity`.
- First run failed with `Could not load credentials from any providers`: the `AWS_ROLE_ARN` variable was not being found, so the action had no role to request.
