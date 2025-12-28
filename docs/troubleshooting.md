# Troubleshooting

## CloudFormation Stack Issues

### Stack Creation Failures

**Symptoms:**
- Stack creation fails with CREATE_FAILED status
- Resources show CREATE_FAILED in stack events
- Deployment script exits with error during stack creation

**Common Causes:**
- Insufficient IAM permissions for resource creation
- Resource naming conflicts with existing resources
- Invalid parameter values in .env configuration
- CloudFormation template syntax errors

**Resolution Steps:**
1. Check CloudFormation events for specific error messages
2. Verify IAM permissions match prerequisites documentation
3. Ensure no existing resources conflict with deployment
4. Validate .env configuration against requirements
5. Run template validation: aws cloudformation validate-template --template-body file://cloudformation/bootstrap.yaml

### Stack Update Failures

**Symptoms:**
- Stack update fails with UPDATE_FAILED or UPDATE_ROLLBACK_COMPLETE status
- Changes not applied despite successful script execution
- Resources stuck in UPDATE_IN_PROGRESS state

**Common Causes:**
- Parameter changes require resource replacement
- Resource dependencies prevent updates
- External changes to resources outside CloudFormation
- Insufficient permissions for update operations

**Resolution Steps:**
1. Review stack events to identify failed resource updates
2. Check if parameter changes require resource replacement
3. Verify no manual changes were made to stack resources
4. Consider stack deletion and recreation for major changes
5. Use AUTO_DELETE_FAILED_STACK=true for automatic cleanup

### Stack Deletion Issues

**Symptoms:**
- Stack deletion fails with DELETE_FAILED status
- Resources remain after deletion attempt
- S3 bucket deletion blocked by retained objects

**Common Causes:**
- S3 bucket contains objects preventing deletion
- IAM roles attached to resources outside stack
- CloudTrail still logging to S3 bucket
- Resource dependencies prevent deletion order

**Resolution Steps:**
1. Empty S3 bucket before stack deletion if not using retention policy
2. Stop CloudTrail logging before deletion
3. Remove external dependencies on stack resources
4. Use destroy.sh script for guided deletion process
5. Manually delete orphaned resources if necessary

## CloudTrail Configuration Problems

### Trail Not Logging

**Symptoms:**
- CloudTrail status shows IsLogging: false
- No events appearing in S3 bucket or CloudWatch Logs
- Trail exists but not capturing events

**Diagnosis Steps:**
1. Check trail status: aws cloudtrail get-trail-status --name governance-cloudtrail-trail
2. Verify trail configuration: aws cloudtrail describe-trails --trail-name-list governance-cloudtrail-trail
3. Check S3 bucket policy allows CloudTrail access
4. Verify CloudWatch Logs role permissions if enabled

**Resolution:**
- Ensure trail is enabled: aws cloudtrail start-logging --name governance-cloudtrail-trail
- Verify S3 bucket policy matches CloudFormation template
- Check CloudWatch Logs role has correct permissions
- Confirm event selectors are properly configured

### Missing Events in Logs

**Symptoms:**
- Some API calls not appearing in CloudTrail logs
- Data events missing despite configuration
- Management events incomplete

**Common Causes:**
- Event selectors configured incorrectly
- Data events disabled for cost optimization
- Regional vs global service event capture
- Service-specific logging limitations

**Resolution Steps:**
1. Review event selector configuration in .env file
2. Enable data events if required: CLOUDTRAIL_INCLUDE_DATA_EVENTS=true
3. Verify multi-region trail captures global service events
4. Check service-specific CloudTrail documentation for limitations
5. Allow time for event delivery (up to 15 minutes typical)

## S3 Bucket Access Issues

### Bucket Policy Errors

**Symptoms:**
- CloudTrail cannot write to S3 bucket
- Access denied errors in CloudTrail status
- Bucket policy validation failures

**Diagnosis Steps:**
1. Check bucket policy: aws s3api get-bucket-policy --bucket governance-cloudtrail-{ACCOUNT_ID}
2. Verify CloudTrail service principal permissions
3. Confirm account ID matches in policy conditions
4. Test bucket access with CloudTrail service

**Resolution:**
- Ensure bucket policy matches CloudFormation template exactly
- Verify account ID substitution in policy conditions
- Check bucket exists and is accessible
- Redeploy stack to reset bucket policy if corrupted

### Lifecycle Policy Problems

**Symptoms:**
- Objects not transitioning to expected storage classes
- Unexpected storage costs from lifecycle rules
- Objects expiring sooner than expected

