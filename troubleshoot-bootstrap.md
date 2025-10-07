# Bootstrap WaitCondition Failure Troubleshooting

## Issue
The `IDEIdeBootstrapWaitCondition689B05CD` resource failed with "Configuration failed" message from instance `i-03f0a496ab6758092`.

## Root Cause Analysis

The bootstrap process involves:
1. EC2 instance launches with basic UserData
2. Lambda function triggers SSM document execution on the instance
3. SSM document runs extensive bootstrap script
4. Script signals success/failure to CloudFormation WaitCondition
5. **FAILURE**: Script signaled failure, causing WaitCondition to fail

## Troubleshooting Steps

### 1. Check CloudWatch Logs
```bash
aws logs describe-log-groups --log-group-name-prefix "/aws/ssm"
aws logs get-log-events --log-group-name "IDEIdeLogGroup2C309711" --log-stream-name "LATEST"
```

### 2. Check SSM Command History
```bash
aws ssm describe-instance-information --filters "Key=InstanceIds,Values=i-03f0a496ab6758092"
aws ssm list-command-invocations --instance-id i-03f0a496ab6758092 --details
```

### 3. Check Instance Status
```bash
aws ec2 describe-instances --instance-ids i-03f0a496ab6758092
aws ssm describe-instance-information --instance-information-filter-list key=InstanceIds,valueSet=i-03f0a496ab6758092
```

## Common Failure Reasons

### 1. Network/Internet Connectivity Issues
- Instance can't reach internet for package downloads
- Security group blocking outbound traffic
- Route table misconfiguration

### 2. IAM Permission Issues
- Instance profile missing required permissions
- Secrets Manager access denied
- ECR/S3 access issues

### 3. Resource Constraints
- Insufficient disk space for installations
- Memory issues during compilation
- Timeout during long-running operations

### 4. Package Installation Failures
- Terraform download/installation failure
- Docker installation issues
- Code-server installation problems
- Go/Node.js installation failures

### 5. Application-Specific Issues
- Terraform state bucket access issues
- EKS cluster creation failures
- Container build/push failures

## Quick Fixes

### 1. Increase Timeout
The WaitCondition timeout is set to 900 seconds (15 minutes). The bootstrap script is extensive and may need more time.

### 2. Check Required Resources
Ensure these resources exist and are accessible:
- S3 bucket: `TerraformRunnerTerraformStateBucket72F4F64A`
- Secrets Manager secret: `IDEIdePasswordSecret13714D56`
- GitHub repository: `https://github.com/aws-samples/sample-agentic-frameworks-on-aws`

### 3. Verify IAM Permissions
The `SharedRoleD1D02F7E` role needs:
- AdministratorAccess (already granted)
- AmazonSSMManagedInstanceCore (already granted)
- Access to Secrets Manager secret
- Access to S3 bucket

## Resolution Steps

1. **Delete the failed stack** (if safe to do so)
2. **Check AWS service limits** (EC2, EKS, etc.)
3. **Verify region availability** for all required services
4. **Re-deploy with monitoring** enabled
5. **Consider splitting bootstrap** into smaller chunks

## Monitoring During Re-deployment

```bash
# Monitor SSM command execution
aws ssm list-commands --instance-id i-03f0a496ab6758092

# Monitor CloudWatch logs in real-time
aws logs tail IDEIdeLogGroup2C309711 --follow

# Check instance system logs
aws ec2 get-console-output --instance-id i-03f0a496ab6758092
```

## Prevention

1. **Test bootstrap script** in isolation
2. **Add more granular error handling** in the script
3. **Implement retry logic** for network-dependent operations
4. **Use CloudWatch alarms** for early failure detection
5. **Consider using AWS Systems Manager Patch Manager** for OS updates