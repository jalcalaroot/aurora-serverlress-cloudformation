# Aurora Serverless v2 Cluster (CloudFormation)

CloudFormation template that deploys an Aurora Serverless v2 cluster (MySQL or PostgreSQL engine) into an existing VPC, with Secrets Manager credentials, an optional reader and RDS Proxy, a custom-resource Lambda that creates application DB users, and SSM parameters for the endpoints. (The repo name `aurora-serverlress` is a typo of "serverless".)

> **Status:** legacy reference template (2026), not actively maintained. Review and update before deploying.

## Architecture

Template: `aurora-serverless-template.yaml`

- Aurora cluster (`EngineMode: provisioned`, `ServerlessV2ScalingConfiguration`), storage encrypted, deletion protection on, 7-day backups, CloudWatch log exports
- Writer instance (`db.serverless`) and an optional reader (`pNeedsReplica`)
- Security group allowing 3306 (MySQL) or 5432 (PostgreSQL) from the VPC CIDR and from a managed prefix list (VPN)
- DB subnet group using the private or internal subnets read from SSM
- Secrets Manager secrets: master user, app user, and (when IAM auth is enabled) an IAM-auth user
- Optional RDS Proxy (TLS required, optional IAM auth) with its IAM role, optionally assumable from another account
- Lambda `CreateDbUser` (Python 3.12, in the VPC) + `Custom::CreateDbUser` that creates the app users and runs PostgreSQL `GRANT` statements. **Its code is not in this repo**; it is loaded from `s3://<pLambdaBucket>/create-db-user.zip`
- SSM parameters: `/rds/<db>/endpoint`, `/rds/<db>/port`, `/rds/<db>/passdbsecretarn`, and proxy endpoint/role ARN when the proxy is enabled
- Organisation-style tags (`workload:*`, `automation:*`, `security:*`) on most resources

## Prerequisites

SSM parameters `/vpc/id`, `/vpc/cidr`, `/vpc/subnets/private/ids` and `/vpc/subnets/internal/ids`, a prefix list for VPN access, and the Lambda zip in S3.

## Parameters

| Name | Purpose | Default |
|------|---------|---------|
| `pPrefix` | Name prefix for some tags | `account` |
| `VPCId`, `VPCCidr`, `SubnetsPrivate`, `SubnetsInternal` | SSM parameter names for network values | `/vpc/...` |
| `pSourcePrefixListId` | Prefix list allowed to reach the DB port | (required) |
| `pSubnets` / `pLambdaSubnets` | Subnet tier for the DB / Lambda (`private` or `internal`) | `internal` / `private` |
| `pDBName`, `pClusterId`, `pInstanceId` | Database name, cluster and writer identifiers | (required) |
| `pROInstanceId`, `pNeedsReplica` | Reader identifier and toggle | `reader-instance`, `false` |
| `pNeedsRDSProxy`, `pRDSProxyIAMAuth` | Create RDS Proxy / require IAM auth on it | `false`, `true` |
| `pDBSecretName` | Master secret name | (required) |
| `pDBEngine`, `pEngineVersion` | `aurora-mysql` or `aurora-postgresql` and version | (required) |
| `pMasterUsername`, `pAppUsername`, `pIAMUsername` | DB usernames | `adminuser`, `appuser`, `iamuser` |
| `pMinCapacity`, `pMaxCapacity` | Serverless v2 ACU range | `2`, `8` |
| `pEnableHttpEndpoint` | Data API | `false` |
| `pRoleCrossAccount` | Role ARN allowed to assume the proxy role | empty |
| `pLambdaBucket` | Bucket with `create-db-user.zip` | (required) |
| `pTagValue*` | Tag values (owner emails, environment, cost center, backup flags...) | various; cost center `<COST_CENTER>` |

## Deploy

```bash
aws cloudformation deploy \
  --stack-name <STACK_NAME> \
  --template-file aurora-serverless-template.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    pDBName=<DB_NAME> pClusterId=<CLUSTER_ID> pInstanceId=<INSTANCE_ID> \
    pDBSecretName=<SECRET_NAME> pDBEngine=aurora-postgresql pEngineVersion=<ENGINE_VERSION> \
    pSourcePrefixListId=<PREFIX_LIST_ID> pLambdaBucket=<LAMBDA_BUCKET> \
    pTagValueTechOwner=<OWNER_EMAIL> pTagValueBussOwner=<OWNER_EMAIL> pTagValueEnvironment=dev
```

## Known issues / Security notes

- The template is effectively PostgreSQL-only for the proxy and user creation: RDS Proxy `EngineFamily` is hardcoded to `POSTGRESQL` and the grants are PostgreSQL SQL.
- The master secret hardcodes `"username": "adminuser"` regardless of `pMasterUsername`; the master password is only 16 characters and no secret has rotation.
- Grants give the app/IAM users `ALL PRIVILEGES` on the database and `ALL` on schema `public`.
- RDS Proxy has `DebugLogging: true`, which can log SQL statements; disable outside troubleshooting.
- The Lambda role allows `ec2:*NetworkInterface` on `Resource: "*"` (standard for VPC Lambdas) and reuses the DB security group.
- Metadata references a non-existent `pDBPassSecret` parameter and `pSubnets` is described as an "Application Load Balancer" setting (copy-paste leftovers). There are no stack Outputs.

## Cleanup

```bash
aws rds modify-db-cluster --db-cluster-identifier <CLUSTER_ID> --no-deletion-protection --apply-immediately
aws cloudformation delete-stack --stack-name <STACK_NAME>
```

`DeletionProtection` is hardcoded to `true`, so disable it first. With no explicit `DeletionPolicy`, CloudFormation takes a final cluster snapshot; delete it manually if not needed. Secrets are scheduled for deletion (recovery window) and their names cannot be reused until then.

## License

MIT — see [LICENSE](LICENSE).