**Common Causes:**
- Lifecycle rule configuration errors in .env
- Conflicting lifecycle rules
- Object versioning affecting rule application
- Minimum storage duration requirements not met

**Resolution Steps:**
1. Review lifecycle configuration: aws s3api get-bucket-lifecycle-configuration --bucket governance-cloudtrail-{ACCOUNT_ID}
2. Verify .env lifecycle parameters are reasonable values
3. Check object versioning is enabled for lifecycle rules
4. Understand storage class minimum duration requirements
5. Monitor storage class transitions in S3 console

## CloudWatch Logs Integration Issues

### Log Group Not Receiving Events

**Symptoms:**
- CloudWatch Logs log group exists but empty
- CloudTrail status shows no CloudWatch Logs delivery
- Events appearing in S3 but not CloudWatch Logs

**Diagnosis Steps:**
1. Verify ENABLE_CLOUDWATCH_LOGS=true in .env configuration
2. Check CloudWatch Logs role exists and has correct permissions
3. Confirm log group exists: aws logs describe-log-groups --log-group-name-prefix governance-cloudtrail
4. Review CloudTrail configuration for CloudWatch Logs ARN

**Resolution:**
- Ensure CloudWatch Logs integration is enabled in configuration
- Verify IAM role has logs:CreateLogStream and logs:PutLogEvents permissions
- Check log group ARN matches CloudTrail configuration
- Redeploy stack to recreate CloudWatch Logs integration

### Log Retention Issues

**Symptoms:**
- Logs expiring sooner than expected
- Retention policy not applied correctly
- Unexpected CloudWatch Logs costs

**Common Causes:**
- CLOUDWATCH_LOGS_RETENTION_DAYS configured incorrectly
- Retention policy not applied to existing log streams
- Multiple log groups with different retention settings

**Resolution Steps:**
1. Verify retention setting in .env: CLOUDWATCH_LOGS_RETENTION_DAYS
2. Check applied retention: aws logs describe-log-groups --log-group-name-prefix governance-cloudtrail
3. Understand retention applies to new log streams
4. Consider cost implications of retention period
5. Update configuration and redeploy if needed

## Deployment Script Failures

### Prerequisites Verification Failures

**Symptoms:**
- verify-prerequisites.sh script fails with tool or permission errors
- Deployment blocked by prerequisite checks
- AWS credential or permission issues

**Resolution Steps:**
1. Install missing tools: AWS CLI v2, jq, git
2. Configure AWS credentials: aws configure or environment variables
3. Verify IAM permissions match prerequisites documentation
4. Check git repository state and remote configuration
5. Validate .env file contains all required variables

### Environment Configuration Errors

**Symptoms:**
- Deployment script fails with missing variable errors
- Invalid parameter values cause CloudFormation failures
- Configuration validation errors

**Common Issues:**
- Missing or empty variables in .env file
- Invalid values for boolean or numeric parameters
- Incorrect AWS region specification
- Malformed tag values

**Resolution Steps:**
1. Copy .env.example to .env if missing
2. Verify all required variables are set and non-empty
3. Check boolean values are exactly 'true' or 'false'
4. Validate numeric values are within acceptable ranges
5. Ensure tag values follow AWS tagging requirements

## Resource Discovery Issues

### SSM Parameters Missing

**Symptoms:**
- list-deployed-resources.sh shows no SSM parameters
- Consuming projects cannot find infrastructure outputs
- Service discovery integration broken

**Diagnosis Steps:**
1. Check SSM parameters exist: aws ssm get-parameters-by-path --path /governance/cloudtrail/
2. Verify CloudFormation stack created SSM parameter resources
3. Confirm parameter names match expected paths
4. Check parameter values contain correct resource identifiers

**Resolution:**
- Redeploy stack to recreate missing SSM parameters
- Verify CloudFormation template includes all SSM parameter resources
- Check IAM permissions for SSM parameter creation
- Validate parameter paths match consuming project expectations

### Resource Verification Discrepancies

**Symptoms:**
- CloudFormation reports resources but AWS APIs show different state
- Resource counts don't match between stack and actual resources
- Orphaned resources detected outside CloudFormation management

**Common Causes:**
- Manual changes to resources outside CloudFormation
- Failed resource deletions leaving orphans
- Resource retention policies preventing cleanup
- External tools modifying stack resources

**Resolution Steps:**
1. Use list-deployed-resources.sh to identify discrepancies
2. Import orphaned resources into CloudFormation stack if possible
3. Manually clean up resources not managed by stack
4. Avoid manual changes to CloudFormation-managed resources
5. Use stack drift detection to identify configuration changes
