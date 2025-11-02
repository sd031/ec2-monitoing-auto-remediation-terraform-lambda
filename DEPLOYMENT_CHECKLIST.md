# Deployment Checklist

Use this checklist to ensure successful deployment and testing.

## Pre-Deployment

### ✅ Prerequisites
- [ ] AWS Account created
- [ ] AWS CLI installed and configured
- [ ] Terraform installed (>= 1.0)
- [ ] Python 3.11+ installed
- [ ] pip3 installed
- [ ] Git installed (optional)

### ✅ Verify AWS Access
```bash
# Check AWS credentials
aws sts get-caller-identity

# Expected output: Account ID, User ARN, User ID
```

### ✅ Configuration
- [ ] Copy `terraform.tfvars.example` to `terraform.tfvars`
- [ ] Set `alarm_email` to your email address
- [ ] Review and adjust thresholds if needed
- [ ] Verify AWS region setting

## Deployment

### ✅ Build Lambda Package
```bash
# Make build script executable
chmod +x scripts/build_lambda.sh

# Build Lambda package for x86_64
./scripts/build_lambda.sh

# Verify package created
ls -lh lambda_function.zip
```

### ✅ Initialize Terraform
```bash
terraform init

# Expected: Success message, .terraform directory created
```

### ✅ Validate Configuration
```bash
terraform validate

# Expected: "Success! The configuration is valid."
```

### ✅ Plan Deployment
```bash
terraform plan

# Review the plan:
# - Should create ~25 resources
# - Check resource names match project_name
# - Verify no unexpected changes
```

### ✅ Deploy Infrastructure
```bash
# Option 1: Using deploy script
./scripts/deploy.sh

# Option 2: Using Terraform
terraform apply

# Type 'yes' when prompted
```

### ✅ Verify Deployment
```bash
# Check outputs
terraform output

# Expected outputs:
# - instance_id
# - instance_public_ip
# - dashboard_url
# - sns_topic_arn
# - lambda_function_name
# - cloudwatch_log_group
# - alarm_names
```

## Post-Deployment

### ✅ Confirm SNS Subscription
- [ ] Check email inbox
- [ ] Find "AWS Notification - Subscription Confirmation"
- [ ] Click "Confirm subscription" link
- [ ] Verify confirmation page loads

### ✅ Verify EC2 Instance
```bash
# Check instance is running
aws ec2 describe-instances \
  --instance-ids $(terraform output -raw instance_id) \
  --query 'Reservations[0].Instances[0].State.Name'

# Expected: "running"
```

### ✅ Verify CloudWatch Agent
```bash
# Wait 5 minutes after deployment

# Check if metrics are being reported
aws cloudwatch list-metrics \
  --namespace CWAgent \
  --dimensions Name=InstanceId,Value=$(terraform output -raw instance_id)

# Expected: List of custom metrics (mem_used_percent, disk_used_percent, etc.)
```

### ✅ Verify Alarms
```bash
# List all alarms
aws cloudwatch describe-alarms \
  --alarm-name-prefix ec2-monitoring

# Expected: 4 alarms in OK or INSUFFICIENT_DATA state
```

### ✅ Verify Lambda Function
```bash
# Check Lambda function exists
aws lambda get-function \
  --function-name $(terraform output -raw lambda_function_name)

# Expected: Function configuration details
```

### ✅ Access Dashboard
```bash
# Get dashboard URL
terraform output dashboard_url

# Open in browser
open $(terraform output -raw dashboard_url)

# Verify:
# - Dashboard loads successfully
# - Widgets display (may show "No data" initially)
```

## Testing

### ✅ Wait for Metrics (Important!)
- [ ] Wait 5-10 minutes after deployment
- [ ] CloudWatch Agent needs time to start reporting
- [ ] Refresh dashboard to see metrics appear

### ✅ Test CPU Alarm
```bash
# Connect to instance
aws ssm start-session --target $(terraform output -raw instance_id)

# Run CPU stress test (on EC2 instance)
sudo stress --cpu $(nproc) --timeout 300s

# Expected after 10 minutes:
# - Email notification (alarm triggered)
# - Dashboard shows CPU spike
# - Instance reboots (auto-remediation)
# - Email notification (remediation completed)
```

### ✅ Test Memory Alarm
```bash
# On EC2 instance
sudo stress --vm 1 --vm-bytes 512M --timeout 300s

# Expected after 10 minutes:
# - Email notification (alarm triggered)
# - Dashboard shows memory spike
# - Instance reboots (auto-remediation)
# - Email notification (remediation completed)
```

### ✅ Verify Lambda Logs
```bash
# View Lambda logs
aws logs tail /aws/lambda/ec2-monitoring-auto-remediation --follow

# Expected:
# - Log entries for Lambda invocations
# - Remediation action details
# - Success/error messages
```

### ✅ Verify EC2 Logs
```bash
# View EC2 logs
aws logs tail /aws/ec2/ec2-monitoring --follow

# Expected:
# - System log entries
# - CloudWatch Agent logs
# - Application logs
```

## Validation

### ✅ Dashboard Validation
- [ ] CPU widget shows data
- [ ] Memory widget shows data
- [ ] Disk widget shows data
- [ ] Network widget shows data
- [ ] Status checks widget shows data
- [ ] Log widget shows recent entries
- [ ] Single-value widgets show current values

