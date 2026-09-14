# Orders DynamoDB Table — AWS SAM

**Live deployment:** `orders-dev` and `orders-prod` tables in `eu-west-1`,
deployed via the [dev](../../actions/workflows/deploy-dev.yml) and
[prod](../../actions/workflows/deploy-prod.yml) GitHub Actions pipelines.

A DynamoDB table (`orders-<env>`) deployed with AWS SAM, with fully separate
dev/prod GitHub Actions pipelines and dedicated per-environment S3 artifact
buckets.

## Table design

| Item | Value |
|---|---|
| Partition key | `OrderId` (String) |
| Sort key | `CreatedAt` (String) |
| Additional attributes | `CustomerId` (String), `Status` (String) |
| GSI 1 | `CustomerIndex` — PK `CustomerId`, SK `CreatedAt` |
| GSI 2 | `StatusIndex` — PK `Status`, SK `CreatedAt` |
| Billing mode | `PAY_PER_REQUEST` (On-Demand) |
| Table class | `STANDARD_INFREQUENT_ACCESS` (non-default) |

Defined in [template.yaml](template.yaml) as a plain `AWS::DynamoDB::Table`
resource rather than `AWS::Serverless::SimpleTable`, since `SimpleTable`
cannot express GSIs or a non-default `TableClass`.

## Why two hand-written workflows instead of `sam pipeline init`

The lab's challenge explicitly calls for environment-specific pipelines
*instead of* SAM's auto-generated multi-stage workflow, so
`deploy-dev.yml`/`deploy-prod.yml` are hand-authored rather than produced by
`sam pipeline init` — each triggers only on its own branch, deploys only its
own stack, and reads only its own environment's config out of
`samconfig.toml`. Deployment parameters (stack name, per-env S3 bucket,
region, `Environment` parameter override) are likewise split into dedicated
`[dev.deploy.parameters]` / `[prod.deploy.parameters]` stanzas rather than
one shared config block.

## Repo layout

```
template.yaml                       SAM template: the DynamoDB table
samconfig.toml                      dev / prod deploy config (per-env S3 bucket, params)
pipeline/artifact-buckets.yaml      IaC: dedicated per-env SAM artifact bucket
pipeline/github-oidc-role.yaml      IaC: per-env IAM role GitHub Actions assumes (OIDC, no static keys)
.github/workflows/deploy-dev.yml    Pipeline for the dev environment (branch: develop)
.github/workflows/deploy-prod.yml   Pipeline for the prod environment (branch: main)
```

## Status

Both environments are bootstrapped and deployed already: artifact buckets,
OIDC deploy roles, GitHub environment secrets, and the `orders-dev` /
`orders-prod` tables all exist. The steps below are what was run, kept here
so the setup is reproducible (e.g. in a fresh AWS account).

## One-time setup (per environment)

Do this twice — once with `Environment=dev`, once with `Environment=prod`.
You need AWS CLI configured locally with an account admin/deploy-capable
profile for this bootstrap step only; the ongoing pipeline runs with no
long-lived keys.

1. **Create the dedicated artifact bucket:**
   ```bash
   aws cloudformation deploy \
     --template-file pipeline/artifact-buckets.yaml \
     --stack-name orders-artifacts-dev \
     --parameter-overrides Environment=dev \
     --capabilities CAPABILITY_IAM
   ```
   Repeat with `orders-artifacts-prod` / `Environment=prod`. Note the bucket
   name each produces (`orders-sam-artifacts-<env>-<account-id>`).

2. **Create the GitHub OIDC deploy role:**
   ```bash
   aws cloudformation deploy \
     --template-file pipeline/github-oidc-role.yaml \
     --stack-name orders-deploy-role-dev \
     --parameter-overrides \
         Environment=dev \
         GitHubOrg=<your-github-username-or-org> \
         GitHubRepo=<this-repo-name> \
         BranchName=develop \
         ArtifactBucketName=orders-sam-artifacts-dev-<account-id> \
     --capabilities CAPABILITY_NAMED_IAM
   ```
   Repeat for prod with `Environment=prod`, `BranchName=main`, and
   `CreateOidcProvider=false` (an account can only have one GitHub OIDC
   provider — the dev run already created it).

   Grab each stack's `DeployRoleArn` output.

3. **In the GitHub repo**, create two Environments (Settings → Environments):
   `dev` and `prod`. In each, add:
   - Secret `AWS_ROLE_ARN` = the matching `DeployRoleArn` output
   - Secret `ARTIFACT_BUCKET` = the matching artifact bucket name

   Optionally add a required reviewer on the `prod` environment so
   production deploys need manual approval before running.

4. Push to `develop` to deploy dev, push to `main` to deploy prod — or
   trigger either workflow manually from the Actions tab
   (`workflow_dispatch`).

## Deploying locally (optional, for testing before pushing)

```bash
sam build
sam deploy --config-env dev    # or --config-env prod
```

(Update the placeholder bucket names in `samconfig.toml` first, or pass
`--s3-bucket` explicitly.)

## Verifying in the AWS Console

1. **DynamoDB → Tables** → open `orders-dev` (or `orders-prod`).
2. **Explore table items → Create item** and add a few items, e.g.:
   ```json
   { "OrderId": "ord-1", "CreatedAt": "2026-09-14T10:00:00Z", "CustomerId": "cust-1", "Status": "PENDING" }
   { "OrderId": "ord-2", "CreatedAt": "2026-09-14T10:05:00Z", "CustomerId": "cust-1", "Status": "SHIPPED" }
   { "OrderId": "ord-3", "CreatedAt": "2026-09-14T10:10:00Z", "CustomerId": "cust-2", "Status": "PENDING" }
   ```
3. **Query** the base table by `OrderId`, then switch the index selector to
   `CustomerIndex` to query by `CustomerId`, and to `StatusIndex` to query by
   `Status` — confirming both GSIs return the expected items.
