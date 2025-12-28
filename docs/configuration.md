# Configuration

## Environment Variables Reference

### AWS Configuration

**AWS_REGION**
- Purpose: Target AWS region for all resource deployment
- Valid Values: Any valid AWS region (e.g., us-east-1, eu-west-1)
- Default: us-east-1
- Impact: Determines where CloudFormation stack and all resources are created

### Resource Tagging

**TAG_ENVIRONMENT**
- Purpose: Environment identifier for resource categorization
- Valid Values: prd, stg, tst, dev
- Required: Yes
- Impact: Applied to all resources for cost allocation and environment identification

**TAG_OWNER**
- Purpose: Resource owner identifier for accountability
- Valid Values: Any string identifier
- Example: StephenAbbot, TeamName, DepartmentCode
- Impact: Applied to all resources for ownership tracking

**TAG_COST_CENTER**
- Purpose: Cost allocation identifier for billing
- Valid Values: Any string identifier
- Default: default
- Impact: Used for cost tracking and chargeback processes

### Deployment Behavior

**AUTO_DELETE_FAILED_STACK**
- Purpose: Automatic cleanup of failed CloudFormation stacks
- Valid Values: true, false
- Default: false
- Impact: When true, automatically deletes stacks in failed states before retry

### CloudTrail Configuration

**CLOUDTRAIL_IS_MULTI_REGION**
- Purpose: Enable multi-region trail for global event capture
- Valid Values: true, false
- Default: true
- Impact: When true, captures events from all AWS regions; when false, only current region

**CLOUDTRAIL_INCLUDE_MANAGEMENT_EVENTS**
- Purpose: Capture AWS management API calls
- Valid Values: true, false
- Default: true
- Impact: Controls logging of control plane operations (create, delete, modify resources)

**CLOUDTRAIL_INCLUDE_DATA_EVENTS**
- Purpose: Capture data plane API calls
- Valid Values: true, false
- Default: false
- Impact: When true, logs S3 object operations and Lambda function invocations (increases costs)

**CLOUDTRAIL_EVENT_SELECTORS**
- Purpose: Types of events to capture
- Valid Values: WriteOnly, ReadOnly, All
- Default: WriteOnly
- Impact: WriteOnly captures create/modify/delete; ReadOnly captures read operations; All captures everything

### CloudWatch Logs Integration

**ENABLE_CLOUDWATCH_LOGS**
- Purpose: Stream CloudTrail events to CloudWatch Logs
- Valid Values: true, false
- Default: true
- Impact: Enables real-time log analysis and alerting capabilities

**CLOUDWATCH_LOGS_RETENTION_DAYS**
- Purpose: CloudWatch Logs retention period
- Valid Values: 1, 3, 5, 7, 14, 30, 60, 90, 120, 150, 180, 365, 400, 545, 731, 1827, 3653
- Default: 90
- Impact: Controls how long logs are retained in CloudWatch Logs (affects costs)

### S3 Lifecycle Management

**S3_INTELLIGENT_TIERING_DAYS**
- Purpose: Days before transitioning to Intelligent-Tiering storage class
- Valid Values: 0 or positive integer
- Default: 0 (immediate transition)
- Impact: Optimizes storage costs by automatically moving objects between access tiers

**S3_GLACIER_TRANSITION_DAYS**
- Purpose: Days before transitioning to Glacier Instant Retrieval
- Valid Values: Positive integer (minimum 30 for Glacier IA)
- Default: 90
- Impact: Reduces storage costs for infrequently accessed audit logs

**S3_EXPIRATION_DAYS**
- Purpose: Days before objects are permanently deleted
- Valid Values: Positive integer
- Default: 2555 (approximately 7 years)
- Impact: Automatic cleanup of old audit logs to manage storage costs

## Configuration Examples

### Production Environment