### ✅ Alarm Validation
- [ ] All alarms in OK state (after initial data)
- [ ] Alarm history shows state changes
- [ ] Alarm actions configured (SNS + Lambda)

### ✅ Notification Validation
- [ ] Received alarm notification email
- [ ] Email contains alarm details
- [ ] Received remediation notification email
- [ ] Email contains remediation details

### ✅ Remediation Validation
- [ ] Lambda function triggered by alarm
- [ ] Instance rebooted successfully
- [ ] Cooldown period enforced
- [ ] No repeated actions within 30 minutes

## Troubleshooting

### ❌ No Metrics in Dashboard
**Cause**: CloudWatch Agent not started or misconfigured

**Solution**:
```bash
# SSH to instance
aws ssm start-session --target $(terraform output -raw instance_id)

# Check agent status
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a query -m ec2 -c default -s

# Restart agent if needed
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 -s \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/config.json
```

### ❌ Alarm Not Triggering
**Cause**: Threshold not exceeded or insufficient data

**Solution**:
- Increase stress test duration
- Lower threshold temporarily
- Check metric is being reported
- Verify evaluation periods

### ❌ Lambda Not Executing
**Cause**: Permission issue or alarm action not configured

**Solution**:
```bash
# Check Lambda permissions
aws lambda get-policy \
  --function-name $(terraform output -raw lambda_function_name)

# Check alarm actions
aws cloudwatch describe-alarms \
  --alarm-names ec2-monitoring-high-cpu \
  --query 'MetricAlarms[0].AlarmActions'
```

### ❌ SNS Subscription Not Confirmed
**Cause**: Email not received or link expired

**Solution**:
```bash
# Resend subscription
aws sns subscribe \
  --topic-arn $(terraform output -raw sns_topic_arn) \
  --protocol email \
  --notification-endpoint your-email@example.com
```

### ❌ Terraform Apply Fails
**Cause**: Various (permissions, quotas, syntax)

**Solution**:
- Read error message carefully
- Check AWS service quotas
- Verify IAM permissions
- Validate Terraform syntax: `terraform validate`
- Check Terraform state: `terraform state list`

## Cleanup

### ✅ Before Destroying
- [ ] Export any important logs
- [ ] Save dashboard configuration
- [ ] Document any customizations
- [ ] Verify no production data

### ✅ Destroy Resources
```bash
# Option 1: Using destroy script
./scripts/destroy.sh

# Option 2: Using Terraform
terraform destroy

# Type 'yes' when prompted
```

### ✅ Verify Cleanup
```bash
# Check no EC2 instances
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=ec2-monitoring-*" \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name]'

# Check no alarms
aws cloudwatch describe-alarms \
  --alarm-name-prefix ec2-monitoring

# Check no Lambda functions
aws lambda list-functions \
  --query 'Functions[?starts_with(FunctionName, `ec2-monitoring`)]'

# Expected: Empty results for all
```

### ✅ Clean Local Files
```bash
# Remove generated files
rm -f lambda_function.zip
rm -f tfplan
rm -f terraform.tfstate*
rm -rf .terraform/
rm -f .terraform.lock.hcl

# Keep configuration
# Don't delete: terraform.tfvars (if you want to redeploy)
```

## Cost Monitoring

### ✅ Set Up Billing Alerts
```bash
# Create billing alarm (optional)
aws cloudwatch put-metric-alarm \
  --alarm-name high-billing \
  --alarm-description "Alert when charges exceed $50" \
  --metric-name EstimatedCharges \
  --namespace AWS/Billing \
  --statistic Maximum \
  --period 21600 \
  --evaluation-periods 1 \
  --threshold 50 \
  --comparison-operator GreaterThanThreshold
```

### ✅ Monitor Costs
- [ ] Check AWS Cost Explorer daily
- [ ] Review CloudWatch costs
- [ ] Monitor EC2 usage
- [ ] Track Lambda invocations
- [ ] Review data transfer costs

## Documentation

### ✅ Review Documentation
- [ ] Read README.md (comprehensive guide)
- [ ] Read QUICKSTART.md (5-minute setup)
- [ ] Read ARCHITECTURE.md (detailed architecture)
- [ ] Read PROJECT_OVERVIEW.md (high-level overview)

### ✅ Understand Components
- [ ] Terraform infrastructure code
- [ ] Lambda function logic
- [ ] CloudWatch Agent configuration
- [ ] Dashboard JSON structure
- [ ] Deployment scripts

## Success Criteria

You've successfully completed the deployment when:

- ✅ All resources deployed without errors
- ✅ Dashboard shows metrics from EC2 instance
- ✅ Alarms configured and in OK state
- ✅ SNS subscription confirmed
- ✅ Stress test triggers alarm
- ✅ Lambda function executes remediation
- ✅ Email notifications received
- ✅ Instance reboots automatically
- ✅ Logs visible in CloudWatch
- ✅ All tests pass

## Next Steps

After successful deployment:

1. **Explore**: Experiment with different thresholds
2. **Customize**: Modify remediation actions
3. **Extend**: Add more instances or metrics
4. **Learn**: Study the architecture and code
5. **Share**: Show your work to others
6. **Improve**: Suggest enhancements

## Support

If you encounter issues:

1. Check this checklist
2. Review error messages
3. Check AWS CloudWatch Logs
4. Verify IAM permissions
5. Consult AWS documentation
6. Search Stack Overflow
7. Review Terraform state

---

**Good luck with your deployment! 🚀**
