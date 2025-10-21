# Setup OpenSearch Domain

1. Identify a VPC and a private subnet where this OpenSearch domain will be deployed in. Take note of the VPC ID and Subnet ID.

- List VPC with their name and ID.

```bash
aws ec2 describe-vpcs --query "Vpcs[*].[VpcId,CidrBlock,Tags[?Key=='Name'].Value|[0]]" --output table
```

- List Subnets and respective AZ in a VPC.

```bash
aws ec2 describe-subnets --query "Subnets[*].[VpcId,AvailabilityZone,SubnetId,Tags[?Key=='Name'].Value|[0]||'No Name',MapPublicIpOnLaunch]" --output table
```

For OpenSearch domain, choose one or more private subnet (MapPublicIpOnLaunch = false).

2. Determine the CIDR range of the proxy server that will access the OpenSearch domain.

- If proxy server is in the same VPC, use the subnet CIDR range
- If proxy server is in a different VPC, use the VPC CIDR range or specific subnet range
- You can also use the specific private IP of the proxy server with /32

Example: `10.0.1.0/24` or `10.0.1.100/32`

3. Choose a domain name for your OpenSearch cluster.

- Must start with a lowercase letter
- Can contain only lowercase letters, numbers, and hyphens
- Example: `my-demo`, `opensearch-dev`, `search-cluster`

4. Deploy the CloudFormation template file.

**Note:** Master user credentials are now automatically generated and stored in AWS Secrets Manager. You no longer need to provide a password parameter.

```bash
aws cloudformation deploy --template-file setup-opensearch-domain-with-secret.yaml --stack-name opensearch-my-demo --parameter-overrides DomainName="my-demo" --capabilities CAPABILITY_IAM
```

The template will automatically:
- Create a Secrets Manager secret with auto-generated credentials
- Store the master username (`admin`) and a secure random password
- Configure the OpenSearch domain to use these credentials

5. (Optional) If any error occurs, rectify the problem; delete the stack before redeploying it again.

```bash
aws cloudformation delete-stack --stack-name opensearch-my-demo
```

6. Get the output of the CloudFormation stack.

```bash
aws cloudformation describe-stacks --stack-name opensearch-my-demo --query "Stacks[0].Outputs" --output table
```

7. The CloudFormation stack outputs the following values:

- **OpenSearchDomainEndpoint**: VPC endpoint to access OpenSearch API (format: `vpc-*-*.*.es.amazonaws.com`)
- **OpenSearchDashboardsURL**: URL to access OpenSearch Dashboards with web browser
- **OpenSearchSecurityGroupId**: Security group ID of the OpenSearch domain
- **MasterUserSecretArn**: ARN of the Secrets Manager secret containing master user credentials

8. Retrieve the master user credentials from Secrets Manager.

```bash
# Get the secret ARN from stack outputs
SECRET_ARN=$(aws cloudformation describe-stacks --stack-name opensearch-my-demo --query "Stacks[0].Outputs[?OutputKey=='MasterUserSecretArn'].OutputValue" --output text)

# Retrieve the credentials
aws secretsmanager get-secret-value --secret-id $SECRET_ARN --query SecretString --output text | jq -r '{username, password}'
```

This will display:
```json
{
  "username": "admin",
  "password": "your-auto-generated-password"
}
```

## Security Features

This template implements the following security best practices:

1. **Secrets Manager Integration**: Master credentials are auto-generated and stored securely in AWS Secrets Manager, not in CloudFormation parameters or stack history
2. **Restricted Access Policy**: Access is limited to the current AWS account with IP-based conditions for the proxy server CIDR range
3. **Encryption**: Both encryption at rest and node-to-node encryption are enabled
4. **VPC Deployment**: Domain is deployed in private subnets with no public access
5. **HTTPS Enforcement**: TLS 1.2 minimum security policy
6. **Fine-Grained Access Control**: Optional FGAC can be enabled for advanced authentication and authorization
