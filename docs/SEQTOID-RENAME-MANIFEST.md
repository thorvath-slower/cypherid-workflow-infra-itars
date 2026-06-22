# seqtoid Rename Manifest — cypherid-workflow-infra

**Ticket:** CZID-249 (Decision D1). **Status:** spec only — execution is deferred to the UCSF apply-side (see "Why deferred").

This is the authoritative mapping of the old brand tokens (`idseq-*`, `cypherid-*`) to the
single canonical brand **`seqtoid-*`** for the AWS resources defined in this repo. It is a
**static specification**: the actual renames change real AWS resource names and therefore
require `terraform plan`/`apply` against the UCSF accounts, which is out of scope for the
stabilization line (D3 — we have no UCSF account access and do not run plan/apply). UCSF Ops
executes these during the apply-side migration; this file is the contract they work from.

## Why deferred (not done in-repo)

- Renaming a resource's `name`/`bucket` attribute changes the **real AWS resource**. For most
  of these (IAM roles, S3 buckets, Batch compute environments, security groups) a name change
  **forces replacement**, which only Terraform apply can carry out and only the account owner
  can validate. **D3** scopes our work to static correctness; the rename itself is apply-side.
- `moved {}` blocks do **not** help here: they relocate a resource's *Terraform state address*,
  not its real cloud `name`. D1 wants the real names changed, so `moved {}` is not the tool.
- The env-alias files (`EXPECT_AWS_ACCOUNT_ALIAS`) were already reconciled to `seqtoid-*` under
  **CZID-239** — they are not part of this manifest.

## Rename rule

Swap the leading brand token, preserve everything else (env interpolation, suffixes, casing
shape):

- `idseq-…`  → `seqtoid-…`
- `idseq_…`  → `seqtoid_…`  (underscore variants)
- `IDseq-…`  → `Seqtoid-…`  (mixed-case display variants)
- `cypherid-…` → `seqtoid-…`
- embedded `…-idseq-…` → `…-seqtoid-…`

`${var.DEPLOYMENT_ENVIRONMENT}` / `${var.deployment_environment}` interpolations are kept verbatim.

## Resources to rename

### AWS Batch (compute environments, queues, job definitions, service roles)
| Current | Target |
|---|---|
| `idseq-${var.DEPLOYMENT_ENVIRONMENT}-batch-job` | `seqtoid-${var.DEPLOYMENT_ENVIRONMENT}-batch-job` |
| `idseq-${var.DEPLOYMENT_ENVIRONMENT}-batch-main` | `seqtoid-${var.DEPLOYMENT_ENVIRONMENT}-batch-main` |
| `idseq-${var.DEPLOYMENT_ENVIRONMENT}-batch-main-instance` | `seqtoid-${var.DEPLOYMENT_ENVIRONMENT}-batch-main-instance` |
| `idseq-${var.DEPLOYMENT_ENVIRONMENT}-batch-service` | `seqtoid-${var.DEPLOYMENT_ENVIRONMENT}-batch-service` |
| `idseq-${var.DEPLOYMENT_ENVIRONMENT}-batch-spot-fleet-service` | `seqtoid-${var.DEPLOYMENT_ENVIRONMENT}-batch-spot-fleet-service` |
| `idseq-${var.deployment_environment}-batch-${var.alignment_algorithm}` | `seqtoid-${var.deployment_environment}-batch-${var.alignment_algorithm}` |
| `idseq-${var.deployment_environment}-batch-${var.alignment_algorithm}-instance` | `seqtoid-${var.deployment_environment}-batch-${var.alignment_algorithm}-instance` |
| `idseq-${var.deployment_environment}-batch-${var.alignment_algorithm}-database-bucket-read` (IAM) | `seqtoid-…` |
| `idseq_batch_service_role-destroy` | `seqtoid_batch_service_role-destroy` |

