# my-app

React 19 + TypeScript + Vite 8 single-page app. This is a learning project: the app itself is the stock Vite template, and the real work is the CI/CD and AWS hosting around it.

## Purpose and how to work with me

- I'm learning to deploy and host apps the way enterprise teams do, and preparing to talk about it in interviews.
- Teach as you go: explain what each piece does and why it's done that way before or alongside the change, not just the code.
- Prefer the approach a production team would use (IaC, least privilege, OIDC, environments, approvals) over shortcuts, and say when a real company would do something differently at larger scale.
- Work in small stages and stop after each so I can run it, verify it, and ask questions.
- Let me do the hands-on account steps (AWS console/CLI login, GitHub settings) myself; give exact steps.
- After each stage, add to `docs/interview-notes.md`: the concept, what we built, likely interview questions with model answers, and ask me a few mock questions.

## Plan docs

- Every feature or roadmap stage is planned before it is built. Once I approve the plan, save it as `docs/plans/NN-short-name.md` (numbered in order, e.g. `01-github-oidc.md`) before writing any code.
- Each plan doc has: context (why), approach, files to change, verification steps, and a status line (`Planned`, `In progress`, `Done`).
- Keep the doc current: update the status as work progresses, and note anything that ended up different from the plan and why.

## Commands

- `npm run dev` — local dev server
- `npm run lint` — ESLint
- `npm test` — Vitest, single run (used by CI); `npm run test:watch` for watch mode
- `npm run build` — type-check (`tsc -b`) and build to `dist/`

Run lint, test and build before committing.

## Layout

- `src/` — app code; tests sit next to the code as `*.test.tsx`
- `src/setupTests.ts` — jest-dom matchers and RTL cleanup
- `vite.config.ts` — Vite and Vitest config
- `.github/workflows/ci.yml` — `build` job: install, lint, test, build, upload `dist` artifact; `deploy` job (pushes to `main` only): deploy `infra/site.yml` as stack `my-app-staging`, upload `dist` to S3, clear the CloudFront cache
- `infra/` — CloudFormation templates
- `docs/plans/` — one plan doc per feature or stage; `README.md` there is the index
- `docs/interview-notes.md` — interview notes per stage

## Conventions

- Tests import `describe`, `it`, `expect` from `vitest` explicitly (globals are off).
- Never commit directly to `main`; work on a branch and merge through a pull request.
- Commit or push only when I ask.
- GitHub repo: `srini313131/react-app-for-github-actions`. The `gh` CLI is not installed, so PRs and repo settings are done in the browser.

## AWS

- Region `us-east-1`; hosting target is a private S3 bucket behind CloudFront.
- Infrastructure is defined in CloudFormation and deployed by GitHub Actions through an OIDC role — no long-lived AWS keys in GitHub.
- Never put AWS credentials, account secrets or keys in the repo or workflow files.

## Deployment roadmap

1. Identity: GitHub OIDC provider and least-privilege deploy role — done
2. Hosting as code: private S3, CloudFront with OAC, security headers — template done (`infra/site.yml`), first deployed in stage 3
3. Continuous deployment to one environment on push to `main`
4. Staging and production with an approval gate, same artifact promoted
5. Operations: rollback, logging and alarms, branch protection

Update this list as stages are completed.
