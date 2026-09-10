# Deployment

S3 deployment is handled by GitHub Actions. Pushes are deployed to the `building-models-app.concord.org` bucket by the `s3-deploy` job in [`ci.yml`](../.github/workflows/ci.yml), which serves them at `https://sage.concord.org/`. Branches go to `branch/<name>/` and tags go to `version/<name>/`.

A released version is promoted to `staging/` by [`release-staging.yml`](../.github/workflows/release-staging.yml) and then to the top level by [`release_production.yml`](../.github/workflows/release_production.yml), both via `workflow_dispatch`. See the release steps in the [README](../README.md#production-deployment).

## AWS Access

The GitHub actions in this project are allowed to update files in S3 using OIDC. An IAM role has been created in AWS with a trust policy that allows GitHub actions in this specific repository to assume this IAM role.

This repository does **not** use the shared `S3-deploy-by-role-tag` managed policy. That policy only grants access to `models-resources/[RepoName]`, and this project deploys to its own `building-models-app.concord.org` bucket instead. The role therefore carries a single inline policy scoped to that bucket. Both documents are recorded in [`iam/`](iam/):

- [`building-models-trust-policy.json`](iam/building-models-trust-policy.json) — who may assume the role
- [`building-models-app-policy.json`](iam/building-models-app-policy.json) — what the role may do in S3

The trust policy allows both the legacy and the immutable GitHub OIDC subject formats, so the role keeps working if this repository is later opted in to immutable subject claims.

See [deploy-setup.md in starter-projects](https://github.com/concord-consortium/starter-projects/blob/main/doc/deploy-setup.md) for how the AWS side is set up, including the section on deploying somewhere other than `models-resources`.

Two hardening options sometimes suggested in code review — splitting the build and the deploy into separate jobs, and pinning actions to a commit SHA — have been considered and declined. See [Hardening we have chosen not to do](https://github.com/concord-consortium/starter-projects/blob/main/doc/deploy-setup.md#hardening-we-have-chosen-not-to-do) for the reasons.
