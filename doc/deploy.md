# Deployment

S3 deployment is handled by GitHub Actions. Pushes are deployed to the `building-models-app.concord.org` bucket by the `s3-deploy` job in [`ci.yml`](../.github/workflows/ci.yml). Branches go to `branch/<name>/` and tags go to `version/<name>/`.

Two hostnames front that bucket and serve the same content: `sage.concord.org` is the public address and is a CNAME for the bucket, and `building-models-app.concord.org` is the bucket's own name. The deployment status link on a pull request uses the bucket hostname, because that is what `deployRunUrl` in `ci.yml` is set to. Note that `sagemodeler.concord.org` is a different site, built from [sage-modeler-site](https://github.com/concord-consortium/sage-modeler-site).

A released version is promoted to `staging/` by [`release-staging.yml`](../.github/workflows/release-staging.yml) and then to the top level by [`release_production.yml`](../.github/workflows/release_production.yml), both via `workflow_dispatch`. See the release steps in the [README](../README.md#production-deployment).

## AWS Access

The GitHub actions in this project are allowed to update files in S3 using OIDC. An IAM role has been created in AWS with a trust policy that allows GitHub actions in this specific repository to assume this IAM role.

This repository does **not** use the shared `S3-deploy-by-role-tag` managed policy. That policy only grants access to `models-resources/[RepoName]`, and this project deploys to its own `building-models-app.concord.org` bucket instead. The role therefore carries a single inline policy, `building-models-app`, scoped to that bucket. Both documents are recorded in [`iam/`](iam/):

- [`building-models-trust-policy.json`](iam/building-models-trust-policy.json) — who may assume the role
- [`building-models-app-policy.json`](iam/building-models-app-policy.json) — what the role may do in S3

The trust policy allows both the legacy and the immutable GitHub OIDC subject formats, so the role keeps working if this repository is later opted in to immutable subject claims.

The S3 policy grants reads and writes across the whole bucket, but deletes only under `branch/*` and `version/*`. That split is deliberate and worth keeping. Writes have to reach the bucket root because `release_production.yml` copies a version tree to the root and the deploy action writes `index-<branch>.html` there. Deletes never need to: the only delete the pipeline performs is the `aws s3 sync --delete` inside `s3-deploy-action`, whose destination is always `branch/<name>/` or `version/<tag>/`, and both release workflows use `aws s3 cp`, which deletes nothing. Since `ci.yml` runs on every push, widening deletes to the bucket root would let any branch push erase the production site.

`s3:AbortMultipartUpload` is granted separately from `s3:PutObject`. `PutObject` covers starting, uploading, and completing a multipart upload, but not the cleanup call the AWS CLI makes when a transfer fails. Without it a failed upload reports a misleading `AccessDenied` and leaves orphaned parts that keep accruing storage charges and that this role cannot remove. The CLI switches to multipart above 8 MB by default, and the largest deployed file is already about 6.75 MB.

These files are a record of what was applied, not the live configuration. IAM keeps its own copy, so editing them here changes nothing until the policy is pushed to AWS:

```sh
aws iam put-role-policy \
  --role-name building-models \
  --policy-name building-models-app \
  --policy-document file://doc/iam/building-models-app-policy.json
```

Because this role was created by hand rather than by `create-deploy-role.sh`, there is no script to re-run that would reconcile the two copies. To check whether they have drifted, read the live versions back and compare them against the files here:

```sh
aws iam get-role --role-name building-models --query 'Role.AssumeRolePolicyDocument'

aws iam get-role-policy --role-name building-models \
  --policy-name building-models-app --query 'PolicyDocument'
```

See [deploy-setup.md in starter-projects](https://github.com/concord-consortium/starter-projects/blob/main/doc/deploy-setup.md) for how the AWS side is set up, including the section on deploying somewhere other than `models-resources`.

Two hardening options sometimes suggested in code review — splitting the build and the deploy into separate jobs, and pinning actions to a commit SHA — have been considered and declined. See [Hardening we have chosen not to do](https://github.com/concord-consortium/starter-projects/blob/main/doc/deploy-setup.md#hardening-we-have-chosen-not-to-do) for the reasons.
