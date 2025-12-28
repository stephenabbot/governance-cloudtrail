# Prerequisites

## Required Tools

- **AWS CLI** version 2.x for AWS service interaction and CloudFormation operations
- **Bash** version 4.x or higher for deployment script execution
- **jq** for JSON processing in deployment and verification scripts
- **Git** for repository information detection and metadata extraction

## AWS Account Requirements

- AWS account with appropriate permissions for CloudFormation, S3, CloudTrail, CloudWatch Logs, IAM, and SSM
- No existing CloudFormation stack named governance-cloudtrail
- No existing S3 bucket with name governance-cloudtrail-{ACCOUNT_ID}
- No existing CloudTrail trail named governance-cloudtrail-trail

## AWS Permissions Required

The deployment principal must have permissions for:

- **CloudFormation**: CreateStack, UpdateStack, DeleteStack, DescribeStacks, ListStackResources, ValidateTemplate
- **S3**: CreateBucket, PutBucketPolicy, PutBucketVersioning, PutBucketEncryption, PutBucketLifecycleConfiguration, PutBucketPublicAccessBlock
- **CloudTrail**: CreateTrail, UpdateTrail, DeleteTrail, DescribeTrails, GetTrailStatus, PutEventSelectors
- **CloudWatch Logs**: CreateLogGroup, DeleteLogGroup, DescribeLogGroups, PutRetentionPolicy
- **IAM**: CreateRole, DeleteRole, AttachRolePolicy, DetachRolePolicy, PassRole, ListAccountAliases
- **SSM**: PutParameter, DeleteParameter, GetParameter, GetParametersByPath, DescribeParameters
- **STS**: GetCallerIdentity for account metadata collection

## Git Repository Requirements

- Current directory must be a git repository with remote origin configured
- Repository should have clean working directory with no uncommitted changes
- Local branch should be synchronized with remote tracking branch if configured
- Repository URL used for resource tagging and metadata collection

## Configuration Requirements

- Environment configuration file (.env) must exist with all required variables
- CloudFormation template must be present at cloudformation/bootstrap.yaml
- Template must pass AWS CloudFormation validation

## Environment Variables

Required variables in .env file:

- **AWS_REGION** - Target AWS region for deployment
- **TAG_ENVIRONMENT** - Environment identifier (prd, stg, tst, dev)
- **TAG_OWNER** - Resource owner identifier
- **TAG_COST_CENTER** - Cost allocation identifier
- **AUTO_DELETE_FAILED_STACK** - Automatic cleanup of failed stacks (true/false)
- **CLOUDTRAIL_IS_MULTI_REGION** - Enable multi-region trail (true/false)
- **CLOUDTRAIL_INCLUDE_MANAGEMENT_EVENTS** - Capture management events (true/false)
- **CLOUDTRAIL_INCLUDE_DATA_EVENTS** - Capture data events (true/false)
- **CLOUDTRAIL_EVENT_SELECTORS** - Event types to capture (WriteOnly, ReadOnly, All)
- **ENABLE_CLOUDWATCH_LOGS** - Stream events to CloudWatch Logs (true/false)
- **CLOUDWATCH_LOGS_RETENTION_DAYS** - Log retention period in days
- **S3_INTELLIGENT_TIERING_DAYS** - Days before Intelligent-Tiering transition
- **S3_GLACIER_TRANSITION_DAYS** - Days before Glacier IA transition
- **S3_EXPIRATION_DAYS** - Days before object expiration

## Verification Process

Run the prerequisites verification script before deployment:

```bash
./scripts/verify-prerequisites.sh
```

This script validates:
- Required tools installation and versions
- Git repository state and configuration
- AWS credentials and permissions
- CloudFormation template syntax
- Environment configuration completeness
