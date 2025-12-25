# AWS CloudTrail Governance Foundation

https://github.com/stephenabbot/governance-cloudtrail

Foundational audit logging infrastructure using CloudFormation for AWS governance, compliance, and security monitoring.

## Table of Contents

- [Why](#why)
- [How](#how)
  - [Architecture](#architecture)
  - [Script-Based Deployment](#script-based-deployment)
- [Resources Deployed](#resources-deployed)
- [Configuration Reference](#configuration-reference)
  - [Required Configuration](#required-configuration)
  - [CloudTrail Behavior](#cloudtrail-behavior)
  - [CloudWatch Logs Integration](#cloudwatch-logs-integration)
  - [S3 Lifecycle Management](#s3-lifecycle-management)
- [Event Types and Selectors](#event-types-and-selectors)
  - [Management Events](#management-events)
  - [Data Events](#data-events)
  - [Read vs Write Events](#read-vs-write-events)
  - [Recommended Configurations](#recommended-configurations)
- [Multi-Region Behavior](#multi-region-behavior)
- [Cost Estimation](#cost-estimation)
  - [Personal and Demo Configuration](#personal-and-demo-configuration)
  - [Production and HIPAA Configuration](#production-and-hipaa-configuration)
  - [Cost Variables](#cost-variables)
- [Prerequisites](#prerequisites)
  - [Required Tools](#required-tools)
  - [AWS Account Requirements](#aws-account-requirements)
  - [Git Repository Setup](#git-repository-setup)
- [Quick Start](#quick-start)
- [Deployment Architecture](#deployment-architecture)
  - [Single Account vs Organization Deployment](#single-account-vs-organization-deployment)
  - [Compliance Considerations](#compliance-considerations)
  - [Destroy Behavior and S3 Retention](#destroy-behavior-and-s3-retention)
- [Troubleshooting](#troubleshooting)
  - [Stack Creation Failures](#stack-creation-failures)
  - [CloudTrail Not Logging](#cloudtrail-not-logging)
  - [CloudWatch Logs Not Receiving Events](#cloudwatch-logs-not-receiving-events)
  - [S3 Bucket Access Issues](#s3-bucket-access-issues)
  - [SSM Parameter Issues](#ssm-parameter-issues)
- [Technologies and Services](#technologies-and-services)
  - [Infrastructure as Code](#infrastructure-as-code)
  - [AWS Services](#aws-services)
  - [Development Tools](#development-tools)
- [Copyright](#copyright)

## Why

AWS accounts generate thousands of API calls daily from users, applications, and automated processes. Without audit logging, determining who made changes, when they occurred, and what was affected becomes impossible. Security incidents cannot be investigated, compliance requirements cannot be met, and operational troubleshooting lacks critical context.

CloudTrail provides the foundational audit log for AWS governance by recording every API call made in an account. However, manual CloudTrail configuration creates inconsistency across accounts, risks misconfiguration that breaks audit trails, and requires understanding complex interactions between CloudTrail, S3, IAM, and CloudWatch Logs.

This project solves foundational audit logging by deploying CloudTrail with S3 storage, optional CloudWatch Logs streaming, lifecycle management for cost optimization, and SSM parameter publishing for downstream tool discovery. The infrastructure can be enabled selectively for cost management during development or deployed continuously for compliance requirements in regulated environments.

This project is standalone and does not depend on existing Terraform foundation infrastructure, making it suitable as a true governance foundation layer deployed independently of application infrastructure patterns. This project builds on patterns established in other foundational projects including [governance-cloudtrail](https://github.com/stephenabbot/governance-cloudtrail) and [terraform-aws-deployment-roles](https://github.com/stephenabbot/terraform-aws-deployment-roles).

## How

### Architecture

The foundation uses CloudFormation to deploy audit logging infrastructure that compliance tools, security automation, and operational analysis depend on. CloudTrail captures API events, delivers them to S3 for long-term retention, and optionally streams them to CloudWatch Logs for real-time querying. All resource identifiers are published to SSM Parameter Store for discovery by consuming projects.

The deployment supports both single-region and multi-region trails through configuration. Multi-region trails automatically capture API activity across all AWS regions while delivering logs to a single S3 bucket with regional prefixes. Single-region trails limit capture to the deployment region for cost optimization during development.

S3 buckets include versioning, encryption, intelligent tiering for automatic cost optimization, and optional lifecycle transitions to Glacier for long-term retention. CloudWatch Logs integration enables real-time event querying and alerting when enabled, with configurable retention periods. Trail configuration supports management events only for foundational audit logging or optional data events for S3 object access and Lambda invocation tracking.

### Script-Based Deployment

The deployment script handles all complexity including prerequisite validation, metadata collection, parameter preparation, and stack deployment. This eliminates manual CloudFormation CLI operations and reduces deployment errors. The script validates git repository state, AWS credentials, required tools, and account permissions before attempting deployment.

Idempotent operations allow running the deployment script multiple times safely. The script detects existing stacks and performs updates rather than failing. Resource listing scripts provide complete inventory of deployed infrastructure with status checks. Destruction scripts include confirmation prompts and special handling for S3 bucket retention to preserve audit logs.

## Resources Deployed

The foundation creates the following AWS resources:

- CloudTrail trail with event capture for management events and optional data events
- S3 bucket for audit log storage with versioning, encryption, and lifecycle management
- S3 bucket policy allowing CloudTrail service access while blocking other access
- CloudWatch Logs log group for real-time event streaming when integration enabled
- IAM role for CloudTrail CloudWatch Logs delivery when integration enabled
- SSM parameters publishing trail name, S3 bucket, and CloudWatch Logs resources
- CloudFormation stack managing all resources with consistent tagging

All resources include comprehensive tags for cost allocation, ownership tracking, and resource management. Resource names incorporate the AWS account ID to ensure global uniqueness and prevent conflicts across accounts.

## Configuration Reference

All configuration is defined in the .env file at the project root. The deployment script reads these values and passes them as CloudFormation parameters.

### Required Configuration

AWS_REGION: Deployment region for CloudFormation stack and regional resources. Default is us-east-1. All SSM parameters are created in this region regardless of multi-region trail configuration.

TAG_ENVIRONMENT: Environment identifier for resource tagging. Use prod for production, stage for staging, test for testing environments, or dev for development. This tag enables cost allocation and resource filtering.

TAG_OWNER: Resource owner identifier for accountability and contact purposes. Typically set to an individual name, team name, or email address.

TAG_COST_CENTER: Cost allocation identifier for billing and chargeback. Default is default. Change this to match your organization's cost center naming convention.

AUTO_DELETE_FAILED_STACK: Controls automatic deletion of failed CloudFormation stacks during deployment. When true, the deploy script automatically deletes stacks in failed states (ROLLBACK_COMPLETE, CREATE_FAILED, etc.) and retries deployment for automation workflows. When false, failed deployments require manual inspection and cleanup before retry, allowing investigation of deployment failures. Default is false for safety. Set to true only in automated environments where deployment failure investigation is handled separately.

### CloudTrail Behavior

CLOUDTRAIL_IS_MULTI_REGION: Controls whether trail captures events from all AWS regions or only the deployment region. When true, the trail automatically captures API activity across all enabled regions and delivers events to the single S3 bucket with regional prefixes. When false, the trail only captures events in the deployment region. Default is true. Set to false during development to reduce log volume and costs.

CLOUDTRAIL_INCLUDE_MANAGEMENT_EVENTS: Controls whether trail captures management events. Management events are control plane API calls like CreateBucket, RunInstances, and DeleteRole. This should always be true for audit logging. Default is true.

CLOUDTRAIL_INCLUDE_DATA_EVENTS: Controls whether trail captures data events. Data events are data plane operations like S3 GetObject, S3 PutObject, and Lambda Invoke. Data events generate significantly higher volume and cost than management events. Default is false. Enable this only when S3 object access logging or Lambda invocation tracking is required for compliance.

CLOUDTRAIL_EVENT_SELECTORS: Controls whether trail captures read events, write events, or both. Options are WriteOnly for configuration changes only, ReadOnly for read operations only, or All for complete capture. Default is WriteOnly. WriteOnly provides the best cost-to-value ratio by capturing only configuration changes that affect security posture. All is required for complete audit trails in regulated environments but increases costs significantly.

### CloudWatch Logs Integration

ENABLE_CLOUDWATCH_LOGS: Controls whether trail streams events to CloudWatch Logs for real-time querying. When true, CloudFormation creates a log group and IAM role for log delivery. When false, events are only delivered to S3. Default is true. Disable this to avoid CloudWatch Logs ingestion costs if real-time querying is not needed.

CLOUDWATCH_LOGS_RETENTION_DAYS: Controls how long CloudWatch Logs retains events before automatic deletion. Default is 90 days to match the free CloudTrail Event History retention period. S3 retains events according to lifecycle policies regardless of CloudWatch Logs retention. CloudWatch Logs serves as a real-time query cache while S3 serves as the permanent audit archive.

### S3 Lifecycle Management

S3_INTELLIGENT_TIERING_DAYS: Number of days before transitioning objects to Intelligent-Tiering storage class. Default is 0 for immediate transition. Intelligent-Tiering automatically moves objects between access tiers based on usage patterns without retrieval fees, optimizing costs without operational overhead.

S3_GLACIER_TRANSITION_DAYS: Number of days before transitioning objects to Glacier Instant Access storage class. Default is 90 days. Objects in Glacier IA cost less for storage but have retrieval fees. This balances cost optimization with access requirements for older audit logs.

S3_EXPIRATION_DAYS: Number of days before deleting objects permanently. Default is 2555 days which equals 7 years to match AWS Config retention and common regulatory retention requirements. Adjust this based on your compliance requirements. Note that CloudTrail event history in the console is always limited to 90 days regardless of S3 retention.

## Event Types and Selectors

Understanding event types and selectors is critical for balancing audit coverage against costs.

### Management Events

Management events capture control plane API calls that create, modify, or delete AWS resources. Examples include launching EC2 instances, creating S3 buckets, modifying security groups, creating IAM roles, and updating CloudFormation stacks. Every infrastructure change generates management events.

Management events provide foundational audit logging for security analysis, compliance reporting, and operational troubleshooting. The first copy of management events per region is free. Additional copies cost $2.00 per 100,000 events. Most AWS accounts generate between 10,000 and 100,000 management events per month depending on automation and team size.

Management events should always be enabled for audit logging. Disabling management events eliminates visibility into who made infrastructure changes and when.

### Data Events

Data events capture data plane operations on AWS resources. S3 data events include GetObject, PutObject, and DeleteObject for individual object operations. Lambda data events include Invoke for function executions. DynamoDB data events include GetItem, PutItem, and DeleteItem for individual item operations.

Data events generate 10 to 100 times more volume than management events because every object read, write, or function invocation creates an event. An S3 bucket serving a web application might generate millions of GetObject events daily. A Lambda function processing events might generate thousands of Invoke events per minute.

Data events cost $0.10 per 100,000 events. A moderately active S3 bucket with 1 million GetObject calls per day would cost $30 per month just for data event logging. Data events are disabled by default in this project. Enable them only when compliance requirements mandate object-level or function-level auditing.

### Read vs Write Events

CloudTrail can filter events by operation type. WriteOnly captures configuration changes like PutObject, DeleteBucket, UpdateStack, and CreateRole. ReadOnly captures information retrieval like GetObject, ListBuckets, DescribeInstances, and GetRole. All captures both read and write operations.

Write events have higher security value because they represent changes to infrastructure or data. Read events have high volume but lower security impact. An attacker reading sensitive data leaves read events, but most security analysis focuses on write events showing unauthorized changes.

WriteOnly event selectors reduce costs by 50 to 90 percent compared to All while maintaining visibility into configuration changes. All event selectors are required for complete audit trails in regulated environments or when read access auditing is necessary.

### Recommended Configurations

For personal and demonstration use with cost optimization priority:
- CLOUDTRAIL_INCLUDE_MANAGEMENT_EVENTS: true
- CLOUDTRAIL_INCLUDE_DATA_EVENTS: false
- CLOUDTRAIL_EVENT_SELECTORS: WriteOnly
- ENABLE_CLOUDWATCH_LOGS: true
- CLOUDWATCH_LOGS_RETENTION_DAYS: 90

This configuration captures infrastructure changes while minimizing costs. Monthly cost is approximately $1 to $3 depending on activity.

For production environments with compliance requirements:
- CLOUDTRAIL_INCLUDE_MANAGEMENT_EVENTS: true
- CLOUDTRAIL_INCLUDE_DATA_EVENTS: true
- CLOUDTRAIL_EVENT_SELECTORS: All
- ENABLE_CLOUDWATCH_LOGS: true
- CLOUDWATCH_LOGS_RETENTION_DAYS: 90

This configuration captures complete audit trail including data access. Monthly cost ranges from $50 to $500 depending on S3 and Lambda activity volume.

## Multi-Region Behavior

CloudTrail supports two trail types with different regional coverage.

Multi-region trails capture API events from all AWS regions and deliver them to a single S3 bucket. Events from us-east-1 activities are stored under prefix AWSLogs/{AccountId}/CloudTrail/us-east-1/. Events from eu-west-1 activities are stored under prefix AWSLogs/{AccountId}/CloudTrail/eu-west-1/. Regional prefixes enable filtering by region during analysis.

Multi-region trails automatically capture events from regions enabled in the future without reconfiguration. Global service events from IAM, STS, and CloudFront are always captured in us-east-1 regardless of where they originate. This provides complete account visibility with single trail management.

Single-region trails capture API events only from the deployment region. A trail deployed in us-east-1 with multi-region disabled only captures us-east-1 events and global service events. Activities in other regions are not logged. Single-region trails reduce costs during development when most activity occurs in one region.

This project defaults to multi-region trails for comprehensive coverage. Set CLOUDTRAIL_IS_MULTI_REGION to false for single-region trails during cost-conscious development.

## Cost Estimation

CloudTrail costs vary based on event volume and configuration. These estimates assume moderate activity levels.

### Personal and Demo Configuration

Configuration using recommended settings for cost optimization:
- Management events only: WriteOnly selector
- Data events: Disabled
- CloudWatch Logs: Enabled with 90 day retention
- S3 lifecycle: Intelligent-Tiering immediate, Glacier IA after 90 days
- Multi-region: Enabled

Monthly costs:
- CloudTrail management events: $0 to $2 (first copy free, low volume)
- S3 storage: $0.25 (assuming 10GB accumulated over time with lifecycle optimization)
- CloudWatch Logs ingestion: $0.25 to $0.50 (assuming 500MB to 1GB per month)
- CloudWatch Logs storage: $0.15 (assuming 5GB with 90 day retention)

Total monthly cost: $1 to $3

Annual cost: $12 to $36

### Production and HIPAA Configuration

Configuration for regulatory compliance with complete audit coverage:
- Management events: All selector
- Data events: Enabled for S3 and Lambda
- CloudWatch Logs: Enabled with 90 day retention
- S3 lifecycle: Intelligent-Tiering immediate, Glacier IA after 90 days
- Multi-region: Enabled

Monthly costs:
- CloudTrail management events: $5 to $20 (higher volume with read events)
- CloudTrail data events: $30 to $300 (depends heavily on S3 and Lambda activity)
- S3 storage: $2 to $5 (100GB to 200GB accumulated with lifecycle optimization)
- CloudWatch Logs ingestion: $10 to $50 (2GB to 10GB per month with data events)
- CloudWatch Logs storage: $5 to $15 (15GB to 50GB with 90 day retention)

Total monthly cost: $52 to $390

Annual cost: $624 to $4,680

### Cost Variables

Actual costs depend on:
- Number of AWS accounts and services in use
- Level of automation generating API calls
- Team size and development activity
- S3 object access patterns if data events enabled
- Lambda invocation frequency if data events enabled
- CloudWatch Logs retention period
- S3 lifecycle transition timing

Monitor AWS Cost Explorer for actual costs after deployment. Tag all resources consistently to enable cost allocation analysis.

## Prerequisites

### Required Tools

The following tools must be installed and available in your PATH:

- AWS CLI version 2.x for AWS service interaction and authentication
- Bash version 4.x or higher for script execution
- jq for JSON processing in deployment scripts
- Git for repository information detection and version control

### AWS Account Requirements

You need an AWS account with the following:

- IAM permissions to create CloudFormation stacks, CloudTrail trails, S3 buckets, CloudWatch Logs log groups, IAM roles, and SSM parameters
- No existing CloudTrail trail named governance-cloudtrail-trail in the deployment region
- No existing S3 bucket named governance-cloudtrail-{accountId}
- Account must not have reached service quotas for S3 buckets or CloudTrail trails

The deployment uses your current AWS credentials from the AWS CLI configuration. Administrator access is recommended for initial deployment. Reduced permissions can be used after understanding the specific IAM actions required.

### Git Repository Setup

The repository must be:

- Initialized as a git repository with a remote origin configured
- Working directory must be clean with no uncommitted changes or untracked files
- All commits must be pushed to the remote repository

These requirements ensure deployment metadata is accurate and prevent deploying from inconsistent repository states.

## Quick Start

Clone the repository and navigate to the project directory:

```bash
git clone https://github.com/stephenabbot/governance-cloudtrail.git
cd governance-cloudtrail
```

Review and modify the environment configuration file:

```bash
# Edit .env file with your settings
AWS_REGION=us-east-1
TAG_ENVIRONMENT=prod
TAG_OWNER=your-name
TAG_COST_CENTER=default
CLOUDTRAIL_IS_MULTI_REGION=true
ENABLE_CLOUDWATCH_LOGS=true
```

Verify prerequisites and deploy the foundation:

```bash
./scripts/verify-prerequisites.sh
./scripts/deploy.sh
```

The deployment script will automatically detect your git repository information, AWS account details, and create the CloudTrail infrastructure. After successful deployment, trail resource identifiers will be available in SSM Parameter Store for consuming projects.

List deployed resources to verify the foundation:

```bash
./scripts/list-deployed-resources.sh
```

Verify CloudTrail is logging by checking the trail status:

```bash
aws cloudtrail get-trail-status --name governance-cloudtrail-trail
```

## Deployment Architecture

### Single Account vs Organization Deployment

This project is designed for single AWS account deployment. The CloudTrail trail, S3 bucket, and CloudWatch Logs resources are created within a single account. This architecture is appropriate for personal demonstration, development environments, and small organizations with single-account infrastructure.

AWS Organizations support multi-account environments with centralized audit logging through organization trails. An organization trail deployed in the management account automatically captures events from all member accounts and delivers them to a central S3 bucket. This eliminates the need to deploy CloudTrail in every account and provides unified audit logs across the organization.

Extending this project to organization trail architecture would require:
- Deploying the trail in the AWS Organizations management account
- Configuring the trail with organization trail flag enabled
- Updating S3 bucket policy to allow all organization accounts to write logs
- Publishing organization-level resource identifiers to SSM Parameter Store
- Modifying downstream tools to handle multi-account log prefixes

The single account architecture is intentional for this project to demonstrate foundational audit logging patterns without the complexity of organization management. Organizations seeking centralized audit logging should evaluate organization trails and modify this project accordingly.

### Resource Verification Strategy

The project implements dual verification of deployed resources to ensure deployment accuracy and detect configuration drift. CloudFormation and Terraform state can become inconsistent with actual AWS resource state, making independent verification essential for operational confidence.

Verification uses two complementary approaches:

**Tag-based queries**: All project resources include consistent tags enabling discovery via AWS Resource Groups Tagging API. This method finds resources regardless of CloudFormation stack state and detects orphaned resources from previous deployments.

**Resource-specific API calls**: Direct AWS service API calls verify resource configuration and operational status. CloudTrail get-trail-status confirms logging is active, S3 head-bucket verifies bucket existence and configuration, CloudWatch Logs describe-log-groups confirms log group settings, and SSM get-parameters-by-path validates parameter publication.

The list-deployed-resources script reports discrepancies between CloudFormation state and actual AWS resources. This verification runs independently of deployment and destruction operations, enabling ongoing drift detection and operational validation.

After stack destruction, resources are categorized as either "retained by design" (resources with DeletionPolicy: Retain like the S3 bucket) or "potential orphans" (resources that should have been deleted but still exist). This categorization helps distinguish expected retention from cleanup failures.

### Resource Verification Strategy

The project implements dual verification of deployed resources to ensure deployment accuracy and detect configuration drift. CloudFormation and Terraform state can become inconsistent with actual AWS resource state, making independent verification essential for operational confidence.

Verification uses two complementary approaches:

**Tag-based queries**: All project resources include consistent tags enabling discovery via AWS Resource Groups Tagging API. This method finds resources regardless of CloudFormation stack state and detects orphaned resources from previous deployments.

**Resource-specific API calls**: Direct AWS service API calls verify resource configuration and operational status. CloudTrail get-trail-status confirms logging is active, S3 head-bucket verifies bucket existence and configuration, CloudWatch Logs describe-log-groups confirms log group settings, and SSM get-parameters-by-path validates parameter publication.

The list-deployed-resources script reports discrepancies between CloudFormation state and actual AWS resources. This verification runs independently of deployment and destruction operations, enabling ongoing drift detection and operational validation.

After stack destruction, resources are categorized as either "retained by design" (resources with DeletionPolicy: Retain like the S3 bucket) or "potential orphans" (resources that should have been deleted but still exist). This categorization helps distinguish expected retention from cleanup failures.

### Compliance Considerations

This project provides a compliance-ready foundation for regulated environments when configured appropriately. The architecture supports HIPAA, PCI-DSS, and SOC 2 audit requirements with continuous logging, encrypted storage, and log file integrity validation.

For compliance environments:
- Enable multi-region trails to capture all regional activity
- Include both management and data events for complete audit coverage
- Use All event selectors to capture both read and write operations
- Enable CloudWatch Logs for real-time monitoring and alerting
- Set S3 expiration to match regulatory retention requirements
- Deploy continuously rather than enabling selectively
- Implement additional controls like CloudTrail Insights for anomaly detection
- Configure SNS notifications for trail configuration changes
- Implement AWS Config rules to monitor CloudTrail compliance

For personal demonstration and development:
- Single-region trails reduce costs for accounts with limited regional activity
- Management events only provide sufficient visibility for most development workflows
- WriteOnly event selectors reduce costs while maintaining security visibility
- Selective enablement during active development followed by disablement during idle periods is acceptable
- CloudWatch Logs can be disabled to reduce ingestion costs if real-time querying is not needed

The destroy script preserves S3 buckets by default to prevent accidental audit log deletion. This conservative approach protects against compliance violations from unintended infrastructure cleanup.

### Destroy Behavior and S3 Retention

The destroy script implements special handling for S3 bucket retention to prevent accidental audit log deletion.

When running the destroy script, you will be prompted twice:
- First confirmation: Type DESTROY to confirm stack deletion intent
- Second confirmation: Choose whether to retain or delete the S3 bucket

The S3 bucket contains all historical audit logs and should generally be retained even when CloudTrail infrastructure is destroyed. The bucket has DeletionPolicy set to Retain in the CloudFormation template, which preserves the bucket when the stack is deleted.

Choosing to retain the bucket during destroy:
- CloudFormation stack is deleted
- CloudTrail trail is deleted
- CloudWatch Logs log group is deleted if it exists
- IAM role for CloudWatch Logs is deleted if it exists
- SSM parameters are deleted
- S3 bucket remains in the account with all audit logs intact

Choosing to delete the bucket during destroy:
- All resources are deleted including the S3 bucket
- Audit logs are permanently lost
- This action cannot be undone
- Only choose this option if you are certain audit logs are not needed

For compliance environments, always retain the S3 bucket when destroying the stack. For personal development, delete the bucket only if you are certain no audit logs are needed for analysis.

## Troubleshooting

### Stack Creation Failures

If the CloudFormation stack fails to create, check the AWS CloudFormation console for detailed error messages. Common issues include:

Insufficient IAM permissions for the deploying user or role. Verify the user has permissions to create CloudFormation stacks, CloudTrail trails, S3 buckets, S3 bucket policies, CloudWatch Logs log groups, IAM roles, and SSM parameters.

Service quota limits reached for CloudTrail trails or S3 buckets. Check AWS Service Quotas for current limits and request increases if needed.

Existing resources with conflicting names. Verify no trail named governance-cloudtrail-trail exists and no S3 bucket named governance-cloudtrail-{accountId} exists in the account.

Invalid parameter values in the .env configuration file. Review parameter types and allowed values. Boolean parameters must be lowercase true or false.

Review the CloudFormation events tab to identify which resource failed and why. The deployment script includes rollback handling, so failed deployments will clean up partial resources automatically.

### CloudTrail Not Logging

If CloudTrail appears deployed but is not delivering logs to S3, verify the following:

Check trail status shows IsLogging as true using aws cloudtrail get-trail-status command. If IsLogging is false, the trail was stopped manually and needs to be restarted.

Verify S3 bucket policy allows CloudTrail service principal to write objects. Review the bucket policy in the S3 console and confirm it includes PutObject permissions for the CloudTrail service.

Check IAM role permissions for CloudWatch Logs delivery if CloudWatch Logs integration is enabled. The role must have PutLogEvents permissions for the CloudWatch Logs log group.

Verify no service control policies in AWS Organizations block CloudTrail logging. Organization SCPs can prevent CloudTrail from functioning even with correct IAM permissions.

Wait 15 minutes after trail creation before expecting log delivery. CloudTrail delivers logs every 5 to 15 minutes, so new trails may not show logs immediately.

### CloudWatch Logs Not Receiving Events

If CloudWatch Logs integration is enabled but events are not appearing in the log group, verify:

The CloudWatch Logs log group exists with correct name governance-cloudtrail-logs. Check the CloudWatch Logs console to confirm log group creation.

The IAM role for CloudWatch Logs delivery exists and has the correct trust policy allowing CloudTrail to assume it. Review the role trust policy in the IAM console.

The trail configuration includes CloudWatchLogsLogGroupArn and CloudWatchLogsRoleArn properties. Describe the trail using aws cloudtrail describe-trails to verify these properties are set.

No service control policies block CloudWatch Logs permissions. Organization SCPs can prevent CloudWatch Logs delivery even with correct IAM configuration.

Wait 5 minutes after trail creation before expecting CloudWatch Logs delivery. Initial log stream creation may take a few minutes.

### S3 Bucket Access Issues

If downstream tools cannot access CloudTrail logs in S3, verify:

The S3 bucket exists and is named correctly as governance-cloudtrail-{accountId}. List buckets using aws s3 ls to confirm bucket creation.

The bucket policy allows the consuming tool's IAM role to access objects. CloudTrail bucket policy only grants CloudTrail service access by default. Modify the policy to grant GetObject permissions to consuming roles.

Objects exist in the bucket at the expected prefix AWSLogs/{AccountId}/CloudTrail/{Region}/. Use aws s3 ls to browse bucket contents.

No lifecycle policies have transitioned objects to Glacier that the consuming tool cannot access. Check lifecycle rules and object storage class. Tools like Athena can query Glacier Instant Access but not Glacier Flexible Retrieval.

### SSM Parameter Issues

If consuming projects cannot find CloudTrail resource identifiers in SSM Parameter Store, verify:

The SSM parameters exist at /governance/cloudtrail/ prefix in us-east-1 region. Use aws ssm get-parameters-by-path to list all parameters.

The consuming project is querying SSM in the correct region. All CloudTrail parameters are published to us-east-1 regardless of trail deployment region.

The consuming project's IAM role has ssm:GetParameter permissions for /governance/cloudtrail/* path. Check IAM policies on the consuming role.

Parameter names match exactly what the consuming project expects. SSM parameter names are case-sensitive.

## Technologies and Services

### Infrastructure as Code

- CloudFormation for declarative infrastructure deployment with drift detection and change sets
- Bash scripting for deployment automation, prerequisite validation, and operational workflows

### AWS Services

- CloudTrail for API activity logging and audit trail creation
- S3 for durable audit log storage with versioning, encryption, and lifecycle management
- CloudWatch Logs for real-time event streaming and querying when enabled
- IAM for CloudWatch Logs delivery role and service trust policies
- SSM Parameter Store for resource identifier publishing and service discovery
- CloudFormation for declarative infrastructure management and automated rollback

### Development Tools

- AWS CLI for service interaction and credential management
- Git for version control and repository metadata detection
- jq for JSON processing in deployment scripts and parameter handling
- Bash for cross-platform script execution and automation

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

© 2025 Stephen Abbot - MIT License