```bash
# Production configuration with comprehensive logging
AWS_REGION=us-east-1
TAG_ENVIRONMENT=prd
TAG_OWNER=SecurityTeam
TAG_COST_CENTER=security-ops
AUTO_DELETE_FAILED_STACK=false
CLOUDTRAIL_IS_MULTI_REGION=true
CLOUDTRAIL_INCLUDE_MANAGEMENT_EVENTS=true
CLOUDTRAIL_INCLUDE_DATA_EVENTS=true
CLOUDTRAIL_EVENT_SELECTORS=All
ENABLE_CLOUDWATCH_LOGS=true
CLOUDWATCH_LOGS_RETENTION_DAYS=365
S3_INTELLIGENT_TIERING_DAYS=0
S3_GLACIER_TRANSITION_DAYS=90
S3_EXPIRATION_DAYS=2555
```

### Development Environment

```bash
# Development configuration with cost optimization
AWS_REGION=us-east-1
TAG_ENVIRONMENT=dev
TAG_OWNER=DevTeam
TAG_COST_CENTER=development
AUTO_DELETE_FAILED_STACK=true
CLOUDTRAIL_IS_MULTI_REGION=false
CLOUDTRAIL_INCLUDE_MANAGEMENT_EVENTS=true
CLOUDTRAIL_INCLUDE_DATA_EVENTS=false
CLOUDTRAIL_EVENT_SELECTORS=WriteOnly
ENABLE_CLOUDWATCH_LOGS=false
CLOUDWATCH_LOGS_RETENTION_DAYS=30
S3_INTELLIGENT_TIERING_DAYS=0
S3_GLACIER_TRANSITION_DAYS=30
S3_EXPIRATION_DAYS=365
```

### Compliance Environment

```bash
# Compliance configuration with extended retention
AWS_REGION=us-east-1
TAG_ENVIRONMENT=prd
TAG_OWNER=ComplianceTeam
TAG_COST_CENTER=compliance
AUTO_DELETE_FAILED_STACK=false
CLOUDTRAIL_IS_MULTI_REGION=true
CLOUDTRAIL_INCLUDE_MANAGEMENT_EVENTS=true
CLOUDTRAIL_INCLUDE_DATA_EVENTS=true
CLOUDTRAIL_EVENT_SELECTORS=All
ENABLE_CLOUDWATCH_LOGS=true
CLOUDWATCH_LOGS_RETENTION_DAYS=3653
S3_INTELLIGENT_TIERING_DAYS=0
S3_GLACIER_TRANSITION_DAYS=365
S3_EXPIRATION_DAYS=7300
```

## Cost Optimization Guidelines

### Low-Cost Configuration

- Set CLOUDTRAIL_INCLUDE_DATA_EVENTS=false to avoid data event charges
- Use CLOUDTRAIL_EVENT_SELECTORS=WriteOnly to capture only essential events
- Set ENABLE_CLOUDWATCH_LOGS=false if real-time analysis not required
- Use shorter CLOUDWATCH_LOGS_RETENTION_DAYS for cost savings
- Set aggressive S3_GLACIER_TRANSITION_DAYS for faster cost reduction

### High-Compliance Configuration

- Set CLOUDTRAIL_INCLUDE_DATA_EVENTS=true for comprehensive audit trail
- Use CLOUDTRAIL_EVENT_SELECTORS=All to capture all event types
- Set ENABLE_CLOUDWATCH_LOGS=true for real-time monitoring
- Use longer retention periods for both CloudWatch Logs and S3
- Consider regulatory requirements for retention periods

## Security Considerations

### Event Capture Scope

- Management events capture control plane operations essential for security monitoring
- Data events provide detailed audit trail but increase costs significantly
- Multi-region trails ensure global visibility of AWS API activity
- Event selectors should balance security requirements with cost constraints

### Access Control

- S3 bucket policy restricts access to CloudTrail service only
- CloudWatch Logs integration requires dedicated IAM role with minimal permissions
- SSM parameters enable service discovery without exposing sensitive information
- Resource tagging provides accountability and access tracking

### Retention Policies

- S3 lifecycle policies balance cost optimization with compliance requirements
- CloudWatch Logs retention should align with real-time monitoring needs
- Consider legal and regulatory requirements when setting retention periods
- Intelligent Tiering optimizes costs while maintaining access capabilities
