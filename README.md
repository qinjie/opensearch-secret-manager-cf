# OpenSearch Domain with AWS Secrets Manager

Deploy a secure AWS OpenSearch domain in a VPC using CloudFormation with automated credential management via AWS Secrets Manager.

## Overview

This CloudFormation template deploys a production-ready OpenSearch domain with the following security features:

- **Automated Credential Management**: Master user credentials are auto-generated and stored in AWS Secrets Manager
- **VPC Isolation**: Domain deployed in private subnets with no public access
- **Encryption**: Both encryption at rest and node-to-node encryption enabled
- **Restricted Access**: IP-based access control limited to specific CIDR ranges
- **Fine-Grained Access Control**: Optional FGAC for advanced authentication
- **HTTPS Enforcement**: TLS 1.2 minimum security policy

## Quick Start

```bash
aws cloudformation deploy \
  --template-file setup-opensearch-domain-with-secret.yaml \
  --stack-name opensearch-my-demo \
  --parameter-overrides DomainName="my-demo" \
  --capabilities CAPABILITY_IAM
```

## Documentation

See [setup-opensearch-domain-with-secret.md](setup-opensearch-domain-with-secret.md) for detailed deployment instructions.

## Key Features

### Security Best Practices

1. **No Hardcoded Credentials**: Uses AWS Secrets Manager with dynamic secret resolution
2. **Account-Level Access Control**: Restricts access to current AWS account only
3. **IP Whitelisting**: Additional layer restricting access to specified CIDR ranges
4. **Comprehensive Encryption**: At-rest and in-transit encryption enabled
5. **Audit Trail**: CloudFormation changes tracked, secrets access logged

### Architecture

- **OpenSearch Version**: 2.11
- **Deployment**: 2 nodes across 2 availability zones
- **Instance Type**: m6g.large.search (ARM-based Graviton2)
- **Storage**: 20GB GP3 EBS per node
- **Network**: VPC deployment in private subnets

## Template Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `VpcId` | VPC ID for OpenSearch deployment | (Required) |
| `PrivateSubnetIds` | List of private subnet IDs (min 2) | (Required) |
| `ProxyServerCidrRange` | CIDR range allowed to access OpenSearch | `152.0.0.0/16` |
| `DomainName` | OpenSearch domain name | `my-demo` |
| `EnableFineGrainedAccess` | Enable fine-grained access control | `true` |

## Outputs

| Output | Description |
|--------|-------------|
| `OpenSearchDomainEndpoint` | VPC endpoint URL for OpenSearch API |
| `OpenSearchDashboardsURL` | URL for OpenSearch Dashboards |
| `OpenSearchSecurityGroupId` | Security group ID for the domain |
| `MasterUserSecretArn` | ARN of Secrets Manager secret with credentials |

## Retrieving Credentials

```bash
# Get secret ARN from stack outputs
SECRET_ARN=$(aws cloudformation describe-stacks \
  --stack-name opensearch-my-demo \
  --query "Stacks[0].Outputs[?OutputKey=='MasterUserSecretArn'].OutputValue" \
  --output text)

# Retrieve credentials
aws secretsmanager get-secret-value \
  --secret-id $SECRET_ARN \
  --query SecretString \
  --output text | jq -r '{username, password}'
```

## Security Improvements

This template has been enhanced with the following security fixes:

### Critical Fixes
- ✅ Migrated master password from CloudFormation parameters to Secrets Manager
- ✅ Restricted access policy from `Principal: "*"` to account-level with IP conditions

### Recommended Next Steps
- Add CloudWatch log groups for audit logging
- Configure automated backups (SnapshotOptions)
- Add DeletionPolicy: Snapshot for data protection
- Implement customer-managed KMS keys
- Add CloudWatch alarms for monitoring

## Prerequisites

- AWS CLI configured with appropriate credentials
- VPC with at least 2 private subnets in different availability zones
- Proper IAM permissions to create OpenSearch domains and Secrets Manager secrets

## Clean Up

```bash
aws cloudformation delete-stack --stack-name opensearch-my-demo
```

**Note**: Deleting the stack will also delete the Secrets Manager secret after a recovery window (default 7 days).

## License

MIT License - See project root for details