### Step Functions / index generation / swipe
| Current | Target |
|---|---|
| `idseq-${var.DEPLOYMENT_ENVIRONMENT}-index-generation` | `seqtoid-${var.DEPLOYMENT_ENVIRONMENT}-index-generation` |
| `idseq-start_index_generation-${var.DEPLOYMENT_ENVIRONMENT}` | `seqtoid-start_index_generation-${var.DEPLOYMENT_ENVIRONMENT}` |
| `idseq-swipe-${var.DEPLOYMENT_ENVIRONMENT}` | `seqtoid-swipe-${var.DEPLOYMENT_ENVIRONMENT}` |
| `idseq-${var.deployment_environment}-${each.key}` | `seqtoid-${var.deployment_environment}-${each.key}` |

### VPC / security-group / instance `Name` tags
| Current | Target |
|---|---|
| `idseq-${var.DEPLOYMENT_ENVIRONMENT}` | `seqtoid-${var.DEPLOYMENT_ENVIRONMENT}` |
| `idseq-${var.DEPLOYMENT_ENVIRONMENT}-ci-cd` | `seqtoid-${var.DEPLOYMENT_ENVIRONMENT}-ci-cd` |
| `idseq-${var.DEPLOYMENT_ENVIRONMENT}-ci-cd-YOUR_GITHUB_REPO` | `seqtoid-${var.DEPLOYMENT_ENVIRONMENT}-ci-cd-YOUR_GITHUB_REPO` |

### SNS / SSM / DynamoDB / Glue
| Current | Target |
|---|---|
| `${var.deployment_environment}-idseq-heatmap-topic` (SNS) | `${var.deployment_environment}-seqtoid-heatmap-topic` |
| `/idseq-${var.DEPLOYMENT_ENVIRONMENT}-web/SFN_NOTIFICATIONS_QUEUE_ARN` (SSM) | `/seqtoid-${var.DEPLOYMENT_ENVIRONMENT}-web/SFN_NOTIFICATIONS_QUEUE_ARN` |
| `IDseq-pipeline-runs-${var.DEPLOYMENT_ENVIRONMENT}` (DynamoDB) | `Seqtoid-pipeline-runs-${var.DEPLOYMENT_ENVIRONMENT}` |
| `idseq-${var.DEPLOYMENT_ENVIRONMENT}-batch-taxon-indexing` (Glue) | `seqtoid-${var.DEPLOYMENT_ENVIRONMENT}-batch-taxon-indexing` |

### S3 buckets
| Current | Target | Note |
|---|---|---|
| `idseq-prod-system-test` | `seqtoid-prod-system-test` | |
| `cypherid-public-references-dev-<account-id>` | `seqtoid-public-references-dev-<account-id>` | `terraform/index-generation.tf` already references the `seqtoid-public-references` target in a TODO; reconcile both at apply time. |

### IAM secret ARNs (in lambda JSON policy templates)
- `arn:aws:secretsmanager:*:<account-id>:secret:idseq/*` → `…:secret:seqtoid/*`
  (the secret **path prefix** `idseq/` renames with the brand; coordinate with the secret store).

## Needs UCSF Ops coordination (not a mechanical swap)

- **GitHub Actions cross-account role** `gha-cypherid-workflow-infra-terraform` (referenced from
  gql-fed `.happy` deploy and elsewhere) — renaming an IAM role used by CI breaks the OIDC trust
  until both ends are updated together. Sequence with UCSF.

## Do NOT rename (out of scope)

- **`czi-infectious-disease-*`** S3 buckets — customer (CZI) data buckets, not ours.
- **`github.com/chanzuckerberg/*`** module sources (e.g. swipe) — external upstream; repointed by
  SHA/HTTPS elsewhere, never rebranded.
- **Our own repo names** (`cypherid-workflow-infra`, `cypherid-web-infra`) where they appear as
  `authorized_github_repos` / workflow display names — renaming the GitHub repos is a separate
  decision, not part of D1's resource-naming scope.
- **ECR repositories** — already brand-neutral (`amr`, `benchmark`, `consensus-genome`, …); no change.
- Product/app tokens (`czid`, `idseq`) in application/pipeline code, docs, and test fixtures —
  D1 covers infra resource naming, not the product identity.
